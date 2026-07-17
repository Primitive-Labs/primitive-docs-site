# Agent Guide to Primitive Scripts

A **script** is a sandboxed [Rhai](https://rhai.rs/) program that transforms JSON and returns JSON. Scripts are the escape hatch for data shaping too involved for a templated `transform` step — nested reshaping, derived fields, array map/filter/reduce, parsing JSON-string columns. They run inside [workflows](AGENT_GUIDE_TO_PRIMITIVE_WORKFLOWS.md) as `script` steps.

The sandbox is **side-effect-free** — no network, no storage — and deterministic with one deliberate exception: the `ulid()` builtin (see below) reads the clock and randomness to mint ids. A script that never calls `ulid()` always produces the same output for the same input, so it is safe to retry and reproducible to test; a script that calls it is intentionally non-reproducible in exactly those id values.

## The Script model

A script converges on the same shape as managed prompts:

- A **`Script`** header is a named record, unique per app, carrying an `activeConfigId`.
- Versioned **`ScriptConfig`** rows each hold a Rhai body plus that config's `limits`.
- The active body is the `ScriptConfig` that `activeConfigId` points at.

You never edit these records directly. A script body lives in your sync directory as `transforms/<name>.rhai`, where `<name>` (the filename without `.rhai`) is the script's unique name. `primitive sync push` mirrors the file to the server — creating a new `ScriptConfig` and activating it — and `primitive sync pull` writes it back. Authoring is sync-only: no CLI command writes a script body. The `primitive scripts` command is read/inspect plus test-case management on top of that — list scripts, inspect a script's versions, and create or run its test cases (see [Testing](#testing)).

### Live resolution at run time

The runner resolves the body **on every execution** as `step.configId || script.activeConfigId` — there is no publish-time snapshot and no fan-out. Pushing a changed `.rhai` file activates a new config, and every workflow that references the script by name picks up the new body on its **next run**, with no re-publish step.

When you need a body pinned for determinism, set `configId` on the step to bypass the active-config lookup:

```toml
[[steps]]
id = "normalize"
kind = "script"
ref = "normalize-order"     # required — the Script name (unique per app)
# configId = "..."          # optional — pin a specific ScriptConfig
saveAs = "order"
```

## Input and output

The step's `with` table is the JSON context handed to the script. Inside the script the whole table is exposed as **`input.*`** (with **`ctx.*`** as an alias) — **not** as bare top-level variables:

```toml
[[steps]]
id = "normalize"
kind = "script"
ref = "normalize-order"
saveAs = "order"
[steps.with]                # templated by the engine before the script runs
raw = "{{ steps.fetch.body }}"
currency = "{{ input.currency }}"
```

```
// transforms/normalize-order.rhai
let items = input.raw.items.filter(|i| i.qty > 0);
let total = 0.0;
for i in items { total += i.qty * i.price; }
#{
  currency: input.currency,
  itemCount: items.len(),
  total: total,
}
```

The script's return value is its last expression. Given `steps.fetch.body = { "items": [{ "sku": "a1", "qty": 2, "price": 5.0 }, { "sku": "b2", "qty": 0, "price": 9.0 }] }` and `input.currency = "USD"`, this records `steps.normalize.output = { "result": { "currency": "USD", "itemCount": 1, "total": 10.0 }, "ok": true }` — the return value is wrapped under `result` (see Result nesting below).

**Contract rules to write to:**

- **`input.*` only.** `let x = payload;` fails with `Variable not found: payload` — write `input.payload`.
- **`with` is a reserved Rhai keyword** — a script can't declare a variable named `with`.
- **Result nesting.** A script step's output is an envelope: `steps.<id>.output = { result: <return value>, ok: <bool> }`. The return value lives at `steps.<id>.output.result.*` and the success verdict at `steps.<id>.output.ok` (also mirrored at `steps.<id>.ok`); per-run telemetry sits in a sibling `steps.<id>.scriptMetrics`, not inside `output`. Unlike `transform`, whose result is the templated table directly (`steps.<id>.<field>`), wire downstream templates and `runIf` as `{{ steps.normalize.output.result.total }}` — not `{{ steps.normalize.output.total }}` or `{{ steps.normalize.total }}`.
- **Missing keys read as `()`** (Rhai unit). Test with `input.h.symbol != ()` before using a possibly-absent field.
- **`NaN`/`Infinity` can't survive JSON output** — they serialize as `null`. Return a sentinel value instead of relying on them.
- **Map key order is normalized** at serialization, so output is byte-stable regardless of insertion order.

## `parse_json`

`parse_json(str)` parses a strict-JSON string into the matching Rhai value: a JSON object becomes a map, an array becomes an array, and primitives and `null` map to their Rhai equivalents. A database or document field that stores a JSON **array** as a string (a weights array, a composition list) parses straight into an array — no wrapper needed.

```
let weights = parse_json(input.weightsJson);   // "[0.2, 0.5, 0.3]" → [0.2, 0.5, 0.3]
let config  = parse_json(input.configJson);    // "{\"limit\": 10}" → #{ "limit": 10 }
```

Input must be **strict JSON**. Trailing commas, `//` or `/* */` comments, single-quoted strings, and hex literals are rejected. Invalid input fails the step with a non-retryable `SCRIPT_RUNTIME_ERROR` carrying a positioned message — `parse_json: invalid JSON at line N column M: …` — and, when the input matches one of those non-standard forms, a hint pointing at strict JSON. Map keys are sorted at output, so a parsed-then-returned object is byte-stable.

## `ulid()`

`ulid()` (zero-arg) returns a fresh 26-character uppercase Crockford-base32 ULID string — a 48-bit millisecond timestamp prefix plus 80 bits of randomness, matching the `^[0-9A-HJKMNP-TV-Z]{26}$` format the `document.bulkUpdate` step requires for `create` ids. Its purpose is pre-minting record ids when a script builds a `document.bulkUpdate` `operations` array, so cross-record references can be wired without a server round-trip:

```
rows.map(|row| #{ id: ulid(), model: "Holding", action: "create", fields: row })
```

- Ids from calls within one invocation share the timestamp prefix and differ in the random suffix; there is **no monotonic ordering guarantee** between successive calls.
- This is the sandbox's one non-deterministic surface (clock + randomness). In a durable workflow run, step-output memoization keeps minted ids stable across that run's retries.
- For a statically-known number of ids, minting with the <span v-pre>`{{ ulid }}`</span> template helper in the step's `with` table keeps the script itself fully deterministic.

## Per-step limits

Every run is bounded. A step may **lower** the ceilings with a `limits` table; requested values are clamped at the app ceiling and never raised:

```toml
[[steps]]
id = "normalize"
kind = "script"
ref = "normalize-order"
[steps.limits]
maxOperations = 50000
maxOutputBytes = 65536
```

| Limit | Bounds |
|---|---|
| `maxOperations` | Total Rhai operations (the real CPU cap — wall time is unreliable in the sandbox) |
| `wallMsHint` | Advisory wall-time hint |
| `maxOutputBytes` | Serialized output size |
| `maxArrayLength` | Length of any output array |
| `maxObjectKeys` | Keys on any output map |
| `maxNestingDepth` | Output nesting depth |
| `maxStringSize` | Length of any output string |
| `maxCallDepth` | Rhai call-stack depth |
| `maxLogBytes` | Cumulative log output |

## Errors

Deterministic failures (the script will fail the same way every time) come back as a **non-retryable** step error, so durable retries don't re-run a guaranteed failure. Transient/transport failures throw and retry normally. The runtime **fails closed** — it never silently passes input through.

| Code | Meaning | Retryable |
|---|---|---|
| `SCRIPT_PARSE_ERROR` | Input or source JSON failed to parse | No |
| `SCRIPT_COMPILE_ERROR` | Rhai source failed to compile | No |
| `SCRIPT_TYPE_ERROR` | Type mismatch in the script body (a string used where a number is required, etc.) | No |
| `SCRIPT_RUNTIME_ERROR` | Arithmetic error, stack overflow, or other runtime fault | No |
| `SCRIPT_OPERATION_LIMIT` | Exceeded `maxOperations` | No |
| `SCRIPT_OUTPUT_LIMIT` | Exceeded an output size/shape cap | No |
| `SCRIPT_MEMORY_LIMIT` | Exceeded the memory ceiling | No |
| `SCRIPT_VALIDATION_ERROR` | Response failed the host-side validation guard | No |
| `SCRIPT_TIMEOUT` | The runtime call ran long enough to look hung | Yes |
| `SCRIPT_RUNTIME_UNAVAILABLE` | The script runtime was unreachable | Yes |

## Telemetry

Each execution records per-step telemetry at `steps.<id>.scriptMetrics` — a sibling of the step's `output` (persisted on `WorkflowStepRun.scriptMetrics`), carrying the operation count, input/output byte sizes, and the pinned runtime version — visible in run detail alongside every step's input and output.

## Testing

Determinism is what makes scripts testable: same input, same output, no hidden state.

Test a script directly with `primitive scripts tests` — fix the input variables and assert on the returned JSON. A test case names a script, carries input `--vars`, and expresses expectations with `--pattern` (a regex over the output), `--contains` (a JSON array of required substrings), and/or `--json-subset` (a JSON object the result must include); `--config` pins the case to a specific version:

```bash
primitive scripts list
primitive scripts tests create normalize-order --name "drops zero-qty" \
  --vars '{"raw":{"items":[{"qty":0}]},"currency":"USD"}' --contains '["USD"]'
primitive scripts tests run-all normalize-order
```

Test cases round-trip through sync in the `transforms/<name>.tests/*.toml` layout, so they live beside the script body in your sync directory.

You can also drive a script end-to-end from the workflow that uses it, with workflow test cases that assert on `steps.<id>.output`:

```bash
primitive workflows tests create <workflow-id> --name "normalize: drops zero-qty" \
  --vars '{"currency":"USD"}'
primitive workflows tests run-all <workflow-id>
```

Because the sandbox has no network and — outside `ulid()` — no clock or randomness, a passing case stays passing; only assertions on `ulid()`-minted id *values* would flake, so assert on shape or count for those.

## Related

- [Workflows](AGENT_GUIDE_TO_PRIMITIVE_WORKFLOWS.md) — the `script` step, step wiring, and the surrounding pipeline.
- [Configuration](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md) — the sync loop that pushes `transforms/<name>.rhai` to the server.

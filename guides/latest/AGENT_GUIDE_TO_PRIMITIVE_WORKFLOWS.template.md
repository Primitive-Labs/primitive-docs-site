# Workflow Agent Guide

Workflows are multi-step server-side automations defined in TOML. Each step is one of a fixed set of `kind`s (LLM call, integration call, prompt execute, database op, email, blob, etc.). Steps run sequentially with template-rendered inputs and a shared output context.

This guide is the source of truth for what's actually in `src/workflows/`. Examples are kept short and load-bearing.

## TOML structure

```toml
[workflow]
key = "my-workflow"               # required, unique per app
name = "My Workflow"              # required
description = "..."               # optional
accessRule = "hasRole('admin')"  # optional CEL
runAs = "caller"                  # caller (default) | system — see Execution identity
capabilities = ["membership"]    # system-only opt-in grants
perUserMaxRunning = 4             # default 4
perUserMaxQueued = 100            # default 100
dequeueOrder = "fifo"             # fifo | lifo (default fifo)

[workflow.inputSchema]            # JSON Schema as native TOML tables
type = "object"
required = [ "name" ]

[workflow.inputSchema.properties.name]
type = "string"

[workflow.outputSchema]
type = "object"

[workflow.outputSchema.properties.greeting]
type = "string"

[[steps]]
id = "step-1"
kind = "transform"
saveAs = "output"
[steps.output]
greeting = "Hello {{ input.name }}"
```

Schema form rules: the sub-table headers must come after the scalar `[workflow]` keys (a sub-table header ends the parent table). `config pull` always writes schemas as native tables regardless of the form the file had before; a JSON-encoded string (`inputSchema = "{\"type\":\"object\"}"`) is accepted on push, and pull only emits one when the schema contains a `null` TOML can't carry (logged with the workflow and field name). An absent schema omits the key entirely; an empty schema is an empty table. `primitive config migrate-toml` rewrites legacy JSON-string schemas in local files to native form.

A schema may also be a discriminated union: a top-level `oneOf` whose members are all `type = "object"` and share one property declared as a distinct single-literal `enum` — the server validates a value by branch-selecting on that property and checking only the matched member, and `oneOf` must be the schema's only top-level constraint (put shared fields inside each member).

Workflow-level fields the engine actually reads:
`perUserMaxRunning`, `perUserMaxQueued`, `perAppMaxRunning` (default 25), `perAppMaxQueued` (default 10000), `queueTtlSeconds` (default 43200), `dequeueOrder`, `accessRule`, `runAs` (default `caller` — see "Execution identity" below), `capabilities` (system-only grants), `inputSchema`, `outputSchema`, `requiresClientApply` (default `true` — see "Client apply" below), `syncCallable` (default `false` — see "Synchronous invocation" below). `config push` forwards every one of these — `name`, `description`, `status`, `accessRule`, `runAs`, queue settings, schemas, `requiresClientApply`, `syncCallable`, `capabilities`, step list — on both a fresh create and a push to an existing workflow. The file is the whole picture: what it declares is what the workflow has after the push, and an empty `capabilities` array revokes every grant.

**What it does not declare, it removes.** A push to an existing workflow sends every `[workflow]` field, so deleting a line converges the server rather than leaving the old value live: `description`, `accessRule`, `runAs`, `capabilities`, `inputSchema`, `outputSchema`, `[workflow.lock]` and `syncCallable` are **cleared**, and the five queue limits, `dequeueOrder` and `requiresClientApply` are **reset to their defaults** (4, 100, 25, 10000, 43200, `fifo`, `true`). `status` is on neither list: availability is server-owned (#2803), so a push never sends it and a file that still carries the key fails the push. `name` is the exception — it is required, so a `[workflow]` table without a non-empty `name` fails the push locally, naming the file. Pull before editing a workflow you did not author locally; a workflow edited in web-admin since your last sync is reported as a push conflict (a modified-timestamp check, overridable with `--force`) rather than silently overwritten. That check rides the update request, so it covers the files a push sends — a workflow unchanged since your last sync is skipped without a request, and `primitive config diff` is what shows its drift.

An unrecognized key in the `[workflow]` table — or an unrecognized top-level table beside it (a workflow file carries `[workflow]`, `[[steps]]`, `[[configs]]`, `[metadata]`, `secrets`, `vars`, `[expr.cel]` and `include`, and nothing else) — is an error, not a shrug: `config push` names the file and the offending key or table and applies nothing, so a typo fails locally instead of being silently dropped. The mirror case is a CLI older than the server — `config pull` names every `[workflow]` key the server returned that this CLI version does not know and leaves it out of the file, rather than writing a key the next push would reject. Nothing is dropped without a message, and the round trip stays safe: push only sends what it knows, so the server keeps its stored values for the rest.

### Per-step common fields

All steps support these in addition to their own:

| Field | Purpose |
|---|---|
| `id` (req) | Unique within the workflow |
| `kind` (req) | Step type — see list below |
| `name`, `description` | Optional human-readable labels surfaced in CLI/admin views |
| `runIf` | CEL expression; skip step if false |
| `selector` | Override `selected` context (`{ source = "step", stepId = "..." }` or `{ source = "context", path = "outputs.x" }`) |
| `saveAs` | Also store output under `outputs[saveAs]` |
| `forEach` | Iterate over a list expression (path to array, or to `{items: [...]}`), or a `{ zip, as }` table for parallel-array iteration |
| `as` | Loop variable name (default exposes `selected`) |
| `maxItems` | forEach cap (default 500 — clears a provider's native page size; raise it per step) |
| `concurrency` | Parallel forEach lanes (integer 1-100; default 1 = sequential). Results preserve insertion order. |
| `successWhen` | CEL predicate evaluated per forEach iteration to classify the result as functionally succeeded vs empty (see `forEach` below) |
| `continueOnError` | Capture errors as `{ error, errorDetails, ok: false, errored: true }` instead of failing the workflow |
| `skipWhenSkipped` | Array of earlier step ids. Before this step's own `runIf` is evaluated, skip this step (with the same `{ ok: false, skipped: true }` stub) if any listed upstream was skipped. Transitive; reacts only to `skipped: true`, not `errored: true`. Unknown/forward ids are tolerated at run time and warned at save time. |

## Data flow

A run threads one JSON context through every step:

1. **Input**: one JSON object per run — the `start()`/`runSync()` input, the webhook's mapped payload, or the cron trigger's configured input. Validated against `inputSchema` when declared. Top-level scalar properties are coerced to the declared type before the strict check where it's safe: number→`string` via `String()`, numeric string→`number`/`integer` via `Number()` (rejects `""`/NaN), number→`integer` only when integral, `"true"`/`"false"` (case-insensitive)→`boolean`. A non-coercible value (e.g. `"abc"`→number) fails the run with a clear type error. Per-property `coerce: false` opts out; unions/enums/null/nested are not coerced. Applies at every input site (durable run, `start`, `run-sync`, admin preview/run, and `workflow.call` child input), which makes explicit `| string`/`| number` filters at typed call sites optional no-ops.
2. **Step config is templated at execution time**: steps have no implicit input argument — `{{ ... }}` expressions in config strings resolve against the run context (`input`, `steps.<id>`, `outputs.<saveAs>`, `secrets`, `meta`, forEach vars) just before the step runs (see [Templating](#templating)).
3. **Output recording**: each step's JSON result is stored as `steps[id]`; `saveAs = "name"` also registers it as `outputs.name` — a stable alias that survives step-id renames. The engine stamps the uniform verdict (`ok`, the execution `state`, its `succeeded`/`failed`/`skipped` booleans, plus `errored` on a captured failure) on every object entry; array and primitive results pass through unstamped.
4. **Final result**: `outputs.output` if any step used `saveAs = "output"`, otherwise the full `outputs` map (see [Output contract](#output-contract)). Every step's input and output stays on the run record.

## Step types

Every kind below is registered in `src/workflows/runner/default-registry.ts`. If a kind isn't listed here, it doesn't exist.

The network-reaching kinds are bounded by a per-step timeout: `llm.chat` and `gemini.generate` default to 120000 ms, the `database.*` steps and `email.send` to 30000 ms. Override per step with a `timeout` field (milliseconds). A step that exceeds its timeout fails non-retryably and records a failed step-run instead of hanging the run.

### `transform`

Returns the templated `output` field. Use this for shaping the workflow's final output.

```toml
[[steps]]
id = "final"
kind = "transform"
saveAs = "output"
[steps.output]
greeting = "Hello {{ input.name }}"
items = "{{ steps.fetch.data }}"     # single-expression mode preserves arrays
```

### `noop`

Returns `{ message, payload }`. For testing.

### `switch`

First-match branching. Cases are CEL `when` expressions evaluated top-to-bottom in the same scope as `runIf`; the first truthy case's `output` is templated and returned. Non-selected cases are NOT templated, so they can safely reference outputs of steps that didn't run.

```toml
[[steps]]
id = "tier"
kind = "switch"
saveAs = "plan"

[[steps.cases]]
when = "input.score >= 90"
[steps.cases.output]
label = "gold"
discount = 0.2

[[steps.cases]]
when = "input.score >= 70"
[steps.cases.output]
label = "silver"
discount = 0.1

[steps.default]
[steps.default.output]
label = "standard"
discount = 0
```

`default` is opt-in. Omit it and a no-match throws `Switch step '<id>' had no matching case and no default`. An explicit `default = { output = null }` skips on no-match (the step output is `null`).

### `delay`

```toml
[[steps]]
id = "wait"
kind = "delay"
ms = 5000               # number, or "5 seconds" / "200ms"
```

### `event.wait`

Suspends the workflow until a matching event is delivered to the run via the engine's event API.

```toml
[[steps]]
id = "wait-approval"
kind = "event.wait"
type = "user-approval"
timeout = 86400000      # milliseconds (number)
```

### `llm.chat`

OpenRouter chat completion.

```toml
[[steps]]
id = "ask"
kind = "llm.chat"
model = "gpt-4o-mini"
saveAs = "answer"

[[steps.messages]]
role = "system"
content = "You are concise."

[[steps.messages]]
role = "user"
content = "{{ input.question }}"
```

Optional: `temperature`, `top_p`, `attachments`, `plugins`, `tools`, `tool_choice`. **No `maxTokens`** — use `prompt.execute` with a managed prompt to control max tokens.

Output shape: whatever the LLM controller returns — typically `{ content, role, metrics }`.

### `gemini.generate`

```toml
[[steps]]
id = "extract"
kind = "gemini.generate"
model = "models/gemini-2.5-flash"
thinkingLevel = "minimal"   # Gemini 3 only — minimal | low | medium | high

[steps.prompt]
[[steps.prompt.messages]]
role = "user"
[[steps.prompt.messages.parts]]
type = "text"
text = "Summarize: {{ input.content }}"
```

`prompt` may be an object (forwarded as the body) or an array of messages. `gemini.generateRaw` is the same shape, forwarded as a raw API payload. `gemini.countTokens` returns a token count.

If Gemini returns no usable content — a `SAFETY`/`RECITATION`/`MAX_TOKENS` finish, or an empty completion — the step fails with `GEMINI_EMPTY_RESPONSE` (HTTP 502), surfacing the `finishReason`/`blockReason`, rather than completing with empty output. The step runs with no retries, so an empty response fails the run instead of silently yielding a blank result downstream.

The effective model — `prompt.model` when `prompt` is an object, otherwise the top-level `model` — is validated against the server's Gemini model allow-list **at save time**: a disallowed model rejects the workflow save (the error names the step index and lists the allowed models) instead of failing at run time. A model containing template syntax resolves at run time and is validated then.

### `prompt.execute`

Execute a managed prompt (configured separately via the prompts API/CLI).

```toml
[[steps]]
id = "summarize"
kind = "prompt.execute"
promptKey = "summarizer"      # required, must be active
saveAs = "summary"
# configId = "..."             # optional, override active config
# modelOverride = "gpt-4o"     # optional
# expect = "text"              # optional, and the default — content is the model's text

[steps.variables]
text = "{{ input.content }}"
```

Output: `{ content, metrics, promptKey, configName, model, renderedSystemPrompt, renderedUserPrompt }`.

**`expect` types `content` — and nothing else does.** The step declares what it expects back; the prompt config's `outputFormat` governs only the provider request and the HTTP execute response, so activating a different config never retypes what downstream steps receive:

| `expect` | `content` |
|---|---|
| omitted or `"text"` | The model's output text, exactly as the provider returned it — never auto-parsed (even when it is valid JSON) and never fence-stripped (even when the active config sets `outputFormat = "json"`). |
| `"json"` | The parsed JSON value — object, array or primitive. The step strips markdown code fences itself, then parses. Unparseable output **fails the step** with a non-retryable error naming the prompt key and config. |

`expect` must be a literal `"json"` or `"text"` — it is a static control field, never templated; any other value is rejected when you save the workflow.

```toml
[[steps]]
id = "sentiment"
kind = "prompt.execute"
promptKey = "sentiment-extractor"   # a prompt whose config constrains the model to JSON
expect = "json"                     # content is the parsed object
saveAs = "sentiment"

[steps.variables]
text = "{{ input.content }}"

[[steps]]
id = "record"
kind = "database.mutate"
databaseId = "{{ input.databaseId }}"
operationName = "saveSentiment"

[steps.params]
mood = "{{ steps.sentiment.content.mood }}"
score = "{{ steps.sentiment.content.score }}"
```

Pair `expect = "json"` with a config that actually constrains the provider — that is what makes the parse reliable, and the constraint is provider-specific:

- **openrouter**: `outputFormat = "json"` sends `response_format: { type: "json_object" }` (provider JSON mode).
- **gemini**: `outputFormat = "json"` only normalizes the response (fence stripping on the HTTP execute path); the request-side constraint comes from the config's `outputSchema`, which is the gemini-only structured-output field (see the prompts guide).

Either way the step parses what it gets (it strips fences itself), so an unconstrained config is not a syntax error — it is just a model free to answer with prose, which then fails the step.

Two failure modes to know:

- **Declared json, unparseable output** → the prompt step fails non-retryably (a retry would just re-bill another arbitrary generation). It never silently degrades to a string.
- **Traversing a text-typed `content`** → `{{ steps.summarize.content.title }}` against a text step is an unresolved reference, so it **fails the step that references it** (strict templating). Either declare `expect = "json"` on the prompt step, or rescue the reference: `{{ steps.summarize.content.title | default: "" }}`.

Migrating a workflow that relied on the old behavior (the step used to guess by trying `JSON.parse`): add `expect = "json"` to the prompt step. Note the step output has **no `raw` field** — the completion is stored exactly once, in `content`.

### `integration.call`

```toml
[[steps]]
id = "fetch"
kind = "integration.call"
integrationKey = "weather-api"     # required, must match a configured integration
saveAs = "weather"
# bodyMode = "json"                 # json (default) | raw | multipart

[steps.request]
method = "GET"
path = "/current"

[steps.request.query]
city = "{{ input.city }}"

[steps.request.headers]
X-Custom = "value"

[steps.request.form]                # application/x-www-form-urlencoded body
grant_type = "client_credentials"
```

Optional: `attachments`, `multipartFields` (for `bodyMode = "multipart"`).

`[steps.request.form]` sends an `application/x-www-form-urlencoded` body — a table of `{{ }}`-templated key/value pairs the platform URL-encodes (null/undefined/empty values omitted; array values repeat the key; the platform sets `Content-Type: application/x-www-form-urlencoded` unless a header already sets it). It's mutually exclusive with `body` and with `bodyMode = "raw"|"multipart"`; combining them is rejected at push time by the TOML validator and at runtime with `REQUEST_BODY_CONFLICT` (400). (This is a workflow-step surface only — the typed client `integrations.call` request has no `form` field.)

Integration `defaultHeaders` and `staticQuery` resolve `{{secrets.KEY}}` from app secrets — workflow steps cannot put secrets into `request.headers` directly without exposing them in step output snapshots. Put secrets in the integration config.

The integration's own `accessRule` gates this step (the `prompt.execute` step is gated the same way by the prompt's `accessRule`). `fromWorkflow()` is true here and false for a direct client call, so `accessRule = "fromWorkflow()"` (or `fromWorkflow('<this-workflow-key>')`) is the usual rule for a workflow-only integration. A `runAs: "system"` run binds no user, so a `user.*` or `isMemberOf(...)` rule is false for it and there is no admin bypass. An integration with **no** stored rule refuses the step in both run modes — the step fails non-retryably with `Integration access denied`. See the integrations guide's caller-access section.

### `database.query` / `mutate` / `count` / `aggregate` / `pipeline` / `applyToQuery`

All take `databaseId`, `operationName`, optional `params`. Query takes `limit`, `sort`, `cursor`, `direction`. All accept `dryRun = true`.

```toml
[[steps]]
id = "list"
kind = "database.query"
databaseId = "{{ input.dbId }}"
operationName = "listActiveUsers"
limit = 50
saveAs = "users"
[steps.params]
status = "active"
# Output: { data: [...], hasMore?, nextCursor? }

[[steps]]
id = "create-task"
kind = "database.mutate"
databaseId = "{{ input.dbId }}"
operationName = "createTask"
[steps.params]
title = "{{ input.title }}"
# Output: { results: [{ op: "save", success: true, id, values? }] }
# One entry per definition step, in definition order (a save+increment op
# reports both). Each entry has `op` (the step's kind) + `success` (not `ok`)
# + `id`; increment entries carry post-increment counters in `values`.
# Failures throw — a result entry's presence implies it succeeded. In `runIf`,
# prefer the step-level verdict `steps['create-task'].ok` (true iff every
# result item succeeded) or `size(steps['create-task'].?results) > 0`.

[[steps]]
id = "mark-overdue"
kind = "database.applyToQuery"
databaseId = "{{ input.tasksDbId }}"
operationName = "mark-overdue"      # NOTE: uses `operationName`, not `operation`
[steps.params]
now = "2026-04-27T00:00:00Z"
# Output: { matched, affected, failed } — matched === affected + failed.
# If the definition sets a `source.limit`, the response may add
# `truncated: true` + `appliedLimit`; without one, a match over the
# default cap (1000) fails the step instead of truncating.
```

To update every record matching a server-side filter without enumerating items, use `database.applyToQuery`; to write a *known set* of records, use `database.batch` below.

### `database.batch` / `database.batchForUser`

Write a set of records through ONE registered mutation operation, in one step and one operation call — the shape a paginated ingest needs (a page of an external sync landed as a single step run, instead of one call and one step-run row per record).

```toml
[[steps]]
id = "fetch"
kind = "integration.call"
integrationKey = "plaid"

[steps.request]
method = "POST"
path = "/transactions/sync"

[[steps]]
id = "land-page"
kind = "database.batch"
databaseId = "{{ input.dbId }}"
operationName = "upsertTransaction"       # must name a MUTATION operation
batch = "{{ steps.fetch.data.added }}"    # array of objects; each item = one call's params
# Output: { importedOps, failedOps, results: [{ index, success, ids, error? }] }
```

- `batch` must be an **array of objects** after templating; each item is one invocation's params, passed through untouched (there is no step-level `params` to merge). One operation call, one database round trip and **one step-run row** regardless of size. Cap: 100 000 items — but batch per external *page*, not per whole dataset: `results` grows with N and rides in the run state.
- `importedOps` / `failedOps` count emitted **ops**, not items — a multi-step mutation definition emits several ops per item, so the counts can exceed `results.length`. They are deliberately not named `imported`/`failed`: the engine reserves `failed` (with `state`/`succeeded`/`skipped`) on every step entry and overwrites it with the boolean verdict.
- `results` is **item-level and positional** — exactly one entry per `batch` item, in order. An item whose definition emitted no op at all (e.g. a patch step skipped because its id resolved empty) reports `success: false`; a skipped write is never reported as a success.
- **A per-item failure does not throw.** The underlying batch is non-atomic, so 99 of 100 records can land: the step COMPLETES, `steps['land-page'].ok` is false, and every failure is named in `results`. Gate downstream work with `runIf = "steps['land-page'].ok"`. (Contrast `database.mutate`, which is all-or-nothing and fails the step.)
- An **empty `batch` succeeds** as a no-op (`{ importedOps: 0, failedOps: 0, results: [] }`, `ok` true) without calling the server — the last page of a sync returning zero rows must not fail the pipeline.
- **Two tiers of validation.** Rejections the platform can make *before* writing anything fail the whole batch, non-retryably: unknown parameters or a failed coercion, an operation that does not exist or is not a mutation, an access denial on ANY item (the operation's access rule and every per-parameter rule are enforced per item), more than 100 000 items. Problems the database only sees as it applies an op — a missing/empty `upsertOn` value, an unregistered unique index, a unique-constraint violation, a failed condition, a hook denial — are **per-item** failures: that item reports `success: false` while its siblings commit.
- **Re-running converges** when the operation's save is keyed with `upsertOn`: each item resolves to the same canonical record id, so a step re-run after a crash updates instead of duplicating — nothing to configure on the step. Caveat for duplicates *inside one batch*: items matching an already-existing record converge (applied in order, last write wins), but two items sharing a key no record has yet each get a fresh id and the later one fails with a unique-constraint violation. **Dedupe the page before the step.**
- `database.batchForUser` is the system-only subject variant (`runAs = "system"`, plus `userId` — `step.userId ?? input.userId`), like the other `database.*ForUser` kinds: CEL `user.*` and the database's triggers describe the SUBJECT user while the system actor stays audit metadata.
- `dryRun` is not supported on this kind.

### `document.query` / `queryOne` / `count` / `save` / `patch` / `delete`

Read and write records in a document's models server-side. All take `documentId` + `modelName`. Writes are durable when the step completes and reach connected clients like any other document change — they do not use the client-apply flow (`requiresClientApply` is for results only clients write).

| Kind | Additional fields | Output |
|---|---|---|
| `document.query` | optional `filter`, `options` | `{ data: [...], hasMore?, nextCursor? }` — `collect`-compatible |
| `document.queryOne` | optional `filter`, `options` | `{ record }` — `null` when nothing matches (does not fail) |
| `document.count` | optional `filter` | `{ count }` |
| `document.save` | `recordId`, `data` | `{ record }` — creates or replaces the record at `recordId` |
| `document.patch` | `recordId`, `data` | `{ record }` — merges `data` fields into the record |
| `document.delete` | `recordId` | `{ deleted: true, id }` |

`filter` uses the same operator syntax as client-side model queries; empty or omitted matches all records. `options` supports `sort` (`{ field = 1 }` / `-1`), `limit`, and cursor pagination (`uniqueStartKey` — feed it the previous page's `nextCursor`).

Writing a `stringset` field (which also backs `refersToMany`) in `save`/`patch` `data` takes either a plain array — `{ tagNames: ["a", "b"] }`, which **replaces** the whole set — or a per-member delta op-object `{ $add?: [...], $remove?: [...], $clear?: true }`. The delta applies `$clear` first, then `$remove`, then `$add` (so `$add` wins a tie), touching only the named members, so it merges cleanly with concurrent client `.add()` / `.delete()`. A returned stringset field reads back as a member array. The op-object is rejected on a non-collection field; a declared stringset given a non-array, non-op value also fails.

```toml
[[steps]]
id = "overdue"
kind = "document.query"
documentId = "{{ input.docId }}"
modelName = "Invoice"
saveAs = "invoices"
[steps.filter]
status = "overdue"
amount = { "$gt" = 100 }
[steps.options]
sort = { dueDate = 1 }
limit = 50

[[steps]]
id = "mark-paid"
kind = "document.patch"
documentId = "{{ input.docId }}"
modelName = "Invoice"
recordId = "{{ input.invoiceId }}"
[steps.data]
status = "paid"
```

A missing `documentId`/`modelName` (or `recordId` on a write), or a `documentId` that doesn't resolve to a document, fails the step non-retryably.

**Caller-mode ACL.** When the run is `runAs: "caller"` (the default — see [Execution identity](#execution-identity-runas-system-workflows)), every op enforces the caller's per-document permission: `query`/`queryOne`/`count` need `reader`, `save`/`patch`/`delete` all need `read-write`. A `null`/insufficient permission throws `DocumentAccessDeniedError` (non-retryable) — a templated/user-supplied `documentId` can't bypass the ACL. `runAs: "system"` runs app-privileged (no per-caller check). Note `document.delete` deletes **records inside** a document; deleting the document itself (`client.documents.delete`) is a different, stricter operation with no workflow step — see [Deleting Documents](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#deleting-documents) for its permission rule.

**Targeting by alias.** Supply a `documentAlias { scope, aliasKey }` block instead of `documentId` (exactly one of the two). `scope` is `"user"`. In a caller run, it resolves the caller's own alias (forces `userId = caller`). In a **system** run, it requires an explicit subject `userId` on the block — `documentAlias { scope = "user", aliasKey, userId }` — resolved app-privileged (see [subject-user methods](#subject-user-methods-system-workflows)). A non-resolving alias fails the step like a bad `documentId` (Option A — hard fail).

```toml
[[steps]]
id = "load"
kind = "document.query"
modelName = "Habit"
saveAs = "habits"
[steps.documentAlias]
scope = "user"
aliasKey = "tracker"
```

### `document.resolveAlias`

Resolve an alias to its id for conditional branching, without failing on a miss (Option B). Top-level `scope` (`"user"`) + `aliasKey`. Output `{ documentId }`, or `{ documentId: null }` when the alias doesn't resolve (or the caller lacks access — never leaks existence). Caller-mode only.

```toml
[[steps]]
id = "find-tracker"
kind = "document.resolveAlias"
scope = "user"
aliasKey = "tracker"
saveAs = "tracker"

[[steps]]
id = "seed"
kind = "document.save"
runIf = "outputs.tracker.documentId == null"
# ... create the user's tracker document
```

### Document writes obey the model's declared field types

Every document write — `save`/`patch` in both modes, and `bulkUpdate` — type-checks each supplied value against the model's declared field types inside the write transaction, before anything is persisted:

- **Convertible → converted.** The `1`/`0` a database read yields for a `boolean` column (SQLite has no boolean type — the common case when `records` comes straight from `database.query`) is stored as `true`/`false`; a numeric string in a `number` field becomes a number; a number in a `string` field is stringified. Same conversion table as `inputSchema` and database-operation params.
- **Not convertible → non-retryable step failure** naming the model, field, declared type and offending value. `"maybe"` or `2` in a `boolean` field, an object in a `number` field. The rejection rolls back its own transaction: a single write and a `bulkUpdate` write nothing, while a batch `records` write is one transaction per 100-record chunk, so a failure in a later chunk leaves earlier chunks committed.
- **Untouched:** `null`, fields the model doesn't declare, and `stringset` payloads (arrays / `{ $add, $remove, $clear }`) — but an array or op-object aimed at a declared *scalar* field is a type error. `date` fields take an ISO-8601 date string (`"2026-08-14"`, optionally with a time — including the space-separated `"2026-08-14 12:30:00"` a SQL read yields) *or* an epoch number, and nothing else: `"not-a-date"` fails, and so does a string only a locale parser understands (`"08/14/2026"`, `"August 14, 2026"`) or a day the calendar doesn't have (`"2026-02-30"`).
- Enforcement reads the model schema embedded in the document, so a model no client has saved yet has nothing to enforce.
- **Not only workflow steps.** The contract sits in the shared server-direct write path, so the document-records REST API reaches it too — `POST` / `PATCH` / `DELETE documents/:documentId/records/:model` and `records/bulk`, which the `primitive documents records` commands drive. Those now answer **400**, naming the field, for an unconvertible value they previously accepted and stored (`2` or `"maybe"` in a `boolean` field), so a script corrupting a record that way starts failing instead; a convertible value still succeeds, so a script feeding a `1` into a `boolean` field keeps its `200` and now stores `true`. Collaborative client writes over the live connection are unaffected: they apply document updates directly and never enter this path.

### `document.save` / `patch` / `delete` — batch mode (`records` / `recordIds`)

Each write step works in exactly one of two modes, never mixed: singular (`recordId` [+ `data`]) or batch (`records` for `save`/`patch`, `recordIds` for `delete`). Batch mode is keyed off the array field being present at all (not off `recordId` being absent) — supplying both a singular and an array field on the same step is a hard non-retryable error. Batch mode resolves the document ACL once and applies every record in one Yjs transaction + one persist + one broadcast **per chunk**, instead of the per-record cost of a `forEach` fan-out.

| Mode | Input | Output |
|---|---|---|
| singular | `recordId` (+ `data`) | unchanged: `save`/`patch` → `{ record }`; `delete` → `{ deleted: true, id }` |
| batch | `records` (save/patch) / `recordIds` (delete) | `save` → `{ saved, savedIds }`; `patch` → `{ patched, patchedIds }`; `delete` → `{ deleted }` |

```toml
[[steps]]
id = "upsert-holdings"
kind = "document.save"
documentId = "{{ outputs.doc.documentId }}"
modelName = "holding"
records = "{{ steps.rows.output.result.rows }}"   # Array<{ id?, ...fields }> — id optional, minted server-side (ulid()) when absent

[[steps]]
id = "merge-holdings"
kind = "document.patch"
documentId = "{{ outputs.doc.documentId }}"
modelName = "holding"
records = "{{ steps.rows.output.result.rows }}"   # Array<{ id, ...fields }> — id REQUIRED per element

[[steps]]
id = "remove-obsolete"
kind = "document.delete"
documentId = "{{ outputs.doc.documentId }}"
modelName = "holding"
recordIds = "{{ steps.plan.output.result.obsoleteIds }}"   # string[] — a non-existent id is a no-op, not an error
```

Rules and edge cases:

- **Chunk cap = 100 records** (server constant). A batch of N records runs as `ceil(N / 100)` sequential DO round-trips, not one unbounded transaction.
- **Validation-first, then fail-fast.** A structural pass runs over every record before the first chunk is sent (well-formed objects; `patch` elements require a non-empty `id`; `recordIds` entries are non-empty strings). A structural failure fails the whole step non-retryably, names the offending index, and writes nothing.
- **Per-chunk atomicity, not whole-batch atomicity.** Each chunk (≤100 records) commits atomically, but a batch spanning multiple chunks is not atomic across chunks — a failure on chunk *k* leaves chunks `0..k-1` committed. The error names the offending record. Keep a batch ≤100 (one chunk) for all-or-nothing, or design the workflow for idempotent re-runs.
- **Duplicate ids within one batch:** allowed, last write wins (records apply in id order within the transaction).
- **Batch failures are non-retryable** — a validation or constraint failure fails the run immediately rather than retrying.
- **Id-less batch `save` and retries.** An omitted `id` mints a fresh ULID server-side on every attempt — retrying a batch save whose records had no `id` (e.g. after a transient error) creates duplicates rather than upserting the originals. Supply explicit ids, or make the workflow idempotent some other way, if the step might retry.
- **Projection freshness.** A batch marks the document's query projection dirty once (rebuilt on the next read) instead of updating it inline per record — the first `document.query`/`count` after a batch pays one rebuild rather than N inline upserts. Single-record writes still update the projection inline.

### `document.bulkUpdate` — multi-model atomic write (`operations`)

Batch mode is per-model and per-chunk atomic; `document.bulkUpdate` is the whole-blob atomic form: an ordered `operations` array spanning **any of the document's models**, applied in one transaction with one persist and one change broadcast. All-or-nothing across the entire list — a mid-list failure rolls back everything before it.

Takes `documentId` **or** `documentAlias` (same alias rules as the other document steps); no step-level `modelName` — each operation names its own `model`:

```toml
[[steps]]
id = "apply-rebalance"
kind = "document.bulkUpdate"
documentId = "{{ outputs.doc.documentId }}"
saveAs = "applied"
operations = [
  { model = "Account",  action = "create", id = "{{ input.newAccountId }}", data = { name = "Growth", balance = 0 } },
  { model = "Ledger",   action = "patch",  id = "{{ input.ledgerId }}", data = { rebalancedAt = "{{ now }}" } },
  { model = "Position", action = "delete", id = "{{ input.staleId }}" },
]
```

Operation shape: `{ model, action, id, data, precondition? }`. The `action` vocabulary is exactly `create` | `patch` | `delete` — not add/update/save. `data` holds the record fields and is **required and non-empty** on `create` and `patch`; `delete` takes none. Any other key in an operation is rejected before anything is written — including the retired `fields` and `record` spellings, whose error names `data`.

| `action` | `id` | Semantics |
|---|---|---|
| `create` | **required, caller-supplied ULID** (must match `^[0-9A-HJKMNP-TV-Z]{26}$` — 26 uppercase Crockford chars) | Strict create — fails if the id already exists. The server never mints ids for this step. |
| `patch` | required, non-empty string | Merges `data` — fails if the id doesn't exist. |
| `delete` | required, non-empty string | Removes the record; an id that doesn't exist is a no-op (contributes 0 to `deleted`). |

Output: `{ applied, added, updated, deleted }` — `added`/`updated` list `{ model, id }` per committed create/patch in input order; `deleted` is the count actually removed.

Rules and edge cases:

- **Cap = 500 operations**, a separate constant from batch mode's 100-record chunk — and **never chunked** (chunking would break the atomicity that is this step's point). Over 500 is rejected non-retryably before any write.
- **Mint create ids with `{{ ulid }}`** (template helper) for statically-known counts, or the script `ulid()` builtin when a script builds the operations array dynamically — pre-minting is what lets a later operation in the same blob reference a record an earlier operation creates.
- **Validation-first, then fail-fast.** A structural pass (shape, cap, ULID well-formedness, known `action`) runs before any write; failures are non-retryable and name the offending operation index and model. State-dependent failures (create-collision, patch-miss, unique-index violation) abort inside the transaction — nothing commits — and are also non-retryable. Transient storage/broadcast failures stay retryable.
- **Operations apply in listed order** within the one transaction.
- **Caller-mode ACL is checked once** at write level: the caller needs `read-write` on the document; denial is non-retryable and nothing commits. An empty `operations` array is a clean no-op.

### `writeOptions` — conditional (compare-and-set) writes

A single-record `document.save`/`patch`/`delete` accepts a `writeOptions` block that guards the write against the record's current state. It's the workflow-side compare-and-set: two runs racing on the same record converge without a queue, because the guard is evaluated against the live record inside the write transaction — exactly one write applies, the loser fails cleanly.

```toml
[[steps]]
id = "cas-balance"
kind = "document.patch"
documentId = "{{ input.docId }}"
modelName = "Account"
recordId = "{{ input.accountId }}"
data = { balance = "{{ steps.calc.output.next }}" }
saveAs = "updated"
[steps.writeOptions.precondition]
version = "{{ steps.read.output.record.version }}"   # apply only if version still matches
```

- **`precondition`** — a **field-equality** map `{ field: scalar, … }`. The write applies only when every field on the live record equals the given value; a mismatch, or a record that doesn't exist, fails closed. `precondition` on `document.delete` is a conditional delete (delete only if it still matches).
- **`ifNotExists`** (save) — create-only: fails if a record already exists at `recordId`.
- **`upsertOn`** (save) — resolve an existing record by a unique field's value and update it in place instead of creating a new record at the supplied `recordId` (no match → creates at `recordId`).

Failure semantics:

- A failed `precondition`/`ifNotExists` throws `CONDITION_NOT_MET` — a **non-retryable** classified failure. Nothing is written (the transaction rolls back). It is not auto-retried, so a step that must retry until it wins writes an explicit read → compute → conditional-write loop (e.g. `forEach` over a small range, or `runIf` on the prior attempt's `code`), not a bare retry. Capture the attempt with `continueOnError = true` and read the code back from `steps.<id>.code` for a plain step, or `steps.<id>.errors[0].code` / `steps.<id>.items[0].code` for an attempt made inside a `forEach`.
- **Field-equality only.** Operators (`$gt`, `$in`, …) and combinators (`$and`/`$or`) in a `precondition` are rejected non-retryably at validation with a clear error — they are not silently ignored. Express richer conditions by reading first and comparing in a `transform`/`script`, then gating the write on a scalar field.
- **`document.bulkUpdate`** takes a per-operation `precondition` too (`{ model, action, id, data, precondition? }`); a failed precondition on any operation is fail-fast for the **whole blob** — nothing commits, matching the step's all-or-nothing semantics.

### `document.create` — mint a new document mid-run

Creates a brand-new document (no alias) and returns `{ documentId, created: true }`. Built for the mint-then-write pattern: the run decides a document is needed (e.g. after a diff shows real changes), creates it, then targets the returned `documentId` with `document.bulkUpdate` / `document.save`. Works in **both** execution modes.

```toml
[[steps]]
id = "make-doc"
kind = "document.create"
title = "Holdings — {{ input.importDate }}"   # required, 1–255 chars
tags = ["import"]                              # optional, ≤10 tags
saveAs = "doc"
[steps.metadata]                               # optional provenance
source = "csv"
accountId = "{{ input.accountId }}"

[[steps]]
id = "apply"
kind = "document.bulkUpdate"
documentId = "{{ steps.make-doc.documentId }}"
operations = [
  { model = "Holding", action = "create", id = "{{ ulid }}", data = { symbol = "VTI", shares = 10 } },
]
```

- **Caller mode** (`runAs:"caller"`, default): delegates to the same path as the client `documents.create` — the **starting user becomes the document `owner`**. `userId` / `permission` params are ignored (a caller can't mint a document owned by someone else or attributed to the system).
- **System mode** (`runAs:"system"`): requires the `resource-provision` capability (deny-by-default — intentionally stricter than the alias-keyed `document.getOrCreateWithAliasForUser`, which gates on system-run only) **and** an explicit subject `userId` (`step.userId ?? input.userId`, same as the other subject-user steps). The document is created `createdBy: "sys:<appId>"` and the subject is granted **`owner`** by default; override with `permission = "read" | "write" | "owner"`. (The system principal also holds an owner grant from the create — it owns and grants; it does not retain zero access.)
- `tags` and `metadata` are set at creation so an import's provenance rides on the document and a collection listing serves as the history index without opening old documents. `metadata` is validated + size-checked (4 KB) exactly as the REST create.
- **Per-run cap**: a single run is capped at about **1000** documents; a create past the cap fails non-retryably (documents minted before the cap persist). Lower the cap for a run with `input.maxDocumentCreates` (clamped ≤ 1000 — it can only tighten, never raise). The cap is a durable per-run counter keyed on the run id, shared across `forEach`, inline `workflow.call`, and compensation in the same run — so it holds across concurrency and a durable replay. It runs on a fixed wall-clock window, so it is an abuse guardrail, not an exact quota: a long run that straddles a window boundary can mint up to twice the cap.
- Errors: validation failures (missing title, >10 tags, oversized/invalid metadata, missing subject, missing capability) are non-retryable; transient model/controller errors stay retryable (429/5xx). `created` is always `true` (this step always creates — it never gets).

### `group.addMember` / `removeMember` / `removeAll` / `checkMembership` / `listMembers` / `listUserMemberships`

```toml
[[steps]]
id = "add"
kind = "group.addMember"
groupType = "team"
groupId = "{{ input.teamId }}"
userId = "{{ input.userId }}"   # OR email = "...", not both
```

`addMember` is idempotent. With `email`, returns `{ status: "pending_signup", invitationId, inviteToken, ... }` if the email has no AppUser yet.

`group.removeAll` clears all of one user's memberships within a single group type. Params: `groupType` (required; no `#`, no `_`-prefixed system types) and `userId` (required; no `groupId`, no `email`). Returns `{ userId, groupType, removed, groupIds }`. Type-scoped via the `userId#groupType#` sort-key prefix (no cross-type bleed — `sub` ≠ `subscription`); a no-op returns `removed: 0`; caps at 1000 removals per run (refused up front if exceeded). Caller runs pre-flight every affected group's `member.delete` CEL rule (all-or-nothing); system runs need the `membership` capability instead.

Group operations evaluate the group type's CEL rules. For workflow-issued operations, rules can match `fromWorkflow("workflowKey")`. In a system run, `addMember`/`removeMember` record `sys:<appId>` as the membership's `addedBy` (not the admin/cron/webhook that initiated the run).

### `collection.addDocument` / `collection.removeDocument`

Manage a document's membership in a collection. Both take `collectionId` + `documentId` (both required, both templatable) and enforce the same authorization as the equivalent client calls: `addDocument` checks the collection type's `document.add` rule plus caller owner/read-write on the document; `removeDocument` checks `document.remove` only.

```toml
[[steps]]
id = "publish"
kind = "collection.addDocument"
collectionId = "{{ input.collectionId }}"
documentId = "{{ steps.create.documentId }}"
# Output: { collectionId, documentId, added: true, alreadyPresent: false }
#         already a member → { added: false, alreadyPresent: true } (success, not an error)

[[steps]]
id = "retract"
kind = "collection.removeDocument"
collectionId = "{{ input.collectionId }}"
documentId = "{{ input.documentId }}"
# Output: { collectionId, documentId, removed: true, alreadyAbsent: false }
#         not a member → { removed: false, alreadyAbsent: true } (a missing *collection* is still an error)
```

Both converge on retry (already-present add and already-absent remove are no-op successes). Membership presence is checked with a fresh read, so remove → add on the same document within one run works. Errors follow the standard convention: 4xx (except 429) non-retryable, 429/5xx retryable. Both kinds are sync-callable.

**Caller vs. system.** In a caller run, authorization is exactly what's described above (`document.add`/`document.remove` CEL + caller doc access). In a `runAs:"system"` run, both kinds instead require the `resource-grant` capability (see [Sensitive capabilities](#execution-identity-runas-system-workflows)) — bypassing the CEL rule and the caller doc-access check — and are attributed to the system principal. Without the capability, the step is refused non-retryably.

### Resource lifecycle steps

`database.create` / `database.delete`, `group.create` / `group.delete`, `collection.create` / `collection.delete`, and `collection.grantGroupPermission` provision and tear down resources inside a run, so one durable workflow can create a whole constellation (database + groups + collections + grants) with ordered teardown as the rollback story. All seven run in both caller and system workflows.

**Caller mode**: each step acts as the starting user and enforces exactly the CEL rules and delete cascades of the equivalent client call.

**System mode (`runAs:"system"`)**: each step instead requires the matching capability declared in the workflow's `capabilities`, deny-by-default: `resource-provision` for the create steps, `resource-teardown` for the delete steps, `resource-grant` for `collection.grantGroupPermission`. Missing the capability fails the step non-retryably (`<kind> requires the '<capability>' capability in a runAs:"system" workflow (declare it in the workflow's capabilities)`) before anything is written. A system-mode create attributes the resource to the run's system principal (`createdBy: "sys:<appId>"`); a system-mode delete may target only a resource whose `createdBy` starts with `sys:`, unless the workflow also holds the escalated `resource-teardown:any` capability (needed to tear down a resource a member created). See [Sensitive capabilities](#execution-identity-runas-system-workflows) for the full capability reference.

```toml
[workflow]
key = "provision-classroom"
name = "Provision Classroom"
runAs = "system"
capabilities = ["resource-provision", "resource-grant"]

[[steps]]
id = "makeDb"
kind = "database.create"
title = "{{ input.className }}"
databaseType = "classroom"
```

| Kind | Params | Output |
|---|---|---|
| `database.create` | `title` (req), `databaseType` (req), `metadata` (opt object, validated against the type's declared metadata), `initialMetadata` (opt `{ category: { field: value } }`) | The created database: `{ databaseId, title, databaseType, permission: "owner", createdBy, createdAt, ... }` |
| `database.delete` | `databaseId` (req) | `{ databaseId, deleted: true, ... }`; 404 → `{ databaseId, deleted: false, alreadyAbsent: true }` |
| `group.create` | `groupType` (req), `name` (req), `groupId` (opt — caller-supplied id), `description` (opt) | `{ created: true, appId, groupType, groupId, name, ... }`; already exists → `{ groupType, groupId, created: false, alreadyExists: true }` |
| `group.delete` | `groupType` (req), `groupId` (req) | `{ groupType, groupId, deleted: true, ... }`; 404 → `{ deleted: false, alreadyAbsent: true }` |
| `collection.create` | `name` (req), `description`/`collectionType`/`contextId` (opt), `initialMetadata` (opt `{ category: { field: value } }`) | The created collection: `{ collectionId, name, collectionType, contextId, ... }`. A duplicate *name* is a 409 error (not a no-op) |
| `collection.delete` | `collectionId` (req) | `{ collectionId, deleted: true, ... }`; 404 → `{ deleted: false, alreadyAbsent: true }`. Cascades memberships and group grants like the client delete |
| `collection.grantGroupPermission` | `collectionId`, `groupType`, `groupId`, `permission` (all req; `permission` is exactly `"reader"` or `"read-write"` — `"read"` fails non-retryably) | `{ collectionId, groupType, groupId, permission, grantedAt, grantedBy }` (re-grant succeeds) |

```toml
[[steps]]
id = "db"
kind = "database.create"
title = "{{ input.className }}"
databaseType = "classroom"

[[steps]]
id = "teachers"
kind = "group.create"
groupType = "class-teachers"
groupId = "class-{{ input.classCode }}-teachers"   # supplying the id makes re-runs idempotent
name = "{{ input.className }} Teachers"

[[steps]]
id = "posts"
kind = "collection.create"
name = "{{ input.className }} Posts"
collectionType = "class-posts"

[[steps]]
id = "grant"
kind = "collection.grantGroupPermission"
collectionId = "{{ steps.posts.collectionId }}"
groupType = "class-teachers"
groupId = "{{ steps.teachers.groupId }}"
permission = "read-write"
```

`initialMetadata` on `database.create` / `collection.create` stamps resource-metadata categories atomically as the resource is created — identical to the client create call: every category is validated before the resource row is created (an invalid category fails the whole step non-retryably, leaving nothing behind), the category's write rule is waived for the create-time stamp, and at most 10 categories may be stamped. Stamp at create when a later step or an `md.self.*` rule on the new resource must see the value immediately — it closes the fail-closed window a post-create `metadata.write` would leave open. In `runAs:"system"` mode this needs no capability beyond `resource-provision` (create authority covers the stamp).

Idempotency: the `*.delete` kinds map 404 to a no-op success, so an ordered teardown workflow re-run over a partially torn-down constellation converges. `group.create` with a caller-supplied `groupId` is likewise re-runnable. `database.create` and `collection.create` mint server-side ids — a whole-run retry can create a duplicate; sequence creates early and keep the fallible steps after them. Errors: 4xx (except 429) non-retryable, 429/5xx retryable.

### `metadata.write` / `metadata.read` / `metadata.delete` / `metadata.resolve`

```toml
[[steps]]
id = "record-billing"
kind = "metadata.write"
resourceType = "user"
resourceId = "{{ input.userId }}"
category = "billing"
saveAs = "output"
[steps.data]
stripeCustomerId = "{{ input.customerId }}"
status = "active"
# Output: { ok, resourceType, resourceId, category, data, schemaVersion, size }

[[steps]]
id = "load-billing"
kind = "metadata.read"
resourceType = "user"
resourceId = "{{ input.userId }}"
category = "billing"
saveAs = "output"
# Output: { ok, resourceType, resourceId, category, data, schemaVersion, exists }

[[steps]]
id = "clear-billing"
kind = "metadata.delete"
resourceType = "user"
resourceId = "{{ input.userId }}"
category = "billing"
saveAs = "output"
# Output: { ok, resourceType, resourceId, category, deleted }
```

All three route through the same read/write/delete path (and the same category `readRule`/`writeRule` gate) as the client and CLI — `resourceType`/`resourceId`/`category`/`data` are all templated. `metadata.write` is a full replace, not a merge. `metadata.delete` is gated by the `writeRule` (deletion is a write) and is idempotent — an already-absent category returns `deleted: false`, not a 404, and never fails the step; it's the surface a teardown workflow uses to clear categories (including ones whose schema has `required` fields, which a `set` of `{}` can't clear). Gate a category to exactly this workflow with `fromWorkflow('workflowKey')` in its `writeRule`/`readRule`; the workflow identity is a privileged, call-local value the step runner passes in-process, never derived from a request header. A `runAs:"system"` run gets no app-level owner/admin bypass here — only `fromWorkflow('key')` authorizes it; a `runAs:"caller"` run keeps the bypass (the caller's app role — a resource-level permission never bypasses). Errors follow the `database.*` step convention: 4xx (validation, rule denial, reserved category) is non-retryable, 429/5xx is retryable. See the [Resource Metadata guide](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md).

`metadata.resolve` reverse-resolves a resource by a category's `unique` field value (metadata value → resource). System-only (`runAs:"system"`, like `user.resolve`), bypasses `readRule`. Returns `{ resourceId, resourceType }` on a hit and `{ resourceId: null }` on a miss (a miss never throws); a `key` that is not the category's declared `unique` field is a non-retryable error. One exact lookup, no scan.

```toml
[[steps]]
id = "find-user"
kind = "metadata.resolve"
resourceType = "user"
category = "billing"
key = "stripeCustomerId"
value = "{{ input.customerId }}"
saveAs = "output"
# Output: { resourceId, resourceType } on a hit; { resourceId: null } on a miss
```

### `collect`

Auto-paginate through any step that returns `{ items|data: [...], cursor|nextCursor }`.

```toml
[[steps]]
id = "all-users"
kind = "collect"
itemsField = "data"
cursorField = "nextCursor"
maxPages = 20
maxItems = 10000

[steps.step]
kind = "database.query"
databaseId = "{{ input.dbId }}"
operationName = "listUsers"
limit = 100
# Output: { items: [...all merged...], totalPages }
```

### `workflow.call`

Run a child workflow synchronously, inline. Child gets isolated `input` — it does NOT see parent's `steps`/`outputs`. Max call depth 10. Circular calls throw immediately.

```toml
[[steps]]
id = "onboard"
kind = "workflow.call"
workflowKey = "onboard-user"
[steps.input]
userId = "{{ item.userId }}"
# Output: { output: <child's outputs.output or full outputs>, childStepResults: [...] }
```

Add `forEach` to fan out: the child runs once per item (use `concurrency` for parallel lanes) and the step returns the array of child results. For app-wide, restartable fan-out over the whole user roster, use `iterate-users` instead.

Each call is recorded as its own run of the child workflow: `workflows runs list <child-key>` shows it (`running` while it executes, then `completed`/`failed`), with its own step runs to inspect and the calling run in `meta.parentRunId`. Debug a failing child there instead of from the parent's one-line step error.

### `email.send`

Two modes; pass exactly one of `templateType` or `htmlBody`.

```toml
# Template mode — uses a built-in or registered email template
[[steps]]
kind = "email.send"
templateType = "order-confirmation"
to = "{{ input.email }}"          # OR toUserId, not both
[steps.variables]
orderId = "{{ input.orderId }}"

# Inline mode — requires subject + htmlBody (textBody optional)
[[steps]]
kind = "email.send"
to = "{{ input.email }}"
subject = "Your report is ready"
htmlBody = "<p>Download: {{ outputs.upload.signedUrl }}</p>"
textBody = "Download: {{ outputs.upload.signedUrl }}"
```

`to` is a single address (string), not an array. Built-in templates: `magic-link`, `otp`, `document-share`, `document-share-deferred`, `collection-share`, `collection-share-deferred`, `waitlist-invite`, `waitlist-signup-notification`, `admin-invite`, `app-invite`, `access-request-created`, `access-request-resolved`. Register a custom type by authoring `email-templates/<type>.toml` and running `primitive config push`. Hourly rate limit: 100 workflow emails per app per hour.

### `notification.send`

Sends a multi-channel notification (durable in-app inbox row + live WS mirror; push once the recipient has a registered device) to an app user. Full concept, client SDK, idempotency, and rate-limit reference: [Notifications guide](AGENT_GUIDE_TO_PRIMITIVE_NOTIFICATIONS.md).

```toml
[[steps]]
id = "notify"
kind = "notification.send"
toUserId = "{{ input.userId }}"      # required
title = "Your report is ready"       # required
body = "Tap to view this week's summary."   # required
channels = ["in-app", "ios"]         # optional, default ["in-app"]
iconUrl = "https://example.com/icon.png"    # optional
deepLink = "myapp://reports/latest"  # optional
expiresAt = "2026-08-01T00:00:00Z"   # optional, ISO date
idempotencyKey = "{{ input.jobId }}" # optional — see the Notifications guide's Idempotency section
userInfo = { jobId = "{{ input.jobId }}" }   # optional, push-only
collapseId = "report-ready"          # optional, push-only
threadId = "reports"                 # optional, push-only (APNs)
```

Output: `{ results: [{ channel, status, notificationId?, delivered?, failed?, invalidated?, tokenAttempts?, skipReason?, retryable? }], deduplicated?, deduplicatedChannels? }` — one `results` entry per requested channel.

**Verdict.** The step's `ok` is true only when at least one requested channel actually delivered — a send that reaches nobody (every channel failed, was rate-limited, or the recipient had no registered device) reports `steps.<id>.ok === false` even though the step ran and returned a result.

**Retry semantics.** Single-attempt like every other step, but its error classification is more granular: a channel that failed for a transient reason (`retryable: true` on the result, including a push send where only *some* device tokens failed retryably) throws a plain error so the engine retries the whole step; a channel that failed permanently (bad channel name, unknown recipient, every channel rate-limited) throws non-retryably instead, so the run doesn't spin on a failure a retry can't fix. Set `idempotencyKey` so a retry re-sends only the channels/tokens that still need it — without one, a retry re-sends every requested channel from scratch.

### `blob.upload` / `blob.download` / `blob.signedUrl` / `blob.delete`

Four separate kinds, NOT one `blob` step with an `action` field.

```toml
[[steps]]
id = "save"
kind = "blob.upload"
bucketKey = "reports"             # OR bucketId
filename = "{{ meta.workflowRunId }}.pdf"
contentType = "application/pdf"
contentBase64 = "{{ steps.gen.bytesBase64 }}"   # OR content (utf-8 string)
tags = ["monthly"]
# Output: { blobId, bucketId, bucketKey, filename, contentType, numBytes, sha256, tags }

[[steps]]
id = "url"
kind = "blob.signedUrl"
bucketKey = "reports"
blobId = "{{ steps.save.blobId }}"
expiresInSeconds = 3600           # 30..86400, default 300 (5 min)
# Output: { url, token, expiresAt, expiresInSeconds }

[[steps]]
id = "read"
kind = "blob.download"
bucketKey = "reports"
blobId = "{{ steps.save.blobId }}"
asBase64 = true                   # default false (returns utf-8 string)
# Output: { blobId, bucketId, filename, contentType, numBytes, content?, contentBase64? }

[[steps]]
id = "cleanup"
kind = "blob.delete"
bucketKey = "reports"
blobId = "{{ steps.save.blobId }}"
# Output: { deleted: true, blobId, bucketId }

[[steps]]
id = "cleanup-batch"
kind = "blob.delete"
bucketKey = "reports"
blobIds = "{{ steps.expired.output.result | pluck:blobId }}"   # string[], max 500
# Output: { deleted: N, blobIds, bucketId }
```

- Every kind takes exactly one of `bucketId` / `bucketKey`; missing both, an unknown bucket, or a missing required param is non-retryable.
- **`blob.delete` batch mode.** Exactly one of `blobId` / `blobIds` — supplying both (or neither) is a non-retryable config error. `blobIds` caps at 500 ids per step (`"blob.delete 'blobIds' exceeds the maximum batch size of 500"`); an **empty array is a valid no-op**, so a templated list that resolves to nothing doesn't fail the step. All ids are structurally validated (the error names the offending index) and, in a caller run, all are screened against the bucket's `delete` policy **before any delete happens** — one denial fails the whole step with nothing removed. The delete itself is a single storage call, not per-id round-trips. Output `deleted` counts ids *processed* (input length, duplicates included), unlike `document.delete`'s semantics.
- `blob.delete` is idempotent: it reports `deleted: true` uniformly, including when the blob is already gone — a retried run doesn't fail on cleanup it already did. `blob.signedUrl` never checks blob existence; an authorized caller mints a token even for an absent blob.
- **A blob referenced by a run's input must outlive the run's retry window.** Retries re-run against the identical input, so each attempt's `blob.download` re-fetches the same `blobId`; a deleted blob fails the next attempt — and with it the run — with the non-retryable `blob.download blob not found: <blobId>`. Never delete a blob from outside the run (a cancel handler, another workflow) while a run referencing it can still retry — leave it and let the bucket's TTL tier expire it. `blob.delete`'s idempotency covers sequential re-execution of the deleting step only; it is no protection against a concurrent deleter.
- **Caller-mode access.** In a `runAs:"caller"` run, each kind evaluates the bucket's preset/rule set before the storage op, with the same op mapping as direct client calls: `blob.upload` → `write` (evaluated with `blobId: null`, `createdBy` = the caller), `blob.download` → `read`, `blob.signedUrl` → `share` (minting is gated like sharing, not reading), `blob.delete` → `delete`. Download, signedUrl, and delete load the blob's `createdBy` first so uploader-scoped rules (`record.blobCreatedBy == user.userId`) work; download decides access before revealing whether the blob exists, so a denied caller gets an access error, never not-found. Denial is non-retryable (`"<kind> access denied: …"`). A `runAs:"system"` run skips the bucket policy entirely (app-privileged).

### `analytics.write` / `analytics.query`

`analytics.write` emits up to 25 events per step.

```toml
[[steps]]
kind = "analytics.write"
action = "report.generated"
feature = "reports"
[steps.metrics]
durationMs = 1234
```

`analytics.query` runs a server-side analytics query. Always lock down the workflow with `accessRule = "hasRole('admin')"` — the runner rejects non-admin callers by default. Per-run cap of 50 queries.

```toml
[[steps]]
kind = "analytics.query"
queryType = "users.top"            # see list below
windowDays = 7
limit = 25
saveAs = "topUsers"
# cacheTtlSeconds = 0              # 0/null = bypass cache
```

Valid `queryType` values (dotted form, exact strings):
`overview.dau`, `overview.wau`, `overview.mau`, `overview.growth`, `daily-active`, `rolling-active`, `cohort-retention`, `users.top`, `users.search`, `users.detail`, `users.snapshot`, `events`, `events.grouped`, `errors.groups`, `workflows.top`, `prompts.top`, `integrations`. `users.detail` and `users.snapshot` require `userUlid`.

**Step output: `{ queryType, data, ok }`.** The payload sits under `data` — read `{{ outputs.topUsers.data.results }}`, not `{{ outputs.topUsers.results }}`. `data` is the analytics query's response body verbatim (the step dispatches to the same handler as the REST endpoint), so the shape depends on `queryType`:

| `queryType` | `data` |
| --- | --- |
| `overview.dau` / `overview.wau` / `overview.mau` | `{ value, previous, deltaPct }` — `previous` is the preceding non-overlapping window (the current one ends at query time, so it carries a partial final day); `deltaPct` is a fraction (`0.25` = +25%), and `1` is the sentinel for a zero base |
| `daily-active` / `rolling-active` | `{ window_days, rows: [{ day_ts, day_label, active_users }] }` |
| `errors.groups` | `{ window_days, rows: [{ fingerprint, normalized_title, source, status_class, total, daily: [{ day, count }], first_seen, last_seen, exemplar }] }` — `daily` is **sparse** (no bucket for a day with no events), so divide by `window_days`, not `daily.length`, for a per-day baseline. `exemplar` is a raw sample of one failure (`{ message, action, scope_key, step_id, run_id, at }`) or `null` |
| everything else | see the per-type table in [Analytics](AGENT_GUIDE_TO_PRIMITIVE_ANALYTICS.md) |

Every payload also carries a diagnostic `_timing`. `ok` is the engine's uniform step verdict, not a field the query returns.

Optional fields:

- `groupBy` — for `events.grouped`. One of `action`, `feature`, `day`, `route`, `country`, `deviceType`, `plan`.
- `filters` — for `events` and `events.grouped`. Array of `{ field, operator, value }`; values are capped at 200 chars. Same field/operator set as the `/analytics/events` REST endpoint. **At most 10 entries** — an eleventh fails the step non-retryably with the endpoint's 400 body, whose `error` reads `Too many filters: N. Maximum is 10.`, so the run stops rather than returning an aggregate narrowed to the first ten.
- `page` — 0-indexed page number for the `events` feed.
- `query` — search string (email or ULID) for `users.search`.

```toml
[[steps]]
kind = "analytics.query"
queryType = "events.grouped"
windowDays = 7
groupBy = "feature"
filters = [
  { field = "feature", operator = "is",       value = "billing" },
  { field = "action",  operator = "contains", value = "upgrade" },
]
saveAs = "billingUpgrades"
```

### `script`

Runs a sandboxed [Rhai](https://rhai.rs/) script over JSON input and returns JSON. Use it for transforms too involved for a templated `transform` step (nested reshaping, derived fields, array map/filter/reduce). Full concept, input/output contract, `parse_json`, `ulid()`, per-step limits, error codes, and telemetry: [Scripts guide](AGENT_GUIDE_TO_PRIMITIVE_SCRIPTS.md).

```toml
[[steps]]
id = "normalize"
kind = "script"
ref = "normalize-order"     # required — the Script name (unique per app)
saveAs = "order"
# configId = "..."          # optional — pin a specific ScriptConfig for determinism
[steps.with]                # input context passed to the script (templated by the engine)
raw = "{{ steps.fetch.body }}"
currency = "{{ input.currency }}"
```

- `ref` (required) names a `Script` — a stored Rhai body, unique per app, authored only through sync (`transforms/<name>.rhai`, mirrored by `primitive config push`/`pull`); see the Scripts guide's [Script model](AGENT_GUIDE_TO_PRIMITIVE_SCRIPTS.md#the-script-model).
- `with` is the JSON context handed to the script, templated by the engine before the script runs; see the Scripts guide's [Input and output](AGENT_GUIDE_TO_PRIMITIVE_SCRIPTS.md#input-and-output) for how it's exposed inside the script, the `output`/`ok`/`scriptMetrics` result envelope, and `parse_json` gotchas.
- `configId` (optional) pins a specific `ScriptConfig`, bypassing the active-config lookup the runner otherwise does on every execution — see [Live resolution at run time](AGENT_GUIDE_TO_PRIMITIVE_SCRIPTS.md#live-resolution-at-run-time).
- `limits` (optional) lowers the per-run ceilings; see [Per-step limits](AGENT_GUIDE_TO_PRIMITIVE_SCRIPTS.md#per-step-limits) for the fields, bounds, and the fixed (non-lowerable) `with` input cap.

### `block.call`

A single unified step over the four executable block types. It **lowers** at run time to the typed runner for the referenced block and reuses that step's full contract (resolution, success verdict, durable retry policy) — it adds no capability the typed steps don't already have. Reach for the typed step (`prompt.execute`, `integration.call`, `script`, `workflow.call`) directly unless you specifically want one call site parameterized by block type.

```toml
[[steps]]
id = "run-block"
kind = "block.call"
blockType = "prompt"          # required — prompt | integration | script | workflow
blockKey = "summarize"        # required — the block's key (promptKey / integrationKey / script ref / workflowKey)
# configId = "..."            # optional — pin a specific config
# version = "active"          # optional — "active" (the default) uses the block's active config; any other value pins it
[steps.input]                 # unified input, mapped to the lowered step's field
content = "{{ steps.fetch.body }}"
```

- `blockType` selects the runner it lowers to: `prompt` → `prompt.execute`, `integration` → `integration.call`, `script` → `script`, `workflow` → `workflow.call`. An unknown `blockType`, or an empty `blockKey`, fails the step non-retryably.
- `input` is the unified input; per type it maps to the lowered step's own field — prompt `variables`, integration `request`, script `with`, workflow `input`. The type-specific field name still works and takes precedence over `input` (pass `variables` for a prompt, `request` for an integration, `with` for a script).
- `configId` pins a specific config; `version` also pins one unless it's the `"active"` sentinel (the default — use the block's active config). Config/version pins apply to `prompt`, `integration`, and `script` blocks; a `workflow` block ignores them.
- Passthrough fields reach the lowered step: prompts take `modelOverride` and `expect` (the output-type declaration — same contract as on `prompt.execute`); integrations take `attachments` / `bodyMode` / `multipartFields`; scripts take `limits`.

### `iterate-users`

Fans a child workflow out across **every user in the app**, once per user, as a restartable singleton. Built for large per-user batch jobs (backfills, per-user digests, recomputations) that must survive restarts without re-processing completed users or holding the whole user set in memory. **System-only** — the workflow must set `runAs = "system"` (see [Execution identity](#execution-identity-runas-system-workflows)), else it's rejected at save time.

```toml
[[steps]]
id = "backfill"
kind = "iterate-users"
iterationName = "2026-06-prefs-backfill"   # required — stable name; identifies this iteration
saveAs = "backfillResult"
pageSize = 100                # users fetched per page (default 100)
concurrency = 25             # per-page fan-out (default 25, max 100)
onConflict = "skip"          # "skip" (default) | "refuse" if an iteration is already running
onPartialFailure = "continue" # "continue" (default) | "fail"
[steps.source]
mode = "app"                 # iterate the app's full user roster
[steps.perUser]
workflowKey = "process-one-user"   # the per-user workflow to run
[steps.perUser.input]               # static input merged into each child run
reason = "preferences backfill"
```

- **Bounded memory**: users are paged (`pageSize`) rather than loaded all at once, so the step scales to large user counts.
- **Singleton per app**: a per-app lock (`iterationName` keys it) guarantees only one iteration runs at a time and tracks aggregate progress, so a restarted run resumes where it left off instead of starting over. `onConflict` decides whether a second start `skip`s or `refuse`s while one is live.
- **`iterationName` is templated**: `{{ ... }}` is interpolated once when the step runs. A plain string keys one persistent singleton; a date template (e.g. `"digest-{{ today }}"`) makes each day a distinct iteration; `{{ now }}` or `{{ uuid }}` makes every fire unique, disabling the resume-a-singleton behavior.
- **Inspect / reset from the CLI**: `primitive workflows iterations list` shows each iteration's status, acquire mode, and processed/failed counts; `... get <name>` shows one in detail (run/instance ids, lock expiry, succeeded/failed/skipped, last error, a sample of failed user ids); `... reset <name>` clears a **terminal** (completed/failed) iteration so its next trigger runs fresh — a running iteration is refused (409). No `--force`.
- Each user is processed by the `perUser.workflowKey` child workflow. The iterated user's id is injected into the child as `input.userId` automatically, so a child with no `perUser.input` block still reads `{{ input.userId }}`. `onPartialFailure` controls whether per-user failures stop the whole iteration or are tallied and skipped.
- `perUser.input` values are rendered per user against a `user` binding (`user.userId`, `user.role`) with full template syntax — filters and fallbacks included (`{{ user.userId | upper }}`, `{{ user.role || 'member' }}`). A key set in `perUser.input` overrides the injected `userId` default (author-wins).
- Prefer `iterate-users` over a hand-rolled `forEach` across a `users.list` query when the fan-out is app-wide and long-running; `forEach` is better for bounded, in-run collections.
- **`forEach` on an `iterate-users` step is rejected at save time** (`'iterate-users' cannot be used with forEach`): the step already fans out over the entire roster, so nesting it in a loop multiplies the whole user base by the loop length. Put the `iterate-users` step in a child workflow and `forEach` over `workflow.call` if you genuinely need a per-item app-wide pass.

**Filtering by a metadata value.** A `metadataFilter` block fans out over only the users whose `"user"`-resource metadata matches. The roster is still enumerated in full, but non-matching users are skipped **before** dispatch (no child run at all), so don't write the "dispatch a child for everyone and bail inside it" pattern:

```toml
[[steps]]
id = "summaries"
kind = "iterate-users"
iterationName = "nightly-portfolio-summary"
[steps.metadataFilter]
category = "billing"                      # a category defined for resourceType "user"
key = "status"                            # a key declared in that category's schema
in = ["trialing", "active", "past_due"]   # exactly one of: eq | in | exists
[steps.perUser]
workflowKey = "portfolio-summary-one-user"
```

- Exactly one operator: `eq = <string|number|boolean>` (strict — `1` ≠ `"1"`), `in = [<values>]` (non-empty), or `exists = true|false`. `category`/`key` are static (no `{{ }}`, no `#`); `eq`/`in` values may be templates and render **once** at the start of the iteration, so all pages use the same value (an unresolved reference fails the run before any child starts).
- No metadata row for the category, no such key, or a null value → never matches `eq`/`in`, and is exactly what `exists = false` matches.
- `stringset` and `date` fields accept only `exists` (an array can't equal a scalar; a date may be stored as epoch ms or ISO for the same instant).
- The step result gains `filteredOutCount` (rejected rows), present — including `0` — exactly when `metadataFilter` is set; `totalProcessed` still counts only dispatched users.
- An undefined category, an undeclared key, or an operand the declared field type can never hold fails the run by name instead of matching nobody; structural mistakes are rejected at save time (`steps[i].metadataFilter…`).
- **No `readRule` evaluation.** Unlike `metadata.read`, this is a direct system-only read — the step acts *about* users, never *as* one (as `metadata.resolve` does). A category `readRule` does not gate it.

## Templating

`{{ ... }}` resolves paths into the run context. Context vars:

| Var | Source |
|---|---|
| `input` | The `rootInput` passed to start() |
| `selected` | Result of `selector` (or current `forEach` item if no `as`) |
| `steps` | `steps[stepId]` for every prior step |
| `outputs` | `outputs[saveAs]` for every prior `saveAs` |
| `meta` | The `meta` you passed to `start()`, plus `meta.startedAt` — the run's stable start timestamp (ISO 8601), auto-populated and identical across every step and retry (equals `getStatus().startedAt`; an inline `workflow.call` child inherits the parent's). Everything else in `meta` is only what you set. |
| `secrets` | App secrets (read-only) |
| `vars` | App [config vars](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md#config-vars) (read-only) — `{{ vars.KEY }}` |
| `<asVar>` | Current item inside a `forEach` step |
| `loop`, `iteration` | `{ index, count, first, last }` inside `forEach` (use `iteration` in CEL `runIf` since `loop` is reserved) |
| `user` | In a `runAs: "caller"` run, the invoking user (`user.userId`) — bound in step params, `forEach` source expressions, and `forEach` bodies. Inside an `iterate-users` step's `perUser.input`, `user` is instead the iterated user's row (`user.userId`, `user.role`). Unbound in a `runAs: "system"` run (except the `iterate-users` case). |
| `now`, `today`, `uuid`, `ulid` | Built-in zero-arg helpers (see below) |

### Built-in template helpers

Four zero-arg helpers are always available in templates. They re-evaluate on every reference — `{{ now }} {{ now }}` produces two different timestamps.

| Name | Returns |
|---|---|
| `now` | Current ISO 8601 timestamp (e.g. `2026-05-20T14:38:00.123Z`) |
| `today` | Current UTC date as `YYYY-MM-DD` |
| `uuid` | Fresh random UUID v4 |
| `ulid` | Fresh random ULID (lexicographically sortable) |

```toml
filename = "report-{{ today }}-{{ ulid }}.pdf"
correlationId = "{{ uuid }}"
generatedAt = "{{ now }}"
```

User-provided keys win over built-ins. If you bind `as = "uuid"` in a forEach, pass `input.now`, or have a step output named `today`, the user value shadows the built-in for that scope.

**Single-expression mode**: when the entire string is one `{{ ... }}`, the raw value (array/object) is returned, not stringified. Otherwise expressions are coerced to strings and interpolated.

**Fallback** with `||`:

```
"{{ input.title || 'Untitled' }}"
"{{ input.a || input.b || 'default' }}"
```

**Filters** with `|` (single pipe — `||` is fallback):

```
{{ input.data | json }}                         # pretty JSON
{{ input.tags | join: ', ' }}
{{ input.items | length }}
{{ input.users | pluck: 'email' | uniq }}
{{ input.list | where: 'status', 'active' }}
{{ input.list | sort: 'name' | first }}
{{ input.text | upper | truncate: '100' }}
{{ input.amount | toFixed: '2' }}
{{ input.x | default: 'fallback' }}
{{ input.id | expect: 'string' }}              # throws on type mismatch
```

Available filters (see `src/workflows/runner/templates.ts` for full list):
- Type: `json`, `string`, `number`, `boolean`, `default`, `expect`
- String: `upper`/`uppercase`, `lower`/`lowercase`, `trim`, `split`, `replace`, `truncate`, `startsWith`, `endsWith`, `contains`
- Array: `length`/`size`, `first`, `last`, `keys`, `values`, `join`, `pluck`, `where`, `sort`, `reverse`, `flatten`, `uniq`, `compact`, `slice`, `concat`
- Number: `round`, `floor`, `ceil`, `abs`, `toFixed`
- Date: `now`, `toISOString`

Templates have **no arithmetic** (`{{ a + b }}` won't work). Move math into a step or filter chain.

**An unresolved reference fails the step.** A `{{ }}` expression whose path is not in the run context fails the step non-retryably, in both interpolation mode (`"prefix-{{ steps.x.y }}-suffix"`) and single-expression mode (`"{{ steps.x.y }}"` alone). The error names every unresolved expression in the step and the keys that were available; the step is recorded as `failed` with `templateWarnings` naming each path, so it reaches run history and error analytics like any other step failure. Nothing is substituted into the output. A path that *resolves* is never a failure whatever its value: a resolved `null` is typed `null` in single-expression mode and `"null"` in interpolation; a resolved empty string is `""` in both. To mark a reference optional, say so **at the expression** — `{{ steps.x.output.result | default: '' }}` or `{{ input.title || 'Untitled' }}`. `default` (and `now`) rescues a miss from anywhere in the filter chain, not only first position, and a bare `| default` renders `''`. That makes `| default:` required, not optional, for a field built from an optionally-skipped step. There is no workflow-level or step-level setting that changes any of this: `strict` and `strictParams` are accepted and ignored, since the failure they used to opt into is now universal. When a template references a root outside the six valid roots (`input`, `steps`, `outputs`, `meta`, `secrets`, `vars`), the error lists them — catching typos like `{{ inputs.userId }}`. The statically-decidable half of that is caught earlier: `primitive config push` (and the definition-save API) rejects an expression that can never resolve — an unknown root, or a `{{ steps.<id>… }}` / `{{ outputs.<saveAs>… }}` naming something the file does not declare — naming the step, the expression and the valid roots. Contextual roots are scoped: `selected`, `user`, `md` and the built-ins are valid on any step, while `iteration`, `loop` and the loop's binding names (the explicit `as` — one name, or one per array in the `{ zip, as }` form — or `item` when `as` is omitted) are valid only inside a `forEach` step's other fields, never in the `forEach` source, which resolves before any item is bound. An expression carrying `| default` (or `| now`), a fallback chain ending in `''` or `0`, or anything undecidable such as a dynamic bracket key (`{{ steps[input.branch].output }}`) is left alone and stays a run-time concern.

**Bare path fields resolve strictly too.** A `forEach` source, a `selector` path and a `runs` count are bare paths rather than templates, and they have no fallback syntax. The path must resolve: a source that resolves to an empty list (or a paginated object with empty `items`) iterates zero times as always, but one that does not resolve, or that resolves to a non-list, fails the step naming the path or the resolved type — unless the step's own `runIf` is evaluable at step level and false, in which case the step is skipped and the source is never resolved. For a genuinely optional list, make the producing step emit an empty one.

## `runIf` (CEL, not templates)

```toml novalidate
runIf = "input.shouldRun"                        # truthy
runIf = "outputs.text.length < 1000"             # comparison
runIf = "steps.check.isMember && input.amount > 0"
runIf = "steps.previous.ok"                      # uniform verdict on every object result
runIf = "!steps.fetch.skipped"                   # booleans are always present, never absent
runIf = "steps.fetch.succeeded && steps.fetch.output.count > 0"   # guard, then read
runIf = "steps.charge.failed"                    # reacts to a captured failure"
```

CEL context: `input`, `selected`, `steps`, `outputs`, `meta`, `secrets`, `vars`, plus `iteration` (and `as`-var) inside `forEach`, and `expr.<name>` for any [named guard](#named-guards-expr) declared on the workflow. **Do NOT wrap in `{{ }}`** — `runIf` parses CEL directly. A CEL evaluation error fails the step (or is captured by `continueOnError`).

`vars` is the app's full config-vars map — the same values `{{ vars.KEY }}` renders in a step's config, with no `vars = [...]` declaration required in the workflow. (Declared secrets are the opposite: `secrets` in a guard binds ONLY the keys the workflow's manifest declares, never the full app-secret map.) A guard reading a key the app doesn't hold throws `No such key: KEY`, so test presence first with `'KEY' in vars` or `vars.?KEY` when the var may not exist yet.

The map is loaded once per engine invocation and bound into both guards and templates, so the two always agree within an invocation. A parallel `forEach` (`concurrency > 1`) runs each batch as its own durable child that loads its own snapshot: a var edited while such a step is in flight can be visible to one batch and not another, and each batch's own value governs which of its items run.

### Named guards (`expr.*`)

Declare reusable CEL guards in a top-level `[expr.cel]` table — one entry per name, `<name> = "<CEL expression>"` — then reference them as `expr.<name>` (bracket form `expr['name']` / `expr["name"]` also works) in any step guard: `runIf`, a `forEach` step's `successWhen`, and a `switch` case's `when`.

```toml novalidate
[expr.cel]
isPremium   = "input.plan == 'premium' && steps.account.ok"
withinQuota = "steps.usage.count < input.limit"
eligible    = "expr.isPremium && expr.withinQuota"   # definitions may reference each other

[[steps]]
id = "charge"
kind = "database.mutate"
runIf = "expr.eligible"
```

- **Definition scope is fixed and stripped.** Every named guard evaluates against the run-state scope only — `input`, `steps`, `outputs`, `meta`, `secrets`, `vars`, plus `user` in a `runAs: "caller"` run — *never* the reference site's `selected`, `iteration`, or `as`-var bindings. So `expr.isPremium` resolves identically at every reference site; a guard that needs a per-iteration value can't be a named guard.
- **Resolution is by name, not textual expansion.** A referenced guard evaluates as its own expression against that fixed scope, and each guard is memoized per evaluation so wide reuse or a fan-out DAG stays cheap. A guard that throws (or fails to parse) fails the referencing `runIf` or switch `when` with a `RunIfError` named `expr.<name>` — the author's name, not the reference-site expression. At a `successWhen` site the classifier's error fallback applies instead: the error is logged and the iteration is classified `succeeded` (see [`successWhen`](#successwhen-functional-success-vs-empty)).
- **References must form a DAG.** Cross-definition references are allowed; a reference cycle (`a` → `b` → `a`) is rejected at save time.
- **Validation is developer-facing.** `primitive config push` fails fast — and the server 400s — on an `expr.<name>` reference (in a definition body *or* a step guard) that names no declared definition, on a reference cycle, or on an empty definition body. `primitive config pull` writes the `[expr.cel]` table back out, so definitions round-trip.

### Safe navigation

CEL optional types are enabled in every workflow context (`runIf`, `accessRule`, group/database access rules, cron triggers). Use them to collapse multi-conjunct null guards into single-line expressions.

| Syntax | Meaning |
|---|---|
| `steps.foo.?bar` | Optional field access — returns an optional, never throws on missing |
| `steps.foo[?"bar"]` | Optional index access (same, for map/list lookups) |
| `opt.orValue(default)` | Unwrap optional `opt`, falling back to `default` |
| `opt.hasValue()` | True if the optional `opt` is set |
| `optional.of(x)` / `optional.none()` | Construct optionals explicitly |

Common operations work directly on optional values — no `.orValue()` unwrap needed:

| Expression | Semantics |
|---|---|
| `size(steps.x.?data)` | `none` → `0`; a present-but-`null` value → `0`; `some(xs)` → `xs.size()`. Works for lists, maps, strings, bytes. |
| `steps.x.?body.?err == null` / `!= null` | `none == null` is `true` (an absent path compares equal to null), so `!= null` means "present and non-null". |
| `steps.x.?body.?token != 'v'` | **TRUE when the path is absent.** `none` never equals a non-null value, and `!=` is derived as the negation — this is is-distinct-from, not a presence check. |

**Presence checks on strings, lists, maps, and bytes use `size()`, not a bare `!=`.** A guard like `runIf = "input.metadata.?userId != ''"` *passes* for input with no `userId` at all — the opposite of the intent. Write "present and non-empty" as `size(input.metadata.?userId) > 0` (and "absent or empty" as `== 0`): `size()` is `0` for an absent path and for a present-but-null value, so it covers both. For values without a size (numbers, booleans), use `.hasValue()` instead — `size()` on a present number or boolean is an evaluation error (which fails the step), and `.hasValue()` is `true` for a present-but-`null` value. Comparing against `null` is the one safe bare comparison — `.?err != null` correctly means present-and-non-null.

```cel
runIf = "size(steps['fetch'].?data) > 0"
runIf = "steps['profile'].?body.?err != null"
runIf = "size(steps['profile'].?body.?token) > 0"   # present and non-empty — NOT `.?token != ''`
```

You can still use `.orValue()` and `.hasValue()` when you need explicit control over the fallback value:

```cel
runIf = "steps.fetch.?data.?items.orValue([]).size() > 0"
runIf = "steps.profile.?role.hasValue() && steps.profile.role == 'owner'"
```

Without safe navigation, `steps.fetch.data.items` throws on any missing intermediate; with it, the chain short-circuits to `optional.none()`.

## `forEach`

```toml
[[steps]]
id = "notify"
kind = "email.send"
forEach = "steps.team.items"
as = "member"
maxItems = 500
to = "{{ member.email }}"
subject = "Update"
htmlBody = "<p>Hi {{ member.name }}</p>"
```

The source is a **bare path** into the run context (to an array, or to `{items: [...]}`) — not a template, unlike every other field on the step. A whole-value template wrapping one bare path (`forEach = "{{ steps.team.items }}"`) is unwrapped and resolves to the same array. Any other use of `{{ }}` in the source — a partial template (`"rows {{ steps.team.items }}"`), several expressions (`"{{ a }} {{ b }}"`), an empty expression (`"{{ }}"`), or a wrapped expression that is not a bare path (`"{{ steps.team.items || [] }}"`, `"{{ steps.team.items | default(1) }}"` — operators and filters work in the step's other fields, not in the source) — and an empty source string are rejected at push time; they could never resolve, and used to iterate zero times while reporting the step `completed`. A source that *resolves* to an empty list (or to `{items: []}`) iterates zero times with `ok: true` — that is the empty-collection case, not an error — while a source path that is not in the run context fails the step, unless the step's own `runIf` gates it: a gate the engine can evaluate at step level (one that reads no loop variable, e.g. `runIf = "steps.sync.ok"` beside `forEach = "steps.sync.body.added"`) is evaluated first, and when it is false the step is skipped and the source is never resolved. Kebab-case step ids are ordinary paths (`forEach = "{{ steps.list-securities.items }}"` works); a path segment with other punctuation needs the bracket form (`"{{ steps['odd.key'].items }}"`).

Output is always `{ items: [...per-iter results], errors: [{index, error}], totalSucceeded, totalFailed, totalEmpty, ok }` — even when there are no errors. Results are ordered by input index regardless of completion order. `ok` is `true` iff every iteration succeeded (and the source was non-empty); use it in `runIf` on the next step.

When an iteration fails under `continueOnError` and the failure was **classified** (e.g. a conditional write's `CONDITION_NOT_MET`), its `code` and `status` are preserved on both the error entry and the item — `steps.<id>.errors[0].code`, `steps.<id>.items[0].code` — so a following step can tell that failure apart from any other one. This holds for parallel (`concurrency > 1`) fan-out too.

**Parallel forEach** — add `concurrency` to fan out iterations across multiple lanes:

```toml
[[steps]]
id = "notify"
kind = "email.send"
forEach = "steps.team.items"
as = "member"
concurrency = 5           # run up to 5 iterations at once
maxItems = 500
to = "{{ member.email }}"
subject = "Update"
htmlBody = "<p>Hi {{ member.name }}</p>"
```

When `concurrency = 1` (the default), iterations are sequential. When `concurrency > 1`, the engine fans them out in parallel batches — in durable mode each batch runs as a child workflow so restarts don't re-run completed items. For app-wide, restartable fan-out that must survive restarts without re-processing completed users, use `iterate-users`.

When a parallel `forEach` batch's combined output exceeds the inline size limit (~1 MB), the engine automatically offloads the batch result to managed object storage and rehydrates it transparently for the next step. Large per-iteration outputs are handled for you, but the offload adds a storage round-trip — keep per-iteration outputs lean when you can.

### Batch database updates

A set of database writes is a single `database.batch` step — one operation call and one step-run row for the whole set, instead of one of each per record:

```toml
[[steps]]
id = "import-contacts"
kind = "database.batch"
databaseId = "{{ input.dbId }}"
operationName = "createContact"
batch = "{{ input.contacts }}"
# Output: { importedOps, failedOps, results: [{ index, success, ids, error? }] }
# Per-item failures do not throw: the step completes with ok = false and names
# them in `results`. See the `database.batch` section above for the full contract.
```

Each item of `batch` is that invocation's params, so the array's shape must match the operation's parameters (rename fields in a `transform` step first if it doesn't).

Reach for `forEach` over `database.mutate` only when each record genuinely needs its own step run — e.g. per-record retries, or per-record params that a template can't derive from the item. Use `database.applyToQuery` to mutate every record matching a server-side filter without enumerating items at all.

### Zip mode (parallel arrays by index)

When the inputs you want to iterate live in several arrays of the same length, use the `{ zip, as }` form. Each iteration binds one variable per `as` name from the matching `zip` expression at the same index.

```toml
[[steps]]
id = "send"
kind = "email.send"
forEach = { zip = ["steps.list.users", "steps.list.tokens"], as = ["user", "token"] }
to = "{{ user.email }}"
subject = "Your code"
htmlBody = "<p>Code: {{ token }}</p>"
```

This replaces the silently-corrupting `steps.list.tokens[iteration.index]` pattern — if the arrays drift in length, the engine surfaces it instead of indexing past the end.

### `successWhen` (functional success vs. empty)

For iterations that don't throw but didn't accomplish anything meaningful (e.g. an HTTP 200 with `body.matches = []`), use `successWhen` to classify the outcome.

```toml
[[steps]]
id = "lookup"
kind = "integration.call"
forEach = "input.queries"
as = "q"
successWhen = "result.body.matches.size() > 0"
[steps.request]
method = "GET"
path = "/search?q={{ q }}"
```

The predicate runs against each iteration's `result` plus the usual `input`/`steps`/`outputs`/`meta`/`iteration` context. Truthy → `functional_status: "succeeded"`; falsy → `"empty"`; throws → `"failed"` (the predicate is not evaluated). `totalEmpty` on the step output counts the empty bucket. A broken predicate falls back to `"succeeded"` and the workflow continues.

## Error handling

- **Default**: a failed step throws and the workflow fails.
- `continueOnError = true`: failure is captured as `steps[id] = { error, errorDetails, ok: false, errored: true }` and execution continues. A **classified** failure (e.g. a conditional write's `CONDITION_NOT_MET`) also carries `code` and `status` — see the verdict-namespace table below.
- **Unresolved template reference**: any `{{ }}` expression in the step whose path does not resolve throws a non-retryable, path-listing error. Always on; mark a reference optional with `| default` / `||`.
- `expect:` filter (in templates): runtime type check.
- `[[compensate]]` block at the top level runs after a failure (when `continueOnError` is not set). Compensate steps see `steps._error = { message, stepId }`. Compensate runs only in sync execution paths (e.g., `executeWorkflowSync`); not all engine modes invoke it.

```toml
[[steps]]
id = "deduct-token"
kind = "database.mutate"
# ...

[[steps]]
id = "call-api"
kind = "integration.call"
# fails here → compensate runs

[[compensate]]
id = "restore-token"
kind = "database.mutate"
runIf = "steps.deduct-token != null"
# ...
```

Per-step retries on transient errors are handled by the workflow engine automatically and are not configurable from TOML. Mark errors non-retryable by ensuring upstream calls return 4xx (the engine wraps 4xx≠429 as non-retryable).

## Output contract

After all steps run:

- `steps[id]` holds every step's output (including skipped/failed entries).
- `outputs[saveAs]` holds outputs for steps with `saveAs`.
- The workflow's final result is `outputs.output` if any step used `saveAs = "output"`, otherwise the full `outputs` map.

**Best practice**: end with a `transform` step using `saveAs = "output"` that explicitly shapes the return value. Because a single-expression `output` forwards the raw value (see [Templating](#templating)), the minimal form is a passthrough — useful when one step's result *is* the run's result:

```toml
[[steps]]
id = "result"
kind = "transform"
output = "{{ steps.evaluate.output.result }}"   # single expression — forwards the raw value
saveAs = "output"
```

Arrays and primitives pass through byte-for-byte. An **object** result additionally carries the engine's `ok: true` verdict stamp (see [Uniform step verdict](#uniform-step-verdict)) — the stamp overwrites any `ok` key of your own on the forwarded object, so don't use `ok` as a data field in a run result.

### Uniform step verdict

Every **object** entry in `steps[id]` carries a reserved verdict namespace alongside the runner's own fields (a successful array or primitive result passes through unstamped, with no verdict keys):

| Field | When set |
|---|---|
| `state: string` | On every object result. One of `"succeeded"`, `"failed"`, `"skipped"` — the step's execution outcome, derived from the fields below (`skipped` wins over `ok`) |
| `succeeded`, `failed`, `skipped` | On every object result, as booleans mirroring `state`. Always PRESENT (`false` rather than absent), so `!steps.x.skipped` and `steps.x.succeeded && steps.x.output...` are safe on any entry |
| `ok: boolean` | On every object result. `true` if the step ran and the runner classified it as successful; `false` for skipped, errored, or kind-specific failures (e.g. an `integration.call` that returned HTTP 5xx) |
| `errored: true` | The step threw but was captured by `continueOnError = true` |
| `error`, `errorDetails` | Companion fields populated when `errored: true` |
| `code`, `status` | Companion fields populated when `errored: true` **and** the failure was classified (e.g. `CONDITION_NOT_MET` / `409`). Absent when the failure carried no classification, so only branch on `steps.<id>.code` where a classified failure is the case you are testing for |

`state`, `succeeded`, `failed`, `ok`, `skipped`, `errored`, `error`, and `errorDetails` are written by the engine and override any same-named field the runner would have produced. Downstream `runIf` and templates can rely on them whenever the step's result is an object — skipped and errored steps always are:

```toml novalidate
runIf = "steps.fetch.ok"
runIf = "!steps.fetch.skipped"
runIf = "steps.fetch.failed"                # any failure, including one with no response body
runIf = "steps.fetch.state == 'skipped'"
```

`state: "failed"` covers every way a step can fail while the run continues: a thrown failure captured by `continueOnError` (timeouts and transport errors included — no response body needed), a kind-specific failure such as an HTTP 5xx or a `body.error`, a `runIf` CEL error captured the same way, and a `forEach` with at least one errored iteration. Because the booleans are always present, testing `has(steps.x.skipped)` tells you nothing — test the value.

The same verdict rides on steps executed during a `compensate` block, including compensation steps that skip or fail, so a later compensation step can branch on `steps.<earlier>.failed`.

When a step reads `steps.<upstream>.output.*` and that upstream can be skipped, a skipped upstream leaves no `output` key and an unguarded multi-segment read throws `No such key: output`. Three structural fixes: lead with the upstream's verdict (`steps.fetch.succeeded && steps.fetch.output.count > 0` — the guard is `false`, not absent, so `&&` absorbs the read), guard the read with safe navigation (`steps.fetch.?output.?items`), or declare `skipWhenSkipped = ["fetch"]` so the dependent auto-skips whenever `fetch` skips — its `runIf` is never evaluated. The skip is transitive and reacts only to `skipped: true` (an upstream that errored under `continueOnError` does not trigger it). List only earlier step ids; a save-time warning flags an unguarded skippable read or a `skipWhenSkipped` entry naming an unknown or forward step id.

## Access control

`accessRule` is a CEL expression. **A caller workflow with no rule denies every non-admin start** (existing workflows included); `accessRule = "true"` restores open access, and a create without one is refused with `accessRule is required (use "true" for a workflow any app member may start)`. A refused start answers `403 { errorCode: "WORKFLOW_ACCESS_DENIED" }` on all three paths, with the cause (`no access rule` vs `denied`) only in the worker log.
{{#lang swift}}
The refusal reaches Swift as a thrown `HttpError` with `serverCode == "WORKFLOW_ACCESS_DENIED"` — switch on that, not on the message text.
{{/lang}}
Evaluated when:
- A client calls `workflows.start()`.
- A `workflow.call` step invokes the workflow from another workflow.

NOT evaluated for inbound webhook or cron triggers (those bypass it entirely — a webhook handles its own auth, e.g. a Stripe signature). A webhook- or cron-triggered workflow must be `runAs: "system"` (see [Execution identity](#execution-identity-runas-system-workflows)), and a system workflow takes **no** `accessRule` — the rule is never evaluated on a system run, so a non-empty one is rejected at save/push time (see below). What prevents direct client invocation of a webhook workflow is the system-invocation gate plus the webhook's signature verification. Reserve `accessRule` for `runAs: "caller"` workflows, where it genuinely gates who may start a run.

Behavior:
- No rule → **denied** for every caller but an app admin or owner (set `"true"` to allow any app member).
- `admin`/`owner` always bypass.
- Otherwise, evaluate against `user.userId`, `user.role`, plus `hasRole(role)`, `isMemberOf(groupType, groupId)`, `memberGroups(groupType)`.

Set `accessRule` in the `[workflow]` TOML block and push it — an absent or empty rule is sent as "no rule" rather than dropped, so removing the line clears the rule. To change it later, edit `workflow.accessRule` in the file and run `primitive config push --only workflow/<key>`.

## Execution identity (`runAs`, system workflows)

`runAs` declares the principal a run executes as — `caller` (default) or `system`. It's a top-level `[workflow]` field.

- `runAs = "caller"` (default) — the run executes as the invoking user. Every step acts with that member's permissions: `document.*` steps enforce the caller's per-document ACL, group/data ops are checked against their roles, and `blob.*` steps evaluate the bucket's access policy per op (write/read/share/delete — see the blob steps section). `accessRule` gates who may start it.
- `runAs = "system"` — the run executes as the app's synthetic per-app principal (`sys:<appId>`, also the `WorkflowRun` partition key). App-privileged on the **raw** data baseline: direct document/record read/write/delete/manage skips the per-caller ACL, and `blob.*` steps skip the bucket policy. It does **not** bypass a registered database operation's own `access` CEL — a `database.*` step evaluates that rule on every call, system run included (`access = "false"` → `403`; reserve a workflow-only op with `fromWorkflow('key')`, not `"false"`).

The invocation gate is enforced once at top-level start (HTTP start/run-sync, cron, webhook, admin test) — never silently downgraded:

| invoker | `runAs:"caller"` | `runAs:"system"` |
|---|---|---|
| member | runs as the member | **403** "Members cannot run system workflows" |
| admin/owner | runs as the admin | runs as system |
| cron | **403** (cron may only run system workflows) | runs as system |
| webhook | **403** | runs as system |

Nested/child runs (`workflow.call`, durable `forEach` batches) never re-resolve identity — they inherit the parent's verbatim. Every system run records attribution (`initiatedByUserId` + `initiatorKind` of `admin`/`cron`/`webhook`/`test`) for audit; it is not a security control.

**Sensitive capabilities.** A system run gets the raw data baseline unconditionally (registered-operation `access` rules still evaluate — above). Sensitive operations are opt-in via `capabilities` (StringSet, allowlist-validated at save time):

```toml
[workflow]
key = "sync-roster"
runAs = "system"
capabilities = ["membership"]
```

`membership` gates the `group.addMember` / `group.removeMember` / `group.removeAll` steps in a **system** run — without the grant they reject (`group.addMember requires the 'membership' capability`). Read-only group steps (`checkMembership`, `listMembers`, `listUserMemberships`) are never gated, and caller runs are governed by access rules, not capabilities, so the gate never applies to them. When the group type carries no explicit rule set (no group type config, or one with `ruleSetId` unset), a system run holding `membership` is allowed through anyway — the granted capability stands in for a CEL rule the app never configured. A group type with an explicit rule set is still governed by that rule set, never overridden by the capability.

Four more capabilities gate the resource-lifecycle, collection-membership, and `document.create` step kinds in a system run (deny-by-default, checked in `resolveLifecyclePrincipal`/`authorizeSystemLifecycle`):

| Capability | Gates |
|---|---|
| `resource-provision` | `database.create`, `group.create`, `collection.create`, `document.create` |
| `resource-teardown` | `database.delete`, `group.delete`, `collection.delete` |
| `resource-grant` | `collection.grantGroupPermission`, `collection.addDocument`, `collection.removeDocument` |
| `resource-teardown:any` | Escalation: without it, a system-mode `*.delete` may only target a resource whose `createdBy` is `sys:<appId>`; with it, teardown also reaches resources a member created |

A system-mode create attributes the resource to `sys:<appId>`. Missing capability fails the step non-retryably before anything is written; see [Resource lifecycle steps](#resource-lifecycle-steps) and [`collection.addDocument` / `collection.removeDocument`](#collectionadddocument-collectionremovedocument) for the per-kind detail.

Grant or revoke `capabilities` by editing the array in `workflows/<key>.toml` and pushing — `config push` forwards it on create and on update alike, and an empty array revokes every grant.

**`iterate-users` is system-only.** A workflow containing an `iterate-users` step must set `runAs = "system"` — otherwise it's rejected at save time (`'iterate-users' is system-only and may appear only in a runAs:"system" workflow`).

**The resource lifecycle and collection-membership steps run in both modes, capability-gated in system mode.** `database.create`/`delete`, `group.create`/`delete`, `collection.create`/`delete`, `collection.grantGroupPermission`, `collection.addDocument`, and `collection.removeDocument` all run in `runAs:"caller"` workflows under the caller's own CEL rules and doc access, exactly like the equivalent client call. In a `runAs:"system"` workflow, each instead requires its matching capability (`resource-provision`, `resource-teardown`, or `resource-grant` — see above); missing it fails the step non-retryably before anything is written. A system-mode create is attributed to the run's system principal (`createdBy: "sys:<appId>"`); a system-mode delete may only remove a resource with that attribution unless the workflow also holds `resource-teardown:any` — see [Resource lifecycle steps](#resource-lifecycle-steps).

**No `accessRule` on a system workflow.** A `runAs:"system"` workflow with any non-empty `accessRule` is rejected at save time (`runAs:"system" workflows do not evaluate accessRule — remove the accessRule`) — a system run skips the rule entirely, so a constant (`"false"`), a role/group rule (`hasRole('admin')`), or a `user`-principal reference are all equally dead config. The only fix is to remove it. An absent/empty rule is fine. Enforced on every save path (create, version-create, metadata PATCH on the merged effective state, workflow-config create, and `config push`).

### Subject-user methods (system workflows)

A system run can act **about** a specific app user without impersonating them — the run's actor stays `sys:<appId>` for audit; `userId` is an explicit **subject** parameter. The `*ForUser` step kinds are **system-only** (calling one from a `runAs: "caller"` run throws non-retryably: `<kind> is only supported in runAs:"system" workflows`). In each, the subject is the step's `userId` when set, otherwise `input.userId` (e.g. the iterated subject under `iterate-users`) — an empty or whitespace-only `userId` falls back to `input.userId` rather than shadowing it (`step.userId ?? input.userId`). A subject that resolves to empty fails the step with a remediation error (`<kind>: userId is required but resolved to empty — check input.userId or the iterate-users subject`). The subject must be a member of the app, or the step fails non-retryably (`Subject user <id> is not a member of app <appId>`). These reads need no `capabilities` grant — being a system run is the gate.

| Kind | Key fields | Output |
|---|---|---|
| `user.get` | `userId` | `{ userId, email, name, appRole, rootDocId, disabled }` |
| `user.resolve` | `userId` **or** `email` | `{ userId: null }` on no app-member match, else `{ userId, user }` (`user` shaped like `user.get`) |
| `document.resolveAliasForUser` | `userId`, `aliasKey` | `{ documentId: string \| null, aliasKey, userId }` — app-privileged (alias existence ≠ subject access); `null` on miss (vs. the inline `documentAlias.userId` form, which hard-fails) |
| `document.listForUser` | `userId`, `limit?`, `includeRoot?` (default `true`), `ownerOnly?`, `tag?`, `cursor?`, `forward?` | `{ items: [DocumentInfo...], cursor? }` — direct + root grants only; group- or collection-reachable documents are enumerated via the dedicated `document.listForGroup` / `document.listForCollection` steps |
| `document.getForUser` | `userId`, `documentId`, `systemBypass?` | `{ document: DocumentInfo \| null, permission? }` — requires the subject's effective (direct + group) permission by default (`{ document: null }` when none); `systemBypass: true` reads by id app-privileged (`permission: "system"`) |
| `document.getOrCreateWithAliasForUser` | `userId`, `aliasKey`, `title?`, `permission?` (`read`\|`write`\|`owner`, default `write`), `tags?` | `{ documentId, aliasKey, userId, created }` — created with `createdBy = sys:<appId>`; subject gets the alias + the default grant; race-safe |
| `database.queryForUser` / `mutateForUser` / `countForUser` / `aggregateForUser` / `pipelineForUser` / `applyToQueryForUser` | same fields as the base `database.*` kind, plus `userId` | base kind's output; CEL rules + DB triggers evaluate as the **subject** (`user.userId`, `hasRole`, `isMemberOf` refer to the subject), actor stays system. `ok`/verdict semantics match the base kind |
| `analytics.writeForUser` | `userId`, `action`, `feature` | event attributed to the subject; system actor + workflow run id carried in the event context for audit |
| `document.listForGroup` | `groupType`, `groupId`, `permission?` (`reader`\|`read-write`\|`owner` filter), `limit?` (1–200, default 50), `cursor?` | `{ items: [DocumentInfo + nested `grant` `{groupType, groupId, permission, directGrant, sourceCollectionIds}`], cursor? }` — documents the group can access (direct + collection-derived in one query); item `permission` is the group's grant level. The `permission` filter is post-applied per page, so a filtered page can be short with a cursor still present — page until the cursor is absent. The reported grant level can be stale after mixed direct/collection revokes |
| `document.listForCollection` | `collectionId`, `limit?`, `cursor?` | `{ items: [DocumentInfo + `collectionId`/`addedAt`/`addedBy`, `permission: "system"`], cursor? }` — documents contained in a collection (membership, not per-user access) |
| `document.getRootForUser` | `userId` (subject, or inherited from `input.userId`), `create?` (default `false`) | `{ documentId: string \| null, userId, created }` — the subject's root document id; `create: true` race-safely assigns one (subject gets read-write) |

```toml
[[steps]]
id = "ensure"
kind = "document.getOrCreateWithAliasForUser"
userId = "{{ input.userId }}"   # explicit subject, or inherited from input.userId
aliasKey = "profile"
title = "Profile"

[[steps]]
id = "assignments"
kind = "database.queryForUser"
userId = "{{ input.userId }}"
databaseId = "{{ input.classroomDbId }}"
operationName = "listAssignments"
saveAs = "assignments"
```

Once a subject method returns a concrete `documentId`, the app-privileged `document.query` / `save` / `patch` / `delete` steps read/write it — there are **no** `*ForUser` write kinds. The CRUD `document.*` steps also accept `documentAlias { scope = "user", aliasKey, userId }` to resolve a subject's alias inline (hard-fails on miss).

## Inbound webhooks

External services trigger workflows via inbound webhooks. This is the **inbound** half of a third-party integration; the **outbound** half — calling the provider's API — is a configured integration (see the Integrations guide). Most integrations need both. Define webhooks as `webhooks/*.toml` in the config directory and push with `primitive config push`:

```toml
# config/webhooks/stripe-payments.toml
[webhook]
key = "stripe-payments"             # required, unique per app; names the receive URL
displayName = "Stripe Payments"
workflowKey = "process-stripe"
verificationScheme = "stripe"     # stripe | github | slack | discord | jwt | plaid | custom | none
signingSecret = "{{secrets.STRIPE_WEBHOOK_SECRET}}"  # a whole {{secrets.KEY}} reference; nothing else is accepted
# Availability is server-owned and is NOT authored here: a pushed webhook is in
# service, `primitive webhooks enable|disable` is what changes that, and
# `primitive webhooks archive` retires it (soft delete; keeps key and slot).
# Optional: toleranceSeconds, deduplicationEnabled, deduplicationWindowMs,
# secretGracePeriodMs, maxBodyBytes, [webhook.allowedIps] cidrs, [webhook.inputMapping]
```

### Public-key schemes: `[verification.<scheme>]`

`discord`, `jwt` and `plaid` verify with public key material rather than a shared secret, so they take no `signingSecret` and are configured with a `[verification.<scheme>]` table that round-trips through `config pull`/`push`:

The Admin Console edits all three — the same fields, including the jwt key-source choice, and `plaid`'s two credentials as picks from the app's secrets — and shows every scheme's stored configuration on the webhook's page.

```toml
[verification.discord]
publicKey = "<64-char hex Ed25519 application public key>"
# optional previousPublicKey for a rotation grace window
```

```toml
[webhook]
key = "provider-events"
verificationScheme = "jwt"
toleranceSeconds = 300
deduplicationWindowMs = 360000     # must cover the freshness window (below)

[verification.jwt]
header = "X-Provider-Signature"    # required — the header carrying the compact JWS
algorithms = ["ES256"]             # required — ES256/384/512 | RS256 | PS256 | EdDSA
bodyHashClaim = "request_body_sha256"  # required — claim holding SHA-256 of the body
bodyHashEncoding = "hex"           # hex (default) | base64 | base64url
issuer = "https://provider.example.com"   # optional, checked when set
audience = "https://api.example.com/hook" # optional, checked when set
eventIdClaim = "jti"               # optional; defaults to the body-hash claim value
jwks = { keys = [ { kid = "key-1", kty = "EC", crv = "P-256", x = "...", y = "..." } ] }
```

Every key needs a `kid`, the kids must be unique within the JWKS, and each one must be 1–128 characters from `A-Z`, `a-z`, `0-9`, `_`, `.` and `-` — the same shape the delivery path requires of the token's `kid` header. A kid outside that set (one holding a `/` or a `:`, say) is rejected at push time with `JWKS_KID_INVALID`, rather than stored as a key no delivery could ever select. A document fetched from a `jwksUrl` (below) is deliberately more tolerant: the tenant does not control what the provider publishes, so a key whose `kid` is outside that set is dropped and the remaining keys still verify deliveries — one odd key at the provider cannot take the whole webhook down. If dropping leaves no selectable key, the document is rejected and the delivery fails `key_fetch_failed`. The tradeoff is worth knowing: a provider key with an odd `kid` will never verify a delivery (`kid_unknown`) rather than failing loudly at fetch time; the platform logs a warning with the sent and kept key counts when it drops any.

Exactly one key source is required: the inline `jwks` above, or `jwksUrl` — the provider's published JWKS endpoint, which is what Auth0, Okta and OIDC-style signers give you. Supplying both is a `400 JWKS_KEY_SOURCE_CONFLICT`, supplying neither is a `400`.

```toml
[verification.jwt]
header = "X-Provider-Signature"
algorithms = ["ES256"]
bodyHashClaim = "request_body_sha256"
jwksUrl = "https://tenant.auth0.com/.well-known/jwks.json"
```

`jwksUrl` is validated when the webhook is saved and again on every delivery, and a URL that fails is a `400 JWKS_URL_INVALID` at write time: `https` only, port 443 only, a hostname of two or more labels ending in an alphabetic TLD (so IP literals and single-label names are out), no reserved suffix (`.local`, `.internal`, `.home.arpa`, …), no credentials, no fragment, no `{{secrets.KEY}}` template, at most 2048 characters. The fetched document is subject to the same rules as an inline one (unique `kid`s, a `kty` on every key, no private material) — with the one exception above, that an unselectable `kid` is dropped rather than failing the document — plus transport limits — 64 KiB per document, 8 KiB per key, 50 keys, a 5-second timeout, redirects not followed.

At delivery time the document is cached per webhook, so a burst of deliveries costs one fetch, not one each; a `kid` the cached document does not carry triggers at most one extra refresh a minute, which is what picks up a rotation that publishes and signs at the same moment. When a fetch fails, the last document fetched successfully keeps verifying deliveries for up to an hour; with no such document the delivery is rejected. Failures show on the delivery record as `key_fetch_failed`, `key_fetch_throttled` or `kid_unknown` — the same `rejectionReason` vocabulary the inline source uses, so nothing downstream changes.

The `jwt` scheme accepts a delivery only when **all** of these hold, and returns `401` otherwise: the header is present and is a single compact JWS; the token's `alg` is in `algorithms` (the list is a closed asymmetric set — `none` and `HS*` cannot be configured, which closes algorithm confusion); its `kid` names a key in the configured JWKS (inline or fetched) whose type matches the pinned algorithm; the signature verifies; `iat` is present and inside `toleranceSeconds` (up to 60s of future skew allowed); and `bodyHashClaim` matches a SHA-256 of the delivered body bytes. `rejectionReason` on the delivery record distinguishes them: `sig_missing`, `sig_malformed`, `alg_not_allowed`, `kid_missing`, `kid_unknown`, `key_invalid`, `sig_invalid`, `sig_expired`, `body_hash_missing`, `body_mismatch`, `body_too_large`, `scheme_misconfigured`, `verifier_error` — plus `key_fetch_failed` and `key_fetch_throttled` when the keys come from a `jwksUrl`.

### `plaid`: the JWT preset with keys fetched from Plaid

Plaid signs deliveries the same way but publishes no JWKS — each key is fetched from Plaid's API by `kid`. `plaid` is that preset over the same mechanism, so the header (`Plaid-Verification`), the algorithm (`ES256`) and the body-hash claim (`request_body_sha256`, hex) are fixed in code and cannot be configured:

```toml
[webhook]
key = "plaid-events"
verificationScheme = "plaid"
toleranceSeconds = 300
deduplicationWindowMs = 360000     # same JWT-family bound as `jwt`

[verification.plaid]
environment = "sandbox"            # required — sandbox | production (an enum, never a URL)
clientId = "{{secrets.PLAID_CLIENT_ID}}"   # required — must be a {{secrets.KEY}} reference
secret = "{{secrets.PLAID_SECRET}}"        # required — must be a {{secrets.KEY}} reference
```

The key endpoint is derived from `environment` in code, so there is no way to point verification at another host: a `keyEndpoint`, `keyEndpointUrl`, `baseUrl` or `host` key in the table is rejected `400` (`PLAID_ENDPOINT_UNSUPPORTED`), as is an environment outside the enum or a literal credential (`CREDENTIAL_MUST_BE_SECRET_REF`). Those three keys are the whole table — any other key is rejected `400` (`PLAID_UNKNOWN_CONFIG_KEY`) rather than stored and ignored, so a setting that would never be checked cannot round-trip through `config pull`/`push` looking as though it took effect.

### `custom` + `[verification.custom.detachedSignature]`: describe the provider's own scheme

Never reach for `verificationScheme = "none"` because a provider signs differently from the eight built-ins — `none` accepts every request from anyone. Describe the provider's scheme instead. `custom` takes an optional `[verification.custom.detachedSignature]` table that declares how the signature is built; the platform verifies exactly as declared.

The Admin Console edits the same table as JSON on the webhook's page and reads back the parts it covers in order. It checks only that the signature covers the body exactly once and that each part names one source; every other rule below is answered by the server, with its code.

```toml
[webhook]
key = "acme-events"
verificationScheme = "custom"
signingSecret = "{{secrets.ACME_WEBHOOK_SECRET}}"
toleranceSeconds = 300

[verification.custom.detachedSignature]
primitive = "hmac-sha256"          # the only primitive available today
encoding = "base64url"             # hex | base64 | base64url
signature = { header = "X-Acme-Signature" }        # omit `key` when the whole header is the signature
# signature = { header = "X-Acme-Signature", key = "v1", prefix = "sha256=" }
freshness = { from = { header = "X-Acme-Timestamp" }, format = "unixSeconds" }
dedup = { signedBodyPointer = "/event/id" }        # optional — signature-covered id
externalEventId = { untrustedHeader = "X-Acme-Delivery" }  # optional — display only

[[verification.custom.detachedSignature.signedPayload]]
header = "X-Acme-Timestamp"
[[verification.custom.detachedSignature.signedPayload]]
literal = "|"
[[verification.custom.detachedSignature.signedPayload]]
body = true
```

That declares HMAC-SHA256 over `<X-Acme-Timestamp>|<raw body>`.

Rules, all enforced when you push rather than on the first live delivery — one function validates and compiles, so a configuration the write path accepted cannot fail structurally at delivery:

| Rule | Rejection |
|---|---|
| The parts cover the body exactly once: exactly one `{ body = true }` part, `body` is exactly `true`, and no part names `body` alongside another key | `400 SIGNATURE_BODY_PART_REQUIRED` |
| A `literal` separates any two variable-length parts — a `header` / `signatureField` part against `{ body = true }`, or two of them in a row. Without a separator the assembled bytes do not say where one part ends and the next begins, and a capture replays with bytes shifted across the boundary under the same signature. Every `literal` is a non-empty string, and a separating one must not overlap itself (no prefix of it is also a suffix): `\|`, `.`, `->` are fine, `::`, `..`, `abab` are not, because a self-overlapping separator matches across the boundary it marks and leaves the same two-way split. Any single character qualifies | `400 SIGNATURE_CONFIG_INVALID` |
| `freshness.from` must also be one of the `signedPayload` parts — otherwise the value checked for staleness sits outside the signature | `400 FRESHNESS_NOT_SIGNED` |
| `dedup.signedBodyPointer` is a bounded RFC 6901 pointer: ≤ 8 tokens, ≤ 64 chars each, ≤ 256 total, leading `/` | `400 SIGNATURE_POINTER_INVALID` |
| `primitive` must be `hmac-sha256`. `ed25519` and `ecdsa-p256` are named in the vocabulary but not available here — use `verificationScheme = "discord"` for Ed25519, `"jwt"` for a JWS | `400 SIGNATURE_PRIMITIVE_UNSUPPORTED` |
| Every configured header name is an HTTP field-name token; ≤ 8 parts and ≤ 1 KiB of `literal` text (a `header` or `signatureField` part is the sender's text, not yours, and is not counted); a `signatureField` part requires `signature.key`; `signature.key` and a `signatureField` name are ≤ 64 characters and carry no `,` or `=`, and `signature.prefix` is ≤ 64 characters and carries no comma when `signature.key` is set; a part carrying two variant keys where neither is `body`, and unknown keys at any level; the signature header cannot be a signed `{ header }` part and `signature.key` cannot be a `{ signatureField }` part | `400 SIGNATURE_CONFIG_INVALID` |

Those are the write-time parts of the boundary rule; one more is checked per delivery, because it is a property of the delivery rather than of the configuration. A separator only locates the boundary if it does not also occur inside the values beside it, so a delivery whose `header` or `signatureField` value contains a `literal` next to it is refused with `sig_malformed`. With a non-overlapping separator neither value contains, every boundary is the first (or last) occurrence of that separator and the signed bytes read one way. Spell the separator the provider actually uses and expect the provider's own values not to carry it — an `rfc3339` timestamp beside a `.` separator only works for a provider that sends whole seconds, since a fractional second carries a `.` of its own. Some pairings can never verify: `-`, `:`, `T` and `Z` occur in every `rfc3339` timestamp, and `,`, a space and `:` in every `httpDate` one. `primitive webhooks test` refuses to preview a configuration whose timestamp cannot avoid its separator, naming the clash, rather than emitting headers the receiver rejects; where a rendering of the format does avoid it, that is the rendering the preview signs.

`freshness.format` is `unixSeconds`, `unixMilliseconds`, `rfc3339` (offset required) or `httpDate` (IMF-fixdate only — the obsolete RFC 850 and asctime forms are `sig_malformed`). `httpDate` contains a comma, which is the `key=value` grammar's separator, so it can only be read from a header of its own, not a `signatureField`.

`freshness` itself is optional: a provider that signs no timestamp omits it, and that webhook's dedup entries then **never expire** — the `github` rule, for the same reason (nothing bounds a replay but a permanent entry).

`dedup.signedBodyPointer` resolves only after the signature verifies, and only to a non-empty string or a whole number; anything else falls back to `sha256:` of the signed payload, so no delivery is left keyless. `externalEventId.untrustedHeader` populates `meta.externalEventId` and nothing else — no configuration routes a header value into `dedupKey`, so a replay with that header edited is still answered `duplicate`.

Rejection reasons on the declarative path are the existing vocabulary: `sig_missing` (the signature header or a signed header part is absent), `sig_malformed` (the header does not match the declared grammar, an undecodable signature, or a freshness value that fails its format), `sig_expired`, `sig_invalid`, `scheme_misconfigured` (a stored configuration that does not validate — never a silent fall back to the built-in `custom` format). Nothing the sender chooses produces `scheme_misconfigured`: that reason names the operator's configuration, and a configuration the write path accepted has already been checked.

`primitive webhooks test <webhookId>` signs its preview from the declared configuration, so the headers it returns are ones a real delivery accepts. It answers `400 WEBHOOK_TEST_UNSUPPORTED_CONFIG`, naming the blocking part, when it cannot — reachable from a valid configuration by signing over a header an HTTP client may not set (`Host`, `Content-Length`, …); real deliveries from a provider that sets one itself still verify.

Without the table, `custom` is unchanged: `t=<unix>,v1=<hmac hex>` in `X-Webhook-Signature` over `<t>.<body>`, `X-Webhook-Event-Id` as the external id. Other free-form keys under `[verification.custom]` are untouched.

Both credential references must name app secrets that already exist — a create, an update that changes the config, or an update that switches the webhook onto `plaid` is rejected `400` rather than failing later as a `401`. The check is on the config actually changing (or on the scheme moving onto `plaid`), not on the field being sent: re-pushing an unchanged config on an unchanged scheme is accepted even after a referenced secret has been deleted, so one dead reference does not start failing every later `config push`. That webhook then fails closed at delivery instead, rejecting with a `401` (`secret_unresolved`).

Every signature check the `jwt` scheme makes applies unchanged. The `jwt` settings `plaid` does not carry are `issuer`, `audience` and `eventIdClaim` — none of them is checked on a plaid delivery, and none may be written into `[verification.plaid]` (see above), so a plaid delivery's `externalEventId` is always the body-hash claim and there is no issuer or audience pinning. On top of those checks come the key-resolution outcomes, which appear on the delivery record as their own `rejectionReason` values:

| `rejectionReason` | When |
|---|---|
| `secret_unresolved` | a credential reference no longer resolves — no request to Plaid is made |
| `kid_unknown` | Plaid answered that it does not know the token's `kid` (a 400 whose `error_code` names the key) |
| `key_expired` | Plaid returned the key with a non-null `expired_at` |
| `key_fetch_failed` | Plaid was unreachable, timed out, or answered unusably — including any other 400, such as `INVALID_API_KEYS` after a credential rotation |
| `key_fetch_throttled` | the webhook's key-fetch budget for this minute is spent (counted per worker instance, not globally) |

Key rotation needs no configuration change: a new `kid` is simply an uncached one, fetched once and then served from cache (10 minutes fresh, 5 more stale). Unknown and expired keys are remembered for a minute, and an unreachable Plaid for ten seconds, so repeated deliveries do not become one outbound request each. Only an answer Plaid gives *about that `kid`* counts as a verdict on the key: a response the platform cannot recognise, or an error about the request or the credentials, is `key_fetch_failed` and leaves the previously-resolved copy in place. The fetch budget is charged only for `kid`s that have never resolved on that webhook, and a fetch that returns a key is refunded, so a flood of invented `kid`s cannot stop deliveries signed with the keys the webhook actually uses; when the budget is spent, a previously-resolved key is served for up to an hour rather than the delivery rejected. The one case the budget does not cover is a `kid` that is *both* brand-new and arriving while a flood has the budget spent — indistinguishable from an invented one before the fetch — so the first delivery on a freshly rotated key can be delayed by a sustained flood; the key already in use keeps verifying throughout.

`externalEventId` for a `jwt` delivery is the body-hash claim unless `eventIdClaim` names another claim — and a configured `eventIdClaim` the token doesn't carry falls back to the body hash rather than leaving the delivery without a dedup key. So a captured delivery redelivered intact lands as `duplicate` inside the dedup window rather than firing a second run.

That is why, for every scheme except `none`, `deduplicationEnabled` must stay `true`, `toleranceSeconds` must be greater than `0`, and `deduplicationWindowMs` must cover the whole window in which a captured delivery still verifies: `toleranceSeconds * 1000 + 60000` for the JWT family (`jwt` and `plaid`, whose future-skew allowance it is) and `toleranceSeconds * 2000` for the HMAC schemes (whose check is `|now - timestamp| <= tolerance`). A write that violates any of them is rejected `400` (`DEDUP_REQUIRED_FOR_SIGNED_SCHEME`, `DEDUP_WINDOW_TOO_SHORT`, `REPLAY_VALUE_OUT_OF_RANGE`). `deduplicationEnabled` is strictly typed on every scheme, `none` included: a non-boolean such as `0` or `"false"` is rejected (`REPLAY_VALUE_OUT_OF_RANGE`) rather than read as `false` or `true`. The rules are checked only when a write actually changes one of those values, so a webhook created before they existed keeps taking unrelated edits — but the next change to its replay settings has to leave a valid record behind. Those are write-time rules; at delivery time a signed webhook deduplicates whatever its record stores, and over at least the window above, so a record written before the rules existed still suppresses replays.

On a signed scheme the key a delivery is deduplicated on comes only from material the signature covers. Where the provider signs an identifier of its own, that is the key: `stripe` uses the body's `id`, `slack` its `event_id`, `discord` the interaction `id`, and `jwt`/`plaid` the `eventIdClaim`. Where it doesn't, the key is a SHA-256 of the exact payload that was signed — the body for `github` and for `jwt`/`plaid`, `<t>.<body>` for `custom` — and a signed payload carrying no id falls back to the same hash.

Every key carries a tag naming which of those it is: `sha256:<hex>` for a digest the platform computed over the signed payload, `id:<value>` for an identifier the delivery itself carried (including the `X-Webhook-Event-Id` header on `none`), and `cap:<hex>` for the lossless rehash of a key too long to index. The tags keep the two kinds of value from naming each other: without them a sender able to sign for the webhook could set an event id to the exact `sha256:` key a later, id-less delivery would produce, claim that key first, and have the legitimate delivery answered `duplicate` and never dispatched. An id that itself starts with a tag is nested inside `id:` rather than escaping it, and no ordinary derivation can produce a `cap:` key. `github` and `custom` keys are unchanged by the tagging; the `jwt`/`plaid` default key is now the canonical `sha256:` hex regardless of the `bodyHashEncoding` the provider uses. Editing or dropping `X-GitHub-Delivery` or `X-Webhook-Event-Id` on a captured delivery therefore changes nothing: the replay is answered `200 {"received": true, "duplicate": true}` and no second run fires. Those two headers still travel to the delivery log and to the workflow's `meta.externalEventId` as the provider's own id, for display and correlation — never as a security decision. The delivery log carries the derived key as `dedupKey` on accepted deliveries only — the `duplicate` row itself carries none, because the `duplicate` status is already the answer to "was this suppressed?"; the `dedupKey` on the accepted row is what tells you *which* earlier delivery it collapsed into. `primitive webhooks events <webhook-key> --json` carries it too. The reverse join is not available: a `duplicate` row shows `dedupKey` empty, and on `github`/`custom`/`none` its `externalEventId` comes from a header the signature does not cover, so a replay can have changed it. Correlate a suppressed delivery to its original by payload summary and timestamp instead.

One `custom` behavior changes with this: the key is now `sha256:<hex>` of `<t>.<body>`, so the signed timestamp is part of it. A sender that retries the same logical event by **re-signing it with a fresh `t`** — the only retry `toleranceSeconds` lets through, and the normal one for a Stripe-style scheme — now produces a different key and fires the workflow a second time, where it used to be suppressed by the shared `X-Webhook-Event-Id`. That is deliberate (a header the signature does not cover cannot decide anything), but if your `custom` sender retries that way, make the workflow idempotent or move the event id into the signed body.

`github` is the one built-in scheme whose dedup entries **never expire** (a declarative `custom` configuration with no `freshness` is the other case, for the same reason). GitHub signs no timestamp, so a captured delivery stays acceptable forever and no finite `deduplicationWindowMs` bounds it — the receiver ignores the window on that scheme. The consequence to plan for: GitHub's manual **Redeliver** button re-sends a byte-identical body (and reuses the same delivery GUID), so from the second delivery onward it is answered `duplicate` and does not fire the workflow. Re-run the workflow directly (`primitive workflows run`). Rotating the signing secret does **not** clear it: the key is a hash of the body with no key material in it, so GitHub re-signs the identical body and it is still a duplicate. Recreating the webhook is the only thing that starts a fresh dedup history.

On `none` nothing changes: it has no signature to derive anything from, so it still keys off `X-Webhook-Event-Id` and still deduplicates only when `deduplicationEnabled` is `true`. The one exception is shared with every scheme: a delivery whose target workflow does not exist yet records no key at all, so it is not deduplicated — the same reasoning as `workflow_inactive` below, since suppressing it would drop the redelivery you send after creating the workflow. One rejection reason exists for the case that should never happen: a verified delivery on a signed scheme that somehow carries no dedup key is rejected `401` with `rejectionReason: dedup_key_unavailable` rather than dispatched with suppression silently off.

One delivery outcome is a `503`: `Failed to record webhook delivery`, when the platform verified the delivery but could not write the `accepted` row that carries its dedup key. The workflow is **not** started in that case. That row is the only durable record that the delivery ran, so dispatching without it would leave the delivery replayable — for the whole window, and permanently on `github` — and a `503` asks the sender to retry, which is the resolution: the retry re-runs the whole path and either records the delivery and dispatches it, or fails the same way. Size your provider's retry policy to cover it. Unlike the `202` for `workflow_inactive`, this one *wants* the retry. It applies only where a dedup key was actually due: a delivery whose target workflow doesn't exist yet, and a `none` webhook with `deduplicationEnabled: false`, keep their previous best-effort logging and still dispatch.

Any credential inside a `[verification.*]` table must be a `{{secrets.KEY}}` reference, and the reference must be the WHOLE value — the table is stored in cleartext, so a literal would be readable, and so would the literal half of a mixed value like `"sk_live_abcd{{secrets.SUFFIX}}"`. A write carrying either is rejected `400` (`CREDENTIAL_MUST_BE_SECRET_REF`). A `{{secrets.KEY}}` reference is returned verbatim on every read path, so a compliant table survives a `pull` → `push` round trip unchanged. A literal written before this rule existed is **not**: every read path (`GET`, the admin API, and therefore `config pull`) returns it as `[redacted]`. Pushing a file that still holds `[redacted]` is rejected `400` (`CREDENTIAL_MUST_BE_SECRET_REF`) — including on an edit that changed something else entirely — until the real value is moved into the app secret store and the table references it.

The separate rule that a referenced key must EXIST (`400` `MISSING_CONFIG_SECRET_REF`) applies only to the paths the ACTIVE scheme resolves at delivery — today `[verification.plaid]`'s `clientId` and `secret`. A table for some other scheme is inert, so a `stripe` webhook that still carries a `[verification.plaid]` section naming deleted keys stays editable.

Bodies are verified over the raw bytes, so a payload that is not valid UTF-8 verifies correctly on every scheme. A body over the webhook's size cap is rejected `401` (`body_too_large`) before it is read, on every scheme including `none`. The cap defaults to 5 MiB and is set per webhook with `maxBodyBytes` (platform maximum 25 MiB); a value above the maximum, or a non-positive one, is rejected `400` at write time on both the tenant and admin APIs, and an unset value uses the 5 MiB default.

The `[verification.<scheme>]` table owns the stored config once a file declares a `[verification]` section: removing the scheme's table (leaving a bare `[verification]`) clears it on the next push, which is how a compromised key is revoked through `sync`. A file with no `[verification]` section at all leaves the stored config untouched.

`signingSecret` is required for `stripe`, `github`, `slack` and `custom`, and must be a **whole** `{{secrets.KEY}}` reference into the app secret store — nothing else is accepted, on either the tenant or the admin API, on create, update and `rotate-secret`:

| Write | Result |
|---|---|
| a raw value, or a mixed value like `"sk_live_abcd{{secrets.SUFFIX}}"` | `400` `SIGNING_SECRET_MUST_BE_SECRET_REF`, carrying the `primitive secrets set KEY --value <value>` remediation |
| a reference to a key that does not exist | `400` naming the missing key |
| any non-null value on `none` / `discord` / `jwt` / `plaid` | `400` `SIGNING_SECRET_NOT_SUPPORTED_FOR_SCHEME` |
| `signingSecret: null` on a scheme that requires one | `400` — the field is never silently blanked |
| a whole reference to an existing key | accepted, stored trimmed and canonical (`{{secrets.KEY}}`) |

The Admin Console edits the same field as a picker over the app's secrets, with an inline row that creates one and writes the reference; it is shown only for the schemes that take a signing secret.

The reference is stored and returned verbatim (it carries no secret material), so a `pull` → `push` round trip is unchanged. The gate is on the value the record ends up with and fires only when it CHANGES, so re-pushing an unchanged reference whose secret was later deleted is accepted rather than aborting the push.

Switching a webhook onto a scheme that uses no signing secret **clears** `signingSecret`, `previousSigningSecret` and the rotation timestamp; switching back means supplying a reference again (a bare switch back is rejected naming `signingSecret`).

Every read path reports a derived `signingSecretStatus` — and, since the same classification applies to the grace slot, a `previousSigningSecretStatus` with the identical four values:

| value | meaning |
|---|---|
| `reference` | `signingSecret` holds a whole reference and is returned verbatim |
| `legacy-literal` | the stored value is a raw secret, from before this field became reference-only: it is never returned and `config pull` writes no `signingSecret` line, but deliveries still verify with it. Writes are what is blocked — any update to the webhook is rejected `400` `SIGNING_SECRET_MIGRATION_REQUIRED` unless that same update supplies a `{{secrets.KEY}}` reference. (Switching to a scheme with no signing secret also clears the literal, but only if the webhook no longer needs to verify senders — `verificationScheme: "none"` accepts every request, signed or not.) `rotate-secret` is blocked the same way — rotating would move the raw value into the grace slot, where it would keep verifying while the webhook reported itself as migrated — so migrating a literal is a hard cutover with no overlap window |
| `malformed-reference` | the stored value carries reference syntax that no `{{secrets.KEY}}` reference accounts for (`{{secrets.foo}}`, `{secrets.KEY}`, `{{vars.KEY}}`, `whsec_a{{b`), **or** an otherwise-valid reference carrying an invisible character (a zero-width space, a byte-order mark). **Not** a working legacy literal: reference text is public — it round-trips into version-controlled TOML — so it is never used as a signing key, and every signed delivery is already rejected `401` (`rejectionReason: secret_unresolved`, `cause: malformed-reference`). Writes are blocked the same way as `legacy-literal`; the remediation is to store the real signing secret and point the field at it |
| `unset` | no signing secret. Normal for `none` / `discord` / `jwt` / `plaid`; on a signing scheme it is an unusable webhook — every signed delivery is rejected `401` (`rejectionReason: signing_secret_unset`), because an empty HMAC key is as forgeable as none at all |

A second, orthogonal axis reports what is *wrong with* a stored value, as `signingSecretHealth` / `previousSigningSecretHealth`:

| value | meaning |
|---|---|
| `ok` | nothing known to be wrong. Always the value for a `reference`, a `malformed-reference` (which is never used as a key) and `unset` |
| `weak-literal` | the stored value is a grandfathered literal shorter than 16 characters — too short to be a strong HMAC key. Advisory only: deliveries verify exactly as before, and the value, its length and any prefix of it are never disclosed |

Branch on "is it `ok`?", not on the list of members: this axis is deliberately open and more members may be added, so an unrecognized value should read as "needs attention".

Resolution **fails closed**: if the reference can't be resolved at delivery (secret deleted, or the platform's secret encryption key unset) the request is rejected `401` (`rejectionReason: secret_unresolved`) rather than verifying against the literal reference text; an unresolvable grace-window `previousSigningSecret` reference is dropped rather than passed to the verifier. A `legacy-literal` value is not a reference and does not fail closed — it is the key itself. A `malformed-reference` value is not a key either, and does fail closed.

The migration gate is skipped where there is nothing to force off: an **archived** webhook, and a write whose only change is a transition to `paused` or `archived`. So you can always pause or archive a webhook whose signing secret has not been migrated.

**Rotation is key rotation.** A key holds exactly one value, so `primitive secrets set KEY --value <new>` is a sharp cutover. To accept both secrets during a provider's overlap window, create a second key and rotate onto it:

```bash
primitive secrets set STRIPE_WEBHOOK_SECRET_2 --value whsec_new...
primitive webhooks rotate-secret <webhookId> --secret "{{secrets.STRIPE_WEBHOOK_SECRET_2}}"
```

`rotate-secret` requires a scheme that uses a signing secret (`400` otherwise), takes a whole reference to an existing key, moves the previous reference into `previousSigningSecret`, sets the rotation timestamp, and returns the serialized webhook. Rotating onto the key already in use is rejected — it would provide no overlap. It is also rejected `400` `SIGNING_SECRET_MIGRATION_REQUIRED` while the current slot holds a raw value (`legacy-literal` or `malformed-reference`), so nothing withheld ever reaches the grace slot.

The previous reference keeps verifying for `secretGracePeriodMs` (24 hours by default). That window is bounded at 30 days: a larger value is rejected `400` `SECRET_GRACE_PERIOD_OUT_OF_RANGE` on every write surface, and a webhook that already stores a larger one has it capped at the maximum when a delivery is checked — reads report the capped value, so what you see is what is enforced. `0` cuts over the moment the rotation lands. Re-sending an already-stored over-bound value unchanged is accepted, so `config push` on a webhook configured before the bound is not broken by it.

Each referenced credential consumes one of the app's 100 secret slots, so an app with many signed webhooks needs one key per webhook.

Receive endpoint: `POST /app/{appId}/webhook/{webhookKey}`. The platform verifies the signature per `verificationScheme`, then starts `workflowKey` with the event payload as input; `inputMapping` (e.g. `"data.object"`) extracts a nested path first. A webhook-triggered workflow is `runAs: "system"`, so what stops a client from starting it directly with a crafted payload is the system-invocation gate (members get a 403) plus the signature verification — not `accessRule`, which a system workflow doesn't evaluate on the trigger (see [Access control](#access-control)).

CLI: `primitive webhooks list | get | disable | enable | archive | rotate-secret | test | events <webhook-key>` — `events` lists recent deliveries (accepted / rejected / duplicate / `workflow_inactive`); `--json` adds each delivery's `dedupKey`. **Availability is server-owned (#2803)**: `status` is not a TOML key and is refused on every create/update body (admin and tenant alike), a new or pushed webhook is always `active`, and `disable`/`enable` are its only writers. `archived` is written by the delete flow, whose CLI spelling is `primitive webhooks archive <id>` (#2907) — a soft delete that keeps the row, its `webhookKey` and its capped slot, refuses deliveries, and cannot be undone. `list --status` filters by `active`, `inactive` (which also matches a legacy stored `paused`), or `archived`.

An app holds at most **50 webhooks**, counting archived ones — the count is what bounds every per-webhook budget (body cap, delivery rows, remote-JWKS key fetches) for the app as a whole. A create past it is rejected `400` `WEBHOOK_LIMIT_REACHED` on both the tenant and admin APIs (never a `409`, so `config push` cannot mistake it for a key conflict), and `primitive config push` checks up front how many webhooks the `webhooks/` directory would create and refuses, before writing anything, a push that would leave the app past the limit. Only creates count against it: an app at (or already over) the limit can still push its existing config — webhooks, workflows, databases and vars alike. Apps already over the limit keep working; only new creates are refused. An API delete, a console delete and `primitive webhooks archive` all archive, so none of them frees a slot. `primitive config push --prune` does: it is the one prune path that hard-deletes instead of archiving, so a webhook whose TOML you deleted is removed permanently, freeing its slot and releasing its `key` for reuse — deliberate (an archiving prune would burn a slot per cycle with no way to reclaim it from `sync`) but irreversible, and unlike every other pruned resource type. A hard delete does not delete the webhook's delivery rows, but its id stops resolving, so `primitive webhooks events <id>` on it answers "not found" — read the history first. The tenant `GET /app/{appId}/api/webhooks` returns active webhooks only; use `primitive webhooks list --status archived` (or the admin list, which applies no status filter) to see the archived rows that are still holding slots.

`webhooks test <webhook-key> --payload '<json>'` **signs** that JSON object exactly as given and returns the signed request body plus the signature headers — it does **not** post them to the receive endpoint, so it dispatches no workflow and writes no delivery row. Pass the event directly (e.g. `'{"type":"charge.succeeded","data":{"id":"ch_1"}}'`), not wrapped in `{"payload": ...}`. Omitting `--payload` (or passing a non-object) signs a canned `{"type":"webhook.test", ...}` ping instead. To exercise the real path, send the returned body with the returned headers to `POST /app/{appId}/webhook/{webhookKey}` yourself. That delivery is where the `github` consequence bites: the key is `sha256(body)` and `github` entries never expire, so replaying the same signed body is answered `duplicate` the second time and does not dispatch — permanently, with recreating the webhook as the only reset. Vary the payload between real test deliveries (any byte change is a different key). The canned ping carries a fresh timestamp, so it is safe to repeat.

`webhooks test` returns `200` only when the headers it hands back are ones the receiver would accept; every other case is a `400` carrying a `code`, on which the CLI prints the message with its code and exits non-zero. `WEBHOOK_TEST_SIGNING_UNSUPPORTED` means the scheme is verified with public key material only (`discord`, `jwt`, `plaid`) so no preview signature can exist — use `primitive webhooks verify` on a captured delivery instead. `WEBHOOK_TEST_SIGNING_SECRET_WITHHELD` means the `signingSecret` is still a raw stored value: deliveries keep verifying with it, but the row is unmigrated, so store the value with `primitive secrets set` and point the field at a `{{secrets.KEY}}` reference. `SIGNING_SECRET_MUST_BE_SECRET_REF` and `MISSING_CONFIG_SECRET_REF` mean the secret is unset, blank, malformed, or its app secret is gone — deliveries there are already rejected. `verificationScheme = "none"` is the one unsigned `200`, because a `none` delivery genuinely carries no signature — a successful preview, printed with its signed body and headers and exit `0`.

`webhooks verify <webhook-id> --header 'Name: value' [--header …] (--body '<text>' | --body-file <path>)` runs the webhook's configured verifier over a delivery you captured and reports the verdict, the scheme, and the rejection reason — the same vocabulary `webhooks events` shows. Exit `0` when the signature verifies, non-zero otherwise. `--body-file` is read as raw bytes and sent base64-encoded when they are not valid UTF-8, so the verifier sees exactly the captured bytes. Nothing is delivered, and the check is deliberately absent from `webhooks events`, which stays the record of real deliveries. It answers SIGNATURE verification only: status and IP allowlist are checked before verification and handshake rules and deduplication after it, so `verified: true` does not promise the delivery would have dispatched. `none` is refused rather than answered — it checks no signature. Over the API: `POST /admin/api/apps/{appId}/webhooks/{webhookId}/verify` with `{ headers, rawBody }` or `{ headers, bodyBase64 }` (exactly one body form; `headers` as a flat object or ordered `[name, value]` pairs). A bad signature is a `200` verdict; a `400` (`WEBHOOK_VERIFY_BODY_INVALID`, `WEBHOOK_VERIFY_HEADERS_INVALID`, `WEBHOOK_VERIFY_SCHEME_UNSUPPORTED`) means the request itself was malformed.

A `workflow_inactive` delivery means the bound workflow was not dispatchable when the event arrived: the request is acked with HTTP 202 `{ received: true }` (so the sender doesn't retry) but the workflow is **not dispatched**. The event's `rejectionReason` names which case it was — `WORKFLOW_INACTIVE` for a disabled workflow (run `primitive workflows enable <key>` and resend) or `WORKFLOW_ARCHIVED` for a retired one (re-point the webhook, or hard-delete the archived holder and push the workflow's file again). These events are excluded from deduplication, so a resend isn't dropped as a duplicate. Binding a webhook to either succeeds and returns a non-blocking `warning` carrying the matching code.

## Cron triggers

A cron trigger fires a workflow on a clock schedule. It is one of the ways to invoke a workflow — a clock instead of a `workflows.start()` call or an inbound webhook. The trigger points at a workflow by `workflowKey`; the trigger itself runs no code.

**Decision rule:** trigger is a clock → cron trigger. Trigger is a user action or external webhook → `workflows.start()` or an inbound webhook, NOT cron.

### Critical rules

1. **Cron triggers fire workflows, not arbitrary code.** Create the workflow first — push it, and confirm it is in service and has an active config/revision — then point the trigger at it via `workflowKey`. Availability is server-owned: `primitive workflows enable|disable` is what changes it, and a pushed workflow starts in service. Retirement is `primitive workflows archive <id>` / `primitive cron-triggers archive <id>` — the delete flow's soft path, which keeps the row and its key (and the trigger's capped slot) and cannot be undone.
2. **Set `requiresClientApply = false` on the target workflow.** Cron-triggered workflows almost always want this — otherwise the run sits in `apply_pending` forever because no client is listening.
3. **Set an IANA `timezone` whenever the schedule has a user-visible hour.** `0 9 * * *` in UTC is 2am in Los Angeles.
4. **`overlapPolicy` is `"skip"` (default) or `"allow"`.** There is no `"queue"`. `"skip"` checks whether the previous run is still active and increments `skippedCount`; `"allow"` always fires. Use `"allow"` only when each firing is independent and idempotent.
5. **Per-app cap is 50 cron triggers**, archived ones included: `primitive cron-triggers archive` retires a trigger and cancels its schedule but frees neither its slot nor its `triggerKey`. Only a hard delete does — remove the file and run a confirmed `primitive config push --prune`.

### Creating (TOML / `primitive config`)

```toml
# config/cron-triggers/nightly-digest.toml
[cronTrigger]
key = "nightly-digest"
displayName = "Nightly digest email"
cron = "0 9 * * *"
timezone = "America/Los_Angeles"
workflowKey = "send-digest"
overlapPolicy = "skip"
# Availability is server-owned and not authored here: a pushed trigger is in
# service, `primitive cron-triggers enable|disable` is what changes it, and
# `primitive cron-triggers archive` retires it (soft delete; keeps key and slot).

# Optional: static input passed to the workflow on every fire
[rootInput]
digestType = "daily"
environment = "production"

# Optional: dynamic input. `{{now}}` is replaced with the fire-time ISO string.
[inputMapping]
firedAt = "{{now}}"
```

```bash
primitive config push
```

The TOML key `key` maps to the API field `triggerKey`. The field name is `cron` (not `schedule`).

### Creating (client / CLI)

{{ example: scheduling/cron-create }}

From the CLI, a trigger is TOML plus a push — `config create` scaffolds the file from the type's defaults:

```bash
primitive config fields cron-trigger                 # key, displayName, cron, timezone, workflowKey, overlapPolicy, state
primitive config create cron-trigger nightly-digest  # writes cron-triggers/nightly-digest.toml
primitive config push --only cron-trigger/nightly-digest
```

`[rootInput]` in that file is a fixed payload sent on every firing; `[inputMapping]` projects the firing context (`$triggerId`, `$firedAt`) onto the workflow input.

The TOML keys are `key`, `displayName`, `cron`, `timezone`, `workflowKey`, `overlapPolicy` and `state` under `[cronTrigger]`, plus the `[rootInput]` and `[inputMapping]` tables — the field is `cron`, not `schedule`, and `workflowKey`, not `workflow`.

#### Wrong

{{#lang ts}}
```typescript
// WRONG — these field names don't exist
await client.cronTriggers.create({
  key: "nightly-digest",         // should be triggerKey
  schedule: "0 9 * * *",          // should be cron
  input: { ... },                 // should be rootInput
  overlapPolicy: "queue",         // not a valid value
  enabled: true,                  // no such field; use `state`
});
```
{{/lang}}

### Field reference

| Field | Required | Notes |
|-------|----------|-------|
| `triggerKey` | Yes | Per-app unique. Alphanumerics, `-`, `_`. Must start alphanumeric. |
| `displayName` | Yes | Human label. |
| `cron` | Yes | 5-field cron (see Syntax below). |
| `workflowKey` | Yes | Must refer to an existing workflow definition. |
| `timezone` | No | IANA name (validated via `Intl.DateTimeFormat`). Default `"UTC"`. |
| `overlapPolicy` | No | `"skip"` (default) or `"allow"`. |
| `rootInput` | No | JSON object, merged into workflow input. |
| `inputMapping` | No | JSON object, merged AFTER `rootInput`. Supports `{{now}}` substitution. |
| `description` | No | Free text. |
| `status` | Read only | `"active"` / `"inactive"` / `"archived"`. Server-owned: create always produces `"active"`, `disable`/`enable` are the only writers, and delete writes `"archived"`. It is not settable through `update`. |

### Cron expression syntax

Standard 5-field POSIX: `minute hour day-of-month month day-of-week`.

Supported per field: `*`, exact (`5`), range (`5-10`), step on wildcard (`*/5`), step on range (`9-17/2`), comma list (`1,2,3`).

**Not supported:** month/day names, `?`, `L`, `W`, `#`, last-day modifiers, 6/7-field crons.

**Day-of-week:** `0` and `7` both mean Sunday, but `7` is only allowed as a bare value (NOT in ranges). Use `0` in ranges.

**Vixie semantics:** when both day-of-month and day-of-week are restricted, fires when EITHER matches.

| Need | Schedule |
|------|----------|
| Every 5 minutes | `*/5 * * * *` |
| Every hour on the hour | `0 * * * *` |
| Every day at 9am (local) | `0 9 * * *` + `timezone` |
| Every Monday at 9am | `0 9 * * 1` |
| First of every month | `0 0 1 * *` |
| Every 15 min, business hours, Mon–Fri | `*/15 9-17 * * 1-5` |

Invalid expressions are rejected at create time. If the cron later becomes unparseable (rare), the trigger goes to `status: "inactive"` with `lastError` set; alarms stop until you call `enable`.

### Workflow input shape

The workflow receives `rootInput` merged with `inputMapping` (latter wins on key collision), plus the `meta` object on the run record:

```typescript
// Inside the workflow:
input.digestType    // "daily"          (from rootInput)
input.firedAt       // "2026-04-27T..." (from inputMapping with {{now}})

// And on the run record:
run.contextDocId    // "cron:<triggerId>"
run.meta.source     // "cron"
run.meta.triggerId  // <triggerId>
run.meta.triggerKey // "nightly-digest"
run.meta.manual     // true if started via cronTriggers.test()
```

### Lifecycle methods

{{ example: scheduling/cron-lifecycle }}

`.test()`, `.pause()`, `.resume()`, `.delete()`, `.update()`, `.get()` all take the `triggerId` (ULID returned from `.create()`), NOT the `triggerKey`. Use `.list()` to look up `triggerId` by key. The same operations are available on the CLI:

```bash
primitive cron-triggers list
primitive cron-triggers get <trigger-id>          # includes runtime.scheduledAlarmAt
primitive cron-triggers test <trigger-id>         # fire now; does NOT affect schedule
primitive cron-triggers disable <trigger-id>
primitive cron-triggers enable <trigger-id>
```

Deleting a trigger from the CLI is deleting `cron-triggers/<key>.toml` and running `primitive config push --prune`. Editing one is editing that file. `pause`/`resume` are operational rather than configuration: they write a runtime field that is deliberately not a TOML key, so a later push cannot resume a trigger you paused, and `config diff` reports it as "operationally disabled; configuration unchanged" rather than as drift.

### Querying cron-triggered runs

There is no `triggerSource` filter on `workflows.listRuns()`. Cron runs are identifiable by their `contextDocId` starting with `cron:` and `meta.source === "cron"`:

{{ example: scheduling/cron-list-runs }}

### Managing triggers (client)

{{ example: scheduling/cron-manage }}

### Debugging cron triggers

Trigger status:

- `active` — scheduled, alarm armed.
- `inactive` — out of service; the alarm is cancelled until `enable`. Either an operator ran `disable`, or the platform stopped it automatically — when a fire fails hard (the target workflow is **not found**, or a binding/runtime error), when the cron expression becomes unparseable, or after 50 consecutive skips against a target with **no active configuration**. In the automatic cases `lastError` says which (carrying `WORKFLOW_NOT_FOUND` or `WORKFLOW_NO_ACTIVE_CONFIG`); `enable` clears it and reschedules.
- `archived` — soft-deleted; never returns from list. `enable` refuses it: re-create the trigger by pushing its file again.

A target workflow that is **inactive** or **archived** does NOT take the trigger out of service: the fire is skipped and rescheduled indefinitely, and `lastError` is set to `WORKFLOW_INACTIVE` or `WORKFLOW_ARCHIVED`. Both are deliberate operator acts, so the trigger waits rather than punishing the operator with a second thing to undo; an inactive target auto-recovers once `primitive workflows enable <key>` runs, and an archived one waits for the binding to be re-pointed. A target that is active but has **no resolvable configuration** still advances `consecutiveNotActiveCount` and stops the trigger after 50 consecutive skips, and one that is **not found** stops it immediately.

`skippedCount` increments when `overlapPolicy: "skip"` blocks a fire because the prior run is still active — distinct from `consecutiveNotActiveCount`.

### Driving subscriptions from cron

A cron-spawned workflow that writes to a database wakes up every matching database subscription — cron writes → subscription broadcasts → UI renders, with no cron-awareness in the subscriber. The subscription side lives with databases; see the [Databases agent guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#real-time-subscriptions). Cron/workflow writes arrive on the subscription with `originConnectionId: null` / `originUserId: null` and both `isOrigin` flags `false`.

## Client apply (footgun)

By default, `requiresClientApply = true`. After the workflow completes, status becomes `apply_pending` and a connected client must call `claimApply` → run `onApply` → `confirmApply` to finalize. If no client is listening, the run sits in `apply_pending` indefinitely.

For server-only workflows (no client follow-up), set `requiresClientApply = false` in the workflow TOML:

```toml
[workflow]
key = "nightly-digest"
requiresClientApply = false
```

`primitive config push` applies this flag on create and on update alike:

```bash
primitive config set workflow/nightly-digest workflow.requiresClientApply=false
primitive config push --only workflow/nightly-digest
```

## Synchronous invocation

Opt a workflow into synchronous invocation by setting `syncCallable = true` in the TOML and pushing it — on a new workflow and an existing one alike. Once set, a caller can invoke the workflow and receive the final envelope in a single round-trip — useful for short, latency-sensitive tasks like input validation, enrichment lookups, or webhook handlers that must reply with a result.

Server-side constraints on a `syncCallable` workflow:

- **Step kinds are restricted.** The server validates step kinds against a sync-compatible list when the flag is set (or when steps are pushed against a sync-callable workflow). Long-running or suspending kinds (`event.wait`, `delay` over the timeout) reject at save time with `Workflow contains sync-incompatible steps`.
- **Run-scoped `lock:` is honored.** A `syncCallable` workflow may declare a run-scoped `lock:`: the acquire-around-run lifecycle now runs under run-sync and through `workflow.call`, not only on the durable path. All paths share one app-scoped lock namespace, so a durable `start()` run and a `run-sync` run of the same key serialize against each other. A nested `workflow.call` that declares a key an ancestor already holds re-enters it (runs without re-acquiring); concurrent same-run `workflow.call` siblings declaring the same fresh key are NOT mutually excluded (a run does not serialize against itself). The imperative `lock.*` steps are not an escape hatch for this: a `workflow.call` child inherits its parent's run identity, and every workflow-held lock is owned by that identity, so two `lock.acquire` steps on one key inside a single run both return `acquired: true` with the same handle. To serialize concurrent branches, set the `forEach` step's `concurrency = 1`; to let them run concurrently without contending, give each branch its own key.
- **Timeout.** The invocation timeout defaults to 5s and is capped server-side at 30s; anything above the ceiling clamps silently. Exceeding it resolves with `status: "timeout"` in the envelope. The timeout ends the **call**, not the execution: work is not interrupted at the boundary, so side effects from steps still in flight (database writes, emails) can land after the envelope resolves. The run record is finalized `terminated` before the response returns — the envelope's `run.status` reads `terminated`, and the record never transitions to `completed`. `getStatus` targets durable `start()` runs; a sync run's final state comes back in the envelope's `run` and appears in `workflows.listRuns()`. Step-run records are salvaged only when execution settles within a short grace window at the boundary — a timed-out run typically persists no step runs. Recovery: never poll for completion after a timeout; treat the run as incomplete, reconcile against actual resource state with idempotent cleanup or retry, and re-invoke with a **new** `runKey` — the sync path has no `forceRerun`, so the same `runKey` returns the terminated run without re-executing. When the final outcome must be knowable, prefer `start()`: a durable run's record reaches a real terminal status pollable via `getStatus`.
- **Apply still applies.** A sync-callable workflow may still have `requiresClientApply = true`, in which case the synchronous call resolves with `status: "apply_pending"` and the normal `claimApply`/`confirmApply` flow takes over. Most sync-callable workflows want `requiresClientApply = false`.

Long-running workflows should keep using `start()` plus the WebSocket / polling lifecycle.

Call `workflows.runSync` and await the final envelope:

{{ example: workflows/workflow-run-sync }}

`runSync` also accepts `runKey`, `contextDocId`, and `meta` — the same idempotency and scoping fields as `start`.

{{#lang ts}}
An optional `signal` (`AbortSignal`) is accepted as well.

Both invocation methods are generic: `start<I>(options)` types the `input`, and `runSync<I, O>(options)` additionally types the result envelope's `output` as `O` (`RunSyncWorkflowResult<O>`). `getStatus<O>(options)` and `terminate<O>(options)` are generic the same way, typing `WorkflowStatusResult<O>`'s `output` — useful for a terminated run that still carries partial output. Defaults (`Record<string, any>` / `any`) preserve untyped call sites. Rather than hand-writing `I`/`O`, generate them from the workflow's `inputSchema`/`outputSchema` with `primitive workflows codegen` — see [Typed invocation (codegen)](#typed-invocation-codegen).
{{/lang}}

{{#lang swift}}
Both invocation methods are generic: `start<Input>(...)` types the `input`, and `runSync<Input, Output>(...)` additionally types the result envelope's `output` as `Output` (`RunSyncResult<Output>`). `getStatus<Output>(...)` is generic the same way, typing `WorkflowStatus<Output>`'s `output`. Each is `async throws`. Rather than hand-writing `Input`/`Output`, generate them from the workflow's `inputSchema`/`outputSchema` with `primitive workflows codegen --lang swift` — see [Typed invocation (codegen)](#typed-invocation-codegen).
{{/lang}}

## Workflow lifecycle

A workflow needs to be `active` AND have one of (active configuration | published revision) before clients can run it. Availability is one server-owned `status` — `active | inactive | archived` — and is **not** a TOML key: every created or pushed workflow is active, and `primitive workflows disable <key>` / `enable <key>` (or the console's action) plus the delete flow are its only writers. A file that still carries a `[workflow] status` line fails `config push` with a message naming the verb.

| Status | Active config/revision? | Client can run? | CLI preview? |
|---|---|---|---|
| `active` | yes (required) | yes | yes |
| `inactive` | either | no (`409 WORKFLOW_INACTIVE`) | yes — preview is a diagnostic |
| `archived` | either | no (`409 WORKFLOW_ARCHIVED`) | no — it has been deleted |

`primitive workflows enable` on a workflow with no active config or revision returns: `Cannot activate workflow without a configuration`. A row stored before this model may still carry `draft`, which reads `inactive`.

Deleting is two acts. A plain admin `DELETE` (and the console's **Archive**) **retires** the workflow: `status` becomes `archived`, and nothing else changes — its configurations, revisions, runs, step runs and test cases all stay, so the run history keeps resolving what it points at. An archived workflow is refused by every dispatch path, cannot be brought back with `enable`, and goes on holding its `workflowKey`, so re-creating under that key fails with `workflowKey already exists (held by an archived workflow <id>)`. `DELETE ?hard=true` (the console's **Delete permanently**, and what `primitive config push --prune` sends) **destroys**: the full cascade, which is also what frees the key. So the config-as-code way to retire a workflow permanently is to remove `workflows/<key>.toml` and run a confirmed `primitive config push --prune`; to bring the key back after archiving, hard-delete the holder first and then push the file again — a bare re-push cannot clear a server-owned `archived`.

`primitive config push` creates a default configuration automatically when a workflow is first created and updates it on subsequent pushes. Each push of `[[steps]]` updates the active configuration's steps in place, live immediately.

### Configurations vs revisions

- **Configurations** (recommended): named, mutable groupings of steps. One is `activeConfigId`. Created automatically on first config push.
- **Revisions**: immutable snapshots minted by a legacy publish path that no longer exists (the command and its endpoint are gone; the endpoint answers `404`). Read-only history now — the engine still runs a legacy revision for a workflow that has one, but nothing creates or promotes revisions. Use configurations.

#### Migrating a legacy workflow to the configurations model

A workflow with no active configuration is one created before the model existed. `config push` still writes its body to the legacy DRAFT slot, and nothing promotes that slot any more, so the pushed steps never run — the push warns and names this migration. It is a pure TOML change in the app repo:

1. **Identify one — with a read that does not change it.** `primitive workflows list --json` reports `activeConfigId: null` for a legacy workflow. Prefer it over `primitive workflows get`: reading a workflow's DETAIL is not neutral (see the note below), and a `config push` of its steps warning that they landed in the legacy draft slot is the other signal.
2. **`primitive config pull`.** What you get is the active configuration's body when the detail read already migrated the workflow (a copy of the revision that was running), and the DRAFT slot's body when it did not. The draft is not necessarily what runs: if it was edited after the last publish, the two differ. Compare the pulled steps against the latest revision before activating anything — you are about to make the file's body live.
3. **Write the sidecar.** Put the body in `workflows/<key>.configs/<name>.toml` and set `activeConfigName = "<name>"` in the `[workflow]` table of `workflows/<key>.toml`. The sidecar is required: on an existing workflow, `activeConfigName` must name a config that exists or one authored as a sidecar in the same push — a name matching neither fails the push rather than renaming the live config.
4. **`primitive config push`.** It creates the config from the sidecar and activates it. The workflow is on the configurations model from here, and the warning stops.
5. **Optionally `primitive config pull` again** to normalize the layout: the live config's body moves to the top of `workflows/<key>.toml` and its sidecar goes away (pull writes sidecars only for the non-live configs).

> **Reading the detail migrates it.** Fetching a workflow's detail — `primitive workflows get`, `config pull`, `config push`, the console — auto-migrates a workflow that has no configurations: the server copies its latest revision, or the draft slot when there is no revision, into a configuration named `default` and activates it. For a workflow with a published revision that changes no behaviour (what already ran keeps running) and completes the move for you, so steps 2-4 become "pull, edit, push" like any other workflow. For a draft-only workflow it makes the DRAFT live, so look at the draft before that first read if the distinction matters. It is also why a `config push` can land steps in the draft slot and then report that the workflow now runs a configuration copied from its latest revision: push again with `--force` to write those steps to it.

### Run status vocabulary

A **run**'s status (distinct from the workflow's `active`/`inactive` availability above) is always one of nine values. The same set is used by `getStatus`, `listRuns`, `terminate`, the `workflowStatus` event, the CLI, and the persisted run record.

| Status | Terminal? | Meaning |
|---|---|---|
| `queued` | no | Accepted, waiting to start executing. |
| `running` | no | Executing. |
| `apply_pending` | no (settles `waitFor`) | Server-side work finished; waiting for the client apply handler. |
| `apply_claimed` | no (settles `waitFor`) | An apply handler has claimed the run. |
| `completed` | yes | Finished successfully. |
| `failed` | yes | Finished with an error; `error` / `errorMessage` is set. |
| `terminated` | yes | Stopped before finishing — `terminate()`, a `runSync` timeout, or a disabled user. |
| `missing` | — | The run's execution can no longer be resolved. |
| `skipped` | yes | Did not run: its declarative lock was held and the workflow declared `onContention = "ignore"`. Carries `skipReason` (`LOCK_CONTENTION`), no error, and emits no error events. |

Compare against these values directly — do not map spellings client-side. The server reconciles a run's state before answering, so:

- The value is the same across every surface for the same run, with one deliberate exception: for an apply-flow run the `workflowStatus` event reports `completed` with `needsApply: true`, where `getStatus` and `listRuns` report `apply_pending`.
- Reading a status never changes the run's recorded state, and a `terminated` run never later reads as `completed`.
- A run whose execution has just ended can briefly report `running` while the platform publishes its output, so `status` and `output` never contradict each other. Treat a non-terminal read as "not finished yet, keep polling" rather than proof the run is still executing. `run.status` on the same response carries the recorded terminal status throughout, and `waitFor` handles the window for you.
- `listRuns --status <value>` (and `?status=` over the API) accepts these values; an unrecognised value is rejected with `400` rather than silently matching nothing.

A failed run may also carry `errorCode`, the platform's own classification of the failure: `LOCK_CONTENTION` (lost a declarative-lock race under `onContention = "fail"`) or `LOCK_TIMEOUT` (exhausted an `onContention = "block"` budget). It is `null` for every failure the platform does not classify. Branch on `errorCode` / `skipReason` rather than matching `errorMessage` text — both are server-written from a closed set, and an app cannot set either one.

One value outside this set exists and is deliberate: `runSync`'s response envelope reports `status: "timeout"` when the in-request budget expired. That describes the **call**, not the run — the same envelope's `run.status` carries the canonical `terminated`.

## CLI

```bash
# Sync (recommended for everything; sync dir auto-resolves to .primitive/sync/<env>/<appId>/)
primitive config init
primitive config pull
primitive config diff
primitive config push --dry-run
primitive config push

# Authoring (TOML only — there is no workflows create/update/delete)
primitive config fields workflow                    # every [workflow] key, type, required, default
primitive config create workflow order-intake       # scaffold workflows/order-intake.toml
primitive config push --only workflow/order-intake  # apply just this workflow
# Delete: remove workflows/<key>.toml, then `primitive config push --prune`

# Reading
primitive workflows list [--status active|inactive] [--json]
primitive workflows get <workflow-id>

# Inspect / reset iterate-users iterations
primitive workflows iterations list [--app <id>] [--json]
primitive workflows iterations get <iteration-name> [--json]
primitive workflows iterations reset <iteration-name>   # only a completed/failed iteration; a running one is refused (409)

# Expand fragment includes (for debugging)
primitive workflows expand <workflow.toml>

# Preview a workflow
primitive workflows preview <workflow-id> --input '{"x":1}' --wait
primitive workflows preview <workflow-id> --config <config-id> --wait
primitive workflows preview <workflow-id> --draft --wait
# Preview source priority:
#   1. --config <id> if provided
#   2. --draft if flag set
#   3. active configuration (default)
#   4. draft (fallback if no active config)

# Configurations — read from the CLI, written as TOML
primitive workflows configs list <workflow-id>
primitive workflows configs get <workflow-id> <config-id>
# A named config's BODY is a sidecar file: workflows/<key>.configs/<name>.toml.
# The live config is `activeConfigName` in the `[workflow]` table of
# workflows/<key>.toml, and it is authoritative on create AND update.
# `config pull` writes a sidecar for every config except the live one, whose body
# is the `steps` at the top of the workflow file. `config push` applies both.

# Operational, not configuration: take a workflow out of service from anywhere,
# with no checkout. Changes no TOML field, so `config diff` reports the object as
# "operationally disabled; configuration unchanged" rather than as drift.
primitive workflows disable <workflow-id>
primitive workflows enable <workflow-id>

# Run inspection
primitive workflows runs list <workflow-id> [--status queued|running|apply_pending|apply_claimed|completed|failed|terminated|missing|skipped]
primitive workflows runs status <workflow-id> <run-id>
primitive workflows runs steps <workflow-id> <run-id>
primitive workflows runs step-detail <workflow-id> <run-id> <step-id>
primitive workflows runs error <workflow-id> <run-id>
primitive workflows runs failures <workflow-id>

# Test cases — authored at workflows/<key>.tests/<case>.toml, applied by
# `primitive config push`; `config fields workflow` lists the keys.
primitive workflows tests list <workflow-id>
primitive workflows tests run <workflow-id> <test-case-id>
primitive workflows tests run-all <workflow-id>

# Analytics — under the analytics noun, not this one
primitive analytics workflows --window-days 7 --limit 10
```

Workflow analytics live under the `analytics` noun, the single home for per-subject analytics (workflows, prompts, integrations). The `workflows analytics` group was removed. Migrate `workflows analytics top --days N --json` to `primitive analytics workflows --window-days N --json`: same endpoint, same payload, but the default window changed from 7 days to 30 — pass `--window-days 7` explicitly to keep the old default. `workflows analytics overview` was **retired without a replacement**, not moved: the `analytics/workflows/overview` endpoint it called was never registered server-side, so it always returned 404. `analytics workflows` is not that view — it ranks individual workflows by runs (with success rate, median, P95 and tokens per row) rather than reporting app-wide workflow totals. See the analytics guide for what the REST API does expose.

`runs list` includes a `DELAY` column (`queueDelayMs`); `runs status` includes "Execution started" (`executionStartedAt`), "Queue delay" (`queueDelayMs`), and "Create call" (`createCallDurationMs`, the wall-clock time of the run's underlying create call) lines. `executionStartedAt` and `queueDelayMs` are `null` while the run is still queued.

`runs failures` is the triage view: failed runs only, with `STEP` and `ERROR` columns naming the failed step and the cause, so repeated failures group at a glance instead of needing one `runs error` call per run. Its `--json` emits the shared inspection item shape inside `{ items }` — group on `detail.errorTitle`, and pivot to `runs error <workflow-id> <run-id>` for the caret-annotated expression and step input/config. An empty `STEP` means the run failed with no attributable step (a launch-time abort, a reclaimed run, or output-schema validation after every step completed) — the message is still there. The HTTP payload spells that as an explicit `null`; the CLI's `--json` omits the key entirely, so test for "missing or null". See the inspection guide for the full field table.

All inspection commands take `--json`.

Every `<workflow-id>` argument above — get, preview, configs, runs — accepts the workflow **key** as well as the ULID. The value is tried as an id first (the id wins when both exist), then as a key lookup scoped to the app and case-insensitive; an unknown value exits non-zero with `Workflow not found for id or key "<arg>"`. The `workflows tests` commands take the ULID. Don't generalize to cron triggers: `cron-triggers` ops take the `triggerId`, never the trigger key.

{{#lang ts}}
```bash
# Typed invocation wrappers from inputSchema/outputSchema (see Typed invocation below)
primitive workflows codegen [workflow-key] [-o <dir>] [--check] [--json]
```
{{/lang}}

{{#lang swift}}
```bash
# Typed invocation wrappers from inputSchema/outputSchema (see Typed invocation below)
primitive workflows codegen --lang swift [workflow-key] [-o <dir>] [--check] [--json]
```
{{/lang}}

A **durable** run (started with `primitive workflows preview` or `workflows.start()`) that aborts during **setup** — before any step executes (resolving its config/revision, loading steps, or validating `input` against `inputSchema`) — is still marked `failed` with the real error message, and records one synthetic step with id `__setup__` and kind `setup`. So `runs steps` is never empty and `runs error` always names the failure, even when no author-defined step ran. A `syncCallable` run does not synthesize that step: a setup abort there records no step run, so `runs steps` is empty and `STEP`/`failedStepId` is blank — the error message is still recorded on the run.

### Reusable step fragments

A workflow TOML can pull shared `[[steps]]` blocks out of fragment files via `include`:

```toml
# config/workflows/onboard.toml
# `include` is a TOP-LEVEL key: it must come BEFORE the [workflow] header. A
# blank line does not close a table, so writing it under [workflow] parses as
# `workflow.include` and `config push` rejects it as an unrecognized key.
include = ["common-validation", "common-audit"]

[workflow]
key = "onboard"
name = "Onboard new user"
accessRule = "true"

[[steps]]
id = "create-account"
kind = "database.mutate"
# ...
```

Fragments live at `<workflowDir>/../workflow-fragments/<name>.toml` and contain only `[[steps]]` tables (no `[workflow]` block, no further `include`). The CLI expands `include` references before `config push` — the server only ever stores the flattened step list. Step ids must be unique across the expanded set; collisions are reported with both source locations. Use `primitive workflows expand <file>` to print the expanded result.

**Expansion order: all included steps come first, then the workflow's own `[[steps]]`** — no matter where the `include` sits in the file. Fragments expand in the order they appear in the `include` array, each contributing its steps in file order; the workflow's own steps follow, also in file order. Writing `include` below your own `[[steps]]` does not push the fragment's steps later. To run something ahead of an included fragment, lift that work into a fragment of its own and list it earlier in `include`. Steps execute in expanded order, so `primitive workflows expand <file>` is the authoritative check on sequencing.

**`config pull` preserves the fragment form.** Pull never flattens a fragment-authored file just to write the server's step list back. If the file's expansion already equals the running config, pull leaves the file on disk untouched (comments and formatting included). If the server drifted but its step list still *starts* with exactly what the includes expand to — steps appended in the console, or changed workflow metadata — pull rewrites the file with the `include` blocks intact and the uncovered server steps in the workflow's own `[[steps]]`; re-expanding that file reproduces the server's step list in order, so the next `config diff` reads as synced. Only when the running steps no longer begin with the fragment's expansion (a fragment's steps were edited or dropped server-side, or a step was inserted between them) is the fragment form unrepresentable: pull then writes the fully expanded steps and logs which step diverged.

**Parameterized includes.** The `[[include]]` array-of-tables form passes parameters into a fragment and can gate all of its steps behind one condition:

```toml
# config/workflows/onboard.toml
[[include]]
fragment = "lifecycle-email"          # required
runIf = "input.plan == 'pro'"         # optional
[include.with]
id = "welcome"
subject = "Welcome aboard"
retries = 3
```

```toml
# config/workflow-fragments/lifecycle-email.toml
[[steps]]
id = "{{ params.id }}-subject"
kind = "transform"
saveAs = "emailSubject"
runIf = "input.emailOptIn"
[steps.output]
subject = "{{ params.subject }} — {{ input.appName }}"
maxRetries = "{{ params.retries }}"
```

- Allowed keys in an `[[include]]` table: `fragment` (required), `with`, `runIf`. Any other key is an error.
- One form per file: mixing bare strings and `[[include]]` tables in a single `include` array is an error.
- `{{ params.X }}` is substituted at expand time from that include's `with` table, anywhere in the fragment's steps — ids, strings, nested tables, arrays. Dotted paths (`{{ params.email.subject }}`) resolve into nested `with` tables. A reference with no matching key is a hard error naming the fragment and the include.
- A whole-string `{{ params.X }}` splices the raw typed value (`maxRetries` above expands to the number `3`); an embedded reference stringifies the value into the surrounding text.
- Only the `params.` namespace is touched. Every other `{{ ... }}` template (`{{ input.appName }}` above) passes through verbatim for the server to render at run time, and no `params.` reference survives expansion.
- An include-level `runIf` is ANDed onto every expanded step: both present → `(<includeRunIf>) && (<stepRunIf>)`; one present → that one; neither → no `runIf`. The example above yields `runIf = "(input.plan == 'pro') && (input.emailOptIn)"`.
- Include the same fragment more than once with different `with` tables to get several parameterized copies — derive the step ids from `{{ params.* }}` so the expanded ids stay unique.

### Operation `$params` validation

When a workflow references a registered database operation via `database.query`/`mutate`/etc., the CLI checks that every `$params.X` substitution in the operation's TOML maps to a declared `[[operations.params]]` entry at `config push` time. A typo like `$params.proectId` is reported with the file path and line number of the offending `[[operations]]` block instead of silently no-opping at runtime.

## Client SDK

**Default to generated invoker factories, not raw `client.workflows.start`/`runSync` calls with a string-literal `workflowKey`.** Once a workflow's `inputSchema`/`outputSchema` are pushed, run `primitive workflows codegen` and call the workflow through the generated factory it emits — see [Typed invocation (codegen)](#typed-invocation-codegen) just below. A typo'd `workflowKey`, an input built to the wrong shape, or drift between a `result.output` cast and the workflow's actual output schema after a TOML change all fail at build time through the factory instead of only surfacing at run time. [Manual invocation](#manual-invocation-raw-workflowkey) further down — raw calls with a string key and an untyped/hand-cast `input`/`output` — is the fallback for a workflow with no schema to generate from, not the default pattern for application code.

{{#lang ts}}
### Typed invocation (codegen)

Generate typed invocation wrappers from each workflow's `inputSchema`/`outputSchema`:

```bash
primitive workflows codegen [workflow-key] [-o <dir>] [--check] [--json]
```

The command reads `workflows/*.toml` from the auto-resolved config directory (`--dir <path>` overrides; `--app <app-id>` disambiguates when several apps are synced — with more than one match and neither flag, it errors rather than guessing) and emits **one `<key>.generated.ts` per workflow** (default output `<config-dir>/workflows/generated/`; stale generated files for removed workflows are cleaned up on full runs). Reserved `__internal.*` workflows are skipped; malformed schema JSON fails the command. A single `[workflow-key]` argument matches the TOML file stem and generates just that file.

Each generated file exports `<Key>Input` / `<Key>Output` interfaces plus a factory function (camelCase of the key; a leading digit gets a `_` prefix, a reserved word a `_` suffix) that pins the workflow key so it can't drift from its types:

```ts
import { createCheckoutSession } from "./generated/create-checkout-session.generated";

const wf = createCheckoutSession(client);
const res = await wf.runSync({ input: { priceId: "price_123" } }); // input: CreateCheckoutSessionInput
res.output?.checkoutUrl;                                           // output: CreateCheckoutSessionOutput

const status = await wf.getStatus({ runKey: "run-1" });            // status.output: CreateCheckoutSessionOutput | undefined
const ended = await wf.terminate({ runKey: "run-1" });             // ended.output: CreateCheckoutSessionOutput | undefined

await wf.cronTriggers.create({                                     // params.rootInput: Partial<CreateCheckoutSessionInput>
  triggerKey: "nightly-checkout",
  displayName: "Nightly Checkout",
  cron: "0 3 * * *",
  rootInput: { priceId: "price_123" },
});
```

Generated factory members:

- `.start`, `.getStatus`, and `.terminate` are emitted for **every** non-internal workflow. `.getStatus` and `.terminate` bind `output` to `<Key>Output` (schema-less → `unknown`), so an async-only workflow with no `.runSync` still gets a typed status fetch and a typed termination result. `.runSync` is emitted **only** when the workflow has `syncCallable = true` (the server rejects run-sync otherwise).
- `.cronTriggers.create(params)` and `.cronTriggers.update(triggerId, params)` are emitted for every non-internal workflow. They pin `workflowKey` and type the trigger's `rootInput` from the workflow's input schema: `Partial<<Key>Input>` for an object-shaped schema (an `inputMapping` may supply the rest), the full `<Key>Input` otherwise. `update` also accepts `rootInput: null` to clear a stored root input; `create` does not.
- `.define(options)` is emitted **only** for an apply-mode workflow (`requiresClientApply !== false`, the server's default — see [Apply pattern](#apply-pattern)). It binds `<Key>Output` so the `onApply` handler's `output` is the workflow's real output type instead of `any`. A workflow with `requiresClientApply = false` has no apply handler and gets no `.define`.
- `input` is a **required** option when the schema rejects `{}` (i.e. has required properties), optional otherwise. `workflowKey` is pinned after the options spread, so callers can't override it.
- Type mapping mirrors the server's schema validator exactly: scalar `type` → TS scalar, scalar-only type unions → TS unions, `enum` → literal union, `object` + `properties`/`required` → interface (open objects get `[key: string]: unknown`; `additionalProperties: false` omits it), `array` + `items` → `T[]`. A qualifying discriminated-union `oneOf` (see below) → a `type` union alias. Anything else the validator ignores (`$ref`, `allOf`, `format`, tuples, an `anyOf` with a non-object member) → `unknown`. A schema-less workflow gets `Input`/`Output` of `unknown`.
- After a CLI upgrade, `primitive workflows codegen --check` exits non-zero when generated files are out of date — regenerate rather than hand-editing (same CI pattern as `primitive databases codegen --check`).

### Discriminated-union (`oneOf`) schema outputs

`inputSchema`/`outputSchema` support an opt-in tagged-union mode: a `oneOf` array whose members are **all** `type: "object"`, sharing exactly one property declared as a distinct single-literal scalar `enum` in every member — that property is auto-detected as the discriminant (there is no way to name it explicitly). The server validates a value by branch-selecting on the discriminant and checking only the matched member; an unmatched or non-object value is a validation error.

```toml
[[workflow.outputSchema.oneOf]]
type = "object"
required = [ "status", "checkoutUrl" ]
[workflow.outputSchema.oneOf.properties.status]
type = "string"
enum = [ "ok" ]
[workflow.outputSchema.oneOf.properties.checkoutUrl]
type = "string"

[[workflow.outputSchema.oneOf]]
type = "object"
required = [ "status", "message" ]
[workflow.outputSchema.oneOf.properties.status]
type = "string"
enum = [ "error" ]
[workflow.outputSchema.oneOf.properties.message]
type = "string"
```

Codegen renders a qualifying `oneOf` as a `type` union alias (not an `interface`) of the member object types, joined with `|` — the natural shape for a caller to narrow on the discriminant:

```ts
const res = await wf.runSync({ input: { priceId: "price_123" } });
if (res.output?.status === "ok") res.output.checkoutUrl;   // narrowed
```

Rules and failure modes:

- `anyOf` is **not** supported for tagged unions — an all-object `anyOf` throws a codegen error suggesting `oneOf`; an `anyOf` with any non-object member renders `unknown` as before.
- A `oneOf` that looks like a tagged union but is defective — a single member, no discriminator, or an ambiguous one (more than one candidate property) — **throws a codegen error** rather than silently falling back to `unknown`. Fix the schema (add/remove a candidate property, or give every member a distinct literal) rather than working around the error.
- A tagged union must carry `oneOf` as its **only** top-level constraint — combining it with sibling `type`/`properties`/`required`/`additionalProperties`/`items`/`enum` is rejected; put shared constraints inside each member instead.
- A `oneOf` with any non-object member (a scalar/mixed union) is not interpreted as a tagged union at all — it falls through to the `unknown` rendering like any other unsupported keyword, no error.

### Wiring codegen into your build

Run `primitive workflows codegen` alongside model codegen so generated invokers stay current with every push, the same way `primitive databases codegen` does:

```json
{
  "scripts": {
    "codegen": "js-bao-codegen-v2 -i src/models/models.toml -o src/models && primitive databases codegen -o src/types/generated && primitive workflows codegen -o src/types/generated/workflows"
  }
}
```

In a config-as-code repo shared by several clients — the workflow TOML lives outside any one app's checkout — point `--dir` at the shared root instead of relying on auto-resolution:

```bash
primitive workflows codegen --dir ../config -o src/types/generated/workflows
```

`--check` (`primitive workflows codegen --check`) belongs in CI next to `primitive databases codegen --check`, so stale generated invokers fail the build instead of drifting silently.

An app scaffolded from the web starter template ships this wiring already: its `pnpm codegen` script (run by `dev`, `build`, and `test`) regenerates workflow invokers alongside model types whenever workflow TOMLs are present in the sync directory.
{{/lang}}

{{#lang swift}}
### Typed invocation (codegen)

Generate typed invocation wrappers from each workflow's `inputSchema`/`outputSchema`:

```bash
primitive workflows codegen --lang swift [workflow-key] [-o <dir>] [--check] [--json]
```

The command reads `workflows/*.toml` from the auto-resolved config directory (`--dir <path>` overrides; `--app <app-id>` disambiguates when several apps are synced — with more than one match and neither flag, it errors rather than guessing) and emits **one `<key>.generated.swift` per workflow** (default output `<config-dir>/workflows/generated/`; stale generated files for removed workflows are cleaned up on full runs). Reserved `__internal.*` workflows are skipped; malformed schema JSON fails the command. A single `[workflow-key]` argument matches the TOML file stem and generates just that file.

Each generated file declares `<Key>Input` / `<Key>Output` `Codable` types plus a factory function (camelCase of the key; a leading digit gets a `_` prefix, a Swift keyword is backtick-escaped) returning a `<Key>Workflow` struct that pins the workflow key so it can't drift from its types:

```swift
import JsBaoClient

let wf = createCheckoutSession(client)
let res = try await wf.runSync(input: CreateCheckoutSessionInput(priceId: "price_123")) // input: CreateCheckoutSessionInput
res.output?.checkoutUrl                                                                 // output: CreateCheckoutSessionOutput?

let status = try await wf.getStatus(runKey: "run-1")   // status.output: CreateCheckoutSessionOutput?
```

Generated factory members:

- `start` and `getStatus` are emitted for **every** non-reserved workflow, each `async throws`. `getStatus` binds `output` to `<Key>Output?` (schema-less → `JSONValue`), so an async-only workflow with no `runSync` still gets a typed status fetch. `runSync` is emitted **only** when the workflow has `syncCallable = true` (the server rejects run-sync otherwise) and returns `RunSyncResult<<Key>Output>`.
- `input` is a **required** argument when the schema rejects `{}` (i.e. has required properties), optional (`= nil`, sending `{}`) otherwise. The workflow key is pinned inside the wrapper, so callers can't override it.
- The wrappers delegate to the additive generic overloads on the client — `client.workflows.runSync<Input, Output>(...)`, `start<Input>(...)`, and `getStatus<Output>(...)` accept the same type parameters if you prefer to bind them by hand.
- Type mapping mirrors the server's schema validator: scalar `type` → Swift scalar, `enum` → a nested `String`-raw `enum`, `object` + `properties`/`required` → a `struct` (open objects gain an `extra: [String: JSONValue]` catch-all; `additionalProperties: false` omits it), `array` + `items` → `[T]`. A qualifying discriminated-union `oneOf` (see below) → an `enum` with associated values. Anything else the validator ignores (`$ref`, `allOf`, `format`, tuples, an `anyOf` with a non-object member) → `JSONValue`. A schema-less workflow gets `Input`/`Output` of `JSONValue`.
- After a CLI upgrade, `primitive workflows codegen --lang swift --check` exits non-zero when generated files are out of date — regenerate rather than hand-editing (same CI pattern as `primitive databases codegen --check`).

An app scaffolded from the iOS starter template ships this wiring already: `./run.sh` and `./run-ios.sh` regenerate workflow invokers alongside model types through `scripts/codegen.sh`, and `bash scripts/codegen.sh --check` runs the same staleness gate offline (it reads only the local workflow TOMLs — no network or sign-in), so it fits a pre-commit hook or CI.

### Discriminated-union (`oneOf`) schema outputs

`inputSchema`/`outputSchema` support an opt-in tagged-union mode: a `oneOf` array whose members are **all** `type: "object"`, sharing exactly one property declared as a distinct single-literal scalar `enum` in every member — that property is auto-detected as the discriminant (there is no way to name it explicitly). The server validates a value by branch-selecting on the discriminant and checking only the matched member; an unmatched or non-object value is a validation error.

```toml
[[workflow.outputSchema.oneOf]]
type = "object"
required = [ "status", "checkoutUrl" ]
[workflow.outputSchema.oneOf.properties.status]
type = "string"
enum = [ "ok" ]
[workflow.outputSchema.oneOf.properties.checkoutUrl]
type = "string"

[[workflow.outputSchema.oneOf]]
type = "object"
required = [ "status", "message" ]
[workflow.outputSchema.oneOf.properties.status]
type = "string"
enum = [ "error" ]
[workflow.outputSchema.oneOf.properties.message]
type = "string"
```

Codegen renders a qualifying `oneOf` as an `enum` with one case per member (named after the member's discriminant value), each carrying that member's branch `struct`; decode is keyed on the discriminant. The caller narrows with a `switch`:

```swift
let res = try await wf.runSync(input: CreateCheckoutSessionInput(priceId: "price_123"))
if let output = res.output {
  switch output {
  case .ok(let ok): ok.checkoutUrl       // narrowed to the "ok" member
  case .error(let err): err.message      // narrowed to the "error" member
  }
}
```

Rules and failure modes:

- `anyOf` is **not** supported for tagged unions — an all-object `anyOf` throws a codegen error suggesting `oneOf`; an `anyOf` with any non-object member renders `JSONValue` as before.
- A `oneOf` that looks like a tagged union but is defective — a single member, no discriminator, or an ambiguous one (more than one candidate property) — **throws a codegen error** rather than silently falling back to `JSONValue`. Fix the schema (add/remove a candidate property, or give every member a distinct literal) rather than working around the error.
- A tagged union must carry `oneOf` as its **only** top-level constraint — combining it with sibling `type`/`properties`/`required`/`additionalProperties`/`items`/`enum` is rejected; put shared constraints inside each member instead.
- A `oneOf` with any non-object member (a scalar/mixed union) is not interpreted as a tagged union at all — it falls through to the `JSONValue` rendering like any other unsupported keyword, no error.

### Wiring codegen into your build

Run `primitive workflows codegen --lang swift` as part of the app's regular codegen step (a pre-build script or CI job) alongside model codegen, so the generated invokers stay current with every workflow push:

```bash
primitive workflows codegen --lang swift -o Sources/App/Workflows/Generated
```

In a config-as-code repo shared by several clients — the workflow TOML lives outside any one client's checkout, as it typically does for an iOS client paired with a web client — point `--dir` at the shared root instead of relying on auto-resolution:

```bash
primitive workflows codegen --dir ../config --lang swift -o Sources/App/Workflows/Generated
```

`--check` (`primitive workflows codegen --lang swift --check`) belongs in CI next to `primitive databases codegen --check`, so stale generated invokers fail the build instead of drifting silently.
{{/lang}}

### Manual invocation (raw `workflowKey`)

Reach for these only when a workflow has no `inputSchema`/`outputSchema` to generate from, or for one-off exploration outside a codegen'd app — application code should call through the generated factories above instead.

Start a workflow, check its status, and list recent runs:

{{ example: workflows/workflow-start }}

Full options and result shapes for these calls:

{{ example: workflows/workflow-run-options }}

Inspect the per-step debug records of a run:

{{ example: workflows/workflow-list-step-runs }}

The step records carry `{ stepRunId, runId, stepKind, status, input, output, error, startedAt, endedAt, ... }`. A step's `status` is `running` while it executes, then `completed`, `failed`, `skipped`, or `error_captured` (the step errored but the workflow captured the error and continued, so the run is not failed by it). Step records are written as the run goes: reading them on an in-flight run returns the finished steps plus a `running` row for the current one, finalized in place when the step ends — and a run that ends while a step is still `running` (aborted mid-step) closes that row as `failed`.

`runKey` is scoped as `${contextDocId}#${runKey}`. Calling `start` with an existing `runKey` returns `{ existing: true, ... }` unless `forceRerun: true`.

## WebSocket events

Requires an active WebSocket (e.g., from `client.documents.open(docId)`).

{{ example: workflows/workflow-events }}

The `workflowStatus` event and `getStatus` report the same vocabulary — see [Run status vocabulary](#run-status-vocabulary). The event fires only on a terminal transition, so its `status` is always `completed`, `failed`, `terminated`, or `skipped` — and it carries `errorCode` / `skipReason` alongside, so a listener can branch on the structured field.

{{#lang swift}}
Subscribe through `client.stream(for:)` — a `for await` loop in a `.task`, which unsubscribes when the loop ends — or through `client.observeOnMainActor(_:handler:)` when you need a main-actor callback, holding the returned `EventSubscription` for as long as you want the handler live.

`client.nextEvent(_:timeout:where:)` is the one-shot form: it awaits the first event the predicate accepts, throws `JsBaoError` with code `.unavailable` on timeout (default 10 seconds), and throws `CancellationError` if the calling task is cancelled.
{{/lang}}

### `workflows.waitFor`

`workflows.waitFor` wraps the `workflowStatus` frame in a single awaitable call instead of a manual event listener.

- Event-driven: subscribes to the `workflowStatus` frame, plus one reconcile fetch immediately after subscribing to close the started-before-subscribed race; re-runs that reconcile on reconnect. No polling.
- Settles on every terminal state (`completed` / `failed` / `terminated` / `apply_pending` / `apply_claimed` / `skipped`), including `"failed"` — a failing run is a normal result, not an error. `skipped` needs `js-bao-wss-client` >= 2.1.0 (or the Swift client at or after the release that ships it): an older client does not recognise it as terminal and waits out its own timeout.
- Fails only on timeout (`JsBaoError` code `WORKFLOW_WAIT_TIMEOUT`; the default wait is 15 minutes) or when `runId` doesn't resolve to a run (`NOT_FOUND`).

{{#lang ts}}
```ts
const { status, output, error } = await client.workflows.waitFor(runId, { timeoutMs: 60000 });
```

Signature:

```ts
workflows.waitFor(runId: string, options?: { timeoutMs?: number }): Promise<{
  status: "completed" | "failed" | "terminated" | "apply_pending" | "apply_claimed" | "skipped";
  output?: any;
  error?: string;        // set when status === "failed"
  skipReason?: string;   // set when status === "skipped" ("LOCK_CONTENTION")
}>
```

`timeoutMs` is in milliseconds and defaults to `900000`. Pass `timeoutMs: 0` or `Infinity` to disable the timeout.
{{/lang}}
{{#lang swift}}
```swift
let result = try await client.workflows.waitFor(
    runId: runId,
    options: WaitForWorkflowOptions(timeout: 60)
)
// result: WaitForWorkflowResult { status: String, output: JSONValue?, error: String?, skipReason: String? }
// skipReason is set when status == "skipped" ("LOCK_CONTENTION"); nil otherwise
```

`WaitForWorkflowOptions.timeout` is a `TimeInterval` in **seconds** (default 900 — 15 minutes). `waitFor(runId:as:options:)` decodes `output` into a `Decodable` type instead of a raw `JSONValue`. Pass a `timeout` of `0` (or any non-positive value) to disable the timeout — the call and its listeners then live until the run terminates, so only do that for runs you know will end. Cancelling the awaiting `Task` tears the wait down and throws `CancellationError`.
{{/lang}}

## Apply pattern

For workflows with `requiresClientApply = true`, register an apply handler so a client deterministically runs follow-up logic exactly once.

{{ example: workflows/workflow-apply }}

Register `define(...)` before `start(...)` so the apply can't arrive before the handler is in place.

{{#lang swift}}
After an offline gap, `getPendingApplies(contextDocId:)` lists runs still awaiting apply.
{{/lang}}

Manual flow if you need it: `claimApply` → run logic → `confirmApply` (success) or `releaseApply` (failure). 30s lease timeout for crashed clients.

## Footguns and don't-do-this

### Wrong: trying to use `{{ }}` in `runIf`

```toml novalidate
# WRONG — runIf is CEL, not a template
runIf = "{{ steps.check.isMember }}"

# RIGHT
runIf = "steps.check.isMember"
```

### Wrong: stuffing secrets into step config

```toml novalidate
# WRONG — step config is logged to step run records
[steps.request.headers]
Authorization = "Bearer {{ secrets.API_KEY }}"

# RIGHT — put secrets in the integration's defaultHeaders/staticQuery
# (configured once on the integration, never appears in step output)
```

### Wrong: using `{{ now }}` for a value that must stay stable across a run

`now` re-evaluates on every reference, so two steps that each read `{{ now }}` get different timestamps. For one run-wide timestamp — a filename, a batch id, a "generated at" stamp shared across steps — use `{{ meta.startedAt }}`, the run's stable start time (identical across every step and retry). Reserve `now` for a genuinely per-reference clock read.

```toml novalidate
# WRONG — each step reads a different clock value
filename = "{{ now }}.pdf"

# RIGHT — one stable timestamp for the whole run
filename = "{{ meta.startedAt }}.pdf"
```

### Wrong: re-running an idempotent step inside a retry loop

`continueOnError = true` does not retry — it captures the error and moves on. To retry, the workflow has to re-invoke the step explicitly (e.g., another workflow run, or a `forEach` fan-out that re-dispatches failed items). The engine already retries transient errors per step automatically; don't add a second layer.

### Wrong: leaking secrets/PII via `saveAs`

```toml
# WRONG — saveAs result is persisted in step_run records
[[steps]]
id = "fetch-secrets"
kind = "database.query"
saveAs = "creds"
# operationName returns rows with API keys
```

If a step output contains sensitive data, do NOT `saveAs`. Pipe it directly into the next step via `steps.<id>.field` and immediately overwrite/transform it down to non-sensitive fields.

### Wrong: cron triggers that overlap when work isn't idempotent

```toml
# Default overlapPolicy = "skip" → if the previous run is still running, the next firing is dropped.
# Set "allow" only when each firing is independent and idempotent. There is no "queue" policy.
```

### Wrong: forgetting `requiresClientApply = false` for server-only workflows

Cron- and webhook-triggered workflows almost always want `requiresClientApply = false`. Otherwise the run sits in `apply_pending` forever because no client is listening.

```bash
primitive config set workflow/<key> workflow.requiresClientApply=false
primitive config push --only workflow/<key>
```

### Wrong: assuming `email.send` accepts an array `to`

```toml
# WRONG
to = ["a@x.com", "b@x.com"]

# RIGHT — fan out with forEach
[[steps]]
kind = "email.send"
forEach = "{{ input.recipients }}"
as = "addr"
to = "{{ addr }}"
templateType = "..."
```

### Wrong: `workflow.call` thinking the child sees parent state

The child gets ONLY its `[steps.input]` table as `input`. It does not inherit `steps`, `outputs`, `meta`, or `selected`. Pass everything the child needs explicitly.

### Wrong: arbitrary analytics query types

Use the dotted form. `top-users` → `users.top`. `events-grouped` → `events.grouped`. `workflows` → `workflows.top`.

### Wrong: mirroring workflow state into a separate database row

```toml
# WRONG — patching an `importJobs` row at the end of the workflow to track status
[[steps]]
id = "mark-done"
kind = "database.mutate"
operationName = "patch-import-job"
[steps.params]
id = "{{ input.jobId }}"
data = { status = "ready_to_review" }
```

This drifts the moment the workflow ends without your mutate firing — async failure, terminated run, an upstream step that throws between the data write and the status patch. The row sticks on `"processing"` forever and your UI spins indefinitely.

The workflow engine already tracks status (`running` / `apply_pending` / `apply_claimed` / `complete` / `failed` / `terminated`). Use the workflow's own machinery instead of mirroring it:

- **`meta` (≤1KB)** passed to `workflows.start()` for small client-display fields that need to ride alongside the run (filenames, blob IDs, source labels). Surfaces in `listRuns`, `getStatus`, and `workflowStarted` / `workflowStatus` events.
- **Run `output`** (via a final `transform` step with `saveAs = "output"`) for parsed results. Read via `getStatus({ workflowKey, runKey })`.
- **`requiresClientApply = true`** when you need a "ready for user action" stage before finalization — the run sits in `apply_pending`, the client `claimApply`s it, runs whatever it needs locally, then `confirmApply`s. The run's status now encodes the full pipeline including "user has applied this".

Only persist a separate database row when you need durable history beyond the workflow's 45-day TTL — and even then, store a *result record*, not a status mirror.

## Limits / TTLs

- Workflow runs: cleaned up after **45 days** (preview runs: **7 days**).
- `forEach`: 500 items default, override with `maxItems`. Over the cap the step fails non-retryably before any iteration runs.
- `collect`: 10 pages / 10000 items default.
- `analytics.query`: 50 queries per run.
- `analytics.write`: 25 events per step.
- `workflow.call`: max depth 10; circular calls throw.
- Cron triggers: 50 per app.
- Webhooks: 50 per app, archived ones included (hard-delete to free a slot).
- `meta` payload: 1KB.
- App secrets via templates/CEL: read-only, available as `secrets.KEY`.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `404 WORKFLOW_NOT_FOUND` on start/run-sync | No workflow with that key in this app context |
| `409 WORKFLOW_INACTIVE` ("Workflow '<key>' is inactive") | The workflow exists but is out of service — run `primitive workflows enable <key>` |
| `409 WORKFLOW_ARCHIVED` | The workflow was retired by a delete. `enable` will not bring it back: hard-delete the archived holder (delete it in the console, or remove its file and run `primitive config push --prune`), then re-add the file and push |
| `400 WORKFLOW_NO_ACTIVE_CONFIG` | Active status but no active config/revision — run `primitive config push` |
| `400 WEBHOOK_LIMIT_REACHED` on a webhook create | The app holds 50 webhooks, archived ones included. Deleting a webhook's TOML and running `primitive config push --prune` frees a slot; an API or console delete only archives and does not. |
| `409 WEBHOOK_KEY_EXISTS` on a webhook create | The app already has a webhook with that `key` — an archived one still holds it; hard-delete to release it. `primitive config push` converges on this `409` by adopting the existing webhook and updating it in place, so it surfaces only on direct API calls. |
| `Cannot activate workflow without a configuration` | Push steps before activating |
| `Workflow has no draft or configuration to preview` | Run `primitive config push` first — the push creates the workflow's configuration |
| Run stuck in `apply_pending` | `requiresClientApply = true` but no client running `define()` for that key. Set to `false` for server-only workflows. |
| `existing: true` on `start()` | A run with the same `(contextDocId, runKey)` already exists. Use a different `runKey` or `forceRerun: true`. |
| `Step "X": required field "Y" is empty` | A template expression resolved to a real but empty value (`""`). A path that does not resolve at all fails earlier, with `unresolved template expression`. |
| `runIf expression failed` | CEL syntax error or unknown identifier in `runIf`. Don't use `{{ }}` inside. |

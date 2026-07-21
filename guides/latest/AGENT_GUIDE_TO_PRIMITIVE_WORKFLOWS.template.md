# Workflow Agent Guide

Workflows are multi-step server-side automations defined in TOML. Each step is one of a fixed set of `kind`s (LLM call, integration call, prompt execute, database op, email, blob, etc.). Steps run sequentially with template-rendered inputs and a shared output context.

This guide is the source of truth for what's actually in `src/workflows/`. Examples are kept short and load-bearing.

## TOML structure

```toml
[workflow]
key = "my-workflow"               # required, unique per app
name = "My Workflow"              # required
description = "..."               # optional
status = "draft"                  # draft | active | archived
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

Schema form rules: the sub-table headers must come after the scalar `[workflow]` keys (a sub-table header ends the parent table). `sync pull` always writes schemas as native tables regardless of the form the file had before; a JSON-encoded string (`inputSchema = "{\"type\":\"object\"}"`) is accepted on push, and pull only emits one when the schema contains a `null` TOML can't carry (logged with the workflow and field name). An absent schema omits the key entirely; an empty schema is an empty table. `primitive sync migrate-toml` rewrites legacy JSON-string schemas in local files to native form.

A schema may also be a discriminated union: a top-level `oneOf` whose members are all `type = "object"` and share one property declared as a distinct single-literal `enum` — the server validates a value by branch-selecting on that property and checking only the matched member, and `oneOf` must be the schema's only top-level constraint (put shared fields inside each member).

Workflow-level fields the engine actually reads:
`perUserMaxRunning`, `perUserMaxQueued`, `perAppMaxRunning` (default 25), `perAppMaxQueued` (default 10000), `queueTtlSeconds` (default 43200), `dequeueOrder`, `accessRule`, `runAs` (default `caller` — see "Execution identity" below), `capabilities` (system-only grants), `inputSchema`, `outputSchema`, `requiresClientApply` (default `true` — see "Client apply" below), `syncCallable` (default `false` — see "Synchronous invocation" below). `sync push` forwards every one of these — `name`, `description`, `status`, `accessRule`, `runAs`, queue settings, schemas, `requiresClientApply`, step list — on both a fresh create and a push to an existing workflow, with two exceptions that need a direct CLI update instead: `syncCallable` on an existing workflow (the server re-validates it against the currently-active steps, so `sync push` omits it there — use `primitive workflows update --sync-callable true` after the new steps are live) and `capabilities` on an existing workflow (`sync push` persists `capabilities` on create only — grant or revoke them on an existing workflow with `primitive workflows update --capabilities <list>`, comma-separated, `""` to revoke all).

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
| `maxItems` | forEach cap (default 200) |
| `concurrency` | Parallel forEach lanes (integer 1-100; default 1 = sequential). Results preserve insertion order. |
| `successWhen` | CEL predicate evaluated per forEach iteration to classify the result as functionally succeeded vs empty (see `forEach` below) |
| `continueOnError` | Capture errors as `{ error, errorDetails, ok: false, errored: true }` instead of failing the workflow |
| `skipWhenSkipped` | Array of earlier step ids. Before this step's own `runIf` is evaluated, skip this step (with the same `{ ok: false, skipped: true }` stub) if any listed upstream was skipped. Transitive; reacts only to `skipped: true`, not `errored: true`. Unknown/forward ids are tolerated at run time and warned at save time. |
| `strict` | Throw if any template expression in this step is unresolved |
| `strictParams` | Array of top-level param names to treat as strict-on-missing while others stay null-tolerant (a resolved `null` is still tolerated). `strict = true` covers everything and wins when both are set |

## Data flow

A run threads one JSON context through every step:

1. **Input**: one JSON object per run — the `start()`/`runSync()` input, the webhook's mapped payload, or the cron trigger's configured input. Validated against `inputSchema` when declared. Top-level scalar properties are coerced to the declared type before the strict check where it's safe: number→`string` via `String()`, numeric string→`number`/`integer` via `Number()` (rejects `""`/NaN), number→`integer` only when integral, `"true"`/`"false"` (case-insensitive)→`boolean`. A non-coercible value (e.g. `"abc"`→number) fails the run with a clear type error. Per-property `coerce: false` opts out; unions/enums/null/nested are not coerced. Applies at every input site (durable run, `start`, `run-sync`, admin preview/run, and `workflow.call` child input), which makes explicit `| string`/`| number` filters at typed call sites optional no-ops.
2. **Step config is templated at execution time**: steps have no implicit input argument — `{{ ... }}` expressions in config strings resolve against the run context (`input`, `steps.<id>`, `outputs.<saveAs>`, `secrets`, `meta`, forEach vars) just before the step runs (see [Templating](#templating)).
3. **Output recording**: each step's JSON result is stored as `steps[id]`; `saveAs = "name"` also registers it as `outputs.name` — a stable alias that survives step-id renames. The engine stamps the uniform verdict (`ok`, plus `skipped`/`errored`) on every object entry; array and primitive results pass through unstamped.
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

[steps.variables]
text = "{{ input.content }}"
```

Output: result from the prompt config's provider — typically `{ content, role, metrics }`.

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

Workflows have no *database* batch-write step kind. To apply a set of database updates, run `forEach` over a `database.mutate` step (see `forEach` below), or use `database.applyToQuery` for query-driven updates. (Document writes are different: see `document.save`/`patch`/`delete` batch mode and `document.bulkUpdate` below.)

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
  { model = "Account",  action = "create", id = "{{ input.newAccountId }}", fields = { name = "Growth", balance = 0 } },
  { model = "Ledger",   action = "patch",  id = "{{ input.ledgerId }}", fields = { rebalancedAt = "{{ now }}" } },
  { model = "Position", action = "delete", id = "{{ input.staleId }}" },
]
```

Operation shape: `{ model, action, id, fields? }`. The `action` vocabulary is exactly `create` | `patch` | `delete` — not add/update/save.

| `action` | `id` | Semantics |
|---|---|---|
| `create` | **required, caller-supplied ULID** (must match `^[0-9A-HJKMNP-TV-Z]{26}$` — 26 uppercase Crockford chars) | Strict create — fails if the id already exists. The server never mints ids for this step. |
| `patch` | required, non-empty string | Merges `fields` — fails if the id doesn't exist. |
| `delete` | required, non-empty string | Removes the record; an id that doesn't exist is a no-op (contributes 0 to `deleted`). |

Output: `{ applied, added, updated, deleted }` — `added`/`updated` list `{ model, id }` per committed create/patch in input order; `deleted` is the count actually removed.

Rules and edge cases:

- **Cap = 500 operations**, a separate constant from batch mode's 100-record chunk — and **never chunked** (chunking would break the atomicity that is this step's point). Over 500 is rejected non-retryably before any write.
- **Mint create ids with <span v-pre>`{{ ulid }}`</span>** (template helper) for statically-known counts, or the script `ulid()` builtin when a script builds the operations array dynamically — pre-minting is what lets a later operation in the same blob reference a record an earlier operation creates.
- **Validation-first, then fail-fast.** A structural pass (shape, cap, ULID well-formedness, known `action`) runs before any write; failures are non-retryable and name the offending operation index and model. State-dependent failures (create-collision, patch-miss, unique-index violation) abort inside the transaction — nothing commits — and are also non-retryable. Transient storage/broadcast failures stay retryable.
- **Operations apply in listed order** within the one transaction.
- **Caller-mode ACL is checked once** at write level: the caller needs `read-write` on the document; denial is non-retryable and nothing commits. An empty `operations` array is a clean no-op.

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
  { model = "Holding", action = "create", id = "{{ ulid }}", fields = { symbol = "VTI", shares = 10 } },
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

`to` is a single address (string), not an array. Built-in templates: `magic-link`, `otp`, `document-share`, `document-share-deferred`, `collection-share`, `collection-share-deferred`, `waitlist-invite`, `waitlist-signup-notification`, `admin-invite`, `app-invite`, `access-request-created`, `access-request-resolved`. Register custom types with `primitive email-templates set <type>`. Hourly rate limit: 100 workflow emails per app per hour.

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
`overview.dau`, `overview.wau`, `overview.mau`, `overview.growth`, `daily-active`, `rolling-active`, `cohort-retention`, `users.top`, `users.search`, `users.detail`, `users.snapshot`, `events`, `events.grouped`, `workflows.top`, `prompts.top`, `integrations`. `users.detail` and `users.snapshot` require `userUlid`.

Optional fields:

- `groupBy` — for `events.grouped`. One of `action`, `feature`, `day`, `route`, `country`, `deviceType`, `plan`.
- `filters` — for `events` and `events.grouped`. Array of `{ field, operator, value }` (max 10 entries; values capped at 200 chars). Same field/operator set as the `/analytics/events` REST endpoint.
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

Runs a sandboxed [Rhai](https://rhai.rs/) script over JSON input and returns JSON. Use it for transforms too involved for a templated `transform` step (nested reshaping, derived fields, array map/filter/reduce). The sandbox is **side-effect-free** — no network or storage access — and deterministic except for the `ulid()` builtin (mints fresh record ids, e.g. for `document.bulkUpdate` creates), so script steps are safe to retry and easy to test. The [Scripts guide](AGENT_GUIDE_TO_PRIMITIVE_SCRIPTS.md) covers the script model, input/output contract, limits, error codes, and gotchas in full; this section is the step-level reference.

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

The referenced body (`transforms/normalize-order.rhai`):

```
let items = input.raw.items.filter(|i| i.qty > 0);
let total = 0.0;
for i in items { total += i.qty * i.price; }
#{
  currency: input.currency,
  itemCount: items.len(),
  total: total,
}
```

Given `steps.fetch.body = { "items": [{ "sku": "a1", "qty": 2, "price": 5.0 }, { "sku": "b2", "qty": 0, "price": 9.0 }] }` and `input.currency = "USD"`, the step records `steps.normalize.output = { "result": { "currency": "USD", "itemCount": 1, "total": 10.0 }, "ok": true }` — the return value is wrapped under `result` (see Result nesting below).

- `ref` (required) names a `Script` — a stored Rhai body, unique per app. Script bodies live in `transforms/<name>.rhai` in your sync directory and are mirrored to the server by `primitive sync push` (and pulled back by `primitive sync pull`); the `<name>` is the filename without `.rhai`. Script bodies are authored only through sync — no CLI command writes them; the `primitive scripts` command inspects scripts and manages their test cases (see the [Scripts guide](AGENT_GUIDE_TO_PRIMITIVE_SCRIPTS.md)).
- `with` is the JSON context handed to the script. Inside the script the whole table is exposed as **`input.*`** (with `ctx.*` as an alias) — NOT as bare top-level variables. `let x = payload;` fails with `Variable not found: payload`; write `input.payload`. Also, `with` itself is a reserved Rhai keyword — a script can't declare a variable named `with`.
- **Result nesting.** A script step's output is an envelope: `steps.<id>.output = { result: <return value>, ok: <bool> }`. The return value lives at `steps.<id>.output.result.*` and the success verdict at `steps.<id>.output.ok` (also mirrored at `steps.<id>.ok`); per-run telemetry sits in a sibling `steps.<id>.scriptMetrics`, not inside `output`. Unlike `transform`, whose result is the templated table directly (`steps.<id>.<field>`), wire downstream templates/`runIf` as `{{ steps.normalize.output.result.total }}` — not `{{ steps.normalize.output.total }}` or `{{ steps.normalize.total }}`.
- **Script bodies resolve live at run time.** The runner looks up the script's active config body on each execution; there is no publish-time snapshot. Pushing a changed `.rhai` file (`primitive sync push`) creates a new config and activates it — referencing workflows pick up the new body on their next run with no re-publish step. Pin a specific config with `configId = "..."` on the step to bypass the active-config lookup when determinism is required.
- Handy patterns: `parse_json(input.someJsonString)` for JSON-string fields (e.g. payload columns stored as strings); missing keys read as `()` (test with `h.symbol != ()`); `NaN`/`Infinity` can't survive JSON output — they serialize as `null`, so return a sentinel instead.
- **`parse_json` parses any strict-JSON value** — object→map, array→array, plus primitives and `null` — so a field storing a JSON array string parses straight into an array. Input must be strict JSON (no trailing commas, comments, single quotes, or hex literals); invalid input fails the step with a non-retryable `SCRIPT_RUNTIME_ERROR` and a positioned message.
- `limits` (optional) lets a step lower the per-run ceilings (`maxOperations`, `wallMsHint`, `maxOutputBytes`, `maxArrayLength`, `maxObjectKeys`, `maxNestingDepth`, `maxStringSize`, `maxCallDepth`, `maxLogBytes`); requested values are clamped at the app ceiling, never raised.
- Deterministic failures (parse / compile / runtime / limit / validation) come back as a non-retryable step error so durable retries don't re-run a guaranteed failure; transient/transport errors throw and retry normally. The runtime fails closed — it never silently passes input through.
- Each execution records per-step telemetry at `steps.<id>.scriptMetrics` — a sibling of the step's `output`, persisted on `WorkflowStepRun.scriptMetrics` (operation counts, input/output byte sizes, runtime version) — visible in run detail.

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
- Passthrough fields reach the lowered step: prompts take `modelOverride`; integrations take `attachments` / `bodyMode` / `multipartFields`; scripts take `limits`.

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

**Missing path sentinel.** In interpolation mode (`"prefix-{{ steps.x.y }}-suffix"`), an unresolved path renders as `<missing: steps.x.y>` so it's visible in step output and logs. In single-expression mode (`"{{ steps.x.y }}"` alone) the raw value is `null` so downstream `runIf`/comparisons work naturally. A resolved `null` interpolates as `"null"`; a resolved empty string interpolates as `""`. A filter chain that **begins with `default`** (or `now`) rescues a missing path instead: `{{ steps.x.output.result | default: '' }}` renders `''` when the path is unresolved — including when step `x` was skipped and recorded no `output` at all. That makes `default` the idiom for skipped-step collapse in an output table: a field built from an optionally-skipped step yields the fallback value rather than `<missing: …>`. Set `strict = true` on the step to throw on any unresolved path instead, or `strictParams = ["userId", ...]` to make only the named top-level params strict-on-missing (a resolved `null` is still tolerated); `strict = true` covers everything and wins when both are set. When a template references a root outside the six valid roots (`input`, `steps`, `outputs`, `meta`, `secrets`, `vars`), the strict-mode error lists them — catching typos like `{{ inputs.userId }}`.

## `runIf` (CEL, not templates)

```toml novalidate
runIf = "input.shouldRun"                        # truthy
runIf = "outputs.text.length < 1000"             # comparison
runIf = "steps.check.isMember && input.amount > 0"
runIf = "steps.previous.ok"                      # uniform verdict on every object result
runIf = "!steps.fetch.skipped"
```

CEL context: `input`, `selected`, `steps`, `outputs`, `meta`, `secrets`, plus `iteration` (and `as`-var) inside `forEach`, and `expr.<name>` for any [named guard](#named-guards-expr) declared on the workflow. **Do NOT wrap in `{{ }}`** — `runIf` parses CEL directly. A CEL evaluation error fails the step (or is captured by `continueOnError`).

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
- **Validation is developer-facing.** `primitive sync push` fails fast — and the server 400s — on an `expr.<name>` reference (in a definition body *or* a step guard) that names no declared definition, on a reference cycle, or on an empty definition body. `primitive sync pull` writes the `[expr.cel]` table back out, so definitions round-trip.

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

Output is always `{ items: [...per-iter results], errors: [{index, error}], totalSucceeded, totalFailed, totalEmpty, ok }` — even when there are no errors. Results are ordered by input index regardless of completion order. `ok` is `true` iff every iteration succeeded (and the source was non-empty); use it in `runIf` on the next step.

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

Workflows have no *database* batch-write step kind — a set of database updates is a `forEach` over a `database.mutate` step (add `concurrency` to parallelize):

```toml
[[steps]]
id = "import-contacts"
kind = "database.mutate"
forEach = "input.contacts"
as = "contact"
maxItems = 1000
databaseId = "{{ input.dbId }}"
operationName = "createContact"
[steps.params]
name = "{{ contact.name }}"
email = "{{ contact.email }}"
```

To mutate every record matching a server-side filter without enumerating items, use `database.applyToQuery` instead.

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
- `continueOnError = true`: failure is captured as `steps[id] = { error, errorDetails, ok: false, errored: true }` and execution continues.
- `strict = true`: any unresolved template expression in the step throws with a path-listing error.
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
| `ok: boolean` | On every object result. `true` if the step ran and the runner classified it as successful; `false` for skipped, errored, or kind-specific failures (e.g. an `integration.call` that returned HTTP 5xx) |
| `skipped: true` | The step's `runIf` evaluated falsy, or a step listed in its `skipWhenSkipped` was itself skipped. The runner did NOT execute. |
| `errored: true` | The step threw but was captured by `continueOnError = true` |
| `error`, `errorDetails` | Companion fields populated when `errored: true` |

`ok`, `skipped`, `errored`, `error`, and `errorDetails` are written by the engine and override any same-named field the runner would have produced. Downstream `runIf` and templates can rely on them whenever the step's result is an object — skipped and errored steps always are:

```toml novalidate
runIf = "steps.fetch.ok"
runIf = "!steps.fetch.skipped && !steps.fetch.errored"
```

When a step reads `steps.<upstream>.output.*` and that upstream can be skipped, a skipped upstream leaves no `output` key and an unguarded multi-segment read throws `No such key: output`. Two structural fixes: guard the read with safe navigation (`steps.fetch.?output.?items`), or declare `skipWhenSkipped = ["fetch"]` so the dependent auto-skips whenever `fetch` skips — its `runIf` is never evaluated. The skip is transitive and reacts only to `skipped: true` (an upstream that errored under `continueOnError` does not trigger it). List only earlier step ids; a save-time warning flags an unguarded skippable read or a `skipWhenSkipped` entry naming an unknown or forward step id.

## Access control

`accessRule` is a CEL expression. Evaluated when:
- A client calls `workflows.start()`.
- A `workflow.call` step invokes the workflow from another workflow.

NOT evaluated for inbound webhook or cron triggers (those bypass it entirely — a webhook handles its own auth, e.g. a Stripe signature). A webhook- or cron-triggered workflow must be `runAs: "system"` (see [Execution identity](#execution-identity-runas-system-workflows)), and a system workflow takes **no** `accessRule` — the rule is never evaluated on a system run, so a non-empty one is rejected at save/push time (see below). What prevents direct client invocation of a webhook workflow is the system-invocation gate plus the webhook's signature verification. Reserve `accessRule` for `runAs: "caller"` workflows, where it genuinely gates who may start a run.

Behavior:
- No rule → any authenticated app member can start.
- `admin`/`owner` always bypass.
- Otherwise, evaluate against `user.userId`, `user.role`, plus `hasRole(role)`, `isMemberOf(groupType, groupId)`, `memberGroups(groupType)`.

Set `accessRule` once in the `[workflow]` TOML block and it sticks: `primitive sync push` and `primitive workflows create --from-file` both apply it (an absent/empty rule is sent as "no rule" rather than dropped). To change it later without re-pushing the file, `primitive workflows update <id> --access-rule "<CEL>"` sets it (pass `--access-rule ""` to clear; omitting the flag leaves the existing rule untouched).

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

Grant or revoke `capabilities` on a workflow that already exists with `primitive workflows update <id> --capabilities <list>` (comma-separated; `""` revokes all) — `sync push` only persists `capabilities` when creating a new workflow (see above).

**`iterate-users` is system-only.** A workflow containing an `iterate-users` step must set `runAs = "system"` — otherwise it's rejected at save time (`'iterate-users' is system-only and may appear only in a runAs:"system" workflow`).

**The resource lifecycle and collection-membership steps run in both modes, capability-gated in system mode.** `database.create`/`delete`, `group.create`/`delete`, `collection.create`/`delete`, `collection.grantGroupPermission`, `collection.addDocument`, and `collection.removeDocument` all run in `runAs:"caller"` workflows under the caller's own CEL rules and doc access, exactly like the equivalent client call. In a `runAs:"system"` workflow, each instead requires its matching capability (`resource-provision`, `resource-teardown`, or `resource-grant` — see above); missing it fails the step non-retryably before anything is written. A system-mode create is attributed to the run's system principal (`createdBy: "sys:<appId>"`); a system-mode delete may only remove a resource with that attribution unless the workflow also holds `resource-teardown:any` — see [Resource lifecycle steps](#resource-lifecycle-steps).

**No `accessRule` on a system workflow.** A `runAs:"system"` workflow with any non-empty `accessRule` is rejected at save time (`runAs:"system" workflows do not evaluate accessRule — remove the accessRule`) — a system run skips the rule entirely, so a constant (`"false"`), a role/group rule (`hasRole('admin')`), or a `user`-principal reference are all equally dead config. The only fix is to remove it. An absent/empty rule is fine. Enforced on every save path (create, version-create, metadata PATCH on the merged effective state, workflow-config create, and `sync push`).

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

External services trigger workflows via inbound webhooks. This is the **inbound** half of a third-party integration; the **outbound** half — calling the provider's API — is a configured integration (see the Integrations guide). Most integrations need both. Define webhooks as `webhooks/*.toml` in the sync directory and push with `primitive sync push`:

```toml
# config/webhooks/stripe-payments.toml
[webhook]
key = "stripe-payments"
displayName = "Stripe Payments"
workflowKey = "process-stripe"
verificationScheme = "stripe"     # stripe | github | slack | custom | none
signingSecret = "{{secrets.STRIPE_WEBHOOK_SECRET}}"  # raw value, or a {{secrets.KEY}} reference
status = "active"                 # active | paused; create defaults to active
# Optional: toleranceSeconds, deduplicationEnabled, deduplicationWindowMs,
# secretGracePeriodMs, [webhook.allowedIps] cidrs, [webhook.inputMapping]
```

`signingSecret` is required unless `verificationScheme = "none"`. It may be a raw value or a `{{secrets.KEY}}` reference (uppercase-led key, no spaces — the tighter form) into the app secret store; the reference is stored and re-emitted verbatim (never returned as a real secret) and resolved server-side against the app secrets immediately before HMAC verification. The referenced key must exist at create/update or the write is rejected. It **fails closed**: if the reference can't be resolved at delivery (secret deleted, or encryption key unset) the request is rejected with `401` (`rejectionReason: secret_unresolved`) rather than verifying against the literal reference; an unresolvable grace-window `previousSigningSecret` reference is dropped.

Receive endpoint: `POST /app/{appId}/webhook/{webhookKey}`. The platform verifies the signature per `verificationScheme`, then starts `workflowKey` with the event payload as input; `inputMapping` (e.g. `"data.object"`) extracts a nested path first. A webhook-triggered workflow is `runAs: "system"`, so what stops a client from starting it directly with a crafted payload is the system-invocation gate (members get a 403) plus the signature verification — not `accessRule`, which a system workflow doesn't evaluate on the trigger (see [Access control](#access-control)).

CLI: `primitive webhooks list | get | create | update | delete | rotate-secret | test | events <webhook-key>` — `events` lists recent deliveries (accepted / rejected / duplicate / `workflow_not_active`). `active` and `paused` are the settable statuses (`create` and `update` reject anything else, and `create` defaults to `active`); `archived` is reserved for delete and only appears on read — `list --status` filters by `active`, `paused`, or `archived`.

`webhooks test <webhook-key> --payload '<json>'` delivers and signs that JSON object exactly as given — pass the event directly (e.g. `'{"type":"charge.succeeded","data":{"id":"ch_1"}}'`), not wrapped in `{"payload": ...}`. Omitting `--payload` (or passing a non-object) sends a canned `{"type":"webhook.test", ...}` ping instead.

A `workflow_not_active` delivery means the bound workflow was draft or archived when the event arrived: the request is acked with HTTP 202 `{ received: true }` (so the sender doesn't retry) but the workflow is **not dispatched**. Activate the workflow and resend — these events are excluded from deduplication, so the resend isn't dropped as a duplicate. Binding a webhook to a not-yet-active workflow succeeds and returns a non-blocking `warning` carrying `WORKFLOW_NOT_ACTIVE`.

## Cron triggers

A cron trigger fires a workflow on a clock schedule. It is one of the ways to invoke a workflow — a clock instead of a `workflows.start()` call or an inbound webhook. The trigger points at a workflow by `workflowKey`; the trigger itself runs no code.

**Decision rule:** trigger is a clock → cron trigger. Trigger is a user action or external webhook → `workflows.start()` or an inbound webhook, NOT cron.

### Critical rules

1. **Cron triggers fire workflows, not arbitrary code.** Create the workflow first (`status = "active"` with an active config/revision), then point the trigger at it via `workflowKey`.
2. **Set `requiresClientApply = false` on the target workflow.** Cron-triggered workflows almost always want this — otherwise the run sits in `apply_pending` forever because no client is listening.
3. **Set an IANA `timezone` whenever the schedule has a user-visible hour.** `0 9 * * *` in UTC is 2am in Los Angeles.
4. **`overlapPolicy` is `"skip"` (default) or `"allow"`.** There is no `"queue"`. `"skip"` checks whether the previous run is still active and increments `skippedCount`; `"allow"` always fires. Use `"allow"` only when each firing is independent and idempotent.
5. **Per-app cap is 50 cron triggers.**

### Creating (TOML / `primitive sync`)

```toml
# config/cron-triggers/nightly-digest.toml
[cronTrigger]
key = "nightly-digest"
displayName = "Nightly digest email"
cron = "0 9 * * *"
timezone = "America/Los_Angeles"
workflowKey = "send-digest"
overlapPolicy = "skip"
state = "active"

# Optional: static input passed to the workflow on every fire
[rootInput]
digestType = "daily"
environment = "production"

# Optional: dynamic input. `{{now}}` is replaced with the fire-time ISO string.
[inputMapping]
firedAt = "{{now}}"
```

```bash
primitive sync push
```

The TOML key `key` maps to the API field `triggerKey`. The field name is `cron` (not `schedule`).

### Creating (client / CLI)

{{ example: scheduling/cron-create }}

Via the CLI:

```bash
primitive cron-triggers create \
  --key nightly-digest \
  --name "Nightly Digest" \
  --cron "0 9 * * *" \
  --workflow-key send-digest \
  --timezone "America/Los_Angeles" \
  --overlap-policy skip       # skip (default) | allow

# With per-firing input passed to the workflow:
primitive cron-triggers create \
  --key nightly-digest --name "Nightly Digest" --cron "0 9 * * *" \
  --workflow-key send-digest \
  --input '{"reportType":"daily","priority":"high"}'

# Or map fields from the trigger firing context into the workflow input:
primitive cron-triggers create \
  --key nightly-digest --name "Nightly Digest" --cron "0 9 * * *" \
  --workflow-key send-digest \
  --input-mapping '{"runId":"$triggerId","at":"$firedAt"}'
```

The CLI flags are `--key`, `--name`, `--cron`, `--workflow-key`, `--overlap-policy`, `--timezone`, `--input`, `--input-mapping` — NOT `--schedule` or `--workflow`. `--input` is a literal payload; `--input-mapping` projects fields from the firing context into the workflow input. Both are also accepted by `cron-triggers update`.

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
| `state` | Update only | `"active"` / `"paused"` / `"archived"`. Set on update; create always starts `"active"`. |

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

Invalid expressions are rejected at create time. If the cron later becomes unparseable (rare), the trigger transitions to `state: "error_paused"` with `lastError` set; alarms stop until you call `resume`.

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
primitive cron-triggers pause <trigger-id>
primitive cron-triggers resume <trigger-id>
primitive cron-triggers delete <trigger-id>
```

### Querying cron-triggered runs

There is no `triggerSource` filter on `workflows.listRuns()`. Cron runs are identifiable by their `contextDocId` starting with `cron:` and `meta.source === "cron"`:

{{ example: scheduling/cron-list-runs }}

### Managing triggers (client)

{{ example: scheduling/cron-manage }}

### Debugging cron triggers

Trigger states:

- `active` — scheduled, alarm armed.
- `paused` — manual pause; alarm cancelled until `resume`.
- `error_paused` — set automatically when a fire fails hard (the target workflow is **not found**, or a binding/runtime error), when the cron expression becomes unparseable, or after 50 consecutive skips against a **not-active** target. `lastError` is populated (carrying `WORKFLOW_NOT_FOUND` or `WORKFLOW_NOT_ACTIVE`). Call `resume` to clear and reschedule.
- `archived` — soft-deleted; never returns from list.

A target workflow that is draft or archived (**not active**) does NOT error-pause on the first fire: the fire is skipped and rescheduled, `lastError` is set to `WORKFLOW_NOT_ACTIVE`, and `consecutiveNotActiveCount` increments. It auto-recovers once the workflow is activated (the counter resets and `lastError` clears); only after 50 consecutive not-active skips does it escalate to `error_paused`. A target that is **not found**, by contrast, error-pauses immediately.

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

`primitive sync push` pushes this flag. You can also set it via the CLI:

```bash
primitive workflows create --from-file workflow.toml --requires-client-apply false
primitive workflows update <workflow-id> --requires-client-apply false
```

## Synchronous invocation

Opt a workflow into synchronous invocation by setting `syncCallable = true` in the TOML (pushed by `primitive sync push`) or via `primitive workflows update --sync-callable true`. Once set, a caller can invoke the workflow and receive the final envelope in a single round-trip — useful for short, latency-sensitive tasks like input validation, enrichment lookups, or webhook handlers that must reply with a result.

Server-side constraints on a `syncCallable` workflow:

- **Step kinds are restricted.** The server validates step kinds against a sync-compatible list when the flag is set (or when steps are pushed against a sync-callable workflow). Long-running or suspending kinds (`event.wait`, `delay` over the timeout) reject at save time with `Workflow contains sync-incompatible steps`.
- **Timeout.** The invocation timeout defaults to 5s (`timeoutMs`) and is capped server-side at 30s; anything above the ceiling clamps silently. Exceeding it resolves with `status: "timeout"` in the envelope. The timeout ends the **call**, not the execution: work is not interrupted at the boundary, so side effects from steps still in flight (database writes, emails) can land after the envelope resolves. The run record is finalized `terminated` before the response returns — the envelope's `run.status` reads `terminated`, and the record never transitions to `completed`. `getStatus` targets durable `start()` runs; a sync run's final state comes back in the envelope's `run` and appears in `workflows.listRuns()`. Step-run records are salvaged only when execution settles within a short grace window at the boundary — a timed-out run typically persists no step runs. Recovery: never poll for completion after a timeout; treat the run as incomplete, reconcile against actual resource state with idempotent cleanup or retry, and re-invoke with a **new** `runKey` — the sync path has no `forceRerun`, so the same `runKey` returns the terminated run without re-executing. When the final outcome must be knowable, prefer `start()`: a durable run's record reaches a real terminal status pollable via `getStatus`.
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

A workflow needs `status = "active"` AND one of (active configuration | published revision) before clients can run it.

| Status | Active config/revision? | Client can run? | CLI preview? |
|---|---|---|---|
| `draft` | either | no | yes |
| `active` | yes (required) | yes | yes |
| `archived` | – | no | no |

Setting `status = "active"` without an active config or revision returns: `Cannot activate workflow without a configuration`.

`primitive workflows delete <id>` (the default) is a **soft delete**: it archives the workflow and keeps its `workflowKey` reserved, so recreating a workflow under the same key fails with `workflowKey already exists (held by an archived workflow <id>)`. Free the key for reuse with a hard delete — `primitive workflows delete <id> --hard` — which also cascades to the workflow's configurations, runs, step runs, and test cases.

`primitive sync push` creates a default configuration automatically when a workflow is first created and updates it on subsequent pushes. Each push of `[[steps]]` updates the active configuration's steps in place. `primitive workflows update <id> --from-file <toml>` pushes a revised body (metadata + steps) to an existing workflow the same way without the full sync directory — it overwrites the active configuration in place and is live immediately; explicit `update` flags override the TOML's metadata.

### Configurations vs revisions

- **Configurations** (recommended): named, mutable groupings of steps. One is `activeConfigId`. Created automatically on first sync push.
- **Revisions**: immutable snapshots created via `primitive workflows publish`. Prefer configurations.

## CLI

```bash
# Sync (recommended for everything; sync dir auto-resolves to .primitive/sync/<env>/<appId>/)
primitive sync init
primitive sync pull
primitive sync diff
primitive sync push --dry-run
primitive sync push

# Workflow CRUD (when not using sync)
primitive workflows list [--status active] [--json]
primitive workflows get <workflow-id>
primitive workflows create --from-file workflow.toml [--requires-client-apply false]
primitive workflows update <workflow-id> --status active
primitive workflows update <workflow-id> --from-file workflow.toml   # push a revised body (metadata + steps) in place, live immediately; explicit flags above override TOML values
primitive workflows delete <workflow-id>           # archive (soft delete; the workflowKey stays reserved)
primitive workflows delete <workflow-id> --hard --yes   # also frees the workflowKey for reuse

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

# Draft / publish (revision flow)
primitive workflows draft update <workflow-id> --from-file workflow.toml
primitive workflows publish <workflow-id>

# Configurations
primitive workflows configs list <workflow-id>
primitive workflows configs get <workflow-id> <config-id>
primitive workflows configs create <workflow-id> --name "production" --from-file workflow.toml
primitive workflows configs update <workflow-id> <config-id> --from-file workflow.toml
primitive workflows configs activate <workflow-id> <config-id>
primitive workflows configs duplicate <workflow-id> <config-id> --name "experiment-v2"
primitive workflows configs archive <workflow-id> <config-id>

# Run inspection
primitive workflows runs list <workflow-id> [--status pending|running|completed|failed]
primitive workflows runs status <workflow-id> <run-id>
primitive workflows runs steps <workflow-id> <run-id>
primitive workflows runs step-detail <workflow-id> <run-id> <step-id>
primitive workflows runs error <workflow-id> <run-id>
primitive workflows runs failures <workflow-id>

# Test cases
primitive workflows tests list <workflow-id>
primitive workflows tests create <workflow-id> --name "Basic" --vars '{"x":1}' --contains '["expected"]'
primitive workflows tests run <workflow-id> <test-case-id>
primitive workflows tests run-all <workflow-id>

# Analytics (CLI-side)
primitive workflows analytics overview --days 7
primitive workflows analytics top --days 7
```

`runs list` includes a `DELAY` column (`queueDelayMs`); `runs status` includes "Execution started" (`executionStartedAt`), "Queue delay" (`queueDelayMs`), and "Create call" (`createCallDurationMs`, the wall-clock time of the run's underlying create call) lines. `executionStartedAt` and `queueDelayMs` are `null` while the run is still queued.

All inspection commands take `--json`.

Every `<workflow-id>` argument above — CRUD, draft/publish, configs, runs — accepts the workflow **key** as well as the ULID. The value is tried as an id first (the id wins when both exist), then as a key lookup scoped to the app and case-insensitive; an unknown value exits non-zero with `Workflow not found for id or key "<arg>"`. The `workflows tests` commands take the ULID. Don't generalize to cron triggers: `cron-triggers` ops take the `triggerId`, never the trigger key.

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

A run that aborts during **setup** — before any step executes (resolving its config/revision, loading steps, or validating `input` against `inputSchema`) — is still marked `failed` with the real error message, and records one synthetic step with id `__setup__` and kind `setup`. So `runs steps` is never empty and `runs error` always names the failure, even when no author-defined step ran.

### Reusable step fragments

A workflow TOML can pull shared `[[steps]]` blocks out of fragment files via `include`:

```toml
# config/workflows/onboard.toml
[workflow]
key = "onboard"
name = "Onboard new user"
status = "active"

include = ["common-validation", "common-audit"]

[[steps]]
id = "create-account"
kind = "database.mutate"
# ...
```

Fragments live at `<workflowDir>/../workflow-fragments/<name>.toml` and contain only `[[steps]]` tables (no `[workflow]` block, no further `include`). The CLI expands `include` references before `sync push` — the server only ever stores the flattened step list. Step ids must be unique across the expanded set; collisions are reported with both source locations. Use `primitive workflows expand <file>` to print the expanded result.

### Operation `$params` validation

When a workflow references a registered database operation via `database.query`/`mutate`/etc., the CLI checks that every `$params.X` substitution in the operation's TOML maps to a declared `[[operations.params]]` entry at `sync push` time. A typo like `$params.proectId` is reported with the file path and line number of the offending `[[operations]]` block instead of silently no-opping at runtime.

## Client SDK

Start a workflow, check its status, and list recent runs:

{{ example: workflows/workflow-start }}

Full options and result shapes for these calls:

{{ example: workflows/workflow-run-options }}

Inspect the per-step debug records of a run:

{{ example: workflows/workflow-list-step-runs }}

The step records carry `{ stepRunId, runId, stepKind, status, input, output, error, startedAt, endedAt, ... }`.

`runKey` is scoped as `${contextDocId}#${runKey}`. Calling `start` with an existing `runKey` returns `{ existing: true, ... }` unless `forceRerun: true`.

{{#lang ts}}
### Typed invocation (codegen)

Generate typed invocation wrappers from each workflow's `inputSchema`/`outputSchema`:

```bash
primitive workflows codegen [workflow-key] [-o <dir>] [--check] [--json]
```

The command reads `workflows/*.toml` from the auto-resolved sync directory (`--sync-dir <path>` overrides; `--app <app-id>` disambiguates when several apps are synced — with more than one match and neither flag, it errors rather than guessing) and emits **one `<key>.generated.ts` per workflow** (default output `<sync-dir>/workflows/generated/`; stale generated files for removed workflows are cleaned up on full runs). Reserved `__internal.*` workflows are skipped; malformed schema JSON fails the command. A single `[workflow-key]` argument matches the TOML file stem and generates just that file.

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
{{/lang}}

{{#lang swift}}
### Typed invocation (codegen)

Generate typed invocation wrappers from each workflow's `inputSchema`/`outputSchema`:

```bash
primitive workflows codegen --lang swift [workflow-key] [-o <dir>] [--check] [--json]
```

The command reads `workflows/*.toml` from the auto-resolved sync directory (`--sync-dir <path>` overrides; `--app <app-id>` disambiguates when several apps are synced — with more than one match and neither flag, it errors rather than guessing) and emits **one `<key>.generated.swift` per workflow** (default output `<sync-dir>/workflows/generated/`; stale generated files for removed workflows are cleaned up on full runs). Reserved `__internal.*` workflows are skipped; malformed schema JSON fails the command. A single `[workflow-key]` argument matches the TOML file stem and generates just that file.

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
{{/lang}}

## WebSocket events

Requires an active WebSocket (e.g., from `client.documents.open(docId)`).

{{ example: workflows/workflow-events }}

The `workflowStatus` event uses `"completed"`. The `getStatus` method returns `"complete"`. These differ in the SDK — handle both if your code shares logic.

{{#lang swift}}
Subscribe through `client.events.on(_:)`; hold the returned `EventSubscription` for as long as you want the handler live.
{{/lang}}

{{#lang ts}}
### `workflows.waitFor`

`workflows.waitFor(runId, options?)` wraps the `workflowStatus` frame in a single awaitable call instead of a manual event listener:

```ts
const { status, output, error } = await client.workflows.waitFor(runId, { timeoutMs: 60000 });
```

Signature:

```ts
workflows.waitFor(runId: string, options?: { timeoutMs?: number }): Promise<{
  status: "completed" | "failed" | "terminated" | "apply_pending" | "apply_claimed";
  output?: any;
  error?: string;   // set when status === "failed"
}>
```

- Event-driven: subscribes to the `workflowStatus` frame, plus one reconcile fetch immediately after subscribing to close the started-before-subscribed race; re-runs that reconcile on WS reconnect. No polling.
- Resolves on every terminal state, including `"failed"` — a failing run does not reject.
- Rejects only on timeout (`JsBaoError` code `WORKFLOW_WAIT_TIMEOUT`; default `timeoutMs` is 900000 / 15 minutes, pass `0` or `Infinity` to disable) or when `runId` doesn't resolve to a run (`NOT_FOUND`).
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
primitive workflows update <workflow-id> --requires-client-apply false
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
- `forEach`: 200 items default, override with `maxItems`.
- `collect`: 10 pages / 10000 items default.
- `analytics.query`: 50 queries per run.
- `analytics.write`: 25 events per step.
- `workflow.call`: max depth 10; circular calls throw.
- Cron triggers: 50 per app.
- `meta` payload: 1KB.
- App secrets via templates/CEL: read-only, available as `secrets.KEY`.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `404 WORKFLOW_NOT_FOUND` on start/run-sync | No workflow with that key in this app context |
| `409 WORKFLOW_NOT_ACTIVE` ("Workflow '<key>' is not active") | The workflow exists but its status is draft/archived — activate it with `status = "active"` |
| `400 WORKFLOW_NO_ACTIVE_CONFIG` | Active status but no active config/revision — run `primitive sync push` |
| `Cannot activate workflow without a configuration` | Push steps before activating |
| `Workflow has no draft or configuration to preview` | First `primitive sync push` to create a config, or use `primitive workflows draft update` |
| Run stuck in `apply_pending` | `requiresClientApply = true` but no client running `define()` for that key. Set to `false` for server-only workflows. |
| `existing: true` on `start()` | A run with the same `(contextDocId, runKey)` already exists. Use a different `runKey` or `forceRerun: true`. |
| `Step "X": required field "Y" is empty` | A template expression resolved to `""`. Check the path; consider `strict = true` to surface earlier. |
| `runIf expression failed` | CEL syntax error or unknown identifier in `runIf`. Don't use `{{ }}` inside. |

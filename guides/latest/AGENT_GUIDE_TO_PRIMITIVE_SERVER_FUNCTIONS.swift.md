# Agent Guide to Primitive Server Functions

A **server function** is TypeScript authored in the app's config tree and pushed with `primitive config push`. It runs on the platform — never on the device, whatever language the app's clients are written in — **as the app itself**: reviewed code acting with the app's authority, behind an `access` gate that decides who may call it. It answers over HTTP. It is config-as-code beside [prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md) and [integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md): `functions/<key>.toml` states the gate, the entry point and the few capabilities that still need declaring; the code sits next to it; one push builds and ships both as one immutable version.

Two modes, one key:

| Mode | `mode` | What an invocation answers | Run row |
|---|---|---|---|
| **Request** (default) | `"request"` | The result envelope, inside the request | None — nothing to poll or clean up |
| **Task** (the durable one) | `"task"` | A run id, immediately; the run can sleep for hours or days | One per run, polled by run id |

`durable = true` / `durable = false` is the accepted alias for `mode = "task"` / `mode = "request"` through the transition; a file carrying both must have them agree. `config pull` writes a file back in whichever spelling it was authored, so migrating the line to `mode` is an ordinary edit that makes a new version on the next push.

## The config file

```bash
primitive config create function <key>   # scaffolds functions/<key>.toml
primitive config fields function         # every key, its type and default
primitive config push --only function/<key>
```

```toml
# functions/greet.toml
[function]
key = "greet"                       # required; URL-path-safe; one namespace per app (below)
description = "Says hello"
entry = "functions/greet/index.ts"  # required; relative to the config tree root
access = "true"                     # required CEL gate — see The access gate
mode = "request"                    # "task" → a run id instead of an inline answer; the default is "request"
capabilities = []                   # integration:, secret:, and the high-blast strings — see Capabilities

[function.inputSchema]              # JSON Schema, native TOML tables. Input is validated
type = "object"                     # and coerced against it before the handler runs.
required = ["name"]

[function.inputSchema.properties.name]
type = "string"

[function.outputSchema]             # The return value is validated against it; a mismatch
type = "object"                     # answers status "failed".

[function.limits]                   # cpuMs, subRequests, ratePerMinute — may LOWER a platform ceiling,
cpuMs = 2000                        # never raise one. timeoutMs is a request field, not a config key.
```

- **One key namespace per app.** A key is unique per app, case-insensitively; pushing a function onto a key another object already holds fails naming the holder. The key doubles as the URL path segment that addresses the function.
- **Availability is server-owned.** `status` is not a TOML key: `config pull` never writes it and `config push` never sends it. A push to a *disabled* function updates its code and leaves it out of service. Enable, disable and archive are CLI verbs (see Operating).
- Triggers are declared in the same file — see Triggers.

## The handler

```ts
// functions/greet/index.ts
import { defineFunction } from "primitive-functions";
import { salute } from "./greeting.js";

// Keyed: `input` and the return are typed from functions/greet.toml's schemas
// (the declaration `config push` generates — see Codegen). A key this tree
// does not declare is a compile error; one that names another file's function
// is refused at push.
export default defineFunction("greet", async (input, ctx, step) => {
  return { message: salute(input.name) };
});

// Unkeyed, with types you write yourself — still valid:
// export default defineFunction(async (input: { name: string }, ctx, step) => …);
```

| Argument | What it is |
|---|---|
| `input` | The request's `rootInput`, validated and coerced against `inputSchema` when one is declared. On a trigger fire: the verified delivery body (webhook), the entry's `rootInput` (cron), or the committed changes (database-change — see Triggers). |
| `ctx` | The invocation context: `user`, `trigger`, `api`, `db`, `integrations`, `prompts`, `secret`, `configVar`, `users`, `connections`, `channels` — each documented below. |
| `step` | Present in **both** modes. In a task run it is the live engine step (see Task functions). In a request invocation it is a passthrough: `step.do` runs its body, `sleep`/`sleepUntil` resolve at once, and `waitForEvent` throws `STEP_NOT_AVAILABLE` — so a task-authored body can be exercised as a request. |

- The return value is JSON-serialized as the envelope's `output` and validated against `outputSchema`. A throw settles the invocation with `status: "failed"`, `errorCode: "FUNCTION_THREW"` and the thrown message.
- `defineFunction` is typing sugar over the same contract: `export default async function (input, ctx, step) {}` is equivalent.
- `primitive-functions` is the one import the bundle leaves external: the platform supplies it at invoke time, so a function always runs against the SDK the platform is running. The npm package is a stub whose every export throws — a function only ever runs inside a push.

### `ctx.user` and `ctx.trigger`

| Door | `ctx.trigger` | `ctx.user` |
|---|---|---|
| HTTP invoke | `{ kind: "http" }` | `{ userId, email }` — the caller |
| Webhook fire | `{ kind: "webhook", webhookKey, webhookId, externalEventId }` | `null` |
| Cron fire | `{ kind: "cron", name, triggerId, scheduledFor }` — plus `manual: true` on a diagnostic fire | `null` |
| Database-change fire | `{ kind: "database", databaseId, databaseType }` — the changes ride in `input` | `null` |
| `workflow.call` from a parent workflow | `{ kind: "workflow", workflowKey, runId, stepId }` — the parent's key, run and step (see Migrating a workflow tree to functions) | The member who started the parent run, or `null` |

`ctx.trigger` is typed as that union (`FunctionTrigger`), narrowed on `kind`, with every arm's fields typed — the declaration is held identical to what the fire paths build. `ctx.user` is typed `FunctionUser | null`: a function that a trigger can fire reads it as `ctx.user?.userId`. Whichever door the invocation came through, the code runs with the same authority — the app's (see Execution identity). The caller is a **name** the code may use for attribution and scoping; it is not a second principal.

## Pushing

`config push` builds the bundle with esbuild, inlines every npm dependency, and stores the built bundle **together with** the authored TOML bytes and every source file it read.

- Every push that changes anything creates a new **immutable version** and repoints the function at it. "Anything" is the whole envelope: a comment-only or line-ending-only edit makes a new version, and `config pull` on a fresh clone hands back the bytes you wrote.
- **Relative and absolute imports must stay inside the config tree.** An entry or import that resolves outside it — through a symlink included — is refused at push, so a push can never upload a file the tree does not contain.
- **npm packages are imported by name and inlined.** They resolve through Node like any other import; `node_modules` content is never stored as source or written back by `config pull`.
- A missing entry file or a build error names the file and the message, and nothing is pushed for that function — the rest of the tree still applies.
- The built bundle is capped at **5 MB**.
- **Push runs your module scope.** Collecting the manifest (see What a function touches) means building the bundle a second time with `primitive-functions` replaced by a recording stub and executing it in a separate Node process — scrubbed environment, the config tree as working directory, hard timeout. It is the trust you extend by running a project's own build; know it before pushing a tree you did not write. A push whose collection fails is refused naming the error: the manifest is part of the version's identity.
- Capabilities are validated before anything is applied: a malformed string, an `integration:` key the app has no active integration for, an unknown family, or a **retired** grant or key (below) refuses the push by name. Nothing about databases is checked at push, because nothing about databases is declared any more.

## Invoking

```
POST /app/{appId}/api/functions/{key}
```

Any signed-in member may call it, subject to the function's `access` gate. Every request field is optional:

| Field | Meaning |
|---|---|
| `rootInput` | The value the handler receives as `input`. |
| `contextDocId` | A document the invocation is about. On a task start it defaults to the caller's root document and is required when there is none. |
| `meta` | Caller metadata, at most 1 KB encoded. |
| `timeoutMs` | Wall-clock budget. Default 5 000 ms; clamps silently to the 30 000 ms ceiling. Ignored on a task start. |
| `runKey` | Task only — idempotency key (see Task functions). On the request path it is validated and otherwise ignored, since there is no run row to deduplicate against. |

**The request envelope** — always HTTP `200`, because the call reached the function and the function has an answer:

```json
{ "status": "completed", "output": { "message": "hello Ada" } }
```

| `status` | Meaning |
|---|---|
| `completed` | The handler returned; `output` is its value. |
| `failed` | The handler threw (`FUNCTION_THREW`), returned a value JSON cannot carry (`OUTPUT_NOT_SERIALIZABLE`), exported no callable default (`FUNCTION_NO_HANDLER`), returned output `outputSchema` refuses (`OUTPUT_SCHEMA_VIOLATION`), or returned over 1 MiB (`OUTPUT_TOO_LARGE`). `errorCode` says which; `error` carries the message. |
| `timeout` | The handler did not finish inside the budget. |

An envelope from a handler that ran to an answer — `completed`, or `failed` from a throw or an `outputSchema` violation — also carries `limits`, the resolved ceilings that invocation ran under: your `[function.limits]` clamped element-wise to the platform's own values:

```json
{ "status": "completed", "output": { "message": "hello Ada" }, "limits": { "cpuMs": 50, "subRequests": 64, "ratePerMinute": 1200 } }
```

Read it when you need to tell what the platform resolved from what you declared: asking for more than a ceiling gets the platform's value silently, and this is where that is visible. A `timeout`, an `OUTPUT_TOO_LARGE` failure and the HTTP errors below carry no `limits`.

**HTTP errors** mean the platform refused before your code ran:

| HTTP | `errorCode` | Cause |
|---|---|---|
| `400` | `INVALID_META` | `meta` over 1 KB encoded. |
| `400` | `INVALID_RUN_KEY` | A `runKey` carrying a reserved character sequence. |
| `400` | — | `rootInput` failed `inputSchema`; `error` is `Input schema validation failed` and `details` lists the violations. |
| `400` | `CONTEXT_DOC_REQUIRED` | A task start with no `contextDocId` from a caller who has no root document. |
| `403` | `FUNCTION_ACCESS_DENIED` | The `access` gate denied the caller. |
| `403` | `FUNCTION_DISABLED` | The function is disabled — seen only by callers the gate admits. |
| `404` | `FUNCTION_NOT_FOUND` | Unknown key, an archived function, or a hard delete in progress. |
| `409` | `FUNCTION_NOT_PUSHED` | A function with no pushed code. |
| `409` | `FUNCTION_DELETE_IN_PROGRESS` | A task start that raced a hard delete. |
| `429` | `FUNCTION_RATE_LIMITED` | Past the per-function rate ceiling; `Retry-After` is set. |
| `503` | `FUNCTION_RATE_UNAVAILABLE` | The rate limiter could not be consulted; retryable. The platform refuses rather than run unmetered. |

From the client, `client.functions.invoke` returns the envelope typed by its output generic; `input` travels as `rootInput`:

```swift
  struct Greeting: Decodable, Sendable { let message: String }

  let result: FunctionResult<Greeting> = try await client.functions.invoke(
    "greet",
    input: ["name": "Ada"],   // travels as rootInput
    timeout: 10               // seconds; default 5, ceiling 30
  )

  if result.status == "completed" {
    print(result.output?.message ?? "") // "hello Ada"
  } else {
    // "failed" — the handler threw or its output failed outputSchema — or "timeout"
    print(result.status, result.error ?? "")
  }
```

`functions.invoke` on a task function — and `functions.start` on a request function — throws `FUNCTION_MODE_MISMATCH` ("is a task function" / "is a request function") before you are handed a shape the method's types do not describe. The server decides the mode from the pushed config.


`invoke<Input, Output>(_:input:…)` decodes `output` into a caller-chosen `Decodable` (`FunctionResult<Output>`) — there is no untyped entry point; a dynamic caller names the witness (`input: nil as JSONValue?`, bound `as FunctionResult<JSONValue>`). A settled invocation never throws — read `status` — while a platform refusal is an `HttpError` carrying the server's `errorCode` on `serverCode`. The typed `input` is sent as whatever JSON value it encodes to (object, array or scalar), so a function whose schema declares a non-object root receives exactly that.

### The access gate

`access` is a CEL expression (see [Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md)) evaluated on **every call**, and it fails closed:

- A caller it denies gets `403 FUNCTION_ACCESS_DENIED`.
- A function with **no** `access` expression denies everyone — which is why push refuses to ship code for one.
- App owners and admins bypass it.
- It is checked **before** any answer about the function's operational state, so a rejected caller cannot tell a disabled function from a never-pushed one from a live one, and cannot map the app's functions by probing keys.
- A caller the gate rejects, or a request `inputSchema` refuses, consumes no rate-limit budget.

**The gate is the authorization.** Once a caller is through it, the code runs with the app's authority — the gate is the one place a reviewer decides *who* may make this code act. Write it as tightly as the function deserves: `user.role == 'admin'` on a function that deletes, `true` on one that reads its caller's own rows.

## Execution identity

Function code acts as the system. Every platform call a function makes — `ctx.api`, `ctx.db`, the helpers — carries the **app's** authority, not the caller's: the same thing the app's owner could do over the REST API. There is no mode to choose and no `runAs` key; a config still carrying one is refused at push.

| Door | Authority of every platform call | `ctx.user` |
|---|---|---|
| HTTP invoke | The app's | The caller — a name for attribution and scoping |
| Webhook, cron, database-change fire | The app's | `null` |

Three consequences worth stating:

- **A function is not a proxy for the caller.** `ctx.api.documents.get` inside a function reads any document of the app, whoever called; `ctx.api.users.setRole` sets a role a member could not. What a member may reach *through* a function is decided by the `access` gate and by what the code chooses to do with `ctx.user`.
- **Scope reads yourself.** A read the caller should see only part of is filtered by the code — `filter: { owner: ctx.user!.userId }` (non-null on an HTTP invoke), or a `$caller` registered query (see Registered queries). The platform does not narrow a function's read to its caller.
- **A revoked caller is not cut off mid-invocation.** The gate ran when the call arrived; a role change during the run does not change what the running code may do. Re-check `ctx.user` inside the code where that matters.

The `function.invoke` analytics event is still attributed to the caller (see Recording), and the run row of a trigger fire records the app's system principal.

## `ctx.api` — the platform from inside a function

`ctx.api` is a typed client for the platform's own API, generated from its OpenAPI document. The namespaces a function may call mirror the operation ids: `analytics`, `blobBuckets`, `channels`, `collections`, `configVars`, `connections`, `databases`, `documents`, `email`, `gemini`, `groups`, `integrations`, `llm`, `locks`, `notifications`, `prompts`, `resourceMetadata`, `secrets`, `users`. Each method takes one options object carrying the operation's path parameters, query parameters, headers (under their wire names) and `body`; a missing required parameter is a compile error.

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { documentId: string }, ctx) => {
  const doc = await ctx.api.documents.get({ documentId: input.documentId });
  return { title: doc.title, readBy: ctx.user?.userId ?? null };
});
```

A sandbox has **no network access**. Its only route out is the platform, and every operation the gateway publishes is reachable **with no declaration**, as the app, under one of three rules:

- **Admitted as the system** — the ordinary families: `documents`, `databases`, `users`, `groups`, `collections`, `notifications`, `locks`, `blobBuckets`, `resourceMetadata`, `analytics`, `channels`, `email`, `configVars`, `llm`, `gemini`. No grant, no mode. A direct-LLM route still meets the app's `directLlmEnabled` spend gate, because that is the app's own setting.
- **Admitted under a keyed capability** — `integration:<key>` for `ctx.integrations.call`, `secret:<NAME>` for `ctx.secret`. These are declared because each names an outside credential the reviewer should see in the file.
- **Admitted under a high-blast capability** — eleven exact strings for the operations that destroy or re-own (see Capabilities). Declared so that "this function can delete a database" is a line in a reviewed TOML, not a surprise in a log.

Not published to functions at all: `functions.*` (a function cannot invoke or manage a function), `iterations.*`, `databaseTypeConfigs.*`, `integrations.proxy` and `prompts.execute` (the member-facing doors; use `ctx.integrations.call` and `ctx.prompts.run`), `databases.executeOperation` / `executeBatch` / `importBulk` (the operation door — see Where operations are going) and `databases.adminData.*` (duplicates of `databases.records.*`). A method that type-checks can still be refused here.

A refused call rejects with an error carrying `status` and `errorCode`:

| `errorCode` | Meaning |
|---|---|
| `FUNCTION_EGRESS_DENIED` | `fetch` to any host other than the platform. |
| `FUNCTION_ROUTE_NOT_ALLOWED` | An operation the gateway does not publish to functions — the list above. |
| `FUNCTION_INTEGRATION_GRANT_MISSING` | An integration call the capabilities do not cover; the message names the string that would. |
| `FUNCTION_SECRET_GRANT_MISSING` | A secret read the capabilities do not cover. |
| `FUNCTION_HIGH_BLAST_GRANT_MISSING` | A high-blast operation the capabilities do not cover; the message names the string. |
| `FUNCTION_SECRET_NOT_FOUND` / `FUNCTION_VAR_NOT_FOUND` | This environment holds no value of that name. |
| `FUNCTION_CHANNEL_NAME_INVALID` | A channel name outside the grammar (`400`). |
| `FUNCTION_CHANNEL_GRANTEE_REQUIRED` | `ctx.channels.authorize` from a trigger fire without an explicit `userId` — there is no caller to default to. |
| `FUNCTION_CHANNEL_GRANTEE_NOT_FOUND` | The `userId` a grant names is not a member of this app (`404`) — the same answer as a user that never existed. |
| `FUNCTION_SEND_TARGET_NOT_FOUND` | The user is not a member of this app, or the connection is not one of this app's. |
| `FUNCTION_SEND_PAYLOAD_TOO_LARGE` | A send payload over 64 KiB; nothing was delivered. |
| `FUNCTION_SEND_PAYLOAD_INVALID` | A send payload JSON cannot serialize. |

Every platform call carries the invocation's credential, which expires at the invocation's deadline — a function that outlives its budget has late calls denied and side effects that never land (see Ceilings).

## Capabilities

`capabilities` is an array of exact strings. No wildcards, no case folding; a repeated string is the same string. Only three kinds exist:

| Kind | Strings | Unlocks |
|---|---|---|
| Integration | `integration:<key>` | `ctx.integrations.call(key, …)` — checked at push against an active integration |
| Secret | `secret:<NAME>` | `ctx.secret(NAME)` — grammar-checked at push; the value is provisioned per environment |
| High-blast | `databases:create`, `databases:delete`, `databases:transferOwnership`, `databases:addManager`, `databases:revokePermission`, `databases:grantGroupPermission`, `databases:revokeGroupPermission`, `users:remove`, `users:setRole`, `blobBuckets:createBucket`, `blobBuckets:deleteBucket` | The operation of the same name on `ctx.api`, which is otherwise refused `FUNCTION_HIGH_BLAST_GRANT_MISSING` |

**Nothing else is declared.** Database models, prompts, config vars, channels, sends, email and analytics writes are admitted with no declaration, because the code acts as the system. A file carrying `database:<type>/<model>:read|write`, `prompt:<key>`, `var:<NAME>`, `channel:<name>`, `users:send`, `connections:send`, `email:send` or `analytics:writeForUser` is refused at push with a sentence naming the string and what stays — delete the line and push again. A `runAs` or `unscopedReads` key is refused on the same terms.

The platform reads capabilities from the TOML file you review in a pull request, so what a reviewer sees and what the platform enforces are one statement.

### What a function touches is documented, not declared

Push scans the built bundle and records a **manifest** on the version: the model names it saw in `.model("…")` and in the records calls' `modelName`/`model` keys, the `ctx.api` families and helpers it uses, the queries it registered, and whether a model name is resolved at run time. `primitive functions get <function-id>` prints it under `Manifest (documentation, not authorization)`. It is what it says: a reviewer's and an operator's map of what a version reaches, refreshed on every push, and never consulted when a call is admitted. A model missing from the manifest is a scan the code defeated (a name built from a string), not a model the function cannot read.

### Where operations are going

Client-callable database **operations** — the CEL-gated verbs a member runs through `POST …/databases/{id}/operations/{key}` — were built to give a *member* a bounded, reviewed way to write. A function does not need one: it is already reviewed code acting as the app, so it writes through `ctx.db` directly and expresses the bound in code. That is why `databases.executeOperation`, `executeBatch` and `importBulk` are not published to functions (a prepared operation is still reachable through `databases.runOperation`, which never consults the operation's access rule), and why the direction is retirement: the client-callable operation endpoints go in project phase 5 and the concept in phase 6, and an operation that filtered on a user-supplied `$params.userId` becomes a `defineQuery` with a `$caller` parameter under the function's `access` gate. Nothing member-facing changes here; Migrating a workflow tree to functions (below) is the full account, including how a parent workflow keeps working while its leaves move.

## Database records

### On the app's authority

```toml
# functions/monthly-report.toml
[function]
key = "monthly-report"
entry = "functions/monthly-report/index.ts"
access = "user.role == 'admin'"
```

Nothing about databases is declared. A function reads and writes **every database of the app** — every type, every model, every row — as the app, through `ctx.db` and `ctx.api.databases.records.*`. There is no read grant, no write grant and no `unscopedReads` flag; the `access` gate above is where a reviewer decides who may make this code run, and the code is where the reviewer reads what it does with the rows.

- **Scope reads in the code.** A read the caller should see only part of is filtered — `query({ filter: { owner: ctx.user!.userId } })` — or bound through a `$caller` registered query (below). The platform does not narrow a function's read to its caller.
- **Every row is the app's.** The database's own permission rows (owner, managers, groups) govern members calling over REST; they do not narrow a function. Creating, deleting or re-owning a database is the one exception, gated by a high-blast capability (see Capabilities).
- `ctx.db(databaseId, "<type>")` still takes the type key, for typing: the declarations `config push` writes name each type's models.
- The typed `databaseId` a function reaches must belong to this app; another app's id answers as one that does not exist.

### The typed handle

`ctx.db(databaseId, "<type>")` returns the models that database type declares, typed from its schema. The second argument is the database type's key; once `config push` has written the declarations (see Codegen below), a model the type does not declare is a compile error. Before the first push the handle is untyped.

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { databaseId: string }, ctx) => {
  const orders = ctx.db(input.databaseId, "orders").model("Order");

  await orders.save({ id: "o-1", data: { label: "first", total: 10 } });
  const open = await orders.query({ filter: { status: "open" } });
  const total = await orders.count({});

  // Several writes to ONE model in one request; the handle supplies the model.
  await orders.batch({
    operations: [
      { op: "save", id: "o-2", data: { label: "second", total: 4 } },
      { op: "delete", id: "o-3" },
    ],
    atomic: true,
  });

  return { open: open.items.length, total };
});
```

| Handle | Surface |
|---|---|
| `ctx.db(id, type).model(name)` | `query`, `count`, `aggregate`, `save`, `batch` — the model is bound per call |
| `ctx.db(id, type)` | `databaseId`, `databaseType`, `batch` (spans models — each item names its own) |
| `ctx.api.databases.records.*` | The untyped form of the same five operations, plus the rest of the records surface (`get`, `delete`, `find`, `listSchemas`, …), as the app |

Filters and query options are ordinary objects — see [Databases](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) for the cursor parameter. `include` works as it does over REST.

**One paged envelope.** Every paged read the SDK exposes answers `{ items, hasMore, nextCursor? }` — the database handle, the document handle, `ctx.users.list`, and the raw `ctx.api.databases.records.query`, which adds a `prevCursor` and is the only surface that has one. Shared paging code is therefore written once:

```ts
const drain = async (page) => {
  const rows = [];
  while (true) {
    rows.push(...page.items);
    if (!page.hasMore) return rows;
    page = await next(page.nextCursor);
  }
};
```

*Migration.* The database handle used to answer `{ data, … }`. A function reading `.data` off `ctx.db(...).model(...).query()` or off `ctx.api.databases.records.query()` moves to `.items`; the generated declarations make the old key a compile error rather than an `undefined` that pages to nothing. The REST route's own response is unchanged and still carries `data`, so nothing outside a function moves.

**Documents page and narrow the same way.** `ctx.api.documents.records.query({ documentId, model, filter, limit, cursor })` and `documents.records.count({ documentId, model, filter })` take the same `filter` object a database model does — the client JSON-encodes it into the query string; a malformed one answers `400` — and answer the `{ items, hasMore, nextCursor }` page the route and the CLI do, so a function that walks a large model pages it rather than reading it whole.

**Codegen.** `config push` writes the generated files into the tree's `functions/` directory beside your sources, and none is part of what gets pushed:

| File | What it carries |
|---|---|
| `functions/primitive-db-types.d.ts` | The database types behind `ctx.db`, model by model |
| `functions/primitive-functions.d.ts` | The `primitive-functions` module's own types |
| `functions/primitive-function-types.d.ts` | `<Key>Input` / `<Key>Output` for every function, from its `inputSchema` and `outputSchema`, and the `FunctionSchemas` augmentation that types the keyed `defineFunction("<key>", handler)` |
| `functions/generated/<key>.generated.ts` | A typed **client** invoker per function (imports `js-bao-wss-client`): a request function's has exactly `invoke`, a task function's exactly `start`, `getStatus`, `waitFor`, `terminate` — the mode is fixed at generation time, so the wrong verb is a compile error. `input` is required iff the schema rejects `{}`; the task `waitFor` output is `<Key>Output`. `-o <dir>` moves them |
| `functions/tsconfig.json` | Scaffolded once, then yours; push keeps the `primitive-functions` path mapping, the declarations in the program and the `generated/**` exclusion, and restores those if they go missing |

```ts
import { greet } from "../primitive/dev/functions/generated/greet.generated";
const answered = await greet(client).invoke({ input: { name: "Ada" } }); // output typed GreetOutput
```

`config diff` reports when a schema change has left any of them stale; the next push refreshes them.

```bash
primitive functions codegen --check   # CI: non-zero when the generated files are stale; no --check regenerates
primitive functions codegen -o src/generated/functions   # the invokers, somewhere your client code imports from
```

**Swift codegen.** `primitive functions codegen --lang swift` emits one
`<key>.generated.swift` per function instead of the TypeScript artifacts:
`<Key>Input` / `<Key>Output` as `Codable` types from the declared schemas, and a
`<Key>Function` invoker struct reached through a `<key>(client)` factory, bound
over the generic `client.functions` overloads. The mode fixes the verbs exactly
as it does in TypeScript — a request function's invoker has only `invoke`, a
task function's only `start` / `getStatus` / `waitFor` / `terminate` — so the
wrong verb is a compile error. A function with no declared schema gets
`typealias <Key>Input = JSONValue` rather than an empty struct; a key that is
not a legal Swift identifier is mangled (`123-job` → `_123JobFunction`,
`_123Job(client)`) while the call still passes the original key; and a nullable
root input (`type: ["string","null"]`) is a required `String?` parameter whose
`nil` reaches the handler as an explicit JSON `null`.

```bash
primitive functions codegen --lang swift -o Sources/App/Functions/Generated
```

```swift
let answered = try await greet(client).invoke(input: GreetInput(name: "Ada"))
answered.output?.greeting  // String?
```

Swift mode writes no TypeScript artifact and no tsconfig wiring; `--check` names
the stale files and its hint carries `--lang swift`. Both languages may share
`functions/generated/` — each generator sweeps only the files carrying its own
banner. The Swift app template runs this from `scripts/codegen.sh` on every
build path, so the committed invokers never drift.

**Push typechecks what it ships.** The same `config push` that writes those declarations compiles your sources against them — the tree's own `functions/tsconfig.json`, so it is the program your editor loads — and refuses the function with the compiler's own diagnostics when they do not hold. esbuild erases types, so a bundle that builds proves nothing: a handler that dereferences `ctx.user` without checking it — written before that field became nullable — builds cleanly and throws on the request path. The refusal is per function (the rest of the tree still lands), `--dry-run` reports the same diagnostics without shipping, and reproducing one by hand is `tsc -p functions/tsconfig.json --noEmit`. `primitive config push --no-typecheck` skips it. A CI job that runs `primitive config push --dry-run` therefore fails on broken function code with no extra wiring.

### Registered queries

`defineQuery` and `defineMutation` name a read or write over the handle so it can be called from several places, validated once, scoped to its caller by the code, and — when the platform can verify what it touched — cached.

```ts
import { defineQuery, defineMutation } from "primitive-functions";

const myOrders = defineQuery("myOrders", {
  models: ["Order"],
  params: {
    owner: { caller: true },                    // $caller: injected, never supplied
    since: { type: "string", optional: true },
    limit: { type: "number", default: 25 },
  },
  cache: { ttlMs: 30_000 },
  run: async (db, params) => {
    const answer = await db.model("Order").query({ limit: params.limit });
    return answer.items.filter((row) => row.owner === params.owner);
  },
});

// A list parameter — the shape behind every $in filter; params.ids is string[] in run.
const ordersByIds = defineQuery("ordersByIds", {
  models: ["Order"],
  params: { ids: { type: "array", items: { type: "string" } } },
  run: (db, params) => db.model("Order").query({ filter: { id: { $in: params.ids } } }),
});

const addOrder = defineMutation("addOrder", {
  models: ["Order"],
  params: { owner: { caller: true }, label: { type: "string" } },
  run: (db, params) =>
    db.model("Order").save({ data: { owner: params.owner, label: params.label } }),
});

export default async function (input: { databaseId: string }, ctx) {
  const db = ctx.db(input.databaseId, "orders");
  await addOrder(db, { label: "new" });
  return { orders: await myOrders(db, { limit: 10 }) };
}
```

`run`'s `params` is **typed from the declaration** (a `const` type parameter): a scalar as its scalar, `$caller` as a string, an array as `T[]`, an `optional` parameter as an optional key; an undeclared name inside `run` is a compile error, and an explicit `TParams` generic still overrides.

| `params.<name>` key | Meaning |
|---|---|
| `type` | `"string"`, `"number"`, `"boolean"`, `"any"` (default), `"$caller"`, or `"array"` — validated and coerced |
| `items` | For `type: "array"`: `{ type: "string" \| "number" \| "boolean" \| "any" }`, each element coerced and validated like a scalar; absent accepts any array |
| `caller: true` | The `$caller` binding: the platform injects the invoking user; a call that supplies it is refused by name. Not combinable with `array` |
| `optional` | May be omitted |
| `default` | Applied when omitted — **coerced to the declared type at registration** (`"25"` → `25`, `["1"]` → `[1]` for numeric items); a default that cannot be coerced is refused at registration, which `config push` reports |

- **Register at module scope.** A registration inside the handler works for direct calls but is invisible to `config push`, so it is absent from the manifest.
- Names are unique; registering one twice throws. An undeclared parameter is refused, not dropped.
- A `$caller` query throws in an invocation with no initiating user — a webhook, cron or database-change fire; use a query with an explicit parameter there.
- `$caller` is a convenience for the code, not a platform guarantee: the query body is what filters by it. An unfiltered `query()` inside a `$caller` query still reads every row.
- **Caching is off unless the run is verified.** `cache.ttlMs` is honored only when every model the body touched was declared in `models` and nothing was written. TTL is capped at one minute (`MAX_QUERY_CACHE_TTL_MS`). A cache entry belongs to one query, one `databaseId` and resolved type, and one parameter set after injection — two databases of the same type never share an answer, and two callers never see each other's rows. A write through the handle drops the entries that read the written model. The cache lives inside one warm sandbox instance: it bounds staleness, it does not replicate. A mutation is never cached.

### The typed document handle

`ctx.doc(documentId).model("<Model>")` is `ctx.db`'s shape for a document's record models: a thin binding over `ctx.api.documents.records.*` that states the document and the model once and answers the routes' own shapes.

```ts
const orders = ctx.doc(input.documentId).model("Order");
const page = await orders.query({ filter: { status: "open" }, limit: 50, cursor: undefined }); // { items, hasMore, nextCursor? }
const { count } = await orders.count({ filter: { status: "closed" } });               // { count }
const { record } = await orders.save({ id: "o-1", data: { label: "x" } });          // { record }
await orders.patch("o-1", { data: { label: "y" } });                                  // { record }
const { deleted } = await orders.delete("o-1");                                       // { deleted }
```

The rows are typed from the project's `models/models.toml` (fallback: the web client's `src/models/models.toml`), rendered by `config push` into `functions/primitive-document-types.d.ts`: an undeclared model is a compile error, a declared field has its type, and a project with no schema file gets the open form. It confers nothing — the same authority as every other call — and another app's document id is the uniform not-found.

## Calling an integration

A function cannot `fetch` the internet. Under an `integration:<key>` capability, `ctx.integrations.call` asks the platform to make the request on the function's behalf, so the integration's `{{secrets.*}}` and `{{vars.*}}` templates are resolved server-side and the credential never enters the sandbox.

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { orderId: string }, ctx) => {
  const response = await ctx.integrations.call("stripe", {
    method: "POST",
    path: "/v1/refunds",
    body: { payment_intent: input.orderId },
  });
  return { status: response.status, refund: response.body };
});
```

Request: `method` (must be in `allowedMethods`; defaults to the integration's own), `path` (relative to `baseUrl`), `query`, `headers`, `body`, `form` (form-urlencoded; not combinable with `body`), `bodyMode` (`json` | `raw` | `multipart`). Result: `status`, `headers`, `body`, `durationMs`, `traceId`, and `errorCode` when the platform or the upstream refused.

- The path is resolved and normalized before it is checked against `allowedPaths`, so `//elsewhere/x` and `/allowed/../blocked` are refused (`DISALLOWED_PATH`), as is a method outside `allowedMethods` (`DISALLOWED_METHOD`).
- **Redirects are re-checked before they are followed**: a hop is followed only if it stays on the integration's exact origin, lands on an allowed path and uses an allowed method; otherwise the `3xx` comes back unfollowed. Five hops maximum.
- The upstream request is bounded by the integration's `timeoutMs` or the invocation's remaining time, whichever is smaller — including while the response body is still arriving. Past it, `UPSTREAM_TIMEOUT`.
- The integration's `accessRule` is **not** consulted — it governs members calling from a client; the capability in the reviewed TOML is what admits the function. A disabled or deleted integration refuses everyone (`INTEGRATION_INACTIVE`).
- Every call is recorded on the integration's own log with the function's key: `primitive integrations logs <integration-id>`.

See [Integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md) for declaring one.

## Running a prompt

`ctx.prompts.run(key, { variables, modelOverride, configId })` — no capability; the code is the app — runs one of the app's saved prompts through the same path the member endpoint uses and answers the same envelope: `success`, `output`, `error`, `metrics`, `configId` (which version ran).

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { text: string }, ctx) => {
  const result = await ctx.prompts.run("summarize", { variables: { text: input.text } });
  if (!result.success) throw new Error(result.error ?? "prompt failed");
  return { summary: result.output };
});
```

- `configId` pins a config **of that prompt**; any other answers `PROMPT_NO_CONFIG`.
- The prompt's `accessRule` is not consulted; the function acts as the app. An inactive or archived prompt refuses everyone.
- The model call is bounded by the invocation's remaining time. A provider call that passes the deadline answers `PROMPT_UPSTREAM_TIMEOUT` — catch it separately from a model failure: the budget ran out, not the model.

See [Prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md).

## Secrets and config vars

A `secret:<NAME>` capability lets a function read one app secret by name; a config var needs no declaration:

```ts
const region = await ctx.configVar("REGION");
const token = await ctx.secret("PARTNER_TOKEN");
```

```toml
capabilities = ["secret:PARTNER_TOKEN"]
```

- **Prefer an integration when the secret is a credential for one** — there the value is injected into the outbound request and never enters the sandbox. `ctx.secret` is for what an integration cannot express, and it stays declared for that reason: a reviewer sees which secrets can enter the sandbox.
- Secrets and config vars are provisioned per environment, out of band from the push. A secret capability is checked for spelling at push and for existence at call time: a name with no value in this environment answers `FUNCTION_SECRET_NOT_FOUND` / `FUNCTION_VAR_NOT_FOUND`, naming the key.
- The namespaces are separate: `ctx.secret("X")` never reads a config var named `X`, and vice versa.
- Secret values come through a short stale-while-revalidate cache: a secret rotated just now may serve its previous value for up to about a minute.
- **`ctx.configVar` is read once per version.** A config var cannot change within a deployment, so its value is cached keyed on the version's config id — across calls in one invocation and across invocations on a warm sandbox — and a re-push starts cold. A miss is never cached (provision the var, and the next read sees it), and `ctx.secret` is never cached.

See [App Secrets](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md).

## Sending to a connected client

`ctx.users.send` and `ctx.connections.send` push a message to a client that is connected **right now**; no capability is declared. The recipient receives a `direct.message` frame carrying the payload verbatim plus the sending function's key.

```ts
const result = await ctx.users.send(input.userId, { kind: "order-ready", orderId: "o-1" });
return { delivered: result.connections, truncated: result.truncated };
```

- **Presence is not guaranteed and there is no durable record.** A user with no connected client answers `{ connections: 0, truncated: false }` — a success. A client that was offline never receives the frame later; use a notification when the message must survive being missed.
- **One socket, one frame.** A client with several documents open is one connection, delivered once.
- **Fan-out is bounded** at 64 unique connections per user, and the lookup behind it is bounded too; past either, the send delivers what it found and answers `truncated: true` — "there may be more", not an error.
- **Payloads are capped at 64 KiB** serialized (`FUNCTION_SEND_PAYLOAD_TOO_LARGE`; nothing delivered).
- `ctx.connections.send(connectionId, payload)` addresses one connection. An id from another app answers exactly as an id that never existed (`FUNCTION_SEND_TARGET_NOT_FOUND`).

On the client, listen for the `directMessage` event:

```swift
  for await event in client.stream(for: DirectMessageEvent.self) {
    // event.payload (JSONValue?) is what the function passed;
    // event.functionKey is who sent it; event.sentAt is when.
    print(event.functionKey, event.sentAt)
    if let message = event.payload?["message"]?.stringValue {
      print(message)
    }
  }
```

`DirectMessageEvent` carries `payload` (a `JSONValue?`, `nil` when the function sent none), `functionKey` and `sentAt`. It needs no subscription and no grant — the frame arrives on the app socket — and a frame that arrives with no listener is inert.

## Channels

A direct send addresses a user or a connection the function already knows; a **channel** addresses whoever is listening to a named topic — an order's status, a game table, a document's presence lane — without the function knowing who that is. Two calls make one, and neither needs a capability:

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { orderId: string }, ctx) => {
  // The grant names the CALLER (an HTTP invocation's default grantee), so it
  // is not a credential anybody else can present.
  const { grant, expiresAt } = await ctx.channels.authorize(`orders:${input.orderId}`);
  return { grant, expiresAt };
});
```

```ts
// Any function — a database-change fire included — publishes to the topic.
const result = await ctx.channels.publish(`orders:${orderId}`, { status: "shipped" });
// result: { connections, truncated }
```

| Call | Contract |
|---|---|
| `ctx.channels.authorize(channel, { userId?, ttlSeconds? })` → `{ channel, grant, expiresAt }` | Mints a signed, short-lived grant naming this app, this channel and one user. `userId` defaults to the HTTP caller; a trigger fire has no caller and must pass it (`FUNCTION_CHANNEL_GRANTEE_REQUIRED` otherwise). `ttlSeconds` defaults to 300 and is clamped to the 900 ceiling — asking for more gets the ceiling, not an error. `expiresAt` is epoch milliseconds. There is no app-wide grant. |
| `ctx.channels.publish(channel, payload)` → `{ connections, truncated }` | Delivers a `channel.message` frame, payload verbatim, to every connection holding a live membership. Fan-out stops at 500 connections per channel (`truncated: true`). Zero connections is a success — nobody was listening. |

- **Channel names** are one or more `[A-Za-z0-9_-]` segments joined by `:`, at most 200 characters — `orders`, `orders:42`, `orders:42:chat`. Outside the grammar: `400 FUNCTION_CHANNEL_NAME_INVALID`, decided before anything else.
- **Expiry is the only revocation, and it ends membership.** A grant is a token, not a row: past `expiresAt` the server stops delivering even though the socket stays open. Renewal is another authorize and another subscribe — same channel, same socket, new expiry — and replaces the membership rather than adding one.
- **Refusals are uniform.** An expired, tampered, cross-app, cross-user or wrong-channel grant all get the same error frame with the same message; the frame echoes the channel, so a client with several subscribes in flight knows which one failed, but never why. A grant is refused at subscribe time if the socket's user is not the one it names.
- **Reconnect re-issues every held subscribe** with its stored grant; a grant that expired meanwhile is refused, and only that channel's registration is dropped.

On the client, present the grant on the socket it already has:

```swift
  struct Grant: Decodable, Sendable { let grant: String; let expiresAt: Int }

  // 1. Ask the authorizing function for a grant; it names this caller and one channel.
  let result: FunctionResult<Grant> = try await client.functions.invoke(
    "order-room",
    input: ["orderId": orderId]
  )
  guard result.status == "completed", let output = result.output else { return }

  // 2. Present the grant on the socket the client already has.
  let subscription = try await client.subscribeToChannel(
    "orders:\(orderId)",
    grant: output.grant
  )

  // Later: leave. Idempotent — the same as client.unsubscribeFromChannel(channel).
  subscription.unsubscribe()
```

- `subscribeToChannel(channel, grant)` resolves with `channel`, `expiresAt` and an `unsubscribe` closure on the server's ack for that channel, and fails on the uniform refusal. Calling it again with a fresh grant renews the membership; two calls for the same channel run one after the other, so a renewal is answered on its own merits.
- `unsubscribeFromChannel(channel)` — or the subscription's `unsubscribe` — is idempotent and safe on a closed socket; a subscribe still in flight for that channel is cancelled and its call fails.
- `channelMessage` fires for every held membership, carrying `channel`, `payload`, `functionKey` and `sentAt` — a live frame with no durable record, so a client that was offline never receives it later.
- `channelSubscribeFailed` (`channel`, `message`) announces a refusal nothing was waiting on — the reconnect case, where a grant that expired while the socket was down is refused: invoke the authorizing function again and re-subscribe. A refusal that answers a `subscribeToChannel` call surfaces from that call instead, so a failure is never announced twice.


The join fails by throwing `JsBaoError(.channelSubscribeFailed)` from the `async` call, whose `details` name the `channel`; `channelMessage` / `channelSubscribeFailed` arrive as typed events on `client.stream(for:)`. An empty channel or grant throws `.invalidArgument` before any frame is sent.

## Sending email

No capability — a cron-fired function sending a digest is the point of it:

```ts
await ctx.api.email.send({
  body: {
    toUserId: input.userId,
    subject: "Your receipt",
    htmlBody: `<p>Thanks, ${input.name}.</p>`,
    textBody: `Thanks, ${input.name}.`,
    variables: { appName: "Acme" },
  },
});
```

- Template mode and inline mode are exclusive, as are `to` and `toUserId`; a `toUserId` must be a member of this app.
- The route, `POST /app/{appId}/api/emails/send`, is reachable **only from a function** (`403 FUNCTION_ROUTE_FUNCTION_ONLY` otherwise) — sending as the app is the function's declared authority, not a member role.
- The app has **one hourly email budget**; passing it answers `429 EMAIL_RATE_LIMITED`.

## Writing analytics for a user

`ctx.api.analytics.writeForUser({ body: { userId, action, feature, … } })` records an event attributed to a **subject** user rather than to whoever is running. Function-only route (`POST /app/{appId}/api/analytics/write-for-user`; `403 FUNCTION_ROUTE_FUNCTION_ONLY` otherwise). The subject must be a member of this app; `appId` and `userId` are reserved — a `context` object of your own cannot rewrite whose activity the event is. See [Analytics](AGENT_GUIDE_TO_PRIMITIVE_ANALYTICS.md).

## Helpers: `pMap`, `ulid`, `stepPolicy`

```ts
import { pMap, ulid, stepPolicy } from "primitive-functions";

// Bounded concurrency. Results in input order; the first rejection wins and
// nothing new starts after it. An unbounded Promise.all over a page of rows
// spends the sandbox's whole subrequest budget in one line.
const results = await pMap(rows, (row) => handle(row), { concurrency: 4 });

// Mint ids INSIDE step.do in a task run (see Step discipline).
const id = await step.do("mint-id", async () => ulid());

// The platform's timeout/retry policy for the non-idempotent families:
// email, llm, gemini, database. `retries: 0` on email and the AI families is
// deliberate — a retried timeout would send twice or bill twice.
await step.do("send-receipt", stepPolicy.email, () =>
  ctx.api.email.send({ body: { to, subject, htmlBody } })
);
```

`stepPolicy.<family>` is `{ retries: { limit, delay, backoff }, timeout }` — a default you apply, never one the engine forces on your own `step.do` calls. `PMAP_DEFAULT_CONCURRENCY` (8) is the bound `pMap` applies when none is given.

## Task functions

```toml
# functions/order-sync.toml
[function]
key = "order-sync"
entry = "functions/order-sync/index.ts"
access = "true"
mode = "task"
```

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { orderId: string }, ctx, step) => {
  const charged = await step.do("charge", async () =>
    ctx.api.documents.get({ documentId: input.orderId })
  );

  // Hibernate. Nothing runs — and nothing is billed — while it waits.
  await step.sleep("cooling-off", "24 hours");

  return { orderId: charged.documentId, confirmedAt: Date.now() };
});
```

| `step` method | Meaning |
|---|---|
| `do(name, body)` | Run `body` once; its return value is stored and replayed on every later resume |
| `do(name, config, body)` | The same with a retry/timeout config — `stepPolicy.<family>` is one |
| `sleep(name, duration)` | Hibernate for a duration (`"24 hours"`, or milliseconds) |
| `sleepUntil(name, timestamp)` | Hibernate until a `Date` or epoch-ms timestamp |
| `waitForEvent(name, options)` | Hibernate until an event arrives |

### Starting and polling a run

The same route, `POST /app/{appId}/api/functions/{key}`, answers `201` with a run envelope instead of a result:

```json
{ "runId": "01J…", "runKey": "01J…", "instanceId": "app-doc-01J…", "status": "running" }
```

- `runKey` makes the start idempotent per `(caller, contextDocId, runKey)`: a repeat answers the run that already exists with `existing: true`, never a second run — and that holds for a run that FAILED, so retrying takes a new key with the same input, not the key the failure holds. It defaults to the run id.
- `timeoutMs` is accepted and ignored: the durable engine owns how long a task run may take, and a per-request budget cannot hold across a `step.sleep`.
- Poll the run id at `GET /app/{appId}/api/workflows/runs/{runId}/status`; `terminate` takes the function key and the run key.
- A run broadcasts nothing over the WebSocket. Poll.
- A caller over HTTP starts a task run, and so does `ctx.functions.start` from inside any running function — a request function, a task function's slice, or a webhook- or cron-fired invocation. The callee must be a task function (a request callee answers `FUNCTION_MODE_MISMATCH`; import its module instead), its `access` gate is NOT consulted (yours was the authorization), every run in the tree is keyed by the original external initiator, and nesting stops at 4 levels with `FUNCTION_NEST_DEPTH_EXCEEDED`.

### Reporting back

A task run has three ways to deliver its result, and they compose:

| Channel | How | Survives the client being away? |
|---|---|---|
| The run's `output` | Return a value; it lands on the run status once the run completes | Yes — poll the run id any time |
| Server state | Write database rows, documents or notifications as the app, inside `step.do` | Yes |
| A live push | `ctx.users.send` (see Sending to a connected client) | No — a client that is offline never receives it; pair it with state or a notification |

```swift
  struct Confirmation: Decodable, Sendable { let confirmedAt: Int }

  let run = try await client.functions.start(
    "order-sync",
    input: ["orderId": orderId],
    runKey: "order-\(orderId)" // a repeat replays the existing run (existing == true)
  )

  // Poll once…
  let now = try await client.functions.getStatus(runId: run.runId)
  print(now.status)

  // …or wait for a terminal state.
  let settled = try await client.functions.waitFor(
    runId: run.runId,
    as: Confirmation.self,
    options: WaitForWorkflowOptions(timeout: 60) // throws workflowWaitTimeout past it
  )
  print(settled.status, settled.output?.confirmedAt ?? 0)

  // The function key goes where a workflow key would.
  _ = try await client.functions.terminate(
    FunctionRunRef(functionKey: "order-sync", runKey: run.runKey)
  )
```

`functions.waitFor` polls with a backing-off interval (0.4 s doubling to 5 s) and throws a wait-timeout past its budget (default 15 minutes; a non-positive value waits unbounded). `getStatus` and `terminate` take the run id and the function key respectively; a repeated `runKey` on `start` replays the run that already exists.

The typed `waitFor(runId:as:)` decodes `output` into a caller-chosen `Decodable` (`WaitForResult<Output>`); a failed run resolves with `status == "failed"` rather than throwing, while a 404 or a `missing` run throws `.notFound`. `terminate` takes a `FunctionRunRef`.

### Step discipline

The engine re-executes the handler from the top on every resume and replays each completed `step.do` from its **stored result**.

- **Put every side effect inside `step.do`.** A charge, an email or a write outside a step runs again on every resume; inside one it runs exactly once.
- **Do not branch on anything nondeterministic between steps.** `Date.now()`, `Math.random()`, `ulid()` or a fresh read at the top level can take a different path after a resume, and the engine then looks for steps that are not there. Read such values inside a step so the value is stored with it.
- **Your code is pinned; the platform's is not.** A run is pinned to the content hash of the version that started it — pushing mid-run does not change what a running run executes, and new runs get the new code. The `primitive-functions` SDK and the runtime around your handler are whatever is deployed at resume time; treat the SDK as a stable interface, not a frozen artifact.
- **A task function may carry cron entries**, and each fire starts a run. A webhook or a database watch requires request mode: both are waiting inside a request for an answer, so they hand the work over with `ctx.functions.start`.

### Budgets in a task run

The ceilings (see Ceilings) apply **per engine invocation, not per run**. A run is a series of *slices*: the engine runs the handler from the top until it returns or reaches a `sleep`, `sleepUntil` or `waitForEvent` that has not completed, then hibernates; the next wake is a new slice, with completed steps replayed from storage.

| Ceiling | In a task run |
|---|---|
| Wall clock | Continuous, up to 12 hours from the slice's start. Each slice is minted with a 10-minute credential and **every step begins with at least 5 minutes of it**. A single step may use up to 10 minutes of wall clock. When a step finishes with less than 5 minutes left, or is about to start with less than 5 minutes left (a cleanup step after the handler catches a failed step, or after code outside steps spent the slice), the platform REFRESHES the credential through the gateway and the clock rolls on with no pause. The guarantee holds at every `step.do` boundary; code outside steps still spends the clock it runs in, and the next `step.do` entry refreshes it. A handler that never calls `step.do` gets no refresh and dies at the 10-minute slice deadline. The request's `timeoutMs` is ignored; the run as a whole is bounded only by the 12-hour ceiling. |
| `cpuMs` (5 minutes per task slice) | Per slice, not per step — a refreshed slice keeps counting; the paid plan's per-invocation ceiling. A step that exhausts it is killed and the run FAILS (it does not recover from the kill today), so a step's own body must fit in 5 minutes of CPU, and a run whose steps together need more must end the slice between them (a `step.sleep` past the engine's five-minute grace period; the wake starts a fresh CPU budget). |
| `subRequests` (10 000 per task slice) | Counted across the slice; does NOT reset while it refreshes. Replayed steps spend nothing — their bodies do not run — but any platform call *outside* a step re-spends on every wake. |
| `ratePerMinute` (1 200) | Per **start**. A task start reserves a slot and returns it if the start is refused; resumes take none. |
| Output (1 MiB) and `outputSchema` | Once, on the final return value, at settlement. |

When a refresh is refused — past the 12-hour ceiling, more than once a minute (one per 60-second window on the platform's epoch-aligned clock), or during a rate-limiter or engine outage — the slice falls back to a pause of a little over 5 minutes: the engine hibernates only for a sleep longer than its own five-minute grace period, so that is what the fallback sleeps, and the next wake mints a fresh 10-minute slice with a fresh 12-hour ceiling. So a refused refresh costs a little over five minutes, once. Two budgets do NOT refresh: the subrequest count yields instead — when it passes half the resolved limit (5 000 of 10 000, or half a lower configured value), the next `step.do` boundary takes the fallback yield rather than refreshing; and CPU, which is 5 minutes of active CPU per slice — a step that exhausts it is killed, and the run fails rather than recovering. Steps running in parallel (`Promise.all` over two chains) that reach a fallback yield together: it waits for every step in flight to finish before it sleeps, because the engine hibernates only when nothing is running. Because the fallback is a hibernation, what the run printed before it is not recoverable on the log record (see Debugging a failing function). On a local development server the engine never hibernates: a fallback there is a pause with no fresh credential, and a task run is bounded by one slice.

Consequences: `step.sleep` between chunks is for waiting, not for budget — chunk a long job by steps and sleep when there is something to wait for; keep the code outside steps minimal, since it re-executes on every wake; and a refresh does not count toward the engine's maximum number of steps per run, and neither does a fallback yield (`step.sleep` and `step.sleepUntil` are excluded from that allowance; only your `step.do` calls count).

**Observing a slice.** `GET /app/{appId}/api/workflows/runs/{runId}/status` carries a `slice` block beside `status` and `run` for a task run that has one — `sliceId`, `startedAt`, `ceilingAt` (the 12-hour bound), `refreshCount`, `lastRefreshAt`, `settledAt` and `settledStatus`; a request invocation and a workflow run carry no `slice` key. `primitive functions runs` prints the count in a `REFRESHES` column, blank for a request run.

`functions.getStatus` answers the block as `slice` on `WorkflowStatusResult`, and so do `workflows.getStatus` and `workflows.terminate` — a function run IS a run row, so the same run read through either surface answers the same block. The typed overloads carry it too. A block the client cannot read is dropped rather than published with a hole in it, so `slice` being absent means "no record to show you", never "this run never refreshed".

### Deleting a function with live runs

A hard delete destroys the stored bundles a sleeping run reloads when it wakes, so it is refused — including through `primitive config push --prune` — while any run has not settled; the refusal names the live runs. Wait for them or terminate them, then retry. To stop new runs meanwhile, **archive** the function: an archived function keeps its bundles, so an in-flight run finishes on the version it pinned. While a hard delete is running, an invoke answers `404` and a task start already in flight answers `409 FUNCTION_DELETE_IN_PROGRESS`.

## Triggers

A function's config block can declare one inbound **webhook**, up to ten **cron** schedules and up to five **database-change** watches. The platform creates and operates whatever they need. Which modes a kind takes is per kind: a **cron** entry runs on a request or task function, and on a task function each fire starts a run; a **webhook** and a **database-change** watch require request mode, because the delivery or the write is waiting for the function's answer inside the request — those hand slow work to a task function with `ctx.functions.start`.

```toml
# functions/stripe-events.toml
[function]
key = "stripe-events"
entry = "functions/stripe-events/index.ts"
access = "true"

[function.triggers.webhook]           # one per function; answers on the function's own key
verificationScheme = "stripe"          # required; "none" is refused
signingSecret = "{{secrets.STRIPE_WEBHOOK_SECRET}}"

[[function.triggers.cron]]
name = "nightly"                       # required; half of the trigger's identity
cron = "0 3 * * *"                     # five-field expression
timezone = "UTC"                       # IANA; default UTC
rootInput = { mode = "full" }          # what the function receives as input

[[function.triggers.cron]]
name = "hourly-sweep"
cron = "0 * * * *"
overlapPolicy = "skip"                 # skip (default) | allow
```

```ts
export default async function (input, ctx) {
  if (ctx.trigger.kind === "webhook") return { received: input.type };
  return { swept: ctx.trigger.name }; // kind === "cron"
}
```

**Webhook.** The receiver is `POST /app/{appId}/webhook/{functionKey}`. Verification, replay protection and a delivery log are built in. Accepted keys: `verificationScheme`, `signingSecret`, `toleranceSeconds`, `deduplicationEnabled`, `deduplicationWindowMs`, `maxBodyBytes`, `secretGracePeriodMs`, and a `[function.triggers.webhook.verification]` table for scheme-specific settings (a Discord public key, a JWT's JWKS, …). A verified delivery runs the function and answers `200 {"received": true}` **whatever the function did** — the run row records the outcome. A delivery that cannot run right now (function disabled, rate ceiling hit) answers `202` with the reason in the delivery log and no dedup key stored, so the provider's redelivery runs once the condition clears. A rotation via `primitive webhooks rotate-secret <webhook-id>` survives later pushes: a push touches the signing secret only when the TOML value itself changed. Push refuses `mode = "task"` beside a webhook block: for work that outlasts the delivery, verify and acknowledge in the request function and `ctx.functions.start` a task function, keying the start on the provider's event id or a body id so a redelivery replays rather than starting a second run.

**Cron.** Runs on a request or a task function; on a task function each fire starts a run (`ctx.user` is null, `ctx.trigger.kind` is `"cron"`, and the run polls and terminates on the ordinary run routes). With `overlapPolicy = "skip"`, a fire that arrives while the previous run is still going is counted as a skip; for a task run that question goes to the **engine**, so a sleeping run counts as live, and so does one the platform cannot be asked about. `overlapPolicy = "allow"` starts one run per fire. Prefer a request function calling `ctx.functions.start` when you need a run key or an input computed at fire time.

**Database-change.** A `[[function.triggers.database]]` block watches a database **type**; every write request that commits something to any database of that type runs the function once, with the committed changes:

```toml
[[function.triggers.database]]
type = "orders"                        # a database type the app already has; up to five per function
```

```ts
// Watches `orders`, writes `shipments` — a type it does NOT watch (see Loops).
const SHIPMENTS_DB = "…";
export default async function (input, ctx) {
  if (ctx.trigger.kind === "database") {
    // ctx.trigger.databaseId / ctx.trigger.databaseType say which database
    const shipments = ctx.db(SHIPMENTS_DB, "shipments").model("Shipment");
    for (const change of input.changes) {
      // change.op, change.modelName, change.id, change.data, change.previousData
      if (change.op === "save" && change.previousData === null) {
        await shipments.save({ data: { orderId: change.id } });
      }
      await ctx.channels.publish(`orders:${change.id}`, { op: change.op });
    }
  }
  return { seen: input.totalChanges };
}
```

`input` is `{ databaseId, databaseType, changes, totalChanges, truncated }`. The delivery contract:

- **After the write's answer, from committed operations only.** Dispatched off the response path once the write's answer is decided; a database batch is partial-success inside an HTTP `200`, and only the operations that landed reach `changes`. The write's caller never waits for the function and never sees its outcome.
- **Once per write request, not once per row.** A batch of fifty saves is one fire carrying fifty changes, in the order the database committed them; `changes` is capped at 50 and `totalChanges`/`truncated` describe the rest.
- **No ordering promise across write requests.** Fires from separate write requests may run concurrently or arrive out of order; a function that needs a row's current state reads it.
- **No retry, no queue.** A fire that fails, times out or is refused by the rate ceiling leaves the write's response as it was and is not retried.
- **Bounded fanout, as enforced.** Up to 25 functions may watch one type — `config push` refuses the 26th watcher of a type — and a fire reaches every watcher the platform accepted.
- **The rows are in the payload.** Every change carries `op`, `modelName`, `id`, `data` and the pre-image in `previousData` (`null` when there was no row before), whether or not anyone is subscribed to the database.
- **No run row.** A fire leaves an invocation metric and, on failure, an application error event — nothing in `primitive functions runs`.
- **Request mode only, with no caller.** `mode = "task"` and a database trigger are refused together at push; `ctx.user` is `null` inside the fire. Hand slow work to a task function with `ctx.functions.start`.

**Loops.** A fire's own writes fire triggers too. Every database-change fire carries an **origin chain** — the ids of the functions fired so far in its causal chain — and the fired function's writes carry it to the fires they plan (through `ctx.functions.start` as well). The planner refuses a target already in the chain (`LOOP_DETECTED`) and any fire past a depth of 4 (`DEPTH_EXCEEDED`): a function that writes to the type it watches fires itself once and stops, A-writes-B-writes-A stops at the first repeat, and a five-watcher chain fires four. HTTP, webhook, cron and `workflow.call` invocations are roots with an empty chain, so an HTTP-invoked function that writes the type it watches is fired once by that write and that fire's own write is refused. A refused fire is **not an error**: the write commits and answers as before, no application error event is written, and the function's logs show nothing for it. It is **counted** as a `function.invoke` event with `status: "refused"` and the reason, attributed to the user whose write started the chain, and **logged** as one `server-function db-change trigger … outcome=refused` line per function, reason and minute carrying the chain and its depth. Write to a type you do not watch, or make the handler idempotent on `change.data` so the one re-fire is harmless.

A push naming a type the app does not have is refused by name, and a write to a database with no `databaseType` fires nothing.

**Every webhook or cron fire writes a run row** — the only record it leaves, since nobody is waiting for an answer (a database-change fire deliberately writes none; see above). A fire has no caller: `ctx.user` is `null`, and the code acts as the app exactly as it does on an HTTP invoke (see Execution identity).

```bash
primitive functions runs <function-id>      # newest first: status, what fired it, refreshes, timings, failure code
primitive functions get <function-id>       # receiver URL, schedules, watched types, last fired
```

**The file is the whole truth.** Removing the webhook block stops deliveries (the receiver answers `410`); putting it back resumes them on the same URL with the log intact. Removing a cron entry cancels it; re-adding schedules it again.

## Ceilings

`[function.limits]` may **lower** any ceiling and never raise one; a config asking for more gets the platform value. One caveat: `cpuMs` is enforced — past its budget the invocation is stopped and answers `failed` — but the runtime adds an allowance of about **two seconds** before it stops anything, so `cpuMs = 50` and `cpuMs = 200` buy the same thing and neither stops code in milliseconds. To bound a function tightly, use `timeoutMs` on the request: that is wall clock and is held exactly. The table describes one request invocation; a task run is bounded per slice (see Budgets in a task run).

| Limit | Platform value | Key |
|---|---|---|
| CPU per invocation | 5 000 ms request; 5 minutes per task slice | `cpuMs` |
| Outbound subrequests | 128 request; 10 000 per task slice | `subRequests` |
| Invocations per minute, per function | 1 200 | `ratePerMinute` |
| Wall clock, request | 5 000 ms default, 30 000 ms ceiling | `timeoutMs` (on the request) |
| Wall clock, task slice | continuous up to 12 hours; every step begins with at least 5 minutes | — (see Budgets in a task run) |
| Response size | 1 MiB | — |
| Built bundle | 5 MB | — |
| Send payload | 64 KiB | — |

- **Rate.** Buckets are per app and per function. A slot is taken only once a call is going to run. Past the ceiling: `429 FUNCTION_RATE_LIMITED` with `Retry-After`. Limiter unreachable: `503 FUNCTION_RATE_UNAVAILABLE`, retryable.
- **Response size** is measured by the platform on the parsed `output` (and `error`): over 1 MiB is `{"status":"failed","errorCode":"OUTPUT_TOO_LARGE"}` whatever the sandbox did. Page results, or write them to a database and return a handle.
- **The wall clock starts when the platform accepts the invocation**, so a cold load of the function's code comes out of it. Past the budget the caller gets `{"status":"timeout"}` and the sandbox request is aborted. The platform cannot kill a running sandbox, so a function that ignores the abort may execute until `cpuMs` stops it — but it can no longer *act*: the invocation's credential carries an absolute deadline checked on every platform call, so late calls are denied and late side effects never land. The same deadline cuts off `ctx.integrations.call` (`UPSTREAM_TIMEOUT`) and `ctx.prompts.run` (`PROMPT_UPSTREAM_TIMEOUT`).
- The clock does not advance during CPU-only work: a loop that waits for `Date.now()` to move never finishes. Wait on a timer, not on the clock.

## Recording

Every HTTP invocation writes one `function.invoke` analytics event attributed to the **caller** — never the app, although the app's authority is what the code ran with — carrying the function key, the version that ran, the terminal status and the duration. Trigger fires are recorded as run rows instead. The event is a COUNT: it answers "how often", never "why" — for why, read the invocation records below. A failed invocation's event reports `outcome: "error"` through the shared inspection contract, so failures are countable.

## Debugging a failing function

Every invocation the platform dispatched toward the sandbox leaves a record, and `console.log` is where you read it back:

```bash
primitive functions logs <function-id>            # newest first
primitive functions logs <function-id> --json     # the shared inspection items
primitive functions logs <function-id> --follow   # tail as invocations happen
primitive functions logs <function-id> --limit 50 --cursor <cursor>
```

A record carries what the invocation printed (`console.log`/`info`/`debug`/`trace` as stdout, `warn`/`error` as stderr), the thrown error with its code and a bounded stack, and the correlation keys — app, function, config version, the run id for a trigger fire or task run, and what triggered it. **Retention is seven days.**

| | |
|---|---|
| Doors that write one | HTTP invoke, webhook fire, cron fire, database-change fire, `workflow.call` from a DSL workflow, a task run's settling slice |
| Also written | A TIMED-OUT invocation, carrying what the function printed **before** it hung |
| Never written | A gate refusal — 403 (access), 404 (unknown key), 409 (not pushed), 400 (input schema), 429 (rate). Answered in the HTTP response, and debugged from there |
| Who can read | Owner and admin, on the CLI, the admin API and `ctx.api.functions.logs({ functionKey, limit })` |

**Secrets are redacted, best-effort.** A value `ctx.secret()` returned is replaced with `[REDACTED:<NAME>]` in the captured lines, in the lines forwarded to the live console, and in the error message and stack — on both sides of the sandbox boundary. It is best-effort by nature: a secret your code transformed before printing is not detectable. Do not print credentials.

**Task functions have a narrower guarantee.** A task run's record is written when the slice that SETTLES it finishes. A `step.do` body that ran in an earlier slice does not re-execute on replay, so what it printed before a `step.sleep` is not in the record — and the same is true of a fallback yield the platform takes when a refresh is refused (see Budgets in a task run), which is a hibernation like any other sleep. The platform cannot observe a step boundary from outside the sandbox, and a slice that suspends at a sleep never settles. Print what you need in the slice that settles, or write progress as data.

Three markers: `truncated` says a line or the channel hit its cap (2 KiB per line, 16 KiB and 256 entries per invocation) — the invocation is never failed for logging too much; `logsUnavailable` says the platform could not retrieve the buffer at all (an evicted isolate, or a dispatch refused before any code ran), which is not the same as a function that printed nothing; `contentSuppressed` says the platform could not load EVERY secret the version declares at write time (the store did not answer, or a declared `secret:<NAME>` has no value in this environment), so it kept the correlation and the status and dropped the console and the error fields rather than publish them unredacted — an unprovisioned declared secret suppresses every record of that version, so provision it or drop the declaration.

Records outlive the function: archiving does not remove them, and both read surfaces keep answering for an archived function until the seven days are up.

## Operating

```bash
primitive functions list                 # keys, status; --status active|inactive|archived
primitive functions get <function-id>    # active version, capabilities, manifest, triggers
primitive functions runs <function-id>   # trigger fires and task runs, newest first; --limit <n> --cursor <cursor>
primitive functions logs <function-id>   # what it printed and what it threw; --follow --limit <n> --cursor <cursor>
primitive functions disable <function-id>
primitive functions enable <function-id>
primitive functions archive <function-id>
primitive functions codegen --check      # the selected environment's tree; --env <name> picks another
primitive config set function/<key> <path>=<value>
```

Every verb takes `--app <app-id>` and `--json`.

- `disable` refuses invocations and leaves config and code unchanged; pushes still land.
- `archive` retires a function without destroying it: the row keeps its key, pushing code to it is refused, and there is no un-archive. Reclaiming the key is a hard delete — `primitive config pull` (removes the tombstone's local files), then a confirmed `primitive config push --prune`, which destroys the function, every version and their bundles (refused while a run is live — see Deleting a function with live runs).

## Migrating a workflow tree to functions

Move a DSL workflow tree **leaf-first**. Rewrite the leaves — the workflows that call nothing — as functions; leave the parents alone, because a parent's `workflow.call` step names a key and a key may name a function. The parents move last. Moving a whole subtree at once is the only alternative, and an app's parents are its largest workflows.

**Resolution.** `workflow.call` resolves `workflowKey` in this order: a live workflow wins; a key naming no workflow, or an archived one, resolves to a function holding the same key; when neither answers, the step fails with the error it always failed with (`Workflow "<key>" not found`, or the archived workflow's own sentence). The function's output becomes the step's `output`, so `saveAs`, templates and downstream steps read through it unchanged.

```toml
[[steps]]
id = "shape-order"
kind = "workflow.call"
workflowKey = "shape-order"   # a workflow yesterday, a function today
[steps.input]
orderId = "{{ input.orderId }}"
```

**Authorization.** A caller-mode parent (one a member started) has the callee function's own `access` gate evaluated against that member before dispatch, the same way it has always had a child workflow's `accessRule` evaluated; a refusal fails the step with `FUNCTION_ACCESS_DENIED` and never reaches the function. App admins and owners pass. A system parent — a cron or webhook run — skips the gate, exactly as it skips a child workflow's rule. The gate is not ceremony: `workflowKey` is a template, so a dispatcher carrying `workflowKey = "{{ input.target }}"` lets its caller choose the callee.

**Identity.** The function runs with the app's system authority, like every function. `ctx.user` is the member who started the parent run, or `null` when nothing human did. `ctx.trigger` says how it was entered:

```ts
export default defineFunction(async (input: { orderId: string }, ctx) => {
  if (ctx.trigger.kind === "workflow") {
    console.log("called by", ctx.trigger.workflowKey, ctx.trigger.runId, ctx.trigger.stepId);
  }
  return { orderId: input.orderId };
});
```

**Budget and failures.** A bridge invocation gets the full synchronous budget of 30 seconds — the DSL child it replaces had the parent run's whole budget — draws on the function's ordinary per-minute ceiling, and applies its declared `inputSchema` and `outputSchema`. There is no run row: the parent's step run is the record. A throw, a timeout or a refused output fails the step with the function's own code and message, so the parent's `continueOnError` and `retry` apply unchanged; `FUNCTION_ARCHIVED`, `FUNCTION_DISABLED`, `FUNCTION_NOT_PUSHED` and `FUNCTION_MODE_UNAVAILABLE` (task mode — `workflow.call` is synchronous) fail it without a retry, naming the remedy. Every one of those codes lands on `steps.<id>.code`, so a parent can branch on which it was.

The bridge is **transitional**: it retires with the workflow engine in project phase 7, along with `workflow.call`. One direction only — a function never calls a workflow.

**Taking the key over.** A function may take an **archived** workflow's key, so a replacement never carries a temporary name into its public URL:

```bash
primitive workflows archive shape-order   # the workflow keeps its history
primitive config push                     # the function claims the same key
```

The reservation moves; the archived row is untouched and still resolves by id. If the workflow is not archived, the push is refused with `functionKey already exists (held by a workflow …; archive it with 'primitive workflows archive', or remove its file and run 'primitive config push', then re-push this function to take the key)`. Only a function may take a key this way, and only from an archived workflow — an archived **function** still blocks everything, because a function tombstone keeps its key.

**What becomes what.**

- An operation that filtered on a `$params.userId` fed from the user becomes a `defineQuery` with the `$caller` marker: the platform injects the invoking user, a call may not supply one, and the query runs on the app's authority under the function's `access` gate.
- A client-callable operation becomes a function. The operation's `access` becomes the function's gate; its shape becomes `inputSchema` and `outputSchema`.
- A `script` step becomes a pure function — a Rhai transform has no platform access to lose.

**Do not reuse the old envelope as fixtures.** Test fixtures captured from the workflow being replaced **prove nothing about the replacement**: they describe the old envelope, so a port that shapes its answer to them passes its tests while every page reading the new one is wrong. Write the replacement's tests against what the caller needs now.

## Related

- [Configuration](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md) — the sync loop that pushes `functions/<key>.toml` and its code.
- [Databases](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) — database types, models and the records surface.
- [Integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md), [Prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md), [App Secrets](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md) — what a function calls out to.
- [Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md) — the CEL the `access` gate is written in.

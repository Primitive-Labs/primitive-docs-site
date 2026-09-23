# Agent Guide to Primitive Server Functions

A **server function** is TypeScript authored in the app's config tree and pushed with `primitive config push`. It runs on the platform — never on the device, whatever language the app's clients are written in — **as the app itself**: reviewed code acting with the app's authority, behind an `access` gate that decides who may call it. It answers over HTTP. It is config-as-code beside [prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md) and [integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md): `functions/<key>.toml` states the gate, the entry point and the few capabilities that still need declaring; the code sits next to it; one push builds and ships both as one immutable version.

**The caller chooses the RUNTIME, and the ROUTE is how it says so.** A function is a function and its config says nothing about how it runs. `functions.invoke` → `POST functions/{key}` runs it under the **request** runtime (the code runs inside the call and answers its result, no run row); `functions.start` → `POST functions/{key}/start` runs it under the **task** runtime (a run id immediately, one run row, and the code may sleep for hours or days). Every pushed function takes both, with no config change.

| Door | Runtime |
|---|---|
| `functions.invoke`, a webhook delivery | request |
| `functions.start`, `ctx.functions.start`, a cron fire | task |

`ctx.runtime` is the `request` / `task` word for the runtime in use — `"request"` or `"task"` on every invocation, beside `ctx.trigger.kind` (which says which DOOR fired it). A function that must not run under one of the two says so **in code**, with `assertRuntime("request" | "task")` from `primitive-functions` — at module scope to refuse every such invocation, or inside the handler. The platform settles a mismatch as `status: "failed"`, `errorCode: "FUNCTION_RUNTIME_REFUSED"`, with a message naming both runtimes. That assertion is the only lock there is.

**`mode` and `durable` are not keys of a function**, and a `[[function.triggers.cron]]` entry has no `mode` either — every cron fire starts a task run.

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
                                    # No `mode` and no `durable`: the CALLER picks the runtime at
                                    # each call. Use assertRuntime(...) in code to refuse one.
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
| `input` | The request's `rootInput`, validated and coerced against `inputSchema` when one is declared. On a trigger fire: the verified delivery body (webhook) or the entry's `rootInput` (cron). |
| `ctx` | The invocation context: `user`, `trigger`, `api`, `db`, `integrations`, `prompts`, `secret`, `configVar`, `users`, `connections`, `channels` — each documented below. |
| `step` | Present under **both** runtimes, so one body runs either way. Under the TASK runtime it is the live engine step: `step.do` is memoized and replayed, `step.sleep` hibernates (see Task runs). Under the REQUEST runtime `step.do` runs its body **inline** (nothing persisted, nothing replayed) and `sleep`/`sleepUntil` really wait when the remaining request **budget** covers the duration — past it, and for `waitForEvent`, the invocation fails with `errorCode: "FUNCTION_RUNTIME_REFUSED"` naming the step, the duration and the budget left. The thrown error's `name` stays `STEP_NOT_AVAILABLE`. That rule holds on every request-runtime door: HTTP and a webhook delivery. |
| `ctx.runtime` | `"request"` or `"task"` — which runtime the platform is running this invocation under, set from the runner in use. `assertRuntime(required)` from `primitive-functions` throws a typed `FunctionRuntimeError` when it differs, and the platform settles it `failed` with `errorCode: "FUNCTION_RUNTIME_REFUSED"`. |
| `ctx.runId` | The RUN this invocation belongs to, or `null`. Set under the task runtime (every slice) and for a webhook-fired request run; `null` for an HTTP `invoke`, which writes no run row. |
| `ctx.sliceId` | The SLICE of that run, or `null`. A task run is a sequence of slices and this changes when the run hibernates and wakes, so it tells a resume from a first pass. The same identifier `primitive functions runs` and the status route's `slice.sliceId` report. `null` under the request runtime — a request invocation is not a slice. |

- The return value is JSON-serialized as the envelope's `output` and validated against `outputSchema`. A throw settles the invocation with `status: "failed"`, `errorCode: "FUNCTION_THREW"` and the thrown message.
- `defineFunction` is typing sugar over the same contract: `export default async function (input, ctx, step) {}` is equivalent.
- `primitive-functions` is the one import the bundle leaves external: the platform supplies it at invoke time, so a function always runs against the SDK the platform is running. The npm package is a stub whose every export throws — a function only ever runs inside a push.

### `ctx.user` and `ctx.trigger`

| Door | `ctx.trigger` | `ctx.user` |
|---|---|---|
| HTTP invoke | `{ kind: "http", runKey }` | `{ userId, email }` — the caller |
| Webhook fire | `{ kind: "webhook", webhookKey, webhookId, externalEventId }` | `null` |
| Cron fire | `{ kind: "cron", name, triggerId, scheduledFor }` | `null` |
| Nested start (`ctx.functions.start`) | `{ kind: "function", functionKey, runId, runKey }` — the PARENT's key and run | Inherited from the tree's root, or `null` |
| Admin system door | `{ kind: "manual", userId, runKey }` — the admin who asked | `null` |

`ctx.trigger.runKey` carries the caller-supplied run key a start was **coalesced** by, on the `http`, `function` and `manual` arms — `null` when the start named none, and on a request invocation, which coalesces nothing. A trigger root's run key is its own run id, which `ctx.runId` already says, so the `webhook` and `cron` arms carry none.

`ctx.trigger` is typed as that union (`FunctionTrigger`), narrowed on `kind`, with every arm's fields typed — the declaration is held identical to what the fire paths build. `ctx.user` is typed `FunctionUser | null`: a function that a trigger can fire reads it as `ctx.user?.userId`. Whichever door the invocation came through, the code runs with the same authority — the app's (see Execution identity). The caller is a **name** the code may use for attribution and scoping; it is not a second principal.

## Pushing

`config push` builds the bundle with esbuild, inlines every npm dependency, and stores the built bundle **together with** the authored TOML bytes and every source file it read.

- Every push that changes anything creates a new **immutable version** and repoints the function at it. "Anything" is the whole envelope: a comment-only or line-ending-only edit makes a new version, and `config pull` on a fresh clone hands back the bytes you wrote.
- **Relative and absolute imports must stay inside the config tree.** An entry or import that resolves outside it — through a symlink included — is refused at push, so a push can never upload a file the tree does not contain.
- **npm packages are imported by name and inlined.** They resolve through Node like any other import; `node_modules` content is never stored as source or written back by `config pull`.
- A missing entry file or a build error names the file and the message, and nothing is pushed for that function — the rest of the tree still applies.
- The built bundle is capped at **5 MB**.
- **Push typechecks what it ships** against the declarations it generates, per function — see Codegen (under The typed handle).
- Pointing a function back at an earlier version is `primitive functions activate` — see Versions and rollback (under Operating).
- **Push runs your module scope.** Collecting the manifest (see What a function touches) means building the bundle a second time with `primitive-functions` replaced by a recording stub and executing it in a separate Node process — scrubbed environment, the config tree as working directory, hard timeout. It is the trust you extend by running a project's own build; know it before pushing a tree you did not write. A push whose collection fails is refused naming the error: the manifest is part of the version's identity.
- Capabilities are validated before anything is applied: a malformed string, an `integration:` key the app has no active integration for, an unknown family, or a grant string or key that is not part of the grammar (below) refuses the push by name. Nothing about databases is checked at push, because nothing about databases is declared.

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
| `runKey` | Task only — idempotency key (see Task runs). On the request path it is validated and otherwise ignored, since there is no run row to deduplicate against. |

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
{ "status": "completed", "output": { "message": "hello Ada" }, "limits": { "cpuMs": 50, "subRequests": 128, "ratePerMinute": 1200 } }
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

Both verbs work on every function: `invoke` posts to `functions/{key}` and `start` to `functions/{key}/start`, and the route is the only thing that says which runtime the call means. A function that must not be reached one of the two ways says so in its own code with `assertRuntime`, and the refusal comes back as a `failed` invocation with `errorCode: "FUNCTION_RUNTIME_REFUSED"`.


`invoke<Input, Output>(_:input:…)` decodes `output` into a caller-chosen `Decodable` (`FunctionResult<Output>`) — there is no untyped entry point; a dynamic caller names the witness (`input: nil as JSONValue?`, bound `as FunctionResult<JSONValue>`). A settled invocation never throws — read `status` — while a platform refusal is an `HttpError` carrying the server's `errorCode` on `serverCode`. The typed `input` is sent as whatever JSON value it encodes to (object, array or scalar), so a function whose schema declares a non-object root receives exactly that.

### The access gate

`access` is a CEL expression (see [Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md)) evaluated on **every HTTP call**, and it fails closed:

- A caller it denies gets `403 FUNCTION_ACCESS_DENIED`.
- A function with **no** `access` expression denies everyone — which is why push refuses to ship code for one.
- App owners and admins bypass it.
- A trigger fire (webhook, cron) and a nested `ctx.functions.start` do NOT consult it — the gate governs the HTTP route only. A function that only its triggers or another function should run declares `access = "false"`, a gate no member passes, which closes that route.
- It is checked **before** any answer about the function's operational state, so a rejected caller cannot tell a disabled function from a never-pushed one from a live one, and cannot map the app's functions by probing keys.
- A caller the gate rejects, or a request `inputSchema` refuses, consumes no rate-limit budget.

**The gate is the authorization.** Once a caller is through it, the code runs with the app's authority — the gate is the one place a reviewer decides *who* may make this code act. Write it as tightly as the function deserves: `user.role == 'admin'` on a function that deletes, `true` on one that reads its caller's own rows.

## Task runs

Any function starts as a run when the caller says `start`. When running it inside a call makes no sense — a function that sleeps for a day is not something you want a caller waiting on — say so in the CODE with `assertRuntime("task")`, at module scope so every invoke is refused before a line of the handler runs:

```toml
# functions/order-sync.toml — nothing here says how it runs
[function]
key = "order-sync"
entry = "functions/order-sync/index.ts"
access = "true"
```

```ts
import { assertRuntime, defineFunction } from "primitive-functions";

// An invoke of this function settles `failed` with
// errorCode: "FUNCTION_RUNTIME_REFUSED", naming both runtimes.
assertRuntime("task");

export default defineFunction(async (input: { orderId: string }, ctx, step) => {
  const charged = await step.do("charge", async () =>
    ctx.api.documents.get({ documentId: input.orderId })
  );

  // Hibernate. Nothing runs — and nothing is billed — while it waits.
  await step.sleep("cooling-off", "24 hours");

  return { orderId: charged.documentId, confirmedAt: Date.now() };
});
```

| `step` method | Under the TASK runtime | Under the REQUEST runtime |
|---|---|---|
| `do(name, body)` | Run `body` once; its return value is stored and replayed on every later resume | Run `body` inline; nothing stored, nothing replayed |
| `do(name, config, body)` | The same with a retry/timeout config — `stepPolicy.<family>` is one | The config is accepted and the body runs once |
| `sleep(name, duration)` | Hibernate for a duration (`"24 hours"`, or milliseconds) | Really wait, if the remaining request budget covers it; otherwise `FUNCTION_RUNTIME_REFUSED` |
| `sleepUntil(name, timestamp)` | Hibernate until a `Date` or epoch-ms timestamp | The same rule, from the timestamp's distance |
| `waitForEvent(name, options)` | Hibernate until an event arrives | `FUNCTION_RUNTIME_REFUSED`: nothing can deliver an event to a call that must return now |

### Starting and polling a run

`POST /app/{appId}/api/functions/{key}/start` — the same gate, input schema and rate ceiling as the request route — answers `201` with a run envelope instead of a result:

```json
{ "runId": "01J…", "runKey": "01J…", "instanceId": "app-doc-01J…", "status": "running" }
```

- `runKey` makes the start idempotent per `(caller, contextDocId, runKey)`: a repeat answers the run that already exists with `existing: true`, never a second run — and that holds for a run that FAILED, so retrying takes a new key with the same input, not the key the failure holds. It defaults to the run id.
- `timeoutMs` is accepted and ignored: the task engine owns how long a task run may take, and a per-request budget cannot hold across a `step.sleep`.
- Poll the run id with `functions.getStatus` / `functions.waitFor`; `terminate` takes the function key and the run key.
- A run broadcasts nothing over the WebSocket. Poll.
- A caller over HTTP starts a task run with `POST functions/{key}/start`, and so does `ctx.functions.start` from inside any running function. Any callee may be started; one that must not run that way says so with `assertRuntime("request")` and the child run settles `FUNCTION_RUNTIME_REFUSED`. The callee's `access` gate is NOT consulted (yours was the authorization), every run in the tree is keyed by the original external initiator, and nesting stops at 4 levels with `FUNCTION_NEST_DEPTH_EXCEEDED`.

### Reporting back

A task run has three ways to deliver its result, and they compose:

| Channel | How | Survives the client being away? |
|---|---|---|
| The run's `output` | Return a value; the same settlement that records the run's status records it on the run's own record | Yes — for the life of the run row (45 days), whether or not the engine still holds the instance |
| Server state | Write database rows, documents or notifications as the app, inside `step.do` | Yes |
| A live push | `ctx.users.send` (see Sending to a connected client) | No — a client that is offline never receives it; pair it with state or a notification |

**`output` outlives the engine's instance.** `client.functions.getStatus(runId).output` and `waitFor(runId).output` answer it, and so do `primitive functions runs wait <function-id> <run-id>` and `primitive functions runs <function-id> --json`. The `runs` TABLE does not print it — an output can be a megabyte, so the table says where to find it. Over roughly 10 000 characters the value is stored by reference and still answered whole, which is invisible to a reader except that a `runs` page holding many large outputs may be **shorter than the limit you asked for** and continue with a cursor. If the platform could not store a large output when the run settled, the record carries a preview marked `outputTruncated` and is completed from the engine on the next read while the engine still holds the run; once it has forgotten the instance, the preview is what is left. The ceiling is the same **1 MiB** a request invocation's output has.

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
    options: FunctionWaitOptions(timeout: 60) // throws workflowWaitTimeout past it
  )
  // A failed run RESOLVES — branch on the status, don't catch. `error.message`
  // is the reason, a platform refusal leads it with its code, and `error.code`
  // is that code on its own.
  if settled.isFailure {
    print("order-sync failed:", settled.error?.message ?? "no reason given")
  } else {
    print(settled.status, settled.output?.confirmedAt ?? 0)
  }

  // The function key goes where a workflow key would.
  _ = try await client.functions.terminate(
    FunctionRunRef(functionKey: "order-sync", runKey: run.runKey)
  )
```

`functions.waitFor` polls with a backing-off interval (0.4 s doubling to 5 s) and throws a wait-timeout past its budget (default 15 minutes; a non-positive value waits unbounded), taking `FunctionWaitOptions`. `getStatus` and `terminate` take the run id and the function key respectively; a repeated `runKey` on `start` replays the run that already exists.

All three answer a **function run status** — `FunctionRunStatus` (`FunctionRunSettled` once a wait has settled) in TypeScript, `FunctionRunStatus` and `FunctionRunResult<Output>` in Swift. It is a function's own type: `status` is one of `queued`, `running`, `completed`, `failed`, `terminated`, `missing`; `run` carries the function run's own fields (`functionId`, `functionKey`, `executionPrincipal`, `parentFunctionKey`, `parentRunId`, `nestDepth`, `errorCode`); `slice` and `outputTruncated` ride beside them. Which routes the client polls is its business.

**A failed run RESOLVES with `status == "failed"`** — it is an answer, not a transport error, so branch on the status rather than catching. Why it failed rides on the one structured `error`, on `getStatus` and on `waitFor` alike:

| Field | What it is |
|---|---|
| `error.message` | What went wrong. **This is where a platform refusal's code is** — the message leads with it (`OUTPUT_SCHEMA_VIOLATION: …`, `OUTPUT_TOO_LARGE: …`). An ENGINE failure is the exception: its message is the engine's own text and carries no code. |
| `error.code` | The platform's classification, from the closed set in Run error codes below — the same value the run row carries as `run.errorCode`. **Branch on this**, not on the message. |
| `error.name` | The error's own NAME, not the platform's code: your thrown error's class name (`Error` for a plain `throw new Error(…)`) where the engine preserved it. **Absent for a failure read off the persisted row**, which is the ordinary case once a run has settled — the platform keeps only the message there. |
| `error.details` | Anything else the platform sent with the error. Absent when there was nothing. |

Read `error.code` for the classification and `error.message` for the reason; treat `name` as a hint. `error` is absent on a run that has not failed.

**`terminate` tells two 404s apart.** A run it cannot find is `NOT_FOUND` / `.notFound` naming the function key and the run key; a run that EXISTS and that the engine could not stop is `UNAVAILABLE` / `.unavailable`, carrying the engine's own diagnostic verbatim after the message. Only the second means "try again".

### Run error codes

`run.errorCode` on the row and `error.code` on the function run status carry the same closed-set value. Two of these are about the handler, five about the platform's limits, and six — the `ENGINE_*` family — about the engine that ran the code. **Nothing in your code causes an `ENGINE_*` failure**; branch on the prefix to treat the family as one.

| Code | What happened | What to do |
|---|---|---|
| `FUNCTION_THREW` | The handler threw. | Read `error.message` and `primitive functions logs <key> --run <runId>`. |
| `FUNCTION_NO_HANDLER` | The entry file exported no callable default. | Export a default function from the file `entry` names. |
| `FUNCTION_TIMEOUT` | The invocation did not finish inside its budget. | Shorten the work, or run it as a task, where the budget is per slice. |
| `FUNCTION_INPUT_INVALID` | The input did not match `inputSchema`. | Fix the caller's input, or the schema. |
| `FUNCTION_OUTPUT_INVALID` | A webhook delivery's run returned output that did not match `outputSchema`. (An HTTP invoke refusing its output answers `OUTPUT_SCHEMA_VIOLATION` in the envelope instead.) | Fix the returned value, or the schema. |
| `OUTPUT_SCHEMA_VIOLATION` | A task run's output did not match `outputSchema`. | Fix the returned value, or the schema; the message names the failing paths. |
| `OUTPUT_NOT_SERIALIZABLE` | The returned value is not JSON. | Return plain data — no functions, no cycles. |
| `OUTPUT_TOO_LARGE` | The output exceeds the 1 MiB ceiling. | Page the result, or write it to a database and return a handle. |
| `FUNCTION_DELETED` | The function was hard-deleted while the run was starting. | Nothing to retry: the bundle the run pinned is gone. |
| `FUNCTION_RUNTIME_REFUSED` | The code refused the runtime it was handed (`assertRuntime`), or a request `step` was asked for a wait its budget cannot cover. | Call it the other way — `invoke` for a request, `start` for a task. |
| `ENGINE_ISOLATE_EVICTED` | A platform component the invocation depended on went away mid-call, or the isolate was reset for memory. | Nothing in your code caused it. To run the work again, start a NEW run with a new `runKey` and the same input — a repeated key replays the failed run and executes nothing — and make side effects the failed attempt performed idempotent yourself. Report a repeat to the platform. |
| `ENGINE_CODE_UPDATED` | A platform deploy reset the engine mid-attempt. | Nothing in your code caused it. Start a NEW run with a new `runKey` and the same input; a repeated key replays and executes nothing, and side effects already performed need your own idempotency. |
| `ENGINE_STORAGE_ERROR` | The engine's storage timed out or failed and the object was reset. | Nothing in your code caused it. Start a NEW run with a new `runKey` and the same input (a repeated key replays), with your own idempotency over side effects already performed. Report a repeat. |
| `ENGINE_INTERNAL_ERROR` | The engine reported an internal error on the attempt. | Nothing in your code caused it. Start a NEW run with a new `runKey` and the same input (a repeated key replays), with your own idempotency over side effects already performed. Report a repeat. |
| `ENGINE_SLICE_DEADLINE` | A platform call was refused because the slice's token deadline had passed. | The slice outlived its token: find the step that runs longer than the slice ceiling and split it. |
| `ENGINE_INSTANCE_LOST` | The engine no longer reports the instance and nothing recorded how the run ended. | `primitive functions logs <key> --run <runId>` shows what was recorded. Start a new run with a new `runKey` if the work still needs doing. |

An engine failure's `errorMessage` is the ENGINE's own text, so the code is on `error.code` and `run.errorCode` rather than the message's prefix. An attempt whose engine died before the platform wrote its invocation record leaves no record at all — the run row's code is then the whole statement.

The typed `getStatus(runId:)`, `waitFor(runId:as:options:)` and `terminate(_:)` decode `output` into a caller-chosen `Decodable` and answer `FunctionRunResult<Output>`; the untyped ones answer `FunctionRunStatus`. `waitFor` takes `FunctionWaitOptions(timeout:)`; a 404 or a `missing` run throws `.notFound` in function words. `terminate` takes a `FunctionRunRef`. `error` is the `FunctionRunError` above (`message`, `code`, `name`, `details`) on every one of them — a value of a shape the client cannot read leaves it `nil` and still reports the status, so a surprise in this field never costs you the run. `isTerminal` is true for exactly `completed`, `failed` and `terminated`.


### Step discipline

The engine re-executes the handler from the top on every resume and replays each completed `step.do` from its **stored result**.

- **Put every side effect inside `step.do`.** A charge, an email or a write outside a step runs again on every resume; inside one it runs exactly once.
- **Do not branch on anything nondeterministic between steps.** `Date.now()`, `Math.random()`, `ulid()` or a fresh read at the top level can take a different path after a resume, and the engine then looks for steps that are not there. Read such values inside a step so the value is stored with it.
- **Every cron fire starts a task run** — a run row, the task runtime's budgets and `overlapPolicy` asked of the engine. A webhook delivery always runs under the REQUEST runtime: the provider is waiting inside the request for an answer, so it hands slow work over with `ctx.functions.start`.

### Locks

A function takes a named lock through `ctx.api.locks.*`. Under the task runtime there is one rule: **acquire live on every slice, outside `step.do`** — a memoized acquire hands a resumed slice the handle a previous slice was given, and that handle no longer works.

```ts
export default defineFunction(async (input: { userId: string }, ctx, step) => {
  const key = `portfolio-import:${input.userId}`;
  const got = await ctx.api.locks.tryAcquire({
    body: { key, ttlMs: 120_000, owner: ctx.runId },
  });
  if (!got.acquired) return { skipped: true, heldBy: got.owner };
  try {
    await step.do("import", async () => { /* work that must not overlap */ });
  } finally {
    await ctx.api.locks.release({
      body: { key, handle: { handleId: got.handle.handleId } },
    });
  }
});
```

`owner: ctx.runId` is what lets a later slice of the SAME run re-take the lease instead of colliding with its own earlier slice: presenting the owner that already holds the key succeeds with a **fresh handle**, and the previous handle is fenced (`release` answers `not_holder`, `renew` answers `lease_lost`). The match is the same principal, the same kind of caller and the same owner string — so a member who started the task cannot rotate or release the lease their function holds. When a run key coalesces several triggers into one run, name the work that key stands for (`ctx.trigger.runKey ?? ctx.runId`). Never a static string: two unrelated runs presenting one would re-enter each other's hold. `status.owner` tells your own hold from another's.

### What resets a running task

- **A `config push` never resets a running task.** A run is pinned to the content hash of the version that started it and reloads that same immutable bundle at every wake; a push points the function at a new version for the *next* call. Neither archive nor `functions activate` reaches a run in flight either. A developer who saw a long run die shortly after a push was not looking at the cause.
- **A platform deploy does, and so does an isolate eviction.** The platform's own code is **not pinned** — the handler executes inside the platform's own runtime, so a redeploy resets that runtime under every slice that is *executing*. Nothing in the app's code caused it.
- **The engine then re-runs the handler.** About five minutes later it dispatches again: the handler runs from the top, every completed `step.do` replays from its stored result, and the step that was executing runs again **as a further attempt of that step** — charged to that step's retry budget (default 5 attempts). A step whose budget the reset exhausts does not re-run: the run fails with the engine's own internal-error message, recorded as `ENGINE_INTERNAL_ERROR`. Do not lower `retries.limit` on a step that must survive a deploy.
- **So a step body must be safe to repeat.** The interrupted attempt may have got partway through; the replacement starts it over. Upsert rather than insert, carry an idempotency key on anything that charges or sends, and let later steps read the step's return value rather than something it wrote outside itself.
- **How to see it.** A reset run that then finishes reads `completed` with no failure and no error code, so the count is the record: the `RESETS` column of `primitive functions runs`, `primitive functions runs steps` for which step was interrupted, `primitive functions logs <key> --run <id>` for what it printed, and `resets`/`lastResetAt`/`lastResetCause` on the `slice` block both clients read. A step the reset interrupted keeps **one** trace row, `running` until the replay completes it.

### Budgets in a task run

The ceilings (see Ceilings) apply **per engine invocation, not per run**. A run is a series of *slices*: the engine runs the handler from the top until it returns or reaches a `sleep`, `sleepUntil` or `waitForEvent` that has not completed, then hibernates; the next wake is a new slice, with completed steps replayed from storage.

| Ceiling | In a task run |
|---|---|
| Wall clock | Continuous, up to 12 hours from the slice's start. Each slice is minted with a 10-minute credential and **every step begins with at least 5 minutes of it**. A single step may use up to 10 minutes of wall clock. When a step finishes with less than 5 minutes left, or is about to start with less than 5 minutes left (a cleanup step after the handler catches a failed step, or after code outside steps spent the slice), the platform REFRESHES the credential through the gateway and the clock rolls on with no pause. The guarantee holds at every `step.do` boundary; code outside steps still spends the clock it runs in, and the next `step.do` entry refreshes it. A handler that never calls `step.do` gets no refresh and dies at the 10-minute slice deadline. The request's `timeoutMs` is ignored; the run as a whole is bounded only by the 12-hour ceiling. |
| `cpuMs` (5 minutes per task slice) | Per slice, not per step — a refreshed slice keeps counting; the paid plan's per-invocation ceiling. A step that exhausts it is killed and the run FAILS, so a step's own body must fit in 5 minutes of CPU, and a run whose steps together need more must end the slice between them (a `step.sleep` past the engine's five-minute grace period; the wake starts a fresh CPU budget). |
| `subRequests` (10 000 per task slice) | Counted across the slice; does NOT reset while it refreshes. Replayed steps spend nothing — their bodies do not run — but any platform call *outside* a step re-spends on every wake. |
| `ratePerMinute` (1 200) | Per **start**. A task start reserves a slot and returns it if the start is refused; resumes take none. |
| Output (1 MiB) and `outputSchema` | Once, on the final return value, at settlement. |

When a refresh is refused — past the 12-hour ceiling, more than once a minute (one per 60-second window on the platform's epoch-aligned clock), or during a rate-limiter or engine outage — the slice falls back to a pause of a little over 5 minutes: the engine hibernates only for a sleep longer than its own five-minute grace period, so that is what the fallback sleeps, and the next wake mints a fresh 10-minute slice with a fresh 12-hour ceiling. So a refused refresh costs a little over five minutes, once. Two budgets do NOT refresh: the subrequest count yields instead — when it passes half the resolved limit (5 000 of 10 000, or half a lower configured value), the next `step.do` boundary takes the fallback yield rather than refreshing; and CPU, which is 5 minutes of active CPU per slice — a step that exhausts it is killed, and the run fails rather than recovering. Steps running in parallel (`Promise.all` over two chains) that reach a fallback yield together: it waits for every step in flight to finish before it sleeps, because the engine hibernates only when nothing is running. Because the fallback is a hibernation, what the run printed before it is not recoverable on the log record (see Debugging a failing function). On a local development server the engine never hibernates: a fallback there is a pause with no fresh credential, and a task run is bounded by one slice.

Consequences: `step.sleep` between chunks is for waiting, not for budget — chunk a long job by steps and sleep when there is something to wait for; keep the code outside steps minimal, since it re-executes on every wake; and a refresh does not count toward the engine's maximum number of steps per run, and neither does a fallback yield (`step.sleep` and `step.sleepUntil` are excluded from that allowance; only your `step.do` calls count).

**Observing a slice.** A task run's status carries a `slice` block beside `status` and `run` — `sliceId`, `startedAt`, `ceilingAt` (the 12-hour bound), `refreshCount`, `lastRefreshAt`, `settledAt` and `settledStatus`; a request invocation carries no `slice` key. `primitive functions runs` prints the count in a `REFRESHES` column, blank for a request run.

`functions.getStatus` answers the block as `slice` on `FunctionRunStatus`, and the typed overloads carry it too. A block the client cannot read is dropped rather than published with a hole in it, so `slice` being absent means "no record to show you", never "this run never refreshed".

**When a run's status settles.** A task run **settles its own row** as it ends: the settlement that writes the invocation record `primitive functions logs` prints also writes the run's `status`, `endedAt` and **`output`**, so `functions runs`, `functions logs`, `getStatus` and `runs wait` agree **without being polled** and without waiting for a sweep. Do not build a staleness timeout on top of it — a run that finished a day ago and was never polled reads exactly as one that finished a second ago.

`running` means the run is live, **or not yet known to have ended**: executing, hibernating in `step.sleep`, or waiting out a step retry, and nothing that merely reads it writes to it. The `slice` block is how you tell the two apart, and it reads one way only — `settledAt` set (with `settledStatus`) is proof the slice ENDED; **an unsettled slice is not proof the run is live**, because the platform writes the invocation record before it settles the slice record and a settlement it could not write down is logged rather than retried. Treat `settledAt: null` on a `running` row as "not known to have ended".

A `running` row whose run really did end is **repaired on read**: `primitive functions runs`, `getStatus`, `runs wait` and the hard-delete live-run scan each reconcile such a run once, asking the engine first and then the platform's own records — the settled slice record, then the run's **invocation record**, which lives seven days. So a run whose engine instance has aged out of retention is still settled from what was written down about it rather than guessed at. Polling and the platform's 60-second sweep are **fallbacks**; neither is what makes a finished run read finished.

One deliberate exception: a run whose row was written in the last five minutes and about which nothing has been recorded is treated as still STARTING — answered exactly as stored, never written, and a hard delete of its function is still refused, because that run is about to load that function's code.

### Deleting a function with live runs

A hard delete destroys the stored bundles a sleeping run reloads when it wakes, so it is refused — including through `primitive config push --prune` — while any run has not settled; the refusal names the live runs. Wait for them or terminate them, then retry. To stop new runs meanwhile, **archive** the function: an archived function keeps its bundles, so an in-flight run finishes on the version it pinned. While a hard delete is running, an invoke answers `404` and a task start already in flight answers `409 FUNCTION_DELETE_IN_PROGRESS`.

## Execution identity

Function code acts as the system. Every platform call a function makes — `ctx.api`, `ctx.db`, the helpers — carries the **app's** authority, not the caller's: the same thing the app's owner could do over the REST API. There is no execution identity to choose: `runAs` is not a key of a function.

| Door | Authority of every platform call | `ctx.user` |
|---|---|---|
| HTTP invoke | The app's | The caller — a name for attribution and scoping |
| Webhook, cron fire | The app's | `null` |

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

- **Admitted as the system** — the ordinary families: `documents`, `databases`, `users`, `groups`, `collections`, `notifications`, `locks`, `blobBuckets`, `resourceMetadata`, `analytics`, `channels`, `email`, `configVars`, `llm`, `gemini`. No grant, no declaration. A direct-LLM route still meets the app's `directLlmEnabled` spend gate, because that is the app's own setting.
- **Admitted under a keyed capability** — `integration:<key>` for `ctx.integrations.call`, `secret:<NAME>` for `ctx.secret`. These are declared because each names an outside credential the reviewer should see in the file.
- **Admitted under a high-blast capability** — eleven exact strings for the operations that destroy or re-own (see Capabilities). Declared so that "this function can delete a database" is a line in a reviewed TOML, not a surprise in a log.

Of `functions.*`, only `functions.start` (what `ctx.functions.start` calls) and `functions.logs` (a function's invocation logs) are published. Not published to functions at all: every other `functions.*` operation (a function cannot invoke or manage a function), `iterations.*`, `databaseTypeConfigs.*`, and `databases.adminData.*` (duplicates of `databases.records.*`). A prompt runs only through `ctx.prompts.run` and an integration is called only through `ctx.integrations.call`; any other `ctx.api.prompts` / `ctx.api.integrations` route that runs or calls one is refused. A method that type-checks can still be refused here.

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
| `QUERY_IN_LIST_TOO_LARGE` | A single `$in`/`$nin` list over 1 000 values (`400`); the message names the field, the count and the cap. |

Every platform call carries the invocation's credential, which expires at the invocation's deadline — a function that outlives its budget has late calls denied and side effects that never land (see Ceilings).

## Capabilities

`capabilities` is an array of exact strings. No wildcards, no case folding; a repeated string is the same string. Only three kinds exist:

| Kind | Strings | Unlocks |
|---|---|---|
| Integration | `integration:<key>` | `ctx.integrations.call(key, …)` — checked at push against an active integration |
| Secret | `secret:<NAME>` | `ctx.secret(NAME)` — grammar-checked at push; the value is provisioned per environment |
| High-blast | `databases:create`, `databases:delete`, `databases:transferOwnership`, `databases:addManager`, `databases:revokePermission`, `databases:grantGroupPermission`, `databases:revokeGroupPermission`, `users:remove`, `users:setRole`, `blobBuckets:createBucket`, `blobBuckets:deleteBucket` | The operation of the same name on `ctx.api`, which is otherwise refused `FUNCTION_HIGH_BLAST_GRANT_MISSING` |

**Nothing else is declared.** Database models, prompts, config vars, channels, sends, email and analytics writes are admitted with no declaration, because the code acts as the system.

**Push checks the list** before anything is applied: it refuses a malformed string, an `integration:` naming a key the app has no active integration for, and any string outside the three kinds, naming the string. `secret:` is checked for grammar only — values are provisioned per environment, so push the function, then provision the value; a name with no value is `FUNCTION_SECRET_NOT_FOUND` when the function reads it.

The platform reads capabilities from the TOML file you review in a pull request, so what a reviewer sees and what the platform enforces are one statement.

### What a function touches is documented, not declared

Push scans the built bundle and records a **manifest** on the version: the model names it saw in `.model("…")` and in the records calls' `modelName`/`model` keys, the `ctx.api` families and helpers it uses, the queries it registered, and whether a model name is resolved at run time. `primitive functions get <function-id>` prints it under `Manifest (documentation, not authorization)`. It is what it says: a reviewer's and an operator's map of what a version reaches, refreshed on every push, and never consulted when a call is admitted. A model missing from the manifest is a scan the code defeated (a name built from a string), not a model the function cannot read.

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

- The integration's own rules apply to what the request really asks for — path normalization against `allowedPaths`, `allowedMethods`, redirect re-checks, and its `timeoutMs`, which is also bounded by the invocation's remaining time (`UPSTREAM_TIMEOUT`). They are the integration's contract: see [Integrations — The call contract](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md#the-call-contract).
- **Authorization is the function's.** An integration has no client endpoint and no rule of its own: the function's `access` gate decides who may make the code run, and the `integration:<key>` capability in the reviewed TOML is the egress allowlist that admits the call. A disabled or deleted integration refuses every call (`INTEGRATION_INACTIVE`).
- Every call is recorded on the integration's own log with the function's key: `primitive integrations logs <integration-id>`.

See [Integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md) for declaring one.

## Running a prompt

`ctx.prompts.run(key, { variables, modelOverride, configId })` — no capability; the code is the app — runs one of the app's saved prompts and answers an envelope: `success`, `output`, `error`, `metrics`, `configId` (which config ran). `modelOverride` swaps the model for this call only.

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { text: string }, ctx) => {
  const result = await ctx.prompts.run("summarize", { variables: { text: input.text } });
  if (!result.success) throw new Error(result.error ?? "prompt failed");
  return { summary: result.output };
});
```

- `configId` pins a config **of that prompt**; any other answers `PROMPT_NO_CONFIG`.
- **Authorization is the function's.** A prompt has no client endpoint and no rule of its own; the function's `access` gate is the whole authorization. An inactive or archived prompt refuses every run.
- The model call is bounded by the invocation's remaining time. A provider call that passes the deadline answers `PROMPT_UPSTREAM_TIMEOUT` — catch it separately from a model failure: the budget ran out, not the model.

**Do not parse a JSON prompt by hand.** When the prompt declares `[prompt.outputSchema]`, the success arm carries `parsed` — the answer parsed, validated against that schema and typed from it:

```ts
const answer = await ctx.prompts.run("categorize", { variables: { payload } });
if (!answer.success) throw new Error(answer.error ?? "categorize failed");
// `parsed` is typed from the prompt's schema; no JSON.parse, no failure branch.
return { suggested: answer.parsed.suggested_transactions };
```

`parsed` is on the **success arm only**, which is what the `success` check unlocks. Declaring the schema, the `PROMPT_OUTPUT_*` codes a wrong-shaped answer returns, `outputFormat`, and the generated `primitive-prompt-types.d.ts` are the Prompts guide's: see [Prompts — Typed output](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md#typed-output).

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
- **`ctx.configVar` is read once per version.** Its value is cached keyed on the function version's config id — across calls in one invocation and across invocations on a warm sandbox — so a changed var reaches a function reliably only on its next pushed version; a re-push starts cold. A miss is never cached (provision the var, and the next read sees it), and `ctx.secret` is never cached.

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
// Any function publishes to the topic — including the one that just wrote the row.
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

`ctx.api.analytics.writeForUser({ body: { userId, action, feature, … } })` records an event attributed to a **subject** user rather than to whoever is running. Function-only route (`POST /app/{appId}/api/analytics/write-for-user`; `403 FUNCTION_ROUTE_FUNCTION_ONLY` otherwise). The subject must be a member of this app; `appId` and `userId` are reserved — a `context` object of your own cannot rewrite whose activity the event is. Fields and refusal codes: [Analytics — Writing Events from a Server Function](AGENT_GUIDE_TO_PRIMITIVE_ANALYTICS.md#writing-events-from-a-server-function).

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

## Database records

### On the app's authority

```toml
# functions/monthly-report.toml
[function]
key = "monthly-report"
entry = "functions/monthly-report/index.ts"
access = "user.role == 'admin'"
```

Nothing about databases is declared. A function reads and writes **every database of the app** — every type, every model, every row — as the app, through `ctx.db` and `ctx.api.databases.records.*`. There is no read grant and no write grant; the `access` gate above is where a reviewer decides who may make this code run, and the code is where the reviewer reads what it does with the rows.

- **Scope reads in the code.** A read the caller should see only part of is filtered — `query({ filter: { owner: ctx.user!.userId } })` — or bound through a `$caller` registered query (below). The platform does not narrow a function's read to its caller.
- **Every row is the app's.** The database's own permission rows (owner, managers, groups) are records a function can read (`ctx.api.databases.listPermissions`, `listGroupPermissions`) and app admins manage; they do not narrow a function. Creating, deleting or re-owning a database is the one exception, gated by a high-blast capability (see Capabilities).
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

  // A row that came back from a read carries its id, so it can be changed or
  // removed with no cast and no second lookup.
  await orders.patch(open.items[0].id, { data: { label: "renamed" } });
  await orders.delete(open.items[1].id);

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

**Every read row carries `id: string`**, whatever its schema declares — the
record identity is the platform's, not your schema's, and it is what `patch`
and `delete` take. It is there on a queried row even when the model's `.toml`
declares no `id` field, and on one whose `id` is declared optional. A row you
BUILD for a write still needs none: every write takes a partial row.

| Handle | Surface |
|---|---|
| `ctx.db(id, type).model(name)` | `query`, `count`, `aggregate`, `save`, `patch`, `delete`, `batch` — the model is bound per call, and `patch`/`delete` take the record id first |
| `ctx.db(id, type)` | `databaseId`, `databaseType`, `batch` (spans models — each item names its own) |
| `ctx.api.databases.records.*` | The untyped form of the same operations, plus the rest of the records surface — `increment`, `addToSet` / `removeFromSet`, the index and unique-constraint routes (`registerIndex`, `dropIndex`, `listIndexes`, `syncIndexes`, `registerUniqueConstraint`, `dropUniqueConstraint`, `listUniqueConstraints`), `describe` and `models` — as the app |

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

**A list filter holds 1 000 values.** A single `$in` or `$nin` list is capped at 1 000 values; each list in a filter is counted on its own, so two 600-value lists are fine. Past the cap the call is refused with `QUERY_IN_LIST_TOO_LARGE` naming the field and the count, rather than failing inside SQL. The list costs the statement ONE bound parameter, so a thousand keys page and sort no more expensively than two, and a batch of keys is one query rather than a loop. Past a thousand, the split depends on the operator: chunk an `$in` and merge the pages (each chunk matches some of the rows, so the union is the answer), but NEVER merge chunked `$nin` queries — a query excluding one chunk returns the rows the others exclude, so the union is nearly every row. Put `$nin` chunks in ONE filter, where they intersect: `{ $and: [{ f: { $nin: chunk1 } }, { f: { $nin: chunk2 } }] }`, each chunk counted against the cap on its own.

```ts
const page = await positions.query({
  filter: { snapshotKey: { $in: keys } },   // up to 1 000 keys
  options: { sort: { snapshotKey: 1 }, limit: 200 },
});
```

**Documents page and narrow the same way.** `ctx.api.documents.records.query({ documentId, model, filter, limit, cursor })` and `documents.records.count({ documentId, model, filter })` take the same `filter` object a database model does — the client JSON-encodes it into the query string; a malformed one answers `400` — and answer the `{ items, hasMore, nextCursor }` page the route and the CLI do, so a function that walks a large model pages it rather than reading it whole.

**Codegen.** `config push` writes the generated files into the tree's `functions/` directory beside your sources, and none is part of what gets pushed:

| File | What it carries |
|---|---|
| `functions/primitive-db-types.d.ts` | The database types behind `ctx.db`, model by model |
| `functions/primitive-functions.d.ts` | The `primitive-functions` module's own types |
| `functions/primitive-function-types.d.ts` | `<Key>Input` / `<Key>Output` for every function, from its `inputSchema` and `outputSchema`, and the `FunctionSchemas` augmentation that types the keyed `defineFunction("<key>", handler)` |
| `functions/primitive-prompt-types.d.ts` | The `PromptSchemas` augmentation that types `ctx.prompts.run("<key>")` from each prompt's `[prompt.outputSchema]` — see [Prompts — Typed output](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md#typed-output) |
| `functions/generated/<key>.generated.ts` | A typed **client** invoker per function (imports `js-bao-wss-client`): all five verbs — `invoke`, `start`, `getStatus`, `waitFor`, `terminate` — on every function, because the caller picks the runtime at each call. `input` is required iff the schema rejects `{}`; the `waitFor` output is `<Key>Output`. `-o <dir>` moves them |
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
over the generic `client.functions` overloads. Every invoker carries both verb sets
as it does in TypeScript — `invoke` runs the function inside the request, and
`start` / `getStatus` / `waitFor` / `terminate` drive it as a task — so the
caller picks the runtime at each call. A function with no declared schema gets
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

**Push typechecks what it ships.** The same `config push` that writes those declarations compiles your sources against them — the tree's own `functions/tsconfig.json`, so it is the program your editor loads — and refuses the function with the compiler's own diagnostics when they do not hold. esbuild erases types, so a bundle that builds proves nothing: a handler that dereferences `ctx.user` without checking for `null` builds cleanly and throws on the request path. The refusal is per function (the rest of the tree still lands), `--dry-run` reports the same diagnostics without shipping, and reproducing one by hand is `tsc -p functions/tsconfig.json --noEmit`. `primitive config push --no-typecheck` skips it. A CI job that runs `primitive config push --dry-run` therefore fails on broken function code with no extra wiring.

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
    const answer = await db.model("Order").query({ options: { limit: params.limit } });
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
- A `$caller` query throws in an invocation with no initiating user — a webhook or cron fire; use a query with an explicit parameter there.
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

Its rows carry `id: string` too, by the same rule as a database row's: `query().items[0].id`, `save(...).record.id` and `patch(...).record.id` are the id `patch` and `delete` take, whatever the schema declares.

The rows are typed from the project's `models/models.toml` (fallback: the web client's `src/models/models.toml`), rendered by `config push` into `functions/primitive-document-types.d.ts`: an undeclared model is a compile error, a declared field has its type, and a project with no schema file gets the open form. It confers nothing — the same authority as every other call — and another app's document id is the uniform not-found.

`batch` applies an ordered blob of ONE model's ops in a single transaction — all of it commits or none of it does, and a connected client sees one update. Every op's `model` is the handle's, so an op naming another is overwritten; a blob that genuinely spans models is the raw `ctx.api.documents.records.bulk`.

```ts
await ctx.doc(input.documentId).model("Order").batch([
  { action: "create", id: firstUlid, data: { label: "one" } },
  { action: "patch", id: "o-1", data: { status: "closed" } },
  { action: "delete", id: "o-2" },
]);
```

### A function may be a document's first writer

A document's record models carry their schema inside the document (`_meta_<model>`), and a collection field (a `stringset`) cannot be written without it. `config push` carries the project's `models/models.toml` **inside the function's version**, and the first write to a model that version declares seeds the model's metadata from it and writes the record **in the same operation** — one transaction, one persisted update, one broadcast. The record and its stringset read back from a client exactly as if a client had saved the model first, and the seeding write is coerced, defaulted, stamped and uniqueness-checked against the declaration it just wrote.

- A model the version's schema does NOT declare keeps today's behavior: a scalar write succeeds, a collection value is refused with `could not be resolved to a collection (stringset)`. Nothing in a request body can supply a schema — the platform builds it from the running version.
- A client's declaration wins: a model the document already declares is left exactly as it is.
- Editing `models/models.toml` changes every function's envelope hash, so `config diff` reports every function Modified and the next push mints a new version of each.
- A schema declaring a RESERVED field name — `type`, or any name starting with `_` — is refused at the push with a 400 naming the block (`[models.X.fields.type]: Field 'type' is reserved (maps to internal _type column in queries)`). A version is immutable, so it is refused before one is minted rather than seeding a model whose filters would match the model name. Rename the field in `models/models.toml` and push again.
- `config pull` writes the schema back as the generated `functions/primitive-document-schema.generated.toml`; push resolves the project file first and that copy only as the fallback, regenerating the copy on every push.
- `primitive functions get` prints the models the running version can seed.

## Ceilings

`[function.limits]` may **lower** any ceiling and never raise one; a config asking for more gets the platform value. One caveat: `cpuMs` is enforced — past its budget the invocation is stopped and answers `failed` — but the runtime adds an allowance of about **two seconds** before it stops anything, so `cpuMs = 50` and `cpuMs = 200` buy the same thing and neither stops code in milliseconds. To bound a function tightly, use `timeoutMs` on the request: that is wall clock and is held exactly. The table describes one request invocation; a task run is bounded per slice (see Budgets in a task run).

The two RUNTIMES differ in what they may spend, which is most of why the
runtime matters at all:

| Limit | Request runtime | Task runtime | Key |
|---|---|---|---|
| CPU | 5 000 ms per invocation | 5 minutes per slice | `cpuMs` |
| Outbound subrequests | 128 | 10 000 per slice | `subRequests` |
| Wall clock | 5 000 ms default, 30 000 ms ceiling | continuous up to 12 hours; every step begins with at least 5 minutes | `timeoutMs` on the request (request only); see Budgets in a task run |
| — a `step.sleep` has to fit the budget | past it, `FUNCTION_RUNTIME_REFUSED` | hibernates; hours and days are fine | — (start it as a task) |

And the ones that are the same under both:

| Limit | Platform value | Key |
|---|---|---|
| Invocations per minute, per function | 1 200 | `ratePerMinute` |
| Response size | 1 MiB, kept on the run row | — |
| Values per `$in` / `$nin` list | 1 000 | — |
| Built bundle | 5 MB | — |
| Send payload | 64 KiB | — |

- **Rate.** Buckets are per app and per function. A slot is taken only once a call is going to run. Past the ceiling: `429 FUNCTION_RATE_LIMITED` with `Retry-After`. Limiter unreachable: `503 FUNCTION_RATE_UNAVAILABLE`, retryable.
- **Response size** is measured by the platform on the parsed `output` (and `error`): over 1 MiB is `{"status":"failed","errorCode":"OUTPUT_TOO_LARGE"}` whatever the sandbox did. Page results, or write them to a database and return a handle. A TASK run's output is held to the same 1 MiB and is **kept on the run row** for its life, so it outlives the engine's instance (see Reporting back).
- **The wall clock starts when the platform accepts the invocation**, so a cold load of the function's code comes out of it. Past the budget the caller gets `{"status":"timeout"}` and the sandbox request is aborted. The platform cannot kill a running sandbox, so a function that ignores the abort may execute until `cpuMs` stops it — but it can no longer *act*: the invocation's credential carries an absolute deadline checked on every platform call, so late calls are denied and late side effects never land. The same deadline cuts off `ctx.integrations.call` (`UPSTREAM_TIMEOUT`) and `ctx.prompts.run` (`PROMPT_UPSTREAM_TIMEOUT`).
- The clock does not advance during CPU-only work: a loop that waits for `Date.now()` to move never finishes. Wait on a timer, not on the clock.

## Triggers

A function's config block can declare one inbound **webhook** and up to ten **cron** schedules. The platform creates and operates whatever they need. Which RUNTIME a kind uses is the door's, not the declaration's: every **cron** fire starts a task run, and a **webhook** delivery runs under the request runtime, because the provider is waiting for the function's answer inside the request — it hands slow work on with `ctx.functions.start`.

```toml
# functions/stripe-events.toml
[function]
key = "stripe-events"
entry = "functions/stripe-events/index.ts"
access = "false"                       # trigger-only: no member calls it over HTTP; fires skip the gate

[function.triggers.webhook]           # one per function; answers on the function's own key
verificationScheme = "stripe"          # required; "none" is refused
signingSecret = "{{secrets.STRIPE_WEBHOOK_SECRET}}"

[[function.triggers.cron]]
name = "nightly"                       # required; half of the trigger's identity
cron = "0 3 * * *"                     # five-field expression
timezone = "UTC"                       # IANA; default UTC
                                       # No `mode`: every cron fire starts a task run.
rootInput = { scope = "full" }         # what the function receives as input

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

**Webhook.** The receiver is `POST /app/{appId}/webhook/{functionKey}`. Verification, replay protection and a delivery log are built in. Accepted keys: `verificationScheme`, `signingSecret`, `toleranceSeconds`, `deduplicationEnabled`, `deduplicationWindowMs`, `maxBodyBytes`, `secretGracePeriodMs`, and a `[function.triggers.webhook.verification]` table for scheme-specific settings (a Discord public key, a JWT's JWKS, …). A verified delivery runs the function and answers `200 {"received": true}` **whatever the function did** — the run row records the outcome. A delivery that cannot run right now (function disabled, rate ceiling hit) answers `202` with the reason in the delivery log and no dedup key stored, so the provider's redelivery runs once the condition clears. A rotation via `primitive webhooks rotate-secret <webhook-id>` survives later pushes: a push touches the signing secret only when the TOML value itself changed. A delivery always runs under the REQUEST runtime: for work that outlasts it, verify and acknowledge in the delivered function and `ctx.functions.start` another, keying the start on the provider's event id or a body id so a redelivery replays rather than starting a second run. Each webhook trigger is one of the app's webhooks, and an app holds at most **50 webhooks**: a push that would add one past the limit refuses that function, naming the limit.

### Webhook verification schemes

`verificationScheme` is `stripe`, `github`, `slack`, `discord`, `jwt`, `plaid`, or `custom`; `"none"` is refused on a function trigger. `signingSecret` (where the scheme takes one) must be a whole `{{secrets.KEY}}` reference — a raw value is `SIGNING_SECRET_MUST_BE_SECRET_REF`, and a scheme that verifies with public-key material only (`discord`, `jwt`, `plaid`) refuses a supplied one (`SIGNING_SECRET_NOT_SUPPORTED_FOR_SCHEME`). Rotate onto a different secret key with `primitive webhooks rotate-secret <webhook-id> --secret "{{secrets.NEW_KEY}}"`: the previous reference keeps verifying for `secretGracePeriodMs` (24h default, 30 days max, `0` to cut over immediately).

- **`discord`** — `[function.triggers.webhook.verification.discord] publicKey = "<64-char hex>"`, Ed25519.
- **`jwt`** — each delivery arrives as a JWT carrying a body-hash claim: `header` (which request header carries the token), `algorithms` (`ES256`/`384`/`512`, `RS256`, `PS256`, `EdDSA`), `bodyHashClaim`, `bodyHashEncoding` (`hex`/`base64`/`base64url`), optional `issuer`, and exactly one of `jwks` (inline key set) or `jwksUrl` (fetched and cached; `https`, port 443, public hostname only). A delivery is accepted only when the token verifies against a JWKS key, the algorithm is listed, it was issued inside `toleranceSeconds`, and the body-hash claim matches.
- **`plaid`** — a fixed `jwt`-shaped preset with keys fetched from Plaid's API: `[function.triggers.webhook.verification.plaid] environment = "sandbox" | "production"`, `clientId`, `secret` (both `{{secrets.*}}` references).
- **`custom`** — declare the provider's own scheme under `[function.triggers.webhook.verification.custom.detachedSignature]`: `primitive = "hmac-sha256"`, `encoding`, `signature = { header, key?, prefix? }` (where the signature lives), `freshness = { from, format }` (must be one of the signed parts), optional `dedup.signedBodyPointer` (an RFC 6901 pointer into the signed body) and `externalEventId.untrustedHeader` (display only), and an ordered `signedPayload` array of `{ header | signatureField | literal | body = true }` parts with exactly one `body = true` and a `literal` separator between any two variable-length parts. Without a `detachedSignature` table, `custom` verifies its own fixed format: `t=<unix>,v1=<hmac hex>` in `X-Webhook-Signature` over `<t>.<body>`.

**Deduplication.** A signed delivery is deduplicated on the provider's own event id where the scheme carries one, otherwise a SHA-256 of the signed payload; a suppressed replay answers `200 {"received": true, "duplicate": true}` and does not run the function again. `github` (and a `custom` scheme with no `freshness`) keeps its dedup entries forever, since neither signs a timestamp.

**CLI.** `primitive webhooks test <webhook-id> --payload '{…}'` signs a preview without delivering it; add `--deliver` to post it for real (verification, dedup and dispatch all run, and it reports a `dispatched`/`refused`/`unconfirmed` outcome). `primitive webhooks verify <webhook-id> --header '…' --body-file captured.json` checks a captured delivery's signature without running anything — the only way to check `discord`/`jwt`/`plaid`, which `webhooks test` cannot preview. `primitive webhooks events <webhook-id>` lists recent deliveries. All three take the webhook id `primitive functions get <function-id>` prints under the trigger.

**Cron.** Every fire starts a task run (`ctx.user` is null, `ctx.trigger.kind` is `"cron"`, and the run polls and terminates on the ordinary run routes). With `overlapPolicy = "skip"`, a fire that arrives while the previous run is still going is counted as a skip; for a task run that question goes to the **engine**, so a sleeping run counts as live, and so does one the platform cannot be asked about. `overlapPolicy = "allow"` starts one run per fire.

**Following up on a write.** The function that writes a record is the place to do the work that follows it. It has the row, it knows why the write happened, and its caller can see what it did. Do the follow-up in the handler that writes:

```ts
export default async function (input, ctx) {
  const saved = await ctx.db(input.databaseId, "orders").model("Order").save({
    id: input.orderId,
    data: { label: input.label },
  });
  // …then publish, notify, or write another type — in the same handler. `save`
  // answers `{ success, id, appliedFields? }`, not the row: the values you
  // wrote are the values you have, and `appliedFields` carries what the
  // platform stamped on top of them.
  await ctx.channels.publish(`orders:${saved.id}`, { op: "save", label: input.label });
  // …and hand anything slow to a task rather than making the caller wait.
  await ctx.functions.start("reconcile-order", { orderId: input.orderId }, {
    runKey: `reconcile-${input.orderId}`,
  });
  return { placed: input.orderId };
}
```

It runs in order, it cannot loop, and it fails where you can see it.

**Every webhook or cron fire writes a run row** — the only record it leaves, since nobody is waiting for an answer. A fire has no caller: `ctx.user` is `null`, and the code acts as the app exactly as it does on an HTTP invoke (see Execution identity).

```bash
primitive functions runs <function-id>      # newest first: status, what fired it, refreshes, timings, failure code
primitive functions get <function-id>       # receiver URL, schedules, last fired
```

**The file is the whole truth.** Removing the webhook block stops deliveries (the receiver answers `410`); putting it back resumes them on the same URL with the log intact. Removing a cron entry cancels it; re-adding schedules it again.

## Recording

Every HTTP invocation writes one `function.invoke` analytics event attributed to the **caller** — never the app, although the app's authority is what the code ran with — carrying the function key, the version that ran, the terminal status and the duration. Trigger fires are recorded as run rows instead. The event is a COUNT: it answers "how often", never "why" — for why, read the invocation records below. A failed invocation's event reports `outcome: "error"` through the shared inspection contract, so failures are countable.

## Debugging a failing function

Every invocation the platform dispatched toward the sandbox leaves a record, and `console.log` is where you read it back:

```bash
primitive functions logs <function-id>            # newest first
primitive functions logs <function-id> --json     # the shared inspection items
primitive functions logs <function-id> --follow   # tail as invocations happen
primitive functions logs <function-id> --limit 50 --cursor <cursor>
primitive functions logs <function-id> --run <run-id>        # one task run's records
primitive functions logs <function-id> --invocation <id>     # one record, by the id an invoke answered with
```

`--invocation` names ONE record and refuses `--run`, `--follow`, `--cursor` and `--limit` beside it; `--run` narrows to one run and refuses `--follow`. An id that names nothing, one of another function and one whose seven days are up all answer the same not-found.

How `--follow` tails — polling, NDJSON under `--json`, and the look-back that still prints a slow invocation settling after a faster one — is in the `--follow` section of the [Inspection guide](AGENT_GUIDE_TO_PRIMITIVE_INSPECTION.md).

A record carries what the invocation printed (`console.log`/`info`/`debug`/`trace` as stdout, `warn`/`error` as stderr), the thrown error with its code and a bounded stack, and the correlation keys — app, function, config version, the run id for a trigger fire or task run, and what triggered it. **Retention is seven days.**

| | |
|---|---|
| Doors that write one | HTTP invoke, webhook fire, cron fire, a task run's settling slice |
| Also written | A TIMED-OUT invocation, carrying what the function printed **before** it hung |
| Never written | A gate refusal — 403 (access), 404 (unknown key), 409 (not pushed), 400 (input schema), 429 (rate). Answered in the HTTP response, and debugged from there |
| Who can read | Owner and admin, on the CLI, the admin API and `ctx.api.functions.logs({ functionKey, limit })` |

**Secrets are redacted, best-effort.** A value `ctx.secret()` returned is replaced with `[REDACTED:<NAME>]` in the captured lines, in the lines forwarded to the live console, and in the error message and stack — on both sides of the sandbox boundary. It is best-effort by nature: a secret your code transformed before printing is not detectable. Do not print credentials.

**Output from inside a step is captured, and attributed to its step.** A line printed inside a `step.do` body carries the step's NAME, which call of that name it was (`occurrence`, from 0, counting every named step call in the slice — `do`, `sleep`, `sleepUntil` and `waitForEvent` alike, so it lines up with the row of that name in `primitive functions runs steps`) and which `attempt` of the body it was, from 1. A line printed BETWEEN steps carries none of the three. A body that THROWS is recorded under its step as an `err` line, `step "<name>" attempt <n> threw: <message>`; the throw itself is unchanged — the engine still retries it if you configured retries and still fails the run when it runs out. A retried body's lines sit under one name and one occurrence with ascending attempts; a step the engine answers from its memo on a replay runs no body and records nothing new.

`primitive functions logs <function-id> --run <run-id>` prints the run's step trace, then each record oldest first with its lines under `step <name> #<occurrence> attempt <n>` headers, in the order they were PRINTED — stdout and stderr interleaved as they happened, so a `console.error` immediately followed by a `console.log` comes back in that order even though the record stores the two streams separately. A `thrown` line takes a `!` in the gutter. A record whose output hit a cap ends with `… truncated: the platform's per-record cap dropped later lines`, so a shortened record never reads as a complete one.

```
04:39:45  failed  http  cfg_01XYZ  01M2M8...  FUNCTION_THREW  the depot is closed
  +0ms  out  between: starting
  step ship #0 attempt 1
    +103ms  out  about to ship
  ! +104ms  err  step "ship" attempt 1 threw: the depot is closed
```

**Task runs have a narrower guarantee, and it is about SLICES, not steps.** A task run's record is written when the slice that SETTLES it finishes, and it carries everything that slice printed — between steps and inside step bodies alike. What it cannot carry is what an EARLIER slice printed: a slice that hibernated at a `step.sleep` never settles, so it writes no record, and a `step.do` body that ran in it does not re-execute on replay. Output from before a hibernation is **not recoverable**. The same applies to a fallback yield the platform takes when a refresh is refused (see Budgets in a task run), which is a hibernation like any other sleep. So a run that sleeps leaves one record per settled slice and a run that never sleeps leaves one record with everything in it; print what you need in the slice that settles, or write progress as data. A local development server never hibernates, so a task run there is one slice and one record however many times it sleeps.

Three markers: `truncated` says a line or the channel hit its cap (2 KiB per line, 16 KiB and 256 entries per invocation) — the invocation is never failed for logging too much; `logsUnavailable` says the platform could not retrieve the buffer at all (an evicted isolate, or a dispatch refused before any code ran), which is not the same as a function that printed nothing (an attempt that ended in the engine before the handler printed anything carries `endedIn: "engine"`, and `functions logs --run` prints `ended in the engine — no handler output`, or `logs unavailable` / `output suppressed`); `contentSuppressed` says the platform could not load EVERY secret the version declares at write time (the store did not answer, or a declared `secret:<NAME>` has no value in this environment), so it kept the correlation and the status and dropped the console and the error fields rather than publish them unredacted — an unprovisioned declared secret suppresses every record of that version, so provision it or drop the declaration.

Records outlive the function: archiving does not remove them, and both read surfaces keep answering for an archived function until the seven days are up.

## Operating

```bash
primitive functions list                 # keys, status; --status active|inactive|archived
primitive functions get <function-id>    # active version, capabilities, manifest, triggers
primitive functions configs <function-id>                   # every version, newest first; --json --limit <n> --cursor <cursor>
primitive functions activate <function-id> <config-id>      # point it at one of them; -y skips the prompt, makes no version
primitive functions runs <function-id>   # trigger fires and task runs, newest first; --limit <n> --cursor <cursor>
primitive functions runs steps <function-id> <run-id>       # one task run, step by step
primitive functions runs terminate <function-id> <run-id>   # end a run that will not settle; -y skips the prompt
primitive functions logs <function-id>   # what it printed and what it threw; --follow --limit <n> --cursor <cursor>
primitive functions disable <function-id>
primitive functions enable <function-id>
primitive functions archive <function-id>
primitive functions codegen --check      # the selected environment's tree; --env <name> picks another
primitive config set function/<key> <path>=<value>
```

### Running one

```bash
primitive functions invoke <key> --input '{"n":21}'    # the REQUEST runtime: run it, print the result
primitive functions start <key> --input '{"n":21}'     # the TASK runtime: start a run, print its id
primitive functions start <key> --wait                 # …and wait for it
primitive functions runs wait <function-id> <run-id>   # wait for a run already started
```

`invoke` and `start` take the **key** (the public route's argument); `runs`, `runs wait`, `runs steps`, `runs terminate` and `logs` take the **function id**. Every id a verb prints is followed by the command that takes it, so nothing has to be looked up.

BOTH VERBS work on every function, and the ROUTE is what says which runtime a call means. A function that must not be reached one of the two ways says so in its own code with `assertRuntime`; the refusal comes back as a `failed` invocation with `errorCode: "FUNCTION_RUNTIME_REFUSED"` rather than as a mistake the CLI could have caught. `functions runs` prints the runtime each run used in its RUNTIME column, and `functions logs` prints it per invocation.

| Exit code | Meaning |
|---|---|
| `0` | The invocation completed, or the run you waited for completed |
| `1` | It failed, timed out, was terminated, or the platform refused the call |
| `124` | `runs wait` spent its budget with the run still going; it prints the resume command |
| `130` | Ctrl-C, same resume line; a second one exits at once |

`--timeout <seconds>` is the request budget for `invoke` (clamped at 30 s by the platform) and the WAIT budget for `runs wait` and `start --wait` (default 900).

**Who it runs as.** By default your own app user — and the output says so, because an admin or owner BYPASSES a function's `access` expression, so your own invocation does not exercise the gate.

| Flag | Identity | Notes |
|---|---|---|
| *(none)* | Your app user | `Ran as <user-id> (<role>)`; the gate is bypassed for an owner or admin |
| `--user <user-id>` | That app user | Mints a ten-minute token, invokes with it, revokes it on every path including Ctrl-C; the value is never printed. Owner and admin only |
| `--as system` | No caller at all | `ctx.user` null, `ctx.trigger` `{ kind: "manual", userId: <you> }`, the app's system authority — the shape a cron or webhook fire has. Owner and admin only |

The two flags are mutually exclusive and `--as` accepts only `system`. A task started with `--as system` is keyed to the system like a trigger-fired root (`FIRED BY manual`, you recorded as the initiator).

**A task start mints your root document.** `functions start` needs a context document and the app user provisioned for an admin has none, so the CLI asks for it through the same idempotent route sign-in uses — `start` works on a fresh app with nothing done first. `--context-doc-id` names one yourself; `--as system` uses the synthetic `fn:<function-id>` context and asks for nothing.

Every verb takes `--app <app-id>` and `--json`.

- `runs steps` is the step-level view of a **task** run: one row per `step.do` / `step.sleep` / `step.waitForEvent`, with its name, its kind (`function.step` / `function.sleep` / `function.event`), status, the idle GAP before it and its duration. Rows are written as the run goes, so an in-flight run is readable — the step executing right now reads `running` with its elapsed time, and a sleeping run's `function.sleep` step stays `running` until it wakes. A resume replays completed steps from the engine's memo: one row per step, keeping the timing of the slice that executed it.
- `runs terminate` stops the run and settles its row `terminated`. Both happen, and the row stays settled. A run whose instance is already gone still settles — that is the case the verb exists for. A run that finished on its own is reported, not overwritten. Whichever door stops a run — this verb or `client.functions.terminate` — the steps its trace left open are closed with it, and asking again on a run that is already over closes anything still open, so a trace left behind by an older stop is repaired rather than restated.
- `disable` refuses invocations and leaves config and code unchanged; pushes still land.
- `archive` takes a function out of service without destroying it: the row keeps its key, pushing code to it is refused, and there is no un-archive. Reclaiming the key is a hard delete — `primitive config pull` (removes the tombstone's local files), then a confirmed `primitive config push --prune`, which destroys the function, every version and their bundles (refused while a run is live — see Deleting a function with live runs).

### Versions and rollback

`primitive functions configs <function-id>` lists a function's versions newest first — config id, when it was pushed, an `*` on the active one, both hashes, the trigger summary and who pushed it; `--json`, `--limit` and `--cursor` page it. `primitive functions activate <function-id> <config-id>` points the function at one of them.

- **Activation is a repoint, not a new version.** Nothing is created and the version list is the same length afterwards.
- **Request calls and trigger fires use the activated version from the next call** — they resolve the active version per call, so there is no window and nothing to restart — and **that version's own capabilities, limits and triggers apply**: a version IS its declaration, so rolling back to one that declared a cron entry a later version dropped makes that schedule live again, and rolling back past a `secret:` capability stops the function reading that secret.
- **A task run already in flight finishes on the version it started on.** Its code is pinned by content hash and reloaded from the same immutable bundle at every wake.
- A trigger is admitted against the **app**, not against the version, so a rollback that asks for triggers the app can no longer take is refused `FUNCTION_TRIGGERS_NOT_ADMISSIBLE` and **nothing moves**.
- Versions are content-addressed per function, so **pushing a tree that matches a version the function already has re-activates that version and makes none** — which is how a rollback is undone without editing anything — and pushing an edited tree makes a new one.
- `config pull` writes the **active** version's bytes and names a newer version when one exists; `config diff` says *the tree matches version `<newest>`, which is not the active one (`<active>`)* rather than reporting an edit nobody made.
- The version each run and invocation executed is on `primitive functions runs`, `primitive functions logs` and the invocation log record; the active version's id and content hash, and who activated it and when, are on `primitive functions get`.

## Related

- [Configuration](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md) — the sync loop that pushes `functions/<key>.toml` and its code.
- [Databases](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) — database types, models and the records surface.
- [Integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md), [Prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md), [App Secrets](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md) — what a function calls out to.
- [Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md) — the CEL the `access` gate is written in.

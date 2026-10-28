# Agent Guide to Primitive Server Functions

Server functions run TypeScript with the app’s authority. Define the entry
point in `functions/<key>.toml` and an `authorize` method beside the handler,
then deploy with `primitive config push`.

Choose `functions.invoke` for a result in the request or `functions.start` for
a background task. Webhooks use requests; cron schedules start tasks. `invoke`
has a wall-clock budget — 5 000 ms by default, 30 000 ms at most — so work that
can exceed 30 000 ms has to be a task.

## The config file

```toml
# functions/greet.toml
[function]
key = "greet"
entry = "functions/greet/index.ts"
```

`key` identifies the function; `entry` is relative to the configuration root.
Add `inputSchema` and `outputSchema` when inputs and results need validation.

## The handler

```ts
// functions/greet/index.ts
import { defineFunction, requireSignedIn } from "primitive-functions";

export default defineFunction("greet", {
  authorize: requireSignedIn(),
  handler: async (input: { name?: string }) => {
    return { message: `Hello ${input.name ?? "world"}` };
  },
});
```

Every version pushed today must define `authorize` beside the handler — a
version with none is refused `FUNCTION_AUTHORIZE_REQUIRED` at push. See
[Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#the-authorize-method).

```bash
primitive config push
primitive functions invoke greet --input '{"name":"Ada"}'
```

| Argument | Purpose |
| --- | --- |
| `input` | Caller input, webhook payload, or cron `rootInput`. |
| `ctx` | App services and caller/trigger context. |
| `step` | Named operations and waits, with saved results in task runs. |

Return JSON data. Read `ctx.runtime` for `request` or `task`, and
`ctx.trigger.kind` to identify what started the function. `ctx.user` is the
original caller, or `null` for a trigger or system invocation. Call
`assertTrigger(ctx, kind)` first to refuse every other trigger and narrow `ctx`
to that one — a tool function starts with `assertTrigger(ctx, "tool")`.

## Pushing

Push bundles dependencies, generates types, checks function code, and activates
a version. It preserves authored sources for `config pull`.

- Keep source imports inside the configuration directory, or list shared
  directories in `functionSources` in `primitive/config.json`
  (`["web/src/engine"]`, relative to the project root; `..` may climb up to the
  git repository root). Push bundles and typechecks shared files with each
  function that imports them and never stores them; `config pull` writes only
  the configuration directory.
- A shared file's imports resolve with its own nearest `tsconfig.json`
  `paths`/`baseUrl`, and must land in a listed directory: a `@/` alias or a
  linked workspace package outside every listed directory is refused at push.
  All other compiler options come from `functions/tsconfig.json` (no DOM); do
  not set `rootDir` there.
- Import npm packages by name. The built bundle is limited to 5 MB.
- Treat generated declarations as build output. Use `primitive functions
  codegen --check` in CI.
- Push executes module-level code while collecting the manifest; only push a
  tree you trust to run locally.
- The server loads each new version once with no network and no credentials.
  Module-level code that throws, awaits I/O, or runs long refuses that
  function's push with `FUNCTION_PROBE_FAILED`; keep such work in the handler.
- A failed function build can leave other configuration changes applied.
- An agent whose tool names a function that failed to build or typecheck, and
  that the app does not hold (never pushed, or archived), is not pushed. Its
  failure row names that function as the cause; fix the function first.

## Invoking

{{ example: functions/invoke }}

The result has `status: "completed" | "failed" | "timeout"`. Check status before
using `output`. A handler failure returns an envelope; a pre-execution refusal
uses an HTTP error. Use `invocationId` to locate its log.

| Option | Use |
| --- | --- |
| `rootInput` | REST input; client methods call this `input`. |
| `contextDocId` | What the invocation concerns, at most 100 characters — typically a database id. A task run is keyed under it. |
| `meta` | Caller metadata, up to 1 KB. |
| `timeoutMs` | Request deadline: 5 000 ms default, 30 000 ms maximum. |
| `runKey` | Deduplicates task starts; ignored by request invocations. A *failed* run keeps its key — retry with a new one. |

### The access gate

New code defines `authorize` instead — see [Access Control: The authorize method](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#the-authorize-method) and its [porting table](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#porting-from-access). What follows is `access`, the CEL rule an already-deployed legacy version still runs; it is frozen and a new or changed version can no longer set it.

`access` controls member HTTP calls and an agent's tool, context and history
calls, which it evaluates as the turn's initiator. Owners and admins bypass it.
Webhooks, cron, and nested starts do not evaluate it; a legacy function
intended only for these paths uses `access = "false"`, never one an agent calls.

Function operations use app authority, not the caller’s data permissions.
Check the caller's access explicitly when acting on a caller-supplied database ID.

### Input and output validation

Every invoke, task start, nested start, trigger and agent tool call is checked
against `inputSchema` before the handler runs. A failed call or start is `400`
`FUNCTION_INPUT_INVALID` with `details` naming each path and rule; a trigger
records a failed run with that code; a tool call gets
`AGENT_TOOL_ARGUMENTS_INVALID`. The handler's result is checked against
`outputSchema` with the same rules; a result that breaks it fails the call or
run with `FUNCTION_OUTPUT_INVALID`.

```toml
[function.inputSchema]
type = "object"
additionalProperties = false
required = ["householdId"]

[function.inputSchema.properties.householdId]
type = "string"
minLength = 1
maxLength = 64
```

Enforced: `type`, `enum`, `const`, `properties`, `required`,
`additionalProperties = false`, `items`, `minLength`, `maxLength`, `pattern`,
`minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`, `multipleOf`,
`minItems`, `maxItems`, `uniqueItems`, `minProperties`, `maxProperties`,
`anyOf`, `oneOf`, `nullable`. In either schema, push refuses `allOf`, `not`, `if`/`then`/`else`,
`patternProperties`, `propertyNames`, `dependentRequired`, `contains`,
`prefixItems`, a schema-valued `additionalProperties` and a list-valued `items`.

## Task runs

Start background work with `functions.start` and read its status or wait for
completion:

{{ example: functions/start-and-wait }}

### Starting and polling a run

Use `ctx.functions.start(key, input, { runKey })` to start a child task from
function code. A repeated key returns the existing run, including a failed
one. Retry failed work with a new key.

Child tasks inherit the original caller and usually the parent’s context id. A legacy child’s `access` rule is not evaluated; the entry function’s
`access` rule controls who may start the work. A child with an `authorize`
method is different: it runs its own `authorize` all the same, at every level
of the chain — naming the door with `allowCallers({ functions: [parentKey] })`
— see [Doors at a glance](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#doors-at-a-glance).

Use `assertRuntime("task")` for handlers that require background execution.
In request mode, `step.do` executes inline, sleeps must fit the request budget,
and `step.waitForEvent` is unavailable.

Check `status` before reading a result. Failed runs resolve with
`status: "failed"`; inspect `error.code` and `error.message`.

Do not build a staleness timeout around a run. One that shows no progress for
30 minutes is ended with `FUNCTION_STALLED`. The 30 minutes start once the open
step's own `timeout` and next retry delay have passed; sleeps, event waits and
retry delays never count.

### Finding a user's runs

List the signed-in user's task runs with `functions.listRuns`, and one run's
recorded steps, oldest first, with `functions.listRunSteps`:

{{ example: functions/list-runs }}

| Option | Use |
| --- | --- |
| `contextDocId` | Runs keyed under this context id (a database id, say). |
| `functionKey` | One function's runs; matched exactly. |
| `status` | `queued`, `running`, `completed`, `failed`, `terminated` or `missing`. |
| `limit` | Page size: 50 by default, 200 at most. |
| `cursor` | The previous page's `nextCursor`, sent with the same filters. |
| `forward` | `true` reverses the order. |

The list holds only the caller's own runs, never trigger-started ones. It is in
storage order, not start order: most recently updated first, or, with
`contextDocId`, by status and then newest first. Sort by `startedAt` for start
order. Over REST, `GET functions/runs` answers `400` for an unknown query
parameter, `GET functions/runs/{runId}/status` reads one run, and
`GET|POST functions/{functionKey}/instances/{runKey}/status|terminate` read and
stop a run by its run key.

### Step discipline

The engine re-executes your handler from the top on every resume and replays
each completed `step.do` from its **stored result** instead of running it again.
Two rules follow, and both matter:

- **Put every side effect inside `step.do`.** A charge, an email, a write made
  outside a step runs again on every resume. Completed step results are reused on resume. Interrupted or retried steps
  may run again, so make their side effects idempotent.
- **Do not branch on anything nondeterministic between steps.** `Date.now()`,
  `Math.random()` or a fresh read at the top level can take a different path
  after a resume, and the engine then looks for steps that are not there. Read
  such values inside a step, so the value is stored with it.

### Reporting back

Choose how the task exposes its result:

- **Return output.** Read it with `getStatus`, `waitFor`, or
  `primitive functions runs wait`. Results remain on the run for 45 days.
- **Save state.** Write records or notifications inside a step for results the
  app should retain. Return what the step wrote — `{ wrote: true, recordId }`
  or `{ wrote: false, reason }` — so the run's output states the outcome
  without a second lookup.
- **Send a live message.** Use `ctx.users.send` for connected clients. Pair it
  with saved state when offline clients need the result later.

Task completion does not automatically send a WebSocket message, and there is
no feature that pushes a run's settlement to the client — the three options
above are it.

- **For "the task's write landed":** don't poll run status for this. A client
  subscribed to the database the task writes receives the write as a live
  update, same as any other write to a
  [synced database](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#synced-databases-on-a-device).
- **For "the run finished":** call `ctx.users.send(ctx.user.userId, …)` as the
  run's last step, after the work it reports on has committed — a connected
  starter gets it immediately. It never fires for a platform failure
  (`ENGINE_*`, `FUNCTION_STALLED`, a reset that exhausts retries), so keep
  `waitFor` as the fallback that still answers when the send doesn't happen.

### Run error codes

A failed run carries a **code** beside its message: `run.errorCode` on the run
row, and `error.code` on the function run status both clients read. It is a
closed set the platform owns, so you can branch on it rather than matching
prose — and it has no slot for an outcome your own code expects.

For that, return a typed result instead of throwing — `{ ok: false, code,
message }`, shaped however the app needs — and read it from `output` like any
other result. Throwing is for what the handler did not expect: an uncaught
throw settles the run `FUNCTION_THREW`. Inside `step.do`, throw only when the
failure should spend one of the step's retries; a returned failure result is
never retried.

For a terminal `ENGINE_*` failure, start a new run with a new `runKey` if the
work should be retried. Keep side effects idempotent because the failed run
may have completed some work. Report repeated platform failures.

| Code | What happened | What to do |
| --- | --- | --- |
| `FUNCTION_THREW` | Your handler threw. | Read `error.message` and the run's logs (`primitive functions logs <function-id> --run <run-id>`). |
| `FUNCTION_NO_HANDLER` | The entry file exported no callable default. | Export a default function from the file `entry` names. |
| `FUNCTION_TIMEOUT` | The invocation did not finish inside its budget. | Shorten the work, or run it as a task (`functions start`) where the budget is per slice. |
| `FUNCTION_INPUT_INVALID` | The input did not match the declared `inputSchema`. | Fix the caller's input, or the schema. |
| `FUNCTION_OUTPUT_INVALID` | The output did not match `outputSchema` — on an invoke, a webhook delivery's run or a task run. | Fix the returned value, or the schema. The message names the failing paths. |
| `OUTPUT_NOT_SERIALIZABLE` | The returned value is not JSON. | Return plain data — no functions, no cycles, no `undefined` where a value is required. |
| `OUTPUT_TOO_LARGE` | The output exceeds the 1 MiB platform ceiling. | Page the result, or write it to a database and return a handle. |
| `FUNCTION_DELETED` | The function was hard-deleted while the run was starting. | Nothing to retry: the stored bundle the run pinned is gone. |
| `FUNCTION_RUNTIME_REFUSED` | The code refused the runtime it was handed (`assertRuntime`), or a request invocation's `step` was asked for a wait its budget cannot cover. | Call it the other way — `invoke` for a request, `start` for a task. |
| `FUNCTION_STALLED` | The run showed no progress for 30 minutes past its open step's own `timeout` and retry delay, so the platform ended it. The message names the step. | Give the step's awaits a timeout of their own, or split the step. A step that needs longer sets its own `timeout` in `step.do`. Start a new run with a new `runKey` if the work still needs doing. |
| `ENGINE_ISOLATE_EVICTED` | The execution environment was reset. | Retry with a new run key and idempotent steps. |
| `ENGINE_CODE_UPDATED` | A platform update interrupted execution and retries were exhausted. | Retry with a new run key and idempotent steps. |
| `ENGINE_STORAGE_ERROR` | Task storage failed. | Retry with a new run key and idempotent steps. |
| `ENGINE_INTERNAL_ERROR` | The task runtime reported an internal error. | Retry with a new run key and idempotent steps. |
| `ENGINE_SLICE_DEADLINE` | A platform call was refused because the slice's token deadline had passed. | The slice outlived its token. Look for a step that runs longer than the slice ceiling and split it. |
| `ENGINE_INSTANCE_LOST` | The engine no longer reports the instance, and nothing recorded how the run ended. | `primitive functions logs <function-id> --run <run-id>` shows what the platform did record. Start a new run with a new `runKey` if the work still needs doing. |

An engine failure's `errorMessage` is the **engine's own text**, not a
code-leading sentence: for these the code is on `error.code` and
`run.errorCode`, never in the message's prefix.

An attempt whose engine died before the platform could write its
invocation record leaves no record at all — the writer died with it. For such
an attempt the run row's code is the whole statement; there is nothing in
`functions logs` to read.

### Locks

Wait for a named lock with `ctx.locks.withLock(key, { ttlMs, timeoutMs, owner? }, fn)`: it acquires, runs `fn(handle)` while the key is held, and releases in `finally` — on a return and on a throw. It answers `fn`'s value; `fn`'s own error propagates unchanged after the release.

```ts
import { defineFunction, requireSignedIn } from "primitive-functions";

export default defineFunction({
  authorize: requireSignedIn(),
  handler: async (input: { itemId: string }, ctx, step) => {
    return ctx.locks.withLock(`ingest:${input.itemId}`, {
      ttlMs: 600_000,
      timeoutMs: 360_000, // task runtime; a request must fit its remaining budget
    }, async () => {
      await step.do("ingest", async () => { /* work that must not overlap */ });
      return { ingested: true };
    });
  },
});
```

- **Not acquired within `timeoutMs`:** `withLock` throws an error with `name` and `code` `LOCK_TIMEOUT`, its `details` carrying `owner`, `holderKind`, `leaseExpiresAt` and `rateLimited`; `fn` never runs. It settles `FUNCTION_THREW` unless caught.
- **To branch on contention instead**, call `ctx.locks.acquire(key, options)` and `ctx.locks.release(handle)` in a `try`/`finally`. `acquire` never throws for "not acquired": it answers `{ acquired: true, handle }` or `{ acquired: false, timedOut: true, rateLimited, heldBy, holderKind, owner, leaseExpiresAt }`. `release` answers `{ released: true }` or `{ released: false, reason: "not_holder" | "not_held" }`.
- **The owner defaults to `ctx.runId`**, so a run never waits on its own hold: a resumed slice, or a second call on a key the run already holds, re-takes it at once with a **fresh handle**, and the previous handle is fenced (`release` answers `not_holder`, `renew` answers `lease_lost`). The match is the same principal, the same kind of caller and the same owner string — so a member who started the task cannot rotate or release the lease their function holds. When a run key coalesces several triggers into one run, pass `owner: ctx.trigger.runKey ?? ctx.runId`. Never a static string: two unrelated runs presenting one would re-enter each other's hold. A nested `withLock` on the same key re-enters, and its release frees the outer hold: use distinct keys or one wrapper.
- **Cost.** Each attempt asks the platform to wait server-side for up to 30 seconds and answers the moment the key frees: no CPU, one platform call and one acquire attempt per half-minute of waiting. `timeoutMs` is capped at 1 hour (a `TypeError` above it); for longer coordination put a `step.sleep` between `acquire` calls. `ttlMs`, `timeoutMs` and the key are validated at the call (`TypeError` naming the option).
- **Task runtime:** each attempt is a step, so the wait survives a reset or a deploy and the slice clock is refreshed at every attempt. It does not hibernate: the engine suspends nothing shorter than five minutes.
- **Request runtime:** the whole `timeoutMs` must fit the invocation's remaining budget, or the call fails `FUNCTION_RUNTIME_REFUSED` before any attempt. A webhook delivery has 5 seconds in all; hand a longer wait to a task with `ctx.functions.start`.

The one-shot calls stay available as `ctx.api.locks.tryAcquire` / `renew` / `release` / `status`; `tryAcquire` takes `waitMs` for the same server-side wait. Under the task runtime make those raw calls **live, outside `step.do`** — a memoized acquire hands a resumed slice the handle a previous slice was given, and that handle no longer works. `ctx.locks` does this itself. `status.owner` tells your own hold from another's.

### What resets a running task

A running task keeps the function version it started with. Pushing or
activating another version does not change that run.

The platform can interrupt a task during maintenance or a runtime reset.
It resumes the handler from the beginning, reusing completed step results.
The interrupted step may run again and consume another retry attempt.

Make step bodies idempotent, and return values from steps instead of assigning
them to outer variables. Cleanup in a `catch` must also be safe to repeat:
an interruption does not necessarily mean the run has failed permanently.

Read the run’s final status to determine its outcome. Use `functions runs
steps` to see progress; `slice.resets`, `lastResetAt`, and `lastResetCause`
report interruptions.

### Budgets in a task run

A task executes in **slices**. After a pause, the handler resumes from the
beginning and reuses completed step results.

- A slice begins with 10 minutes of wall-clock time. The platform refreshes
  that budget at step boundaries so each step begins with at least 5 minutes.
  A sequence of steps can continue for up to 12 hours before a pause.
- If refresh is unavailable, the task pauses for a little over 5 minutes and
  resumes with a fresh slice.
- Each slice allows `subRequests` of up to 10 000 and 5 minutes of active CPU.
  These budgets do not refresh. The platform pauses at a step boundary after
  half the subrequest allowance is used; exceeding CPU fails the run.
- Keep individual steps within their budgets. A single long operation outside
  `step.do` receives no wall-clock refresh.

Use steps to divide work and `step.sleep` when the task needs to wait. For
CPU-heavy work, a sleep longer than five minutes starts a fresh slice after
hibernation. Output limits and `outputSchema` apply to the final result.

The run status includes a `slice` block with timing, refresh counts and
progress: `lastProgressAt` is when the run last did something, such as
starting or finishing a step; `openStep` and `openStepStartedAt` name the step
it is inside (`charge#0` is the first `step.do("charge", …)`); `parkedAt` is
set while it waits. `functions runs` shows `REFRESHES` and `LAST PROGRESS`.

### A function an agent uses

Archive, hard delete (including `config push --prune`) and a schema change an
agent cannot take are refused with `409 FUNCTION_USED_BY_AGENT` while a
non-archived agent names the function; `details.agents` lists each agent and
its uses. Change the agent in the same push (push applies it first), or retire
it. If the changed agent also names a function the push creates, run
`primitive config push --only function/<new-function-key>` first, then the push.

### Deleting a function with live runs

Hard deletion is refused while runs are active. Archive to stop new work while
existing runs finish. Wait for or terminate those runs before pruning.

## Execution identity

A function can access all data in its app. The caller’s identity provides
attribution; it does not limit the function’s authority. Filter records and
validate resource access in your code when the result must be caller-scoped.

### Checking a user's access

A database id in the input is a claim, not a permission: `ctx.db3` reaches
every database in the app. Gate in `authorize` with the caller's effective
level, `auth.db3(id, type).permissions()` → `"reader" | "read-write" | "owner" | null`:

```ts
authorize: async (input: { db3Id: string }, auth) => {
  if (!auth.user) return deny("FORBIDDEN");
  const level = await auth.db3(input.db3Id, "project").permissions();
  return level === "read-write" || level === "owner" ? allow() : deny("FORBIDDEN");
},
```

For another user, or from the handler: `ctx.api.db3.validateAccess({ db3Id, body: { userId } })` →
`{ hasAccess, permission?, accessSource? }` (`accessSource`: `owner`, `grant`, `group`, `link`, `app`, or `resource` when a platform resource such as an agent session holds the database and its members read it).
Omitting `body.userId` describes the function's own app authority, not the caller — always pass it.
Both resolve grants, groups, collections, link and app access exactly as the database enforces them; an app role
is not a content grant. A sync-off database answers no access for every member — gate those on
records your app keeps (a roster row read through `auth.db3(...).model(...)`).

### Which user a call is about

A call about one user takes `userId`. Omitted, it means the caller
(`ctx.user`). Only an absent `userId` means the caller: an empty, `null` or
non-string `userId` throws `SUBJECT_USER_INVALID` (400), so omit it rather than pass `""`. A run with no caller — cron, a webhook, `--as system`, or a task
started from one — must name the user, or the call throws
`FUNCTION_SUBJECT_REQUIRED` (400):

```ts
// A cron fire has no caller: name the member.
const { unreadCount } = await ctx.api.notifications.unreadCount({ userId: member.userId });
const page = await ctx.api.collections.list({ userId: member.userId });
```

| Calls | `userId` |
| --- | --- |
| `collections.list`; `notifications.list`, `unreadCount`, `markRead`, `markAllRead`, `listDevices`, `registerDevice`, `unregisterDevice`; `ctx.agents.*` | Optional; defaults to the caller |
| `collections.listForUser`, `groups.listUserMemberships` | Required |
| App-wide reads (`groups.list`, `users.list`) | None |
| Calls that hand something to a user (`notifications.send`, `users.setProfile`, `ctx.channels.authorize`) | Names the recipient; not restricted below |

Naming a user other than the caller, with an optional or a required `userId`:

| Run | `userId` omitted | Another member |
| --- | --- | --- |
| No caller | `FUNCTION_SUBJECT_REQUIRED` | Allowed |
| Invoked by an app owner or admin | The caller | Allowed |
| Invoked by any other member | The caller | `SUBJECT_USER_FORBIDDEN` (403) |
| Invoked by any other member, function declares `users:readAny` / `users:actAsAny` | The caller | Allowed on reads / on calls acting as them |
| A tool, context or history run, and anything it starts | The initiator | `SUBJECT_USER_FORBIDDEN` (403) |

The role is read at each call, so a task sees a role change made while it
slept. A named user who is not a member throws `SUBJECT_USER_NOT_FOUND` (404)
on the optional-`userId` calls; the required-`userId` reads keep their own
not-found answers.

Gotchas:

- **A subject grant reaches any member the code names.** A function holding
  `users:readAny` or `users:actAsAny` lets every member who may invoke it name
  anyone; take the user id from your own data, not from input.
- **Writes toward a named user are not restricted.** `users.setProfile`,
  `notifications.send` and `ctx.channels.authorize` accept any member, so a
  tool that passes a model-supplied user id to them changes that user's data.
  Check the id before passing it.
- **`ctx.channels.authorize` keeps its own code.** With no caller and no
  `userId` it throws `FUNCTION_CHANNEL_GRANTEE_REQUIRED`.

## `ctx.api` — the platform from inside a function

Use typed `ctx.api` methods for app services. Methods take an options object
with path/query parameters and `body` where needed. Use `ctx.prompts.run` and
`ctx.integrations.call` for prompt and external-service calls.

Direct internet `fetch` calls are denied. Missing capabilities return
`FUNCTION_INTEGRATION_GRANT_MISSING`, `FUNCTION_SECRET_GRANT_MISSING`, or
`FUNCTION_HIGH_BLAST_GRANT_MISSING`.

A single `$in` or `$nin` list is limited to 1 000 values
(`QUERY_IN_LIST_TOO_LARGE`). How to split a longer list depends on the
operator — see the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#limits).

## Capabilities

`capabilities` is an array of exact strings. No wildcards, no case folding; a repeated string is the same string. Only four kinds exist:

| Kind | Strings | Unlocks |
|---|---|---|
| Integration | `integration:<key>` | `ctx.integrations.call(key, …)` — checked at push against an active integration |
| Secret | `secret:<NAME>` | `ctx.secret(NAME)` — grammar-checked at push; the value is provisioned per environment |
| High-blast | `users:setRole`, `blobBuckets:createBucket`, `blobBuckets:deleteBucket`, `db3:setSync`, `db3:transfer`, `db3:setOwnerRow`, `db3:setAppAccess` | The operation of the same name on `ctx.api`, which is otherwise refused `FUNCTION_HIGH_BLAST_GRANT_MISSING` |
| Naming other members | `users:readAny`, `users:actAsAny` | In a run a plain member invoked, naming another member: `users:readAny` on the user-subject reads (memberships, a member's collections, documents, root document, inbox, devices, agent sessions, user aliases, holdings); `users:actAsAny` on calls that act as them (notification read and device calls, `ctx.agents` session calls other than `listSessions`, `ctx.db3` `asUser`, setting or deleting their aliases). Never in an agent-driven run, never for `custodian: "app"`; a nested start uses its own version's grants |

**Nothing else is declared.** Database records, prompts, config vars, channels, sends, email and analytics writes are admitted with no declaration, because the code acts as the system.

Push checks capability syntax and integration existence. Secret values are
provisioned separately; an unset value returns `FUNCTION_SECRET_NOT_FOUND`.

The platform reads capabilities from the TOML file you review in a pull request, so what a reviewer sees and what the platform enforces are one statement.

### What a function touches is documented, not declared

`functions get` shows the version’s manifest of used models, queries, and
services. It supports review and does not control access.

## Calling an integration

Declare `integration:<key>` in capabilities, then call
`ctx.integrations.call(key, { method, path, query, body })`. The integration
supplies credentials and restricts the destination, methods, and paths.

Check the response’s `errorCode` before using `body`. Calls are bounded by the
function’s remaining time. Each call is logged with its run and, inside
`step.do`, its step (`primitive integrations logs <integration-id> --run <run-id> --step <name>`).
See [Integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md) for configuration
and examples.

## Running a prompt

Call `ctx.prompts.run(key, { variables })`; no capability is needed. Check
`success` before reading `output` or `parsed`. Declare `[prompt.outputSchema]`
for validated, typed `parsed` results. See
[Prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md#typed-output).

## Secrets and config vars

Use `ctx.secret("KEY")` with `secret:KEY` in capabilities. Use
`ctx.configVar("KEY")` for non-secret settings without a capability.

Secrets refresh through a short cache, usually within about a minute. A
running function version caches each config var it reads and keeps that
value until a new version is pushed (any change to its code or TOML) or its
running instance is recycled on its own — there's no fixed expiry, so one
test invocation after changing a var is not proof every invocation sees the
change. Never log credentials. See [App Secrets](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md).

## Sending to a connected client

`ctx.users.send` and `ctx.connections.send` push a message to a client that is connected **right now**; no capability is declared. The recipient receives a `direct.message` frame carrying the payload verbatim plus the sending function's key.

```ts
const result = await ctx.users.send(input.userId, { kind: "order-ready", orderId: "o-1" });
return { delivered: result.connections, truncated: result.truncated };
```

- **Presence is not guaranteed and there is no durable record.** A user with no connected client answers `{ connections: 0, truncated: false }` — a success. A client that was offline never receives the frame later; use a notification when the message must survive being missed.
- **One socket, one frame.** A client with several databases open is one connection, delivered once.
- **Fan-out is bounded** at 64 unique connections per user, and the lookup behind it is bounded too; past either, the send delivers what it found and answers `truncated: true` — "there may be more", not an error.
- **Payloads are capped at 64 KiB** serialized (`FUNCTION_SEND_PAYLOAD_TOO_LARGE`; nothing delivered).
- `ctx.connections.send(connectionId, payload)` addresses one connection. An id from another app answers exactly as an id that never existed (`FUNCTION_SEND_TARGET_NOT_FOUND`).

On the client, listen for the `directMessage` event:

{{ example: functions/direct-message }}

{{#lang swift}}
`DirectMessageEvent` carries `payload` (a `JSONValue?`, `nil` when the function sent none), `functionKey` and `sentAt`. It needs no subscription and no grant — the frame arrives on the app socket — and a frame that arrives with no listener is inert.
{{/lang}}

## Channels

Authorize a user with `ctx.channels.authorize(channel)` and return its `grant`
and `expiresAt`. The caller subscribes using that grant:

{{ example: functions/subscribe-channel }}

Publish with `ctx.channels.publish(channel, payload)`. Any function in the app
can publish; no capability is required.

- Grants belong to a specific user. Triggered functions must pass `{ userId }`
  because they have no caller.
- A grant expires after 300 seconds by default, up to 900. Renew by invoking
  the authorizing function and subscribing with the new grant before expiry.
- Expiry stops delivery even on an open connection. On reconnect, handle
  `channelSubscribeFailed` by obtaining a fresh grant.
- Channel names contain letters, digits, `_`, and `-`, separated by `:`;
  the maximum length is 200 characters.
- Publish returns `{ connections, truncated }`, with a limit of 500 recipient
  connections. Zero recipients is a successful send.

Check the caller’s access to the resource before authorizing its channel.

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

`step.do`'s config parameter is a `StepConfig` — `{ retries?: { limit, delay, backoff? }, timeout? }`, the step configuration, which each `stepPolicy.<family>` is one of. The shape is closed: a key outside it, such as `retry` for `retries`, fails the `config push` typecheck rather than being carried along and ignored.

## Database records

`ctx.db3(db3Id, "<type>").model("<model>")` reads and writes one model of one database:

```ts
const orders = ctx.db3(input.db3Id, "orders").model("orders");
const open = await orders.query({ filter: { status: "open" }, sort: { createdAt: -1 }, limit: 50 });
await orders.patch(open.items[0].id, { data: { status: "picked" }, condition: { status: "open" } });
```

The handle's contract:

- **App authority.** Any database in the app, synced or sync-off, whoever holds it. No grant, no capability. Check the caller in `authorize` when the caller chose the database (see [Checking a user's access](#checking-a-user-s-access)).
- **Typed from `db3-types/`.** `config push` (or `primitive functions codegen`) writes `functions/primitive-db3-types.d.ts`; the type argument accepts only declared types, `.model()` only that type's models, and rows come back typed with `id: string`. The type argument is compile-time only; the server enforces the database's actual type (`DB3_MODEL_NOT_DECLARED`).
- **Untyped and as a member.** `ctx.db3(id)` with no type opens a database created without one (`.model("<key>")` over untyped rows); `ctx.db3(id, type, { asUser: userId })` judges every call by that member's grant instead of the app's authority (a sync-off database refuses them; naming anyone but the caller needs `users:actAsAny`).
- **Verbs:** `get`, `query`, `count`, `aggregate`, `save`, `patch`, `delete`, `increment`, `bulk`. `ctx.db3(id, type).bulk([...])` spans models, each op naming its `model`.
- **Refusals reject:** `DB3_CONDITION_NOT_MET` (409, a failed `condition`), `DB3_RECORD_NOT_FOUND` (404, `increment` on a missing record), `DB3_MODEL_NOT_DECLARED` (400, a model the type does not declare).
- **Retries:** a write that timed out may have applied; resend it with the same `requestId` and it applies once.
- **Paging:** `query` answers 100 records when `limit` is omitted and takes a `limit` up to 10,000 (larger is refused, never clamped); a page stops before 8 MiB of records. Loop on `hasMore`, passing `nextCursor` back as `cursor` with the same `sort`.
- **Includes:** `include` nests up to 3 levels; a `hasMany` include reads at most 1,000 related records per request (`DB3_PAGE_TOO_LARGE` past either budget).

Every operation's options and server-function examples are in the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#records). Creating a database, transferring it, and switching its sync or app access go through `ctx.api.db3` with the `db3:*` capabilities — see [Access](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#access).

### A member's collections and profile

Triggered functions have no caller, so name the member (see
[Which user a call is about](#which-user-a-call-is-about)).
`ctx.api.collections.listForUser({ userId })` pages the collections a member
belongs to, with `permission`; loop on `hasMore`, since a page can be short or
empty while `hasMore` is `true`. A non-member throws 404. No capability gates
it: a run with no caller, or one an app owner or admin invoked, can list any
member's, so keep such a function admin-only or trigger-only.

`ctx.api.users.setProfile({ userId, body: { name, avatarUrl } })` sets a
member's display name and avatar (`avatarUrl` is an `http:` or `https:` URL;
`null` clears it).

## Ceilings

Every function runs under fixed platform ceilings. `[function.limits]` may
**lower** any of them and can never raise one — a config asking for more simply
gets the platform value.

The two runtimes differ in what they are allowed to spend, which is most of
why the runtime matters at all:

| Limit | Request runtime | Task runtime | Key |
| --- | --- | --- | --- |
| CPU | 5 000 ms per invocation | 5 minutes per slice | `cpuMs` (see below) |
| Outbound subrequests | 128 | 10 000 per slice | `subRequests` |
| Wall clock | 5 000 ms default, 30 000 ms ceiling | continuous up to 12 hours; every step begins with at least 5 minutes | `timeoutMs` on the request (request only); see Budgets in a task run |
| — a `step.sleep` has to fit the budget | past it, the invocation fails `FUNCTION_RUNTIME_REFUSED` | hibernates; hours and days are fine | — (start the function as a task) |

And the ones that are the same under both:

| Limit | Platform value | Key |
| --- | --- | --- |
| Invocations per minute, per function | 1 200 | `ratePerMinute` |
| Response size | 1 MiB, kept on the run row | — |
| Values per `$in` / `$nin` list | 1 000 | — |


Use request `timeoutMs` for a precise wall-clock bound. `cpuMs` includes a
runtime allowance of roughly two seconds. A timeout denies further platform
calls but does not undo completed side effects.

## Triggers

```toml
# functions/stripe-events.toml
[function]
key = "stripe-events"
entry = "functions/stripe-events/index.ts"
# authorize: allowCallers({ webhook: true, cron: true }) in the handler — trigger-only, no member calls it over HTTP

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

Webhooks run in request mode after signature verification; cron schedules start
tasks. Both have `ctx.user = null`. Use `ctx.functions.start` to hand long
webhook work to a task, with an idempotent run key.

An app can have up to **50 webhooks**. A function can declare one webhook and
up to ten cron entries. `overlapPolicy = "skip"` skips a scheduled invocation
while the previous run remains active, including while it sleeps.

Operate triggers with `primitive functions triggers`, by function key (and a
cron entry's name):

| Verb | Does |
| --- | --- |
| `events <key>` | The webhook's delivery log. |
| `test <key> [--deliver]` | Signs a test delivery; `--deliver` posts it and runs the function. |
| `verify <key> --header … --body-file …` | Checks captured provider bytes without running code. |
| `rotate-secret <key> --secret "{{secrets.NEXT}}"` | Moves the signing secret, keeping the old one for the grace period. |
| `fire <key> <name>` | Starts a cron entry's task run now. |
| `disable <key> <name>` / `enable <key> <name>` | Takes an entry out of service and back. `enable` also resumes an entry a hard failure paused. |

### Database change triggers

```toml
[[function.triggers.db3]]
type = "orders"                        # the db3 type to watch; at most ten per function, each named once
```

Each batch of writes a database of that type accepts — a device's synced writes, a function's or the CLI's records call — starts one task run. The input is `{ db3Id, type, from, to, principal, changes: [{ model, id, kind }] }` (`kind` is `create`, `update` or `delete`): keys, not values — read records through `ctx.db3(input.db3Id, input.type)`. `ctx.trigger` is `{ kind: "db3", db3Id, type, triggerId, from, to }`; the run is keyed under the database id and has no caller. Admit the door in `authorize` with `allowCallers({ db3: true })`.

- **A watcher that writes the database it watches re-triggers itself**, bounded only by `ratePerMinute`. Inspect `input.changes` and skip what your own writes produced.

### Webhook verification schemes

Choose `stripe`, `github`, `slack`, `discord`, `jwt`, `plaid`, or `custom`.
Shared-secret schemes require a whole `{{secrets.KEY}}` reference. Public-key
schemes use their verification settings instead.

Rotate to a second key with `primitive functions triggers rotate-secret`; the
old key remains valid during `secretGracePeriodMs` (24 hours by default, up to
30 days).

Duplicate signed events do not run the function again. GitHub and custom
signatures without freshness timestamps keep duplicate entries permanently.
Make handlers idempotent because a re-signed logical event can be new.

`triggers test` previews a signed payload. Add `--deliver` only when you intend
real side effects. `triggers verify` checks captured provider bytes without
running code. Find the receiver URL in `functions get`.

A provider that confirms an endpoint first (Slack `url_verification`, Discord
PING) is answered by a handshake rule. `allowedIpCidrs` refuses deliveries
from any other IPv4 address or prefix, before verification; a handshake rule
runs after verification, in turn.

A handshake `when` is a request matcher, not an access rule: by the time it
runs, the IP allowlist and signature verification have already decided
whether the delivery is admitted, so it doesn't authorize anything. It sees
no caller — no `user`, no memberships, no `md`, none of the access-control
identity context — only the shape of this request: `body`, `query`,
`headers`, `rawBody`. The first rule that matches answers with its `response` and the
function does not run.

```toml
[function.triggers.webhook]
verificationScheme = "slack"
signingSecret = "{{secrets.SLACK_SIGNING_SECRET}}"
allowedIpCidrs = ["203.0.113.0/24"]

[[function.triggers.webhook.handshakeRules]]
when = "body.type == 'url_verification'"   # CEL over this request's body, query, headers, rawBody — no caller
response = { status = 200, contentType = "text/plain", body = "{{ body.challenge }}" }
```

Removing a webhook block stops delivery; restoring it resumes the same URL.
Removing a cron entry cancels the schedule.

## Recording

HTTP calls create `function.invoke` analytics events attributed to the caller.
Use logs for diagnostics and run records for tasks and triggers.

## Debugging a failing function

```bash
primitive functions logs <function-id> --invocation <invocation-id>
primitive functions logs <function-id> --run <run-id>
primitive functions runs steps <function-id> <run-id>
```

Logs include output, errors, version, and timing, with step and attempt labels
where applicable. The attempt counts body runs since the task last paused and
restarts at 1 after a pause; integration calls made in the body carry the same
attempt. Owners and admins can read them for seven days.

A shortened record ends with
`… truncated: the platform's per-record cap dropped later lines`.

- Pre-execution refusals have no invocation log; inspect the HTTP error. An
  `authorize` denial is different: it writes no run row, but the call's
  invocation log records it `denied`.
- Task logs contain output from the finishing slice. Output before earlier
  pauses is unavailable. Store important progress as data.
- Never log credentials. Redaction cannot recognize transformed values;
  missing declared secrets suppress log content entirely.

## Operating

```bash
primitive functions list
primitive functions get <function-id>
primitive functions runs <function-id>
primitive functions runs steps <function-id> <run-id>
primitive functions runs terminate <function-id> <run-id>
primitive functions disable <function-id>
primitive functions enable <function-id>
primitive functions archive <function-id>
```

Disable stops new calls; pushing code does not re-enable the function.
Archive preserves versions for existing runs. Prune permanently deletes the
function and is refused until its runs finish or are terminated.

### Running one

```bash
primitive functions invoke <key> --input '{"n":21}'
primitive functions start <key> --input '{"n":21}' --wait
primitive functions runs wait <function-id> <run-id>
```

`invoke` and `start` take the key; inspection commands take the function ID,
which `primitive functions list` prints beside each key.
`--timeout <seconds>` bounds an invoke request or a CLI wait; waits default to
900 seconds. A wait timeout does not stop the task.

| Exit code | Meaning |
|---|---|
| `0` | The invocation completed, or the run you waited for completed |
| `1` | It failed, timed out, was terminated, or the platform refused the call |
| `124` | `runs wait` spent its budget with the run still going; it prints the resume command |
| `130` | Ctrl-C, same resume line; a second one exits at once |
| Flag | Identity | Notes |
|---|---|---|
| *(none)* | Your app user | `Ran as <user-id> (<role>)`; a legacy `access` gate is bypassed for an owner or admin — an `authorize` method is not |
| `--user <user-id>` | That app user | Mints a ten-minute token, invokes with it, revokes it on every path including Ctrl-C; the value is never printed. Owner and admin only |
| `--as system` | No caller at all | `ctx.user` null, `ctx.trigger` `{ kind: "manual", userId: <you> }`, the app's system authority — the shape a cron or webhook fire has. Owner and admin only |


`--user` and `--as system` are mutually exclusive. Use a member identity when
testing a legacy `access` rule, because owners and admins bypass it; an
`authorize` method gives them no such bypass, so invoking it as yourself
already exercises it.

### Versions and rollback

List with `functions configs`; select with
`functions activate <function-id> <config-id>`. New calls use that version’s
code and configuration. Existing tasks keep their starting version.

## Gotchas

- Check a signed-in caller’s access to a database they name explicitly;
  function code has app authority. Triggers have no caller.
- Completed task steps reuse results, but interrupted attempts can run again.
  Keep side effects idempotent and nondeterministic reads inside `step.do`.
- Reusing a run key returns the existing run, even after failure. Retry with a
  new key.
- A timeout blocks later platform calls; it does not undo completed writes.
- A push can partially apply after local validation succeeds. Inspect errors
  and re-run after resolving them.
- Do not hard-delete a function with running tasks; archive it first.

## Related

- [Configuration](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md) — the sync loop that pushes `functions/<key>.toml` and its code.
- [Databases](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) — types, records, synced databases and access.
- [Integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md), [Prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md), [App Secrets](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md) — what a function calls out to.
- [Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md) — the `authorize` method every version defines, and the CEL a legacy version's `access` gate is written in.

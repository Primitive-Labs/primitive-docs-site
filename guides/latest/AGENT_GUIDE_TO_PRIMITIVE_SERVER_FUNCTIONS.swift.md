# Agent Guide to Primitive Server Functions

A **server function** is TypeScript authored in the app's config tree and pushed with `primitive config push`. It runs on the platform — never on the device, whatever language the app's clients are written in — under the invoking user's authority plus the capability grants its TOML declares, and answers over HTTP. It is config-as-code beside [prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md) and [integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md): `functions/<key>.toml` states the gate, the entry point and the grants; the code sits next to it; one push builds and ships both as one immutable version.

Two modes, one flag:

| Mode | `durable` | What an invocation answers | Run row |
|---|---|---|---|
| **Synchronous** (default) | `false` | The result envelope, inside the request | None — nothing to poll or clean up |
| **Durable** | `true` | A run id, immediately; the run can sleep for hours or days | One per run, polled by run id |

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
runAs = "caller"                    # caller (default) | system — see Execution identity
durable = false                     # true → a durable run instead of a synchronous answer
capabilities = []                   # grants — see Capabilities

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

export default defineFunction(async (input: { name: string }, ctx, step) => {
  return { message: salute(input.name) };
});
```

| Argument | What it is |
|---|---|
| `input` | The request's `rootInput`, validated and coerced against `inputSchema` when one is declared. On a trigger fire: the verified delivery body (webhook) or the entry's `rootInput` (cron). |
| `ctx` | The invocation context: `user`, `trigger`, `api`, `db`, `integrations`, `prompts`, `secret`, `configVar`, `users`, `connections` — each documented below. |
| `step` | Present in **both** modes. In a durable run it is the live engine step (see Durable functions). In a synchronous invocation it is a passthrough: `step.do` runs its body, `sleep`/`sleepUntil` resolve at once, and `waitForEvent` throws `STEP_NOT_AVAILABLE` — so a durable-authored body can be exercised synchronously. |

- The return value is JSON-serialized as the envelope's `output` and validated against `outputSchema`. A throw settles the invocation with `status: "failed"`, `errorCode: "FUNCTION_THREW"` and the thrown message.
- `defineFunction` is typing sugar over the same contract: `export default async function (input, ctx, step) {}` is equivalent.
- `primitive-functions` is the one import the bundle leaves external: the platform supplies it at invoke time, so a function always runs against the SDK the platform is running. The npm package is a stub whose every export throws — a function only ever runs inside a push.

### `ctx.user` and `ctx.trigger`

| Door | `ctx.trigger` | `ctx.user` |
|---|---|---|
| HTTP invoke | `{ kind: "http" }` | `{ userId, email }` — the caller |
| Webhook fire | `{ kind: "webhook", webhookKey, webhookId, externalEventId }` | `null` |
| Cron fire | `{ kind: "cron", name, triggerId, scheduledFor }` — plus `manual: true` on a diagnostic fire | `null` |

## Pushing

`config push` builds the bundle with esbuild, inlines every npm dependency, and stores the built bundle **together with** the authored TOML bytes and every source file it read.

- Every push that changes anything creates a new **immutable version** and repoints the function at it. "Anything" is the whole envelope: a comment-only or line-ending-only edit makes a new version, and `config pull` on a fresh clone hands back the bytes you wrote.
- **Relative and absolute imports must stay inside the config tree.** An entry or import that resolves outside it — through a symlink included — is refused at push, so a push can never upload a file the tree does not contain.
- **npm packages are imported by name and inlined.** They resolve through Node like any other import; `node_modules` content is never stored as source or written back by `config pull`.
- A missing entry file or a build error names the file and the message, and nothing is pushed for that function — the rest of the tree still applies.
- The built bundle is capped at **5 MB**.
- **Push runs your module scope.** Collecting the registered-query manifest (see Registered queries) means building the bundle a second time with `primitive-functions` replaced by a recording stub and executing it in a separate Node process — scrubbed environment, the config tree as working directory, hard timeout. It is the trust you extend by running a project's own build; know it before pushing a tree you did not write. A push whose collection fails is refused naming the error: the manifest is part of the version's identity.
- Grants are validated before anything is applied: a malformed grant, a database type the app does not have, a model a schema-declaring type does not declare, or an unknown capability family refuses the push. Database types are applied before functions, so a tree that adds a type and a function granted on it lands in one push.

## Invoking

```
POST /app/{appId}/api/functions/{key}
```

Any signed-in member may call it, subject to the function's `access` gate. Every request field is optional:

| Field | Meaning |
|---|---|
| `rootInput` | The value the handler receives as `input`. |
| `contextDocId` | A document the invocation is about. On a durable start it defaults to the caller's root document and is required when there is none. |
| `meta` | Caller metadata, at most 1 KB encoded. |
| `timeoutMs` | Wall-clock budget. Default 5 000 ms; clamps silently to the 30 000 ms ceiling. Ignored on a durable start. |
| `runKey` | Durable only — idempotency key (see Durable functions). On the synchronous path it is validated and otherwise ignored, since there is no run row to deduplicate against. |

**The synchronous envelope** — always HTTP `200`, because the call reached the function and the function has an answer:

```json
{ "status": "completed", "output": { "message": "hello Ada" } }
```

| `status` | Meaning |
|---|---|
| `completed` | The handler returned; `output` is its value. |
| `failed` | The handler threw (`FUNCTION_THREW`), returned a value JSON cannot carry (`OUTPUT_NOT_SERIALIZABLE`), exported no callable default (`FUNCTION_NO_HANDLER`), returned output `outputSchema` refuses (`OUTPUT_SCHEMA_VIOLATION`), or returned over 1 MiB (`OUTPUT_TOO_LARGE`). `errorCode` says which; `error` carries the message. |
| `timeout` | The handler did not finish inside the budget. |

**HTTP errors** mean the platform refused before your code ran:

| HTTP | `errorCode` | Cause |
|---|---|---|
| `400` | `INVALID_META` | `meta` over 1 KB encoded. |
| `400` | `INVALID_RUN_KEY` | A `runKey` carrying a reserved character sequence. |
| `400` | — | `rootInput` failed `inputSchema`; `error` is `Input schema validation failed` and `details` lists the violations. |
| `400` | `CONTEXT_DOC_REQUIRED` | A durable start with no `contextDocId` from a caller who has no root document. |
| `403` | `FUNCTION_ACCESS_DENIED` | The `access` gate denied the caller. |
| `403` | `FUNCTION_DISABLED` | The function is disabled — seen only by callers the gate admits. |
| `404` | `FUNCTION_NOT_FOUND` | Unknown key, an archived function, or a hard delete in progress. |
| `409` | `FUNCTION_NOT_PUSHED` | A function with no pushed code. |
| `409` | `FUNCTION_DELETE_IN_PROGRESS` | A durable start that raced a hard delete. |
| `429` | `FUNCTION_RATE_LIMITED` | Past the per-function rate ceiling; `Retry-After` is set. |
| `503` | `FUNCTION_RATE_UNAVAILABLE` | The rate limiter could not be consulted; retryable. The platform refuses rather than run unmetered. |


### The access gate

`access` is a CEL expression (see [Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md)) evaluated on **every call**, whatever the function's `runAs`, and it fails closed:

- A caller it denies gets `403 FUNCTION_ACCESS_DENIED`.
- A function with **no** `access` expression denies everyone — which is why push refuses to ship code for one.
- App owners and admins bypass it.
- It is checked **before** any answer about the function's operational state, so a rejected caller cannot tell a disabled function from a never-pushed one from a live one, and cannot map the app's functions by probing keys.
- A caller the gate rejects, or a request `inputSchema` refuses, consumes no rate-limit budget.

## Execution identity

`runAs` decides whose authority `ctx.api` carries. The gate above is unaffected by it.

| | `runAs = "caller"` (default) | `runAs = "system"` |
|---|---|---|
| Caller-authority API families | Reachable with no grant, as the caller | `FUNCTION_CAPABILITY_NOT_AVAILABLE` — there is no caller to be |
| Grant-gated capabilities | Reachable under the grant | Reachable under the grant |
| `ctx.user` | The caller | The caller (who invoked it), for HTTP invokes |

A `runAs = "system"` function invoked over HTTP still runs the `access` gate against the invoking user, still sees that user in `ctx.user`, and still has its `function.invoke` event attributed to them — `runAs` decides which principal the function's *platform calls* carry, not who called it.

**Trigger fires always run as system**, with `ctx.user` null, whatever the file's `runAs` says. A function that a cron or webhook fires reaches only what its grants cover.

## `ctx.api` — the platform from inside a function

`ctx.api` is a typed client for the platform's own API, generated from its OpenAPI document. The namespaces a function may call mirror the operation ids: `analytics`, `blobBuckets`, `collections`, `configVars`, `connections`, `databases`, `documents`, `email`, `gemini`, `groups`, `integrations`, `llm`, `locks`, `notifications`, `prompts`, `resourceMetadata`, `secrets`, `users`. Each method takes one options object carrying the operation's path parameters, query parameters, headers (under their wire names) and `body`; a missing required parameter is a compile error.

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { documentId: string }, ctx) => {
  const doc = await ctx.api.documents.get({ documentId: input.documentId });
  return { title: doc.title, readBy: ctx.user.userId };
});
```

A sandbox has **no network access**. Its only route out is the platform, in three groups distinguished by *whose* authority the call carries:

- **Caller-authority families** — `documents`, `users`, `groups`, `collections`, `notifications`, `locks`, `blobBuckets`, `resourceMetadata`, `analytics`, `llm`, `gemini`. No grant needed, because they confer nothing: every call runs as the invoking user. An admin-only route still needs an admin caller; a private blob is readable only by someone who could read it directly; a direct-LLM route still meets the app's `directLlmEnabled` spend gate. Caller mode only.
- **Database records** — `ctx.db(...)` and `ctx.api.databases.records.*`, under the function's own `database:` grants (see Database records).
- **Grant-gated capabilities** — held by the function in its own right, usable in either mode: `email:send`, `analytics:writeForUser`, and the `integration:`, `prompt:`, `secret:`, `var:`, `users:send` and `connections:send` grants.

A refused call rejects with an error carrying `status` and `errorCode`:

| `errorCode` | Meaning |
|---|---|
| `FUNCTION_EGRESS_DENIED` | `fetch` to any host other than the platform. |
| `FUNCTION_ROUTE_NOT_ALLOWED` | Any operation outside the three groups above — the typed client declares more namespaces (`databaseTypeConfigs`, `functions`) than the gateway publishes to functions, so a method that type-checks can still be refused here. |
| `FUNCTION_CAPABILITY_NOT_AVAILABLE` | A system-mode function reaching a caller-authority family, or a capability the function does not declare. |
| `FUNCTION_DATABASE_GRANT_MISSING` | A records call the grants do not cover; the message names the grant that would. |
| `FUNCTION_INTEGRATION_GRANT_MISSING` | An integration call the grants do not cover; the message names the grant. |
| `FUNCTION_PROMPT_GRANT_MISSING` | A prompt run the grants do not cover. |
| `FUNCTION_SECRET_GRANT_MISSING` / `FUNCTION_VAR_GRANT_MISSING` | A secret or config-var read the grants do not cover. |
| `FUNCTION_SECRET_NOT_FOUND` / `FUNCTION_VAR_NOT_FOUND` | The grant is declared; this environment holds no value of that name. |
| `FUNCTION_SEND_GRANT_MISSING` | A send the grants do not cover. |
| `FUNCTION_SEND_TARGET_NOT_FOUND` | The user is not a member of this app, or the connection is not one of this app's. |
| `FUNCTION_SEND_PAYLOAD_TOO_LARGE` | A send payload over 64 KiB; nothing was delivered. |
| `FUNCTION_SEND_PAYLOAD_INVALID` | A send payload JSON cannot serialize. |
| `FUNCTION_EMAIL_GRANT_MISSING` | An email send the grants do not cover. |
| `FUNCTION_ANALYTICS_GRANT_MISSING` | An analytics write the grants do not cover. |

Every platform call carries the invocation's credential, which expires at the invocation's deadline — a function that outlives its budget has late calls denied and side effects that never land (see Ceilings).

## Capabilities

`capabilities` is an array of exact grant strings. No wildcards, no case folding; a repeated grant is the same grant.

| Grant | Unlocks |
|---|---|
| `database:<type>/<model>:read` | `query`, `count`, `aggregate` on that model, in every database of that type. Requires `unscopedReads = true` or `$caller`-scoped registered queries (below). |
| `database:<type>/<model>:write` | `save`, and every operation inside a batch. |
| `integration:<key>` | `ctx.integrations.call(key, …)` |
| `prompt:<key>` | `ctx.prompts.run(key, …)` |
| `secret:<NAME>` | `ctx.secret(NAME)` |
| `var:<NAME>` | `ctx.configVar(NAME)` |
| `users:send` | `ctx.users.send(userId, payload)` |
| `connections:send` | `ctx.connections.send(connectionId, payload)` |
| `email:send` | `ctx.api.email.send(…)` |
| `analytics:writeForUser` | `ctx.api.analytics.writeForUser(…)` |

The platform reads grants from the TOML file you review in a pull request, so what a reviewer sees and what the platform enforces are one statement. `primitive functions get <function-id>` prints the active version's grants, its `unscopedReads` flag and the queries it registered.

## Database records

### Grants

```toml
# functions/monthly-report.toml
[function]
key = "monthly-report"
entry = "functions/monthly-report/index.ts"
access = "true"
capabilities = [
  "database:orders/Order:read",
  "database:orders/LineItem:read",
  "database:orders/Report:write",
]
unscopedReads = true   # required beside any :read grant — see below
```

- A grant names a **database type key**, not a database id, so it covers every database of that type and the same config works in every environment. A database with no type is unreachable from functions.
- Both components must match `[A-Za-z0-9_-]+`. A type or model whose name carries `/` or `:` cannot be granted — rename it.

**`unscopedReads`.** A read grant is not scoped to the caller: it reads every row of the model for every caller who passes the gate. A config declaring any `:read` grant must therefore either set `unscopedReads = true` — a review artifact, so the reviewer sees that reading the whole model was intended — or bind every read to its caller through registered queries: if every registered query touching a read-granted model declares a `$caller` parameter, the flag is not required. Such a version refuses **every raw read door** inside the sandbox — `db.model(m).query|count|aggregate` outside a registered query, and the same calls on `ctx.api.databases.records` — each with an error naming `unscopedReads`. Writes are untouched. The check is a guard against bugs, living in the SDK inside the sandbox; what stops a function reaching data it was never granted is grant enforcement at the gateway, which is unchanged.

### The typed handle

`ctx.db(databaseId, "<type>")` returns the models that database type declares, typed from its schema. The second argument is the type key the grants name; once `config push` has written the declarations (see Codegen below), a model the type does not declare is a compile error. Before the first push the handle is untyped.

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

  return { open: open.data.length, total };
});
```

| Handle | Surface |
|---|---|
| `ctx.db(id, type).model(name)` | `query`, `count`, `aggregate`, `save`, `batch` — the model is bound per call |
| `ctx.db(id, type)` | `databaseId`, `databaseType`, `batch` (spans models — each item names its own; every model touched needs `:write`) |
| `ctx.api.databases.records.*` | The untyped form of the same five operations, under the same grants |

Filters and query options are ordinary objects; `query` answers `{ data, hasMore?, nextCursor? }` — see [Databases](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) for the cursor parameter. **`include` is refused**: a read grant names one model, and `include` would return rows of another. Query each model under its own grant.

**Codegen.** `config push` writes the type declarations behind `ctx.db` into the tree's `functions/` directory beside your sources — the database-type declaration, the `primitive-functions` declaration and a `functions/tsconfig.json` that loads them. None is part of what gets pushed. `config diff` reports when a schema change has left them stale; the next push refreshes them. The tsconfig is yours to edit — push keeps only the `primitive-functions` path mapping and the generated declarations in the program, and restores those if they go missing.

```bash
primitive functions codegen --check   # CI: non-zero when the generated files are stale; no --check regenerates
```

### Registered queries

`defineQuery` and `defineMutation` name a read or write over the handle so it can be called from several places, validated once, scoped to its caller, and — when the platform can verify what it touched — cached.

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
    return (answer.data ?? []).filter((row) => row.owner === params.owner);
  },
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

| `params.<name>` key | Meaning |
|---|---|
| `type` | `"string"`, `"number"`, `"boolean"`, `"any"` (default) or `"$caller"` — validated and coerced |
| `caller: true` | The `$caller` binding: the platform injects the invoking user; a call that supplies it is refused by name |
| `optional` | May be omitted |
| `default` | Applied when omitted |

- **Register at module scope.** A registration inside the handler works for direct calls but is invisible to `config push`, so it is absent from the manifest and cannot satisfy the `unscopedReads` rule.
- Names are unique; registering one twice throws. An undeclared parameter is refused, not dropped.
- A `$caller` query throws in an invocation with no initiating user — a webhook or cron fire.
- **Caching is off unless the run is verified.** `cache.ttlMs` is honored only when every model the body touched was declared in `models` and nothing was written. TTL is capped at one minute (`MAX_QUERY_CACHE_TTL_MS`). A cache entry belongs to one query, one `databaseId` and resolved type, and one parameter set after injection — two databases of the same type never share an answer, and two callers never see each other's rows. A write through the handle drops the entries that read the written model. The cache lives inside one warm sandbox instance: it bounds staleness, it does not replicate. A mutation is never cached.

## Calling an integration

A function cannot `fetch` the internet. Under an `integration:<key>` grant, `ctx.integrations.call` asks the platform to make the request on the function's behalf, so the integration's `{{secrets.*}}` and `{{vars.*}}` templates are resolved server-side and the credential never enters the sandbox.

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
- The integration's `accessRule` is **not** consulted — it governs members calling from a client; the grant in the reviewed TOML is the authorization here. A disabled or deleted integration refuses everyone (`INTEGRATION_INACTIVE`).
- Every call is recorded on the integration's own log with the function's key: `primitive integrations logs <integration-id>`.

See [Integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md) for declaring one.

## Running a prompt

Under a `prompt:<key>` grant, `ctx.prompts.run(key, { variables, modelOverride, configId })` runs one of the app's saved prompts through the same path the member endpoint uses and answers the same envelope: `success`, `output`, `error`, `metrics`, `configId` (which version ran).

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { text: string }, ctx) => {
  const result = await ctx.prompts.run("summarize", { variables: { text: input.text } });
  if (!result.success) throw new Error(result.error ?? "prompt failed");
  return { summary: result.output };
});
```

- `configId` pins a config **of that prompt**; any other answers `PROMPT_NO_CONFIG`.
- The prompt's `accessRule` is not consulted; the grant authorizes. An inactive or archived prompt refuses everyone.
- The model call is bounded by the invocation's remaining time. A provider call that passes the deadline answers `PROMPT_UPSTREAM_TIMEOUT` — catch it separately from a model failure: the budget ran out, not the model.

See [Prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md).

## Secrets and config vars

`secret:<NAME>` and `var:<NAME>` grants let a function read one app secret or one config var by name:

```ts
const region = await ctx.configVar("REGION");
const token = await ctx.secret("PARTNER_TOKEN");
```

```toml
capabilities = ["secret:PARTNER_TOKEN", "var:REGION"]
```

- **Prefer an integration when the secret is a credential for one** — there the value is injected into the outbound request and never enters the sandbox. `ctx.secret` is for what an integration cannot express.
- Secrets and config vars are provisioned per environment, out of band from the push. A grant is checked for spelling at push and for existence at call time: a granted name with no value in this environment answers `FUNCTION_SECRET_NOT_FOUND` / `FUNCTION_VAR_NOT_FOUND`, naming the key.
- The namespaces are separate: `ctx.secret("X")` never reads a config var named `X`, and vice versa.
- Values come through a short stale-while-revalidate cache: a secret rotated just now may serve its previous value for up to about a minute.

See [App Secrets](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md).

## Sending to a connected client

`users:send` and `connections:send` let a function push a message to a client that is connected **right now**. The recipient receives a `direct.message` frame carrying the payload verbatim plus the sending function's key.

```ts
const result = await ctx.users.send(input.userId, { kind: "order-ready", orderId: "o-1" });
return { delivered: result.connections, truncated: result.truncated };
```

```toml
capabilities = ["users:send"]
```

- **Presence is not guaranteed and there is no durable record.** A user with no connected client answers `{ connections: 0, truncated: false }` — a success. A client that was offline never receives the frame later; use a notification when the message must survive being missed.
- **One socket, one frame.** A client with several documents open is one connection, delivered once.
- **Fan-out is bounded** at 64 unique connections per user, and the lookup behind it is bounded too; past either, the send delivers what it found and answers `truncated: true` — "there may be more", not an error.
- **Payloads are capped at 64 KiB** serialized (`FUNCTION_SEND_PAYLOAD_TOO_LARGE`; nothing delivered).
- `ctx.connections.send(connectionId, payload)` addresses one connection. An id from another app answers exactly as an id that never existed.
- Both grants are exact strings with no key component: the target is named at the call, and authorized then.


## Sending email

Under `email:send`, in either mode — a cron-fired system function sending a digest is the point of it:

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

Under `analytics:writeForUser`, `ctx.api.analytics.writeForUser({ body: { userId, action, feature, … } })` records an event attributed to a **subject** user rather than to whoever is running. Function-only route (`POST /app/{appId}/api/analytics/write-for-user`; `403 FUNCTION_ROUTE_FUNCTION_ONLY` otherwise). The subject must be a member of this app; `appId` and `userId` are reserved — a `context` object of your own cannot rewrite whose activity the event is. See [Analytics](AGENT_GUIDE_TO_PRIMITIVE_ANALYTICS.md).

## Helpers: `pMap`, `ulid`, `stepPolicy`

```ts
import { pMap, ulid, stepPolicy } from "primitive-functions";

// Bounded concurrency. Results in input order; the first rejection wins and
// nothing new starts after it. An unbounded Promise.all over a page of rows
// spends the sandbox's whole subrequest budget in one line.
const results = await pMap(rows, (row) => handle(row), { concurrency: 4 });

// Mint ids INSIDE step.do in a durable run (see Step discipline).
const id = await step.do("mint-id", async () => ulid());

// The platform's timeout/retry policy for the non-idempotent families:
// email, llm, gemini, database. `retries: 0` on email and the AI families is
// deliberate — a retried timeout would send twice or bill twice.
await step.do("send-receipt", stepPolicy.email, () =>
  ctx.api.email.send({ body: { to, subject, htmlBody } })
);
```

`stepPolicy.<family>` is `{ retries: { limit, delay, backoff }, timeout }` — a default you apply, never one the engine forces on your own `step.do` calls. `PMAP_DEFAULT_CONCURRENCY` (8) is the bound `pMap` applies when none is given.

## Durable functions

```toml
# functions/order-sync.toml
[function]
key = "order-sync"
entry = "functions/order-sync/index.ts"
access = "true"
durable = true
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

- `runKey` makes the start idempotent per `(caller, contextDocId, runKey)`: a repeat answers the run that already exists with `existing: true`, never a second run. It defaults to the run id.
- `timeoutMs` is accepted and ignored: the durable engine owns how long a run may take, and a per-request budget cannot hold across a `step.sleep`.
- Poll the run id at `GET /app/{appId}/api/workflows/runs/{runId}/status`; `terminate` takes the function key and the run key.
- A run broadcasts nothing over the WebSocket. Poll.
- Only a caller over HTTP can start a durable run. A trigger fire cannot (a triggered function is synchronous), and a function cannot start another function from inside.

### Reporting back

A durable run has three ways to deliver its result, and they compose:

| Channel | How | Survives the client being away? |
|---|---|---|
| The run's `output` | Return a value; it lands on the run status once the run completes | Yes — poll the run id any time |
| Server state | Write database rows under the function's grants, or documents and notifications as the caller, inside `step.do` | Yes |
| A live push | `ctx.users.send` under `users:send` (see Sending to a connected client) | No — a client that is offline never receives it; pair it with state or a notification |


### Step discipline

The engine re-executes the handler from the top on every resume and replays each completed `step.do` from its **stored result**.

- **Put every side effect inside `step.do`.** A charge, an email or a write outside a step runs again on every resume; inside one it runs exactly once.
- **Do not branch on anything nondeterministic between steps.** `Date.now()`, `Math.random()`, `ulid()` or a fresh read at the top level can take a different path after a resume, and the engine then looks for steps that are not there. Read such values inside a step so the value is stored with it.
- **Your code is pinned; the platform's is not.** A run is pinned to the content hash of the version that started it — pushing mid-run does not change what a running run executes, and new runs get the new code. The `primitive-functions` SDK and the runtime around your handler are whatever is deployed at resume time; treat the SDK as a stable interface, not a frozen artifact.
- A durable function cannot declare triggers — push refuses `durable = true` beside a trigger block.

### Budgets in a durable run

The ceilings (see Ceilings) apply **per engine invocation, not per run**. A run is a series of *slices*: the engine runs the handler from the top until it returns or reaches a `sleep`, `sleepUntil` or `waitForEvent` that has not completed, then hibernates; the next wake is a new slice, with completed steps replayed from storage.

| Ceiling | In a durable run |
|---|---|
| Wall clock | Per slice: every engine invocation mints a fresh credential with a 30 000 ms absolute deadline, so a stretch of code between two sleeps must finish inside it. The request's `timeoutMs` is ignored; the run as a whole has no wall-clock limit. |
| `cpuMs` (5 000 ms) | Per slice. |
| `subRequests` (64) | Per slice. Replayed steps spend nothing — their bodies do not run — but any platform call *outside* a step re-spends on every wake. |
| `ratePerMinute` (60) | Per **start**. A durable start reserves a slot and returns it if the start is refused; resumes take none. |
| Output (1 MiB) and `outputSchema` | Once, on the final return value, at settlement. |

Consequences: budget each stretch between sleeps as if it were a synchronous function; several `step.do` calls with no sleep between them share one slice's budget, so a long job is chunked by sleeps, not by steps; and keep the code outside steps minimal, since it re-executes on every wake.

### Deleting a function with live runs

A hard delete destroys the stored bundles a sleeping run reloads when it wakes, so it is refused — including through `primitive config push --prune` — while any run has not settled; the refusal names the live runs. Wait for them or terminate them, then retry. To stop new runs meanwhile, **archive** the function: an archived function keeps its bundles, so an in-flight run finishes on the version it pinned. While a hard delete is running, an invoke answers `404` and a durable start already in flight answers `409 FUNCTION_DELETE_IN_PROGRESS`.

## Triggers

A function's config block can declare one inbound **webhook** and up to ten **cron** schedules. The platform creates and operates whatever they need. A triggered function is synchronous: push refuses a trigger block beside `durable = true`, so a fire runs inside one bounded invocation and cannot start a durable run.

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

**Webhook.** The receiver is `POST /app/{appId}/webhook/{functionKey}`. Verification, replay protection and a delivery log are built in. Accepted keys: `verificationScheme`, `signingSecret`, `toleranceSeconds`, `deduplicationEnabled`, `deduplicationWindowMs`, `maxBodyBytes`, `secretGracePeriodMs`, and a `[function.triggers.webhook.verification]` table for scheme-specific settings (a Discord public key, a JWT's JWKS, …). A verified delivery runs the function and answers `200 {"received": true}` **whatever the function did** — the run row records the outcome. A delivery that cannot run right now (function disabled, rate ceiling hit) answers `202` with the reason in the delivery log and no dedup key stored, so the provider's redelivery runs once the condition clears. A rotation via `primitive webhooks rotate-secret <webhook-id>` survives later pushes: a push touches the signing secret only when the TOML value itself changed.

**Cron.** With `overlapPolicy = "skip"`, a fire that arrives while the previous run is still going is counted as a skip.

**Every trigger fire writes a run row** — the only record it leaves, since nobody is waiting for an answer. Fires run as system with `ctx.user` null (see Execution identity).

```bash
primitive functions runs <function-id>      # newest first: status, what fired it, timings, failure code
primitive functions get <function-id>       # receiver URL, schedules, last fired
```

**The file is the whole truth.** Removing the webhook block stops deliveries (the receiver answers `410`); putting it back resumes them on the same URL with the log intact. Removing a cron entry cancels it; re-adding schedules it again.

## Ceilings

`[function.limits]` may **lower** any ceiling and never raise one; a config asking for more gets the platform value. The table describes one synchronous invocation; a durable run is bounded per slice (see Budgets in a durable run).

| Limit | Platform value | Key |
|---|---|---|
| CPU per invocation | 5 000 ms | `cpuMs` |
| Outbound subrequests | 64 | `subRequests` |
| Invocations per minute, per function | 60 | `ratePerMinute` |
| Wall clock | 5 000 ms default, 30 000 ms ceiling | `timeoutMs` (on the request) |
| Response size | 1 MiB | — |
| Built bundle | 5 MB | — |
| Send payload | 64 KiB | — |

- **Rate.** Buckets are per app and per function. A slot is taken only once a call is going to run. Past the ceiling: `429 FUNCTION_RATE_LIMITED` with `Retry-After`. Limiter unreachable: `503 FUNCTION_RATE_UNAVAILABLE`, retryable.
- **Response size** is measured by the platform on the parsed `output` (and `error`): over 1 MiB is `{"status":"failed","errorCode":"OUTPUT_TOO_LARGE"}` whatever the sandbox did. Page results, or write them to a database and return a handle.
- **The wall clock starts when the platform accepts the invocation**, so a cold load of the function's code comes out of it. Past the budget the caller gets `{"status":"timeout"}` and the sandbox request is aborted. The platform cannot kill a running sandbox, so a function that ignores the abort may execute until `cpuMs` stops it — but it can no longer *act*: the invocation's credential carries an absolute deadline checked on every platform call, so late calls are denied and late side effects never land. The same deadline cuts off `ctx.integrations.call` (`UPSTREAM_TIMEOUT`) and `ctx.prompts.run` (`PROMPT_UPSTREAM_TIMEOUT`).
- The clock does not advance during CPU-only work: a loop that waits for `Date.now()` to move never finishes. Wait on a timer, not on the clock.

## Recording

Every HTTP invocation writes one `function.invoke` analytics event attributed to the **caller** — never the app, even for a `runAs = "system"` function — carrying the function key, the version that ran, the terminal status and the duration. Trigger fires are recorded as run rows instead.

## Operating

```bash
primitive functions list                 # keys, status, runAs; --status active|inactive|archived
primitive functions get <function-id>    # active version, grants, unscopedReads, registered queries, triggers
primitive functions runs <function-id>   # trigger fires and durable runs, newest first; --limit <n> --cursor <cursor>
primitive functions disable <function-id>
primitive functions enable <function-id>
primitive functions archive <function-id>
primitive functions codegen --check      # --dir <path> to point at another config tree
primitive config set function/<key> <path>=<value>
```

Every verb takes `--app <app-id>` and `--json`.

- `disable` refuses invocations and leaves config and code unchanged; pushes still land.
- `archive` retires a function without destroying it: the row keeps its key, pushing code to it is refused, and there is no un-archive. Reclaiming the key is a hard delete — `primitive config pull` (removes the tombstone's local files), then a confirmed `primitive config push --prune`, which destroys the function, every version and their bundles (refused while a run is live — see Deleting a function with live runs).

## Related

- [Configuration](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md) — the sync loop that pushes `functions/<key>.toml` and its code.
- [Databases](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) — database types, models and the records surface.
- [Integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md), [Prompts](AGENT_GUIDE_TO_PRIMITIVE_PROMPTS.md), [App Secrets](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md) — what the grants unlock.
- [Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md) — the CEL the `access` gate is written in.

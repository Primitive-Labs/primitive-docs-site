# Integrations Guide for Coding Agents

An **integration** is configuration for an outbound third-party API: `integrations/<key>.toml` pins the `baseUrl`, the `allowedMethods` and `allowedPaths`, and the credentials to attach. A **server function** calls it with `ctx.integrations.call(key, request)`; the platform builds the request, injects credentials from App Secrets, enforces the allowlists, and returns the upstream response to the function. The credential never enters the function's sandbox and never reaches a client.

This is the **outbound** half of a third-party integration. The **inbound** half — a webhook the provider calls, verified with a scheme — is a webhook trigger declared on a server function (see [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#triggers)). Most third-party integrations need both.

## Call an integration

```toml
# functions/refund.toml
[function]
key = "refund"
entry = "functions/refund/index.ts"
access = "user.role == 'admin'"
capabilities = ["integration:stripe"]   # the egress allowlist: one line per integration key
```

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { paymentIntent: string }, ctx) => {
  const response = await ctx.integrations.call("stripe", {
    method: "POST",
    path: "/v1/refunds",
    form: { payment_intent: input.paymentIntent },
  });
  if (response.errorCode) throw new Error(`stripe: ${response.errorCode} (${response.status})`);
  return { refund: response.body };
});
```

## Architecture

- Admin defines an integration (per-app, keyed by `integrationKey`) with a `requestConfig` that pins the `baseUrl`, `allowedMethods`, and `allowedPaths`.
- Credentials are stored as **App Secrets** (`primitive secrets set KEY --value ...`), then referenced from `defaultHeaders` or `staticQuery` via `{{secrets.KEY}}` templates.
- Non-secret config values are stored as **Config Vars** (a key in `vars.toml`, applied with `config push`), referenced the same way via `{{vars.KEY}}` templates — see "Vars vs Secrets" below.
- A server function calls it with `ctx.integrations.call(integrationKey, { method, path, query, headers, body | form, bodyMode })`, under an `integration:<key>` line in the function's `capabilities`. There is no client endpoint.
- **Authorization is the function's.** An integration carries no access rule: the function's `access` gate decides who may make the code run, and the `integration:<key>` capability decides which integrations the code may reach. A call without the capability is refused `FUNCTION_INTEGRATION_GRANT_MISSING`, naming the line to add; the capability is checked at push against an active integration of that key.

Credentials always go through App Secrets:

```bash
# Store as an app secret, reference it from the integration's defaultHeaders
primitive secrets set OPENAI_API_KEY --value sk-... --summary "OpenAI prod key"
```

## TOML Schema

Integrations are defined in TOML with two sections: `[integration]` and `[requestConfig]`.

```toml
[integration]
key = "integration-key"              # Required. Unique per app, lowercased on lookup.
displayName = "Display Name"         # Required.
description = "Optional description" # Optional.
timeoutMs = 300_000                  # Optional. Per-request upstream timeout (default 300_000 = 5 min).

[requestConfig]
baseUrl = "https://api.example.com/" # Required. Must start with http://, https://, or test://.
                                     #   Trailing slashes are stripped and one is re-added.
allowedMethods = ["GET", "POST"]     # Optional. Default ["GET"]. Uppercased.
allowedPaths = ["/v1/*", "/status"]  # Optional. Default ["/*"] (allow ALL). Trailing-* prefix match.
defaultMethod = "GET"                # Optional. Default = first allowedMethod. Auto-added to
                                     #   allowedMethods if missing.
forwardHeaders = ["x-trace-id"]      # Optional. Lowercased allowlist of the call's own `headers` sent upstream.
                                     #   Default [] (or ["*"]) = all. host/content-length always stripped.
forwardQueryParams = ["q", "limit"]  # Optional. Lowercased allowlist of the call's own `query` keys sent upstream.
                                     #   Default [] (or ["*"]) = all.
bodyMode = "json"                    # Optional. "json" (default) | "raw" | "multipart".
responsePassthrough = true           # Optional. Parsed but not enforced; the call always
                                     #   returns { status, headers, body } from upstream.

[requestConfig.defaultHeaders]       # Optional. Always-sent headers. {{secrets.KEY}} / {{vars.KEY}} resolved here.
Accept = "application/json"
Authorization = "Bearer {{secrets.OPENAI_API_KEY}}"

[requestConfig.staticQuery]          # Optional. Always-appended query params. {{secrets.KEY}} / {{vars.KEY}} resolved here.
apiVersion = "v2"

[requestConfig.exampleQuery]         # Optional. For docs/test UI only. Not sent on real calls.
q = "search term"

[requestConfig.exampleBody]          # Optional. For docs/test UI only.
model = "gpt-4.1-mini"
input = "Hello"

# bodyMode = "multipart" only
[[requestConfig.multipartFieldMapping]]
fieldName = "file"
type = "attachment"                  # "attachment" or "value"
attachmentIndex = 0                  # or attachmentName = "myfile.pdf"

[[requestConfig.multipartFieldMapping]]
fieldName = "purpose"
type = "value"
value = "fine-tune"
```

### Field Reference

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `integration.key` | string | Yes | — | Lowercased on lookup. Unique per app. |
| `integration.displayName` | string | Yes | — | |
| `integration.description` | string | No | — | |
| `integration.timeoutMs` | int | No | `300000` | Aborts upstream after this long (504 `UPSTREAM_TIMEOUT`). |
| `requestConfig.baseUrl` | string | Yes | — | `http://`, `https://`, or `test://`. |
| `requestConfig.allowedMethods` | string[] | No | `["GET"]` | Uppercased; `defaultMethod` auto-added. |
| `requestConfig.allowedPaths` | string[] | No | `["/*"]` | **Default allows everything.** Trailing-`*` prefix match only. |
| `requestConfig.defaultMethod` | string | No | first allowed | |
| `requestConfig.defaultHeaders` | object | No | `{}` | `{{secrets.KEY}}` / `{{vars.KEY}}` resolved per request. |
| `requestConfig.staticQuery` | object | No | `{}` | string/number/boolean values; `{{secrets.KEY}}` / `{{vars.KEY}}` resolved. |
| `requestConfig.forwardHeaders` | string[] | No | `[]` | **Lowercased.** Filters the call's `headers`: a non-empty list sends only the named ones; `[]` or `["*"]` sends all. |
| `requestConfig.forwardQueryParams` | string[] | No | `[]` | **Lowercased.** Filters the call's `query`: a non-empty list sends only the named keys; `[]` or `["*"]` sends all. |
| `requestConfig.bodyMode` | string | No | `"json"` | `"json"` \| `"raw"` \| `"multipart"`. |
| `requestConfig.multipartFieldMapping` | array | No | `[]` | Required for sane multipart behavior. |
| `requestConfig.responsePassthrough` | bool | No | `true` | Parsed but not enforced. |

### Per-integration limits (set via API/admin UI, not TOML)

- `maxRequestBodyBytes` — default 1 MB. Exceeding → HTTP 413 `REQUEST_BODY_TOO_LARGE`.
- Attachment base64 cap — 10 MB per attachment.

## Validation Rules

- `baseUrl` must start with `http://`, `https://`, or `test://`.
- `allowedMethods` must match `^[A-Z]+$`.
- `allowedPaths`: each path must start with `/`. `*` is only honored as the **last character** (prefix match). `/v1/*` matches `/v1/anything`. There is **no** middle-glob and **no** regex.
- `forwardHeaders` / `forwardQueryParams` are compared case-insensitively. `host` and `content-length` are always stripped from forwarded headers regardless of allowlist.
- `staticQuery` values must be string/number/boolean. Other types are dropped.

## Secret Injection

Secrets are stored once per app and referenced from any integration via templates.

```bash
# 1. Create the app secret (key must match ^[A-Z][A-Z0-9_]{0,63}$)
primitive secrets set OPENAI_API_KEY --value sk-... --summary "OpenAI prod"

# 2. Reference it in the integration TOML (defaultHeaders or staticQuery only)
```

```toml
[requestConfig.defaultHeaders]
Authorization = "Bearer {{secrets.OPENAI_API_KEY}}"

[requestConfig.staticQuery]
api_key = "{{secrets.GOOGLE_API_KEY}}"
```

Behavior:

- Resolution happens server-side when the platform builds the request. The plaintext value never enters the function's sandbox and is never returned in the response.
- Any header/query whose value was substituted from a secret is automatically marked sensitive — its value is replaced with `[redacted]` in admin logs and the test-mode request preview.
- Secret references are validated at save time: creating or updating an integration (or creating a versioned config) whose `defaultHeaders`/`staticQuery` reference a nonexistent app secret fails with a 400 naming the missing key. Whitespace inside the braces is tolerated (`{{ secrets.KEY }}` ≡ `{{secrets.KEY}}`), but not around the dot. If a referenced secret is deleted *after* the config is saved, the platform passes the literal `{{secrets.KEY}}` upstream and logs a server-side warning — re-create the secret or update the config.
- Secret-key constraint: `^[A-Z][A-Z0-9_]{0,63}$` (uppercase letters, digits, underscores; starts with a letter; ≤64 chars).
- Cache: app-secret reads are cached server-side (30s fresh / 60s stale). Updates invalidate the cache for that app.

### Vars vs Secrets

Config vars resolve in the same fields via `{{vars.KEY}}` — same key constraint (`^[A-Z][A-Z0-9_]{0,63}$`), same whitespace tolerance (`{{ vars.KEY }}` ≡ `{{vars.KEY}}`), same two fields (`defaultHeaders`/`staticQuery` only, resolved when the request is built).

```bash
primitive config set vars ACCOUNT_ID=acct_123
primitive config push --only var/ACCOUNT_ID
```

```toml
[requestConfig.staticQuery]
account = "{{vars.ACCOUNT_ID}}"
```

Two behaviors deliberately diverge from secrets:

- **Not redacted.** Only a value substituted from `{{secrets.*}}` is marked sensitive and replaced with `[redacted]` in admin logs and the test-mode request preview. A value substituted from `{{vars.*}}` is never redacted — vars are non-secret and stay visible in both places.
- **Not validated at save time.** Saving/updating an integration whose `defaultHeaders`/`staticQuery` reference a nonexistent `{{secrets.KEY}}` fails with a 400 naming the missing key. The same integration referencing a nonexistent `{{vars.KEY}}` saves successfully — the reference just resolves to the literal `{{vars.KEY}}` placeholder at call time (the fallback secrets only get if the key is deleted *after* save).

Vars are not per-integration rows and have no cache-TTL distinction called out here beyond the shared config-vars cache; see the App Secrets guide's Config Vars section for the full CLI, the `vars.toml` sync shape, and the CEL declared-only binding path (`vars = ["KEY"]`).

### Rotation

```bash
# Overwrite the same key — all integrations using {{secrets.KEY}} pick up the new value
# (after cache TTL or on cache invalidation).
primitive secrets set OPENAI_API_KEY --value sk-new-...
```

## Real Examples

### OpenAI Responses API

```toml
[integration]
key = "open-ai"
displayName = "Open AI"
timeoutMs = 300_000

[requestConfig]
baseUrl = "https://api.openai.com/"
allowedMethods = ["POST"]
allowedPaths = ["/v1/responses"]
defaultMethod = "POST"
forwardHeaders = ["content-type"]
forwardQueryParams = []

  [requestConfig.defaultHeaders]
  Content-Type = "application/json"
  Authorization = "Bearer {{secrets.OPENAI_API_KEY}}"

  [requestConfig.exampleBody]
  model = "gpt-4.1-mini"
  input = "Write a limerick about a llama."
```

### Google YouTube Search (API key as query param)

```toml
[integration]
key = "youtube-search"
displayName = "YouTube Search API"
timeoutMs = 300_000

[requestConfig]
baseUrl = "https://www.googleapis.com/"
allowedMethods = ["GET"]
allowedPaths = ["/youtube/v3/search"]
defaultMethod = "GET"
forwardHeaders = []
forwardQueryParams = ["q"]   # Only the call's `q` reaches the upstream.

  [requestConfig.defaultHeaders]
  Accept = "application/json"

  [requestConfig.staticQuery]
  part = "snippet"
  type = "video"
  maxResults = 10
  key = "{{secrets.YOUTUBE_API_KEY}}"
```

### Postman Echo (test/debug)

```toml
[integration]
key = "postman-echo"
displayName = "Postman Echo"
timeoutMs = 30_000

[requestConfig]
baseUrl = "https://postman-echo.com/"
allowedMethods = ["GET", "POST"]
allowedPaths = ["/get", "/post"]
defaultMethod = "GET"
forwardHeaders = ["x-trace-id"]
forwardQueryParams = ["foo", "bar"]
```

## Footguns and Common Mistakes

### 1. Default `allowedPaths` is `["/*"]` — that allows EVERYTHING

```toml novalidate
# WRONG - omitting allowedPaths defaults to ["/*"]; a call can hit any path on baseUrl
[requestConfig]
baseUrl = "https://api.stripe.com/"
allowedMethods = ["GET", "POST", "DELETE"]

# RIGHT - explicit, narrow allowlist
[requestConfig]
baseUrl = "https://api.stripe.com/"
allowedMethods = ["GET", "POST"]
allowedPaths = ["/v1/customers", "/v1/customers/*", "/v1/charges"]
```

### 2. Wildcard is a prefix match, not a glob

`/users/*` matches `/users/`, `/users/123`, **and** `/users/123/admin/delete`. There is no `/users/*/profile` glob — that pattern would only match a path literally starting with `/users/*/profile`.

```toml novalidate
# WRONG - looks scoped, actually allows everything under /users/
allowedPaths = ["/users/*"]

# RIGHT - if you only want a few endpoints, list them
allowedPaths = ["/users", "/users/me", "/users/me/avatar"]
```

### 3. An empty `forwardHeaders` forwards every header the call sets

`forwardHeaders` (and `forwardQueryParams`) filter what a `ctx.integrations.call` passes in `headers` (and `query`). An empty list is not "none" — it sends every header the call sets, like `["*"]`. When a function builds headers from its input, list the ones the upstream may receive:

```toml novalidate
# RIGHT - explicit allowlist
forwardHeaders = ["x-trace-id", "accept-language"]
```

Never pass a credential in the call's `headers`. Put it in `defaultHeaders` + `{{secrets.KEY}}`, where the platform injects it and the function never holds it.

### 4. Don't hardcode secrets in TOML

```toml novalidate
# WRONG - secret committed to source control and visible in the admin UI
[requestConfig.defaultHeaders]
Authorization = "Bearer sk-abc123..."

# RIGHT - reference an app secret
[requestConfig.defaultHeaders]
Authorization = "Bearer {{secrets.OPENAI_API_KEY}}"
```

### 5. Writing `status` into the TOML

`status` is server-owned and is not a key of `[integration]`. A file that still carries the line fails `config push` with a message naming the verb. Every pushed integration is callable straight away; take one out of service — and put it back — out of band:

```bash
primitive integrations disable <integration-id>   # the ID `primitive integrations list` prints, not the key
primitive integrations enable <integration-id>
```

### 6. `responsePassthrough` has no effect

The call always returns `{ status, headers, body }` from upstream regardless of this flag. Don't rely on toggling it.

### 7. Header names in `forwardHeaders` are case-insensitive — write them lowercased

The platform lowercases the allowlist; `["Content-Type"]` and `["content-type"]` behave identically. Stick with lowercase to match what you'll see in logs.

## CLI Reference

### Auth and app context

```bash
primitive login                  # Browser-based auth
primitive whoami                 # The app this project's environment names
# or pass --app <app-id> to any subcommand
```

### Reading integrations

```bash
primitive integrations list [--status active|inactive|archived] [--json]
primitive integrations get <integration-id> [--json]
```

### Writing integrations (TOML only)

An integration is `integrations/<key>.toml`. There is no create/update/delete command — author the file and push it.

```bash
primitive config fields integration              # every key, type, required, default
primitive config create integration weather-api  # scaffold integrations/weather-api.toml
primitive config push --only integration/weather-api # apply just this one
```

Delete an integration by removing its file and running `primitive config push --prune`.

### Test (admin only — bypasses status check)

```bash
primitive integrations test <id>                                    # CLI defaults: --method GET, no path
primitive integrations test <id> --method POST --path /v1/responses
primitive integrations test <id> --query '{"q":"hello","limit":10}'
primitive integrations test <id> --method POST --body '{"foo":"bar"}'
```

Note: `--method` defaults to `GET` regardless of the integration's `defaultMethod`. Pass `--method` explicitly when the integration only allows non-GET methods.

The test endpoint is a documented diagnostic: it includes a `requestPreview` (with secrets redacted) and works against an INACTIVE integration. An `archived` one is refused.

### Logs

```bash
primitive integrations logs <id> [--limit 50] [--json]
```

### App Secrets (for `{{secrets.KEY}}` resolution)

```bash
primitive secrets list [--app <app-id>] [--json]
primitive secrets set OPENAI_API_KEY --value sk-... --summary "OpenAI prod key"
primitive secrets set OPENAI_API_KEY --value sk-rotated...   # update = same command
primitive secrets delete OPENAI_API_KEY
```

Values are AES-encrypted at rest using `APP_SECRETS_ENCRYPTION_KEY`. Max 100 secrets per app, max 2 KB per value.

### Config Vars (for `{{vars.KEY}}` resolution)

```bash
primitive vars list [--app <app-id>] [--json]
primitive config set vars ACCOUNT_ID=acct_123  # edits vars.toml, no API call
primitive config push --only var/ACCOUNT_ID    # applies it
```

Delete a var by removing its line from `vars.toml` and pushing. Values are plaintext (never encrypted, never masked). Max 100 vars per app, max 2 KB per value.

### Test Cases (regression suite for an integration)

Test cases are authored in TOML, one file per case, and applied by `config push`.
They live in a sidecar directory beside the integration:

```toml
# integrations/stripe.tests/basic.toml
[test]
name = "Basic"
description = "GET /get returns 200"
inputVariables = '{"method":"GET","path":"/get"}'
# configName pins the case to a named config; leave empty for the active one.
configName = ""
expectedOutputPattern = ""
expectedOutputContains = '["\"ok\":true"]'
expectedJsonSubset = '{}'
```

`primitive config fields integration` lists every key. Fixture files for a case
go in `integrations/stripe.tests/basic/` and upload with the same push. Clearing
a field is blanking it (`""`, `[]`, `{}`) and pushing; deleting a case is
removing its file and running `primitive config push --prune`.

For an integration case, `inputVariables` is the request the run issues — the
same object a function passes to `ctx.integrations.call`, at the top level, with
no wrapper key. Its keys are `method`, `path`, `headers`, `query`, and the body:
`body` (JSON), or `form` for form-urlencoded, or `bodyMode` plus
`multipartFields` for multipart. Anything else in the object is ignored.

A key the case omits falls back the same way a live call's does, per field: no
`method` uses `defaultMethod`, no `path` calls `/` on the `baseUrl`,
`defaultHeaders`/`staticQuery` merge into the case's headers and query,
`bodyMode` and `multipartFields` fall back to (and, for the field mapping,
merge with) the configured ones. There is no configured body: a case that
authors no `body` or `form` sends none. So a case against a
`defaultMethod = "POST"` integration can name a `path` and a `body` alone, and
a case that authors nothing runs `defaultMethod` on `/` — which still has to
pass `allowedPaths`.

```toml
# integrations/plaid.tests/mints-a-link-token.toml
[test]
name = "mints a link token"
inputVariables = '''
{
  "method": "POST",
  "path": "/link/token/create",
  "body": { "client_name": "Acme", "products": ["transactions"] }
}
'''
expectedOutputPattern = "link-sandbox-"
```

Run and inspect them from the CLI:

```bash
primitive integrations tests list <id>
primitive integrations tests run <id> <test-case-id>
primitive integrations tests run-all <id>
primitive integrations tests run-all <id> --test-cases 01ABC,01DEF
primitive integrations tests runs <id> [--limit 20] [--group <comparison-group>]
```

`run-all` executes the **registered** cases (the ones a push has sent), not whatever is on disk — see [the case lifecycle](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md#a-case-file-is-local-until-a-push-registers-it) for the local/registered distinction and `config diff`'s counters.

### Sync (TOML <-> server)

Integration configs live at `integrations/<key>.toml`, one file per integration. See the [Configuration guide](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md#the-sync-loop) for the sync loop (`init`/`pull`/`diff`/`push`), how the directory is resolved, snapshots, pruning, and conflict handling.

## The call contract

`ctx.integrations.call(integrationKey, request?)` answers `{ status, headers, body, durationMs, traceId, errorCode? }` — `errorCode` is set when the platform or the upstream refused, and absent on success. Branch on `errorCode`, not on `status` alone.

### Request

| Field | Type | Notes |
|-------|------|-------|
| `method` | string | Optional. Must be in `allowedMethods`; defaults to `defaultMethod`. |
| `path` | string | Optional. Relative to `baseUrl`; must match `allowedPaths`. Resolved and normalized before the check, so `//elsewhere/x` and `/allowed/../blocked` are refused. |
| `query` | `Record<string, unknown>` | Filtered by `forwardQueryParams`. Arrays append multiple values; objects are JSON-stringified. |
| `headers` | `Record<string, string>` | Filtered by `forwardHeaders`. `host`/`content-length` always stripped. |
| `body` | unknown | JSON by default; for non-JSON bodies, set `bodyMode`. |
| `form` | `Record<string, unknown>` | form-urlencoded body; not combinable with `body`. |
| `bodyMode` | `"json"` \| `"raw"` \| `"multipart"` | Overrides the integration's `bodyMode` for this call. |

- **Redirects are re-checked before they are followed**: a hop is followed only if it stays on the integration's exact origin (same scheme, host and port), lands on an allowed path and uses an allowed method; otherwise the `3xx` comes back unfollowed. Five hops maximum.
- The upstream request is bounded by the integration's `timeoutMs` or the invocation's remaining time, whichever is smaller — including while the response body is still arriving, not merely while it waits for a reply. Past it, `UPSTREAM_TIMEOUT`.
- Every call is recorded on the integration's own log with the function's key: `primitive integrations logs <integration-id>`.

### Error codes (`errorCode`)

| Code | HTTP | Meaning |
|------|------|---------|
| `FUNCTION_INTEGRATION_GRANT_MISSING` | 403 | The function's `capabilities` carry no `integration:<key>` line for this key. |
| `INTEGRATION_INACTIVE` | 404 | `status != "active"`. |
| `DISALLOWED_METHOD` | 422 | Method not in `allowedMethods`. |
| `DISALLOWED_PATH` | 422 | Path doesn't match `allowedPaths`. |
| `REQUEST_BODY_TOO_LARGE` | 413 | Body exceeds `maxRequestBodyBytes`. |
| `UPSTREAM_TIMEOUT` | 504 | Upstream took longer than `timeoutMs`, or the invocation's deadline passed. |
| `UPSTREAM_NETWORK_ERROR` | 502 | Upstream unreachable. |
| `UPSTREAM_ERROR` | matches upstream | Upstream returned 4xx/5xx; body still passed through. |
| `INVALID_BASE64` | 400 | Bad attachment data. |

See [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#calling-an-integration) for the function side.

## Body Modes

- `"json"` (default) — body is JSON-stringified; `Content-Type: application/json` set unless overridden.
- `"raw"` — first attachment is sent as the request body; `Content-Type` defaults to the attachment's `type`.
- `"multipart"` — `multipart/form-data` constructed from `multipartFieldMapping`. Each entry is either `type = "value"` (literal field) or `type = "attachment"` (binary part by `attachmentIndex` or `attachmentName`). If no mapping is provided and at least one attachment is supplied, the first attachment is auto-mapped to field `file` and any object body becomes additional text parts.

## Status Lifecycle

`status` is server-owned (`active | inactive | archived`), never a TOML key, and written only by `primitive integrations disable`/`enable`, the console's matching action, and the delete flow — whose CLI spelling is `primitive integrations archive <id>`. Archiving is the delete lifecycle, not availability: the row is kept and goes on holding its `integrationKey`, `enable` refuses it, and there is no un-archive (reclaim the key with a confirmed `primitive config push --prune` after removing the file, then re-add and push).

| Status | `ctx.integrations.call`? | `primitive integrations test`? |
|--------|----------------------|--------------------------------|
| `active` | Yes | Yes |
| `inactive` | No (404) | Yes — it is the diagnostic |
| `archived` | No (404) | No |

Deleting an integration over the API or in the console **archives** it. `primitive config push --prune` — remove the TOML file, then push — hard-deletes the row permanently.

## Files on Disk

- `primitive/<env>/integrations/<key>.toml` — one file per integration
- `primitive/<env>/.sync-state.json` — sync state (last pull/push hashes), committed with the tree

## Quick Triage

```bash
primitive integrations get <id> --json     # see exact stored requestConfig
primitive integrations test <id> --method GET --path /probe --query '{"x":"1"}'
primitive integrations logs <id> --limit 20
primitive secrets list                      # confirm referenced secret keys exist
primitive vars list                         # confirm referenced var keys exist
```

If a function's calls return `INTEGRATION_INACTIVE` (404) despite the integration existing, check its status — inactive and archived both refuse. `primitive integrations enable <integration-id>` puts it back in service.

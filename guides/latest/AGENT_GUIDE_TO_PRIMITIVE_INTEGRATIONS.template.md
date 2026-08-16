# Integrations Guide for Coding Agents

Integrations let tenant apps call third-party APIs through a server-side proxy without exposing credentials to clients. The proxy enforces an allowlist of methods and paths, injects credentials from App Secrets, and returns the upstream response to the caller.

This is the **outbound** half of a third-party integration. The **inbound** half — a webhook the provider calls to trigger a workflow, configured with a verification scheme — is in the Workflows guide ("Inbound webhooks"). Most third-party integrations need both: outbound calls plus an inbound webhook that triggers a workflow.

## Call an integration

{{ example: integrations/integration-call }}

This guide is verified against `js-bao-wss` source. Anything not described here is unsupported.

## Architecture

- Admin defines an `AppIntegration` (per-app, keyed by `integrationKey`) with a `requestConfig` that pins the `baseUrl`, `allowedMethods`, and `allowedPaths`.
- Credentials are stored as **App Secrets** (`primitive secrets set KEY --value ...`), then referenced from `defaultHeaders` or `staticQuery` via `{{secrets.KEY}}` templates.
- Non-secret config values are stored as **Config Vars** (a key in `vars.toml`, applied with `sync push`), referenced the same way via `{{vars.KEY}}` templates — see "Vars vs Secrets" below.
- Clients call `client.integrations.call({ integrationKey, method, path, ... })`. The platform routes through `POST /app/{appId}/api/integrations/{integrationKey}/proxy`.
- Workflows can call integrations via the `integration.call` step.

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
status = "draft"                     # Optional. "draft" | "active" | "archived" (default "draft").
                                     #   draft/archived integrations CANNOT be called by clients.
accessRule = "true"                  # REQUIRED at creation. CEL expression deciding who may
                                     #   invoke this integration. "true" = any app member.
                                     #   Empty/absent = DENY everyone but app admins/owners.
timeoutMs = 300_000                  # Optional. Per-request upstream timeout (default 300_000 = 5 min).

[requestConfig]
baseUrl = "https://api.example.com/" # Required. Must start with http://, https://, or test://.
                                     #   Trailing slashes are stripped and one is re-added.
allowedMethods = ["GET", "POST"]     # Optional. Default ["GET"]. Uppercased.
allowedPaths = ["/v1/*", "/status"]  # Optional. Default ["/*"] (allow ALL). Trailing-* prefix match.
defaultMethod = "GET"                # Optional. Default = first allowedMethod. Auto-added to
                                     #   allowedMethods if missing.
forwardHeaders = ["x-trace-id"]      # Optional. Lowercased allowlist of CLIENT headers to forward.
                                     #   Default [] = none. ["*"] = all. host/content-length always stripped.
forwardQueryParams = ["q", "limit"]  # Optional. Lowercased allowlist of CLIENT query params to forward.
                                     #   Default [] = none. ["*"] = all.
bodyMode = "json"                    # Optional. "json" (default) | "raw" | "multipart".
responsePassthrough = true           # Optional. Parsed but not enforced; the proxy always
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
| `integration.status` | string | No | `"draft"` | Only `"active"` is callable by clients. |
| `integration.accessRule` | string | **Yes** | — | CEL rule for WHO may call it. Empty/absent = deny all but app admins/owners. See [Caller access control](#caller-access-control). |
| `integration.timeoutMs` | int | No | `300000` | Aborts upstream after this long (504 `UPSTREAM_TIMEOUT`). |
| `requestConfig.baseUrl` | string | Yes | — | `http://`, `https://`, or `test://`. |
| `requestConfig.allowedMethods` | string[] | No | `["GET"]` | Uppercased; `defaultMethod` auto-added. |
| `requestConfig.allowedPaths` | string[] | No | `["/*"]` | **Default allows everything.** Trailing-`*` prefix match only. |
| `requestConfig.defaultMethod` | string | No | first allowed | |
| `requestConfig.defaultHeaders` | object | No | `{}` | `{{secrets.KEY}}` / `{{vars.KEY}}` resolved per request. |
| `requestConfig.staticQuery` | object | No | `{}` | string/number/boolean values; `{{secrets.KEY}}` / `{{vars.KEY}}` resolved. |
| `requestConfig.forwardHeaders` | string[] | No | `[]` | **Lowercased.** `[]`=none, `["*"]`=all. |
| `requestConfig.forwardQueryParams` | string[] | No | `[]` | **Lowercased.** `[]`=none, `["*"]`=all. |
| `requestConfig.bodyMode` | string | No | `"json"` | `"json"` \| `"raw"` \| `"multipart"`. |
| `requestConfig.multipartFieldMapping` | array | No | `[]` | Required for sane multipart behavior. |
| `requestConfig.responsePassthrough` | bool | No | `true` | Parsed but not enforced. |

### Per-integration limits (set via API/admin UI, not TOML)

- `maxRequestBodyBytes` — default 1 MB. Exceeding → HTTP 413 `REQUEST_BODY_TOO_LARGE`.
- Attachment base64 cap — 10 MB per attachment, enforced by the client payload normalizer.

## Caller access control

`accessRule` is a CEL expression, stored on the integration and evaluated on every proxy call, in the same language as a workflow `accessRule` and a metadata category rule.

Decision order: app admin/owner → allow; no rule stored → **deny**; otherwise evaluate the expression.

| Binding | Client proxy call | Workflow step, `runAs: "caller"` | Workflow step, `runAs: "system"` |
|---|---|---|---|
| `user.userId`, `user.role` | the authenticated caller | the run's caller | unbound — a `user.*` rule is false |
| `isMemberOf(type, id)`, `memberGroups(type)`, `hasRole(role)` | caller's app memberships | caller's app memberships | false |
| `fromWorkflow()` | `false` | `true` | `true` |
| `fromWorkflow("<workflowKey>")` | `false` | true for that key | true for that key |
| `workflow.workflowKey`, `workflow.runId`, `workflow.stepId` | unbound | the running workflow | the running workflow |
| `md.self.<category>.<key>` | the integration's own stored metadata | same | same |
| admin/owner bypass | applies | applies | does not apply (no app role) |

Rules you can write for `[integration] accessRule`:

| Rule | Allows |
|---|---|
| `"true"` | any app member |
| `"fromWorkflow()"` | workflow steps only — no client call |
| `"fromWorkflow('daily-digest')"` | one named workflow |
| `"isMemberOf('staff', 'core')"` | one group |
| `"user.userId == 'usr_123'"` | one user |
| `"md.self.access.clientCallable == true"` | whatever the integration's own metadata says |

Enforcement:

- A denied call returns HTTP `403` with `{"errorCode": "INTEGRATION_ACCESS_DENIED"}`, before any upstream request is built and before any app secret is read. The response carries no rule text.
- A denied integration is omitted from `GET /app/{appId}/api/integrations` for that caller, so its `allowedPaths` are not disclosed.
- A denied workflow step fails non-retryably with `Integration access denied`.
- Denials appear in `primitive integrations logs` with `errorCode = INTEGRATION_ACCESS_DENIED` and the caller's identity.
- A syntactically invalid rule is refused at save time: `400 Invalid accessRule CEL expression: <message>`.

**Migrating an existing integration.** Integrations that predate this field have no rule stored and therefore deny every member call and every workflow step, `runAs: "system"` included. Set a rule on each:

```bash
primitive sync pull                  # writes integrations/<key>.toml
# add accessRule to [integration] — "true" restores the previous open behavior
primitive sync push
```

Removing `accessRule` from an existing integration's TOML CLEARS the stored rule on the next push, which locks the integration down — it never opens it up. Creating a new integration without one is refused: `400 accessRule is required`.

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

- Resolution happens server-side at proxy time. The plaintext value is never returned to the client.
- Any header/query whose value was substituted from a secret is automatically marked sensitive — its value is replaced with `[redacted]` in admin logs and the test-mode request preview.
- Secret references are validated at save time: creating or updating an integration (or creating a versioned config) whose `defaultHeaders`/`staticQuery` reference a nonexistent app secret fails with a 400 naming the missing key. Whitespace inside the braces is tolerated (`{{ secrets.KEY }}` ≡ `{{secrets.KEY}}`), but not around the dot. If a referenced secret is deleted *after* the config is saved, the proxy passes the literal `{{secrets.KEY}}` upstream and logs a server-side warning — re-create the secret or update the config.
- Secret-key constraint: `^[A-Z][A-Z0-9_]{0,63}$` (uppercase letters, digits, underscores; starts with a letter; ≤64 chars).
- Cache: app-secret reads are cached server-side (30s fresh / 60s stale). Updates invalidate the cache for that app.

### Vars vs Secrets

Config vars resolve in the same fields via `{{vars.KEY}}` — same key constraint (`^[A-Z][A-Z0-9_]{0,63}$`), same whitespace tolerance (`{{ vars.KEY }}` ≡ `{{vars.KEY}}`), same two fields (`defaultHeaders`/`staticQuery` only, resolved at proxy time).

```bash
primitive config set vars ACCOUNT_ID=acct_123
primitive sync push --only var/ACCOUNT_ID
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
status = "draft"
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
forwardQueryParams = ["q"]   # Only let clients control the search term.

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
status = "draft"
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
# WRONG - omitting allowedPaths defaults to ["/*"]; client can hit any path on baseUrl
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

### 3. `forwardHeaders = ["*"]` leaks every client header upstream

```toml novalidate
# WRONG - forwards Cookie, Authorization, X-Internal-* everything from the client
forwardHeaders = ["*"]

# RIGHT - explicit allowlist
forwardHeaders = ["x-trace-id", "accept-language"]
```

Never put `authorization` in `forwardHeaders` unless the client genuinely owns the upstream credential. Use `defaultHeaders` + `{{secrets.KEY}}` instead.

### 4. Don't hardcode secrets in TOML

```toml novalidate
# WRONG - secret committed to source control and visible in the admin UI
[requestConfig.defaultHeaders]
Authorization = "Bearer sk-abc123..."

# RIGHT - reference an app secret
[requestConfig.defaultHeaders]
Authorization = "Bearer {{secrets.OPENAI_API_KEY}}"
```

### 5. Forgetting to flip status to `active`

Newly-pushed integrations land in `status = "draft"`. Drafts can be exercised via `primitive integrations test` but **not** via `client.integrations.call()` — clients get HTTP 404 / `INTEGRATION_NOT_FOUND`.

```bash
primitive config set integration/weather-api integration.status=active
primitive sync push --only integration/weather-api
```

### 6. `responsePassthrough` has no effect

The proxy always returns `{ status, headers, body }` from upstream regardless of this flag. Don't rely on toggling it.

### 7. Header names in `forwardHeaders` are case-insensitive — write them lowercased

The platform lowercases the allowlist; `["Content-Type"]` and `["content-type"]` behave identically. Stick with lowercase to match what you'll see in logs.

## CLI Reference

### Auth and app context

```bash
primitive login                  # Browser-based auth
primitive use <app-id>           # Persist current app
# or pass --app <app-id> to any subcommand
```

### Reading integrations

```bash
primitive integrations list [--status draft|active|archived] [--json]
primitive integrations get <integration-id> [--json]
```

### Writing integrations (TOML only)

An integration is `integrations/<key>.toml`. There is no create/update/delete command — author the file and push it.

```bash
primitive config fields integration                  # every key, type, required, default
primitive config new integration weather-api         # scaffold integrations/weather-api.toml
primitive config set integration/weather-api integration.status=active
primitive sync push --only integration/weather-api # apply just this one
```

Delete an integration by removing its file and running `primitive sync push --prune`.

### Test (admin only — bypasses status check)

```bash
primitive integrations test <id>                                    # CLI defaults: --method GET, no path
primitive integrations test <id> --method POST --path /v1/responses
primitive integrations test <id> --query '{"q":"hello","limit":10}'
primitive integrations test <id> --method POST --body '{"foo":"bar"}'
```

Note: `--method` defaults to `GET` regardless of the integration's `defaultMethod`. Pass `--method` explicitly when the integration only allows non-GET methods.

The test endpoint runs with `allowInactive: true`, includes a `requestPreview` (with secrets redacted), and works against `draft` integrations.

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
primitive config set vars ACCOUNT_ID=acct_123     # edits vars.toml, no API call
primitive sync push --only var/ACCOUNT_ID       # applies it
```

Delete a var by removing its line from `vars.toml` and pushing. Values are plaintext (never encrypted, never masked). Max 100 vars per app, max 2 KB per value.

### Test Cases (regression suite for an integration)

```bash
primitive integrations tests list <id>
primitive integrations tests create <id> --name "Basic" --input '{"method":"GET","path":"/get"}'
primitive integrations tests run <id> <test-case-id>
primitive integrations tests run-all <id>
primitive integrations tests run-all <id> --test-cases 01ABC,01DEF
primitive integrations tests runs <id> [--limit 20] [--group <comparison-group>]
```

### Sync (TOML <-> server)

Integration configs live at `integrations/<key>.toml`, one file per integration. See the [Configuration guide](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md#the-sync-loop) for the sync loop (`init`/`pull`/`diff`/`push`), the `--dir` override, snapshots, pruning, and conflict handling.

## Calling from the Client SDK

{{ example: integrations/integration-call-response }}

### `IntegrationCallRequest`

| Field | Type | Notes |
|-------|------|-------|
| `integrationKey` | string | Required. |
| `method` | string | Optional. Must be in `allowedMethods`; defaults to `defaultMethod`. |
| `path` | string | Optional. Must match `allowedPaths`. Must start with `/`. Cannot be absolute (`http(s)://...` is rejected). |
| `query` | `Record<string, any>` | Filtered by `forwardQueryParams`. Arrays append multiple values; objects are JSON-stringified. |
| `headers` | `Record<string, string>` | Filtered by `forwardHeaders`. `host`/`content-length` always stripped. |
| `body` | any | For non-JSON bodies, set `bodyMode` on the integration. |

### Error handling

{{ example: integrations/integration-call-errors }}

### Server-side proxy error codes (`errorCode` on response or in `JsBaoError.details`)

| Code | HTTP | Meaning |
|------|------|---------|
| `INTEGRATION_INACTIVE` | 404 | `status != "active"`. |
| `DISALLOWED_METHOD` | 422 | Method not in `allowedMethods`. |
| `DISALLOWED_PATH` | 422 | Path doesn't match `allowedPaths`. |
| `REQUEST_BODY_TOO_LARGE` | 413 | Body exceeds `maxRequestBodyBytes`. |
| `UPSTREAM_TIMEOUT` | 504 | Upstream took longer than `timeoutMs`. |
| `UPSTREAM_NETWORK_ERROR` | 502 | Upstream unreachable. |
| `UPSTREAM_ERROR` | matches upstream | Upstream returned 4xx/5xx; body still passed through. |
| `INVALID_BASE64` | 400 | Bad attachment data. |

## Calling from a Workflow

Use the `integration.call` step kind:

```typescript
{
  kind: "integration.call",
  integrationKey: "open-ai",
  request: {
    method: "POST",
    path: "/v1/responses",
    body: { model: "gpt-4.1-mini", input: "hi" },
  },
}
```

The step pulls App Secrets and resolves `{{secrets.KEY}}` the same way the user-facing proxy does. 4xx (except 429) become non-retryable errors; 5xx and 429 are retryable.

## Body Modes

- `"json"` (default) — body is JSON-stringified; `Content-Type: application/json` set unless overridden.
- `"raw"` — first attachment is sent as the request body; `Content-Type` defaults to the attachment's `type`.
- `"multipart"` — `multipart/form-data` constructed from `multipartFieldMapping`. Each entry is either `type = "value"` (literal field) or `type = "attachment"` (binary part by `attachmentIndex` or `attachmentName`). If no mapping is provided and at least one attachment is supplied, the first attachment is auto-mapped to field `file` and any object body becomes additional text parts.

## Status Lifecycle

| Status | Callable by clients? | `primitive integrations test`? |
|--------|----------------------|--------------------------------|
| `draft` | No (404) | Yes |
| `active` | Yes | Yes |
| `archived` | No (404) | Yes |

Deleting an integration — remove its TOML file, then `primitive sync push --prune` — sets it `archived`. A hard delete, which removes the row permanently, is available over the API and in the console.

## Files on Disk (sync mode)

- `config/integrations/<key>.toml` — one file per integration
- `config/.primitive-sync.json` — sync state (last pull/push hashes)

## Quick Triage

```bash
primitive integrations get <id> --json     # see exact stored requestConfig
primitive integrations test <id> --method GET --path /probe --query '{"x":"1"}'
primitive integrations logs <id> --limit 20
primitive secrets list                      # confirm referenced secret keys exist
primitive vars list                         # confirm referenced var keys exist
```

If client calls return `INTEGRATION_NOT_FOUND` despite the integration existing, check status — drafts/archived return 404.

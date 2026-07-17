# Integrations Guide for Coding Agents

Integrations let tenant apps call third-party APIs through a server-side proxy without exposing credentials to clients. The proxy enforces an allowlist of methods and paths, injects credentials from App Secrets, and returns the upstream response to the caller.

This is the **outbound** half of a third-party integration. The **inbound** half — a webhook the provider calls to trigger a workflow, configured with a verification scheme — is in the Workflows guide ("Inbound webhooks"). Most third-party integrations need both: outbound calls plus an inbound webhook that triggers a workflow.

## Call an integration

```typescript
  const response = await client.integrations.call({
    integrationKey: "crm-api",
    method: "POST",
    path: "/contacts",
    body: { email: "user@example.com", name: "Alice" },
  });
```

This guide is verified against `js-bao-wss` source. Anything not described here is unsupported.

## Architecture

- Admin defines an `AppIntegration` (per-app, keyed by `integrationKey`) with a `requestConfig` that pins the `baseUrl`, `allowedMethods`, and `allowedPaths`.
- Credentials are stored as **App Secrets** (`primitive secrets set KEY --value ...`), then referenced from `defaultHeaders` or `staticQuery` via `{{secrets.KEY}}` templates.
- Non-secret config values are stored as **Config Vars** (`primitive vars set KEY --value ...`), referenced the same way via `{{vars.KEY}}` templates — see "Vars vs Secrets" below.
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
primitive vars set ACCOUNT_ID --value acct_123 --summary "Default account id"
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
primitive integrations update <integration-id> --status active
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

### Integration CRUD

```bash
primitive integrations list [--status draft|active|archived] [--json]
primitive integrations get <integration-id> [--json]

# Create from TOML (preferred)
primitive integrations create --from-file path/to/integration.toml

# Create inline
primitive integrations create \
  --key weather-api \
  --name "Weather API" \
  --base-url "https://api.openweathermap.org/data/2.5" \
  --allowed-methods "GET" \
  --allowed-paths "/weather,/forecast" \
  --timeout-ms 30000

primitive integrations update <id> --name "..." --description "..." \
                                   --status active --timeout-ms 60000

primitive integrations delete <id>            # archive (soft)
primitive integrations delete <id> --hard     # permanent
primitive integrations delete <id> -y         # skip confirm
```

### Test (admin only — bypasses status check)

```bash
primitive integrations test <id>                                    # CLI defaults: --method GET, no path
primitive integrations test <id> --method POST --path /v1/responses
primitive integrations test <id> --query '{"q":"hello","limit":10}'
primitive integrations test <id> --method POST --body '{"foo":"bar"}'
```

Note: `--method` defaults to `GET` regardless of the integration's `defaultMethod`. Pass `--method` explicitly when the integration only allows non-GET methods.

The test endpoint runs with `allowInactive: true` and `allowMissingSecret: true`, includes a `requestPreview` (with secrets redacted), and works against `draft` integrations.

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
primitive vars set ACCOUNT_ID --value acct_123 --summary "Default account id"
primitive vars delete ACCOUNT_ID
```

Values are plaintext (never encrypted, never masked). Max 100 vars per app, max 2 KB per value.

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

```bash
primitive sync init     # create config tree + .primitive-sync.json
primitive sync pull     # server -> local TOML
primitive sync push [--dry-run] [--force]
primitive sync diff
```

The sync directory auto-resolves to `.primitive/sync/<env>/<appId>/`; pass `--dir <path>` to override it. `init` creates these subdirs: `integrations/`, `webhooks/`, `cron-triggers/`, `blob-buckets/`, `prompts/`, `workflows/`, `database-types/`, `rule-sets/`, `group-type-configs/`, `collection-type-configs/`, `email-templates/`. Integration files live at `integrations/<key>.toml`.

## Typical Workflow

# This worked example pins a fixed sync dir (`--dir ./config`) so the file paths
# below are concrete; omit `--dir` to use the default `.primitive/sync/<env>/<appId>/`.

```bash
# Initial setup
primitive sync init --dir ./config
primitive sync pull --dir ./config

# Add a new integration
cat > config/integrations/openai.toml <<'EOF'
[integration]
key = "open-ai"
displayName = "OpenAI"
status = "draft"

[requestConfig]
baseUrl = "https://api.openai.com/"
allowedMethods = ["POST"]
allowedPaths = ["/v1/responses", "/v1/chat/completions"]
defaultMethod = "POST"

  [requestConfig.defaultHeaders]
  Content-Type = "application/json"
  Authorization = "Bearer {{secrets.OPENAI_API_KEY}}"
EOF

# Push and configure
primitive sync push --dir ./config --dry-run
primitive sync push --dir ./config
primitive secrets set OPENAI_API_KEY --value sk-...
primitive integrations test <integration-id> --method POST --path /v1/responses \
  --body '{"model":"gpt-4.1-mini","input":"hi"}'
primitive integrations update <integration-id> --status active
```

If `sync push` reports a conflict, someone modified the server since your last `pull`. Either re-pull and merge, or `--force` to overwrite the server.

## Calling from the Client SDK

```typescript
  const response = await client.integrations.call({
    integrationKey: "open-ai",
    method: "POST",
    path: "/v1/responses",
    body: { model: "gpt-4.1-mini", input: "hi" },
  });

  console.log(response.status); // upstream HTTP status (number)
  console.log(response.headers); // Record<string, string>
  console.log(response.body); // upstream body, parsed if JSON
  console.log(response.traceId); // optional, correlates with admin logs
  console.log(response.durationMs); // optional, proxy-side duration
  console.log(response.errorCode); // optional, set when status >= 400
```

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

```typescript
  try {
    await client.integrations.call({
      integrationKey: "crm",
      method: "POST",
      path: "/contacts",
      body: { email: "user@example.com" },
    });
  } catch (error) {
    if (isJsBaoError(error)) {
      switch (error.code) {
        case "INTEGRATION_NOT_FOUND":
          break; // 404, also fires for status != active
        case "INTEGRATION_SECRET_MISSING":
          break; // 409, MISSING_SECRET upstream
        case "INTEGRATION_REQUEST_INVALID":
          break; // 400/413/422 (method/path/body)
        case "INTEGRATION_PROXY_FAILED":
          break; // upstream timeout/network/malformed
        case "ACCESS_DENIED":
          break; // 401/403 (client auth)
        case "INVALID_ARGUMENT":
          break; // missing integrationKey
        case "OFFLINE":
          break; // SDK in offline mode
      }
    }
  }
```

### Server-side proxy error codes (`errorCode` on response or in `JsBaoError.details`)

| Code | HTTP | Meaning |
|------|------|---------|
| `INTEGRATION_INACTIVE` | 404 | `status != "active"`. |
| `MISSING_SECRET` | 409 | A stored integration-scoped secret row is required but missing. Integrations using `{{secrets.KEY}}` app-secret templates don't trigger this. |
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

`primitive integrations delete <id>` (soft) sets `archived`. `--hard` permanently removes the row.

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

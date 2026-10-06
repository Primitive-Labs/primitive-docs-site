# Integrations Guide for Coding Agents

An integration defines the external API a server function can call. Configure
its base URL, allowed methods and paths, and credential references. Primitive
adds credentials to the request without exposing them to function code.

## Define an integration

Create a TOML config file for each external API:

```toml
# primitive/dev/integrations/weather-api.toml
[integration]
key = "weather-api"
displayName = "Weather API"

[requestConfig]
baseUrl = "https://api.weather.com/v1"
allowedMethods = ["GET"]
allowedPaths = ["/forecast/*", "/current"]

[requestConfig.defaultHeaders]
X-API-Key = "{{secrets.WEATHER_API_KEY}}"
```

```bash
primitive config push
```

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
```## Request and response

`ctx.integrations.call(key, request)` returns
`{ status, headers, body, durationMs, traceId, errorCode? }`. Check `errorCode`
before using the result.

| Field | Type | Notes |
|-------|------|-------|
| `method` | string | Optional. Must be in `allowedMethods`; defaults to `defaultMethod`. |
| `path` | string | Optional. Relative to `baseUrl`; must match `allowedPaths`. Resolved and normalized before the check, so `//elsewhere/x` and `/allowed/../blocked` are refused. |
| `query` | `Record<string, unknown>` | Filtered by `forwardQueryParams`. Arrays append multiple values; objects are JSON-stringified. |
| `headers` | `Record<string, string>` | Filtered by `forwardHeaders`. `host`/`content-length` always stripped. |
| `body` | unknown | JSON by default; for non-JSON bodies, set `bodyMode`. |
| `form` | `Record<string, unknown>` | form-urlencoded body; not combinable with `body`. |
| `bodyMode` | `"json"` \| `"raw"` \| `"multipart"` | Overrides the integration's `bodyMode` for this call. |

## Test and inspect

```bash
primitive integrations test <integration-id> --path /current
primitive integrations logs <integration-id>
primitive integrations logs <integration-id> --run <run-id>
primitive integrations logs <integration-id> --run <run-id> --step <name>
```

A call made inside a function's `step.do` names its step: the STEP column, and
under `--json` `correlation.stepId`, `correlation.stepOccurrence` (which use
of the name, from 0) and `detail.stepAttempt` (which run of the body, from 1).
The attempt is the function log's own count, so it restarts at 1 after the
task pauses. A step name is recorded with secret values redacted; it is
dropped when a declared secret cannot be loaded or the name exceeds 256 bytes.

Create regression cases in `integrations/<key>.tests/<case>.toml` and push
before running them. `inputVariables` contains the request object as JSON text:

```toml
# integrations/weather-api.tests/current.toml
[test]
name = "current"
inputVariables = '{"method":"GET","path":"/current"}'
expectedOutputPattern = "temperature"
```

```bash
primitive config push --only integration/weather-api
primitive integrations tests run-all <integration-id>
```

Remove a case file and push with `--prune` to delete it. Omitted request fields
use integration defaults; no body is sent unless the case supplies one.

## Gotchas

- Declare `integration:<key>` in the function’s capabilities. Its `access`
  rule controls callers; integrations have no separate member access rule.
- Set `allowedPaths` explicitly. Its default is `/*`, and a trailing wildcard
  matches a prefix rather than an arbitrary glob.
- Empty `forwardHeaders` and `forwardQueryParams` allow all caller-supplied
  values. Set explicit lists when they need restriction.
- Store credentials as app secrets and reference `{{secrets.KEY}}` in
  `defaultHeaders` or `staticQuery`. Never inline credentials in TOML.
- Secrets must exist when referenced during save. Missing vars remain literal
  placeholders; var values are not redacted.
- Redirects stay on the configured origin and must match the allowed path and
  method. A disallowed redirect is returned without being followed.
- Requests are limited by the smaller of the integration timeout and the
  function’s remaining time. A timeout does not imply the provider performed
  no side effects; use provider idempotency keys for repeatable writes.
- Use `disable`/`enable` for availability. Archiving stops calls while retaining
  the key; pruning permanently deletes the integration.

## Errors

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

See [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#calling-an-integration) for the function side.## Body modes

- `json`: JSON body; the default.
- `form`: use the request’s `form` field for URL-encoded data, without `body`.
- `raw`: sends attachment data as the body.
- `multipart`: maps attachments and values to multipart fields.

Use `primitive config fields integration` for the configuration fields.
For inbound events, declare a webhook on the receiving server function.

## Related guides

- [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#calling-an-integration)
- [App Secrets](AGENT_GUIDE_TO_PRIMITIVE_APP_SECRETS.md)
- [Configuration](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md)

# Agent Guide to Primitive Error Handling

Use an error’s `code` to choose a user-facing message or recovery action. Codes distinguish failures that share an HTTP status. Human-readable messages can change; do not parse them.

## Branching by Code

```typescript
  try {
    await client.documents.open(documentId);
  } catch (err) {
    if (err instanceof JsBaoApiError) {
      switch (err.code) {
        case "DOC_ACCESS_DENIED":
        case "ACCESS_DENIED":
          show(t("errors.noAccessToDocument"));
          return;
        case "NOT_FOUND":
          show(t("errors.documentGone"));
          return;
        case "UNAUTHENTICATED":
        case "INVALID_TOKEN":
          show(t("errors.signInAgain"));
          return;
        case "RATE_LIMITED":
          show(t("errors.slowDown"));
          return;
        case "STATE_CONFLICT":
          show(t("errors.alreadyExists"));
          return;
        case undefined:
          // No code: the response did not come from the platform's error
          // path — an older server, or an intermediary's HTML 502.
          show(t("errors.unknown"));
          return;
        default:
          show(t("errors.unknown"));
          return;
      }
    }
    throw err;
  }
```

Read `JsBaoApiError.code`, including for file transfers and failed token refreshes.

`JsBaoApiError` is the transport error; `isJsBaoError` deliberately excludes it, so a client-side `JsBaoError` (`LOCK_TIMEOUT`, `INVALID_ARGUMENT`, …) and a server code never get confused even when they spell a value the same way.

The CLI's `ApiError` exposes the same value as `code`, with `statusCode` and `details` beside it.

## The Envelope

Every 4xx and 5xx response with a body, on both `/app/{appId}/api/*` and `/admin/api/*`, is `application/json`:

```json
{
  "error": "Document not found",
  "status": 404,
  "code": "NOT_FOUND",
  "timestamp": "2026-09-13T04:12:57.219Z",
  "details": {}
}
```

| Field | Contract |
|---|---|
| `code` | Non-empty string, always present. The handler's own explicit code where it has one (`DOC_ACCESS_DENIED`, `FUNCTION_DISABLED`, `VALIDATION_FAILED`, `RATE_LIMITED`, …), otherwise the status-derived default below. An explicit code always wins. **This is the only field to branch on.** |
| `error` | Human-readable text, written for a developer reading a log. **It may change without notice** — never parse it, never show it to a user. |
| `status` | The numeric HTTP status, repeated in the body. |
| `timestamp` | ISO 8601, when the server produced the failure. |
| `details` | Present only when a handler attaches structured context: offending field names on a validation failure, or `retryAfter` (seconds) on a `RATE_LIMITED` refusal such as a document access request or a sign-in verification. Exception: a server function's `FUNCTION_RATE_LIMITED` carries `retryAfter` at the top level of the body, plus a `Retry-After` header. |

Two field rules an agent must encode:

- A few bodies carry the human text under **`message`** instead of `error` (the API integration proxy's refusals). Read `error ?? message`.
- Some bodies also carry **`errorCode`**, the same value as `code`. The two spellings agree; either one is the code.

Every body carries `code`, whichever of these other spellings it also has.

## Default Codes by Status

| Status | `code` |
|---|---|
| 400 | `INVALID_REQUEST` |
| 401 | `UNAUTHENTICATED` |
| 403 | `ACCESS_DENIED` |
| 404 | `NOT_FOUND` |
| 405 | `METHOD_NOT_ALLOWED` |
| 409 | `STATE_CONFLICT` |
| 410 | `GONE` |
| 413 | `PAYLOAD_TOO_LARGE` |
| 415 | `UNSUPPORTED_MEDIA_TYPE` |
| 422 | `UNPROCESSABLE` |
| 429 | `RATE_LIMITED` |
| 500 | `INTERNAL_ERROR` |
| 501 | `NOT_IMPLEMENTED` |
| 502 | `UPSTREAM_ERROR` |
| 503 | `UNAVAILABLE` |
| 504 | `UPSTREAM_TIMEOUT` |

Any other 4xx is `INVALID_REQUEST`; any other 5xx is `INTERNAL_ERROR`.

**The 409 distinction matters.** A generic conflict — a duplicate name, a resource that already exists — is `STATE_CONFLICT`. `CONFLICT` is reserved for the optimistic-concurrency refusal, which also carries `serverModifiedAt` and `expectedModifiedAt`. Do not route a `STATE_CONFLICT` through a merge/retry-on-conflict path.

## Gotchas

### When There Is No Code

`code` is absent when the response **did not come from the platform's error path**: an intermediary answering on its own (a CDN's HTML 502, a proxy timeout page). Treat it as an unknown failure with a generic message. Do not invent a code for it, and do not infer one from the status — the platform's own defaults are already applied server-side, so an absent code means the platform did not answer.

## What Is Not Covered

- **3xx is not an error**: the blob download's `304 Not Modified` is bodyless, and the OAuth initiation redirects.
- **A failing `HEAD` has no body** — HTTP forbids one. It answers the same status and headers its `GET` counterpart would; issue the `GET` when the code is needed.
- **WebSocket error frames** are a separate surface with their own shape.
- **A server function that throws answers HTTP `200`**, not a 4xx/5xx: the envelope carries `status: "failed"` and an `errorCode` (`FUNCTION_THREW`, `OUTPUT_SCHEMA_VIOLATION`, …). Only a refusal before the code ran (gate, unknown key, rate ceiling) is an error response. See [Server Functions — Invoking](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#invoking).

## Related Guides

- [Authentication](AGENT_GUIDE_TO_PRIMITIVE_AUTHENTICATION.md) — the sign-in codes (`INVITATION_REQUIRED`, `DOMAIN_NOT_ALLOWED`, `OTP_MAX_ATTEMPTS`, …).
- [Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md) — where the `<DOMAIN>_ACCESS_DENIED` refusals come from.

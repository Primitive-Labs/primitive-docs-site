# Agent Guide to Primitive Error Handling

Every 4xx/5xx response the platform produces carries a **stable, machine-readable `code`**. Branch on the code; never match on the human text. An app that localizes its interface needs the cause, and the HTTP status alone does not carry it — a 403 can be "not a member of this app" or "this document's access rule says no", and those are different sentences in every language the app ships.

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
| `details` | Present only when a handler attaches structured context (offending field names on a validation failure, `retryAfter` on a rate limit). |

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

## Branching by Code

```swift
  do {
    _ = try await client.documents.open(documentId)
  } catch let error as HttpError {
    switch error.serverCode {
    case "DOC_ACCESS_DENIED", "ACCESS_DENIED":
      show(t("errors.noAccessToDocument"))
    case "NOT_FOUND":
      show(t("errors.documentGone"))
    case "UNAUTHENTICATED", "INVALID_TOKEN":
      show(t("errors.signInAgain"))
    case "RATE_LIMITED":
      show(t("errors.slowDown"))
    case "STATE_CONFLICT":
      show(t("errors.alreadyExists"))
    case nil:
      // No code: the response did not come from the platform's error path —
      // an older server, or an intermediary's HTML 502.
      show(t("errors.unknown"))
    default:
      show(t("errors.unknown"))
    }
  }
```

The code is `HttpError.serverCode` (`String?`), with the parsed human text in `serverMessage` and the raw body in `body`. It is populated on every failing path, including the terminal error thrown after a refused token refresh — that one keeps its `Invalid credentials` message and carries the original 401's `body` and `serverCode` alongside it.

The **raw bytes** transfers carry it too — a blob upload or download, a bucket transfer and an avatar upload read the response body as bytes rather than through the typed request path, and parse the same envelope: the error keeps its message and carries `serverCode` beside it.

`HttpError.authCode` gives a typed `AuthCode` when `serverCode` matches a known case; fall back to the raw `serverCode` string for codes the SDK does not yet name. Client-side sentinels use a `CLIENT_` prefix, so they never collide with a server code.

## When There Is No Code

`code` is absent when the response **did not come from the platform's error path**: an intermediary answering on its own (a CDN's HTML 502, a proxy timeout page). Treat it as an unknown failure with a generic message. Do not invent a code for it, and do not infer one from the status — the platform's own defaults are already applied server-side, so an absent code means the platform did not answer.

## What Is Not Covered

- **3xx is not an error**: the blob download's `304 Not Modified` is bodyless, and the OAuth initiation redirects.
- **A failing `HEAD` has no body** — HTTP forbids one. It answers the same status and headers its `GET` counterpart would; issue the `GET` when the code is needed.
- **WebSocket error frames** are a separate surface with their own shape.
- **A server function that throws answers HTTP `200`**, not a 4xx/5xx: the envelope carries `status: "failed"` and an `errorCode` (`FUNCTION_THREW`, `OUTPUT_SCHEMA_VIOLATION`, …). Only a refusal before the code ran (gate, unknown key, rate ceiling) is an error response. See [Server Functions — Invoking](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#invoking).

## Related Guides

- [Authentication](AGENT_GUIDE_TO_PRIMITIVE_AUTHENTICATION.md) — the sign-in codes (`INVITATION_REQUIRED`, `DOMAIN_NOT_ALLOWED`, `OTP_MAX_ATTEMPTS`, …).
- [Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md) — where the `<DOMAIN>_ACCESS_DENIED` refusals come from.

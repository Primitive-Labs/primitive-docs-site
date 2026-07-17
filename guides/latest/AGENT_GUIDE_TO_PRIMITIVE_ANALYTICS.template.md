# Agent Guide to Primitive Analytics

Guidelines for AI agents implementing analytics tracking in Primitive apps.

## Overview

Primitive provides built-in analytics. The platform tracks user activity and resource lifecycle automatically and stores events server-side for querying. The system handles offline persistence, rate limiting, and automatic lifecycle events out of the box.

Read aggregated analytics (DAU/WAU/MAU, retention, top users, event feeds) through the `primitive` CLI, the REST API, or workflow steps — covered below.

---

## What's Tracked Automatically (Zero Developer Work)

Standing up an app gets you DAU/WAU/MAU tracking, session analytics, document/permission audit trails, and full workflow/prompt/integration observability with no instrumentation.

**Key constraints for custom events:**
- Every analytics event requires an authenticated user. Events without a `user_ulid` are dropped silently. Use the unauthenticated-user constant for pre-auth screens.
- A tenant ID (resolved automatically from the client's `appId`) must also be present, or the event is dropped.

{{#lang ts}}
### Client-Side Auto Events

All enabled by default. The client automatically emits these lifecycle events:

| Action | Feature | Default | When it fires |
|--------|---------|---------|---------------|
| `user_active_daily` | `session` | on | First authenticated activity on a UTC day |
| `user_returned` | `session` | on | Tab becomes visible (suppressed if a previous `user_returned` fired less than `minResumeMs` ago — default 5 min) |
| `session_end` | `session` | on | `beforeunload` or `client.destroy()` (records `duration_ms`) |
| `sync_error` | `sync` | on | Outbound sync fails (rate-limited, default min 30s between events) |
| `blob_upload_started` | `blobs` | on | Blob upload begins |
| `blob_upload_succeeded` | `blobs` | on | Blob upload completes |
| `blob_upload_failed` | `blobs` | on | Blob upload fails |
| `prompt_started` | `llm` | on | `client.llm.chat()` request begins |
| `prompt_succeeded` | `llm` | on | `client.llm.chat()` succeeds (records `duration_ms`) |
| `prompt_failed` | `llm` | on | `client.llm.chat()` fails |
| `prompt_started` | `gemini` | on | Gemini `generate` / `countTokens` / `generateRaw` begins |
| `prompt_succeeded` | `gemini` | on | Gemini `generate` / `countTokens` / `generateRaw` succeeds |
| `prompt_failed` | `gemini` | on | Gemini `generate` / `countTokens` / `generateRaw` fails |
{{/lang}}

### Server-Side Events

The platform emits these from the server. No client code at all.

| Action | Feature | When |
|--------|---------|------|
| `session.refreshed` | `auth` | JWT refresh succeeds |
| `document.created` | `documents` | Document created |
| `document.viewed` | `documents` | Document info fetched — fires on every open (the client fetches info as part of opening) |
| `document.opened` | `documents` | Document opened via alias resolution; plain `documents.open(id)` does not fire it |
| `document.updated` | `documents` | Document metadata updated (title/thumbnail/metadata) — document content edits are not individually evented |
| `document.deleted` | `documents` | Document deleted |
| `document.tag_added` | `documents` | Tag added |
| `document.tag_removed` | `documents` | Tag removed |
| `access_request.created` | `documents` | User requested access to a document |
| `access_request.approved` | `documents` | Access request approved |
| `access_request.denied` | `documents` | Access request denied |
| `permission.granted` | `permissions` | Permission granted |
| `permission.revoked` | `permissions` | Permission revoked |
| `permission.pending.cancelled` | `permissions` | Pending invite-permission cancelled |
| `ownership.transferred` | `permissions` / `ownership` | Document ownership transferred (emitted from both controllers) |
| `invitation.sent` | `invitations` | Invitation sent |
| `invitation.cancelled` | `invitations` | Invitation cancelled |
| `invitation.declined` | `invitations` | Invitation declined |
| `user.removed` | `users` | User removed from app |
| `user.role_changed` | `users` | User role changed |
| `prompt.executed` | `prompts` | Prompt execution completes |
| `workflow.started` | `workflows` | Workflow run begins |
| `workflow.completed` | `workflows` | Workflow run succeeds |
| `workflow.failed` | `workflows` | Workflow run fails |
| `integration.invoke` | _(integration key)_ | Integration proxy call |
| `created` | `token` | API token created |
| `revoked` | `token` | API token revoked |

Workflow and prompt events also record `duration_ms` and LLM token counts (`input_tokens`, `output_tokens`, `total_tokens`) when available.

{{#lang ts}}
### Auto-Populated Fields

Every event (auto or custom) gets these populated automatically:

`tenant_id` (from `client.appId`), `route` (from `window.location.pathname`), `device_type`, `os_name`, `os_version`, `browser_name`, `browser_version`, `plan` (default `"unknown"`), `connection_id`.
{{/lang}}

### Offline Persistence and Rate Limiting

Events are buffered on the device and persisted locally while offline; persisted events are flushed automatically when the WebSocket reconnects.
{{#lang ts}}
A rate limiter caps emission at **300 events/minute with a 60-token burst** — events over the cap are dropped silently. No special code needed.
{{/lang}}
{{#lang swift}}
A rate limiter caps emission at **300 events per 60-second window, with no more than 60 events in the first 10 seconds** — events over the cap are dropped silently. No special code needed.
{{/lang}}

The offline buffer is persisted with a **~1 MiB** cap; when it exceeds the cap the **oldest** events are dropped.

---

## Logging Custom Events

### Basic Event

`action` and `user_ulid` are required; `feature` (defaults to `"unspecified"`) groups related events.

{{ example: analytics/log-event }}

{{#lang ts}}
`action` and `user_ulid` are required by the TypeScript interface. At runtime, `user_ulid` will be back-filled from the queue's user-resolver if you omit it AND the resolver returns a value — but the type checker won't let you omit it, so always pass it explicitly (or use `ANALYTICS_UNAUTHENTICATED_USER`).
{{/lang}}
{{#lang swift}}
`analytics.logEvent` takes an `AnalyticsEventInput`. `user_ulid` is optional on the struct: when omitted it is back-filled from the client's current user, or set to `AnalyticsEventInput.unauthenticatedUser` when no user is signed in.
{{/lang}}

### Event with Context

Pass a `context_json` object for per-event debug data. The serialized payload is bounded at **1 KiB**, so keep it small — don't dump request bodies or full reports.

{{ example: analytics/log-event-context }}

{{#lang ts}}
`context_json` accepts a `Record<string, unknown>` or a JSON string. It is serialized then truncated to 1 KiB of UTF-8 (truncation respects code-point boundaries). If you pass an object and don't override defaults, the queue auto-includes `ua`, `language`, and `screen` dimensions.
{{/lang}}
{{#lang swift}}
`context_json` is a `JSONValue` — construct it with object/array/scalar literals. If its serialized size exceeds 1 KiB, the field is dropped from the event before sending.
{{/lang}}

{{#lang ts}}
### AnalyticsEventInput Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | `string` | Yes | Event name (verb_noun: `"login"`, `"photo_uploaded"`) |
| `user_ulid` | `string` | Yes | User ULID, or `ANALYTICS_UNAUTHENTICATED_USER` |
| `feature` | `string` | No | Feature/module name (defaults to `"unspecified"`) |
| `route` | `string` | No | Page path (defaults to `window.location.pathname`) |
| `plan` | `string` | No | Plan name (auto-resolved, defaults to `"unknown"`) |
| `tenant_id` | `string` | No | App ULID (auto-populated from `client.appId`) |
| `device_type` | `string` | No | `"desktop"`, `"mobile"`, etc. (auto-detected) |
| `os_name` | `string` | No | OS name (auto-detected) |
| `os_version` | `string` | No | OS version (auto-detected) |
| `browser_name` | `string` | No | Browser name (auto-detected) |
| `browser_version` | `string` | No | Browser version (auto-detected) |
| `app_version` | `string` | No | Your app's version (or set via `setAppVersionOverride`) |
| `context_json` | `string \| Record<string, unknown> \| null` | No | Debug context (truncated to 1 KiB) |
| `user_created_at_epoch_s` | `number` | No | User signup timestamp (epoch seconds) |

### Don't

```typescript
// WRONG: No user_ulid — TypeScript error AND silently dropped at runtime
client.analytics.logEvent({ action: "click", feature: "nav" });

// WRONG: huge context payload (will be truncated to 1 KiB, losing data)
client.analytics.logEvent({
  action: "report_generated",
  user_ulid: u,
  context_json: { entireReportBody: hugeString },
});

// WRONG: high-frequency telemetry (will be rate-limited and dropped)
window.addEventListener("mousemove", () => {
  client.analytics.logEvent({ action: "mouse_move", user_ulid: u });
});
```
{{/lang}}

{{#lang swift}}
### AnalyticsEventInput Fields

`AnalyticsEventInput` carries the same fields as the event row (`action`, `feature`, `route`, `plan`, `tenant_id`, `user_ulid`, `device_type`, `os_name`, `os_version`, `browser_name`, `browser_version`, `app_version`, `context_json`, `user_created_at_epoch_s`). Only `action` is required at the initializer: `user_ulid` is back-filled from the signed-in user (or the unauthenticated-user constant), and `context_json` is a `JSONValue`.
{{/lang}}

---

## Logging Snapshots

`logSnapshot` records a state snapshot event. It auto-resolves the user; if no user is authenticated, the call is a no-op (no error).

{{ example: analytics/log-snapshot }}

This logs an event with `action: "_snapshot"`, `feature: "_state"`, and your context as `context_json`.

---

## Pre-Auth Events

Events with no authenticated user are dropped. To log on pre-auth screens (landing pages, sign-up flow), pass the unauthenticated-user constant as the `user_ulid`. Its value is `"UNAUTHENTICATED"`. Use sparingly — most analytics should be tied to real users.

{{ example: analytics/log-event-preauth }}

---

## Manual Flush

Events are buffered and flushed automatically — including when the WebSocket reconnects — so you rarely need to flush manually. Call `flush` to force a send (e.g. before an explicit teardown).

{{ example: analytics/manual-flush }}

{{#lang ts}}
The queue auto-flushes every **100ms** or when the buffer reaches **25 KiB**, and the client also flushes on `beforeunload`, on tab visibility hidden, and on reconnect.

Pre-existing browser hooks (added by the client itself):
- `beforeunload` → fires `session_end`, then flushes
- `visibilitychange` (hidden) → flushes
- `visibilitychange` (visible) → triggers `user_returned`
- `status` → on reconnect, flushes

So **don't** add your own `beforeunload → flush` listener — it's redundant and `session_end` is already handled.
{{/lang}}
{{#lang swift}}
The queue auto-flushes every **100ms** (or earlier when batched). `await client.destroy()` cancels the flush timer and triggers a final flush before storage closes, so you don't need a manual flush on teardown.
{{/lang}}

---

## Plan and App Version Overrides

If your app reports its plan/version dynamically (e.g. after an in-app upgrade), set them on the client. They flow into every subsequent event automatically. Pass `null`/`nil` to clear an override.

{{ example: analytics/analytics-overrides }}

---

## Configuring Auto Events

Pass `analyticsAutoEvents` to the constructor. All sub-options default to enabled.

{{ example: analytics/auto-events-config }}

{{#lang ts}}
Accepted shapes:
- `dailyAuth`, `returnActive`, `sessionEnd`: `boolean`
- `minResumeMs`: `number` (ms before another `user_returned` will fire)
- `syncErrors`: `boolean | { enabled?: boolean; minIntervalMs?: number }`
- `blobUploads`: `{ start?: boolean; success?: boolean; failure?: boolean }`
- `llm`, `gemini`: `boolean | { start?: boolean; success?: boolean; failure?: boolean }`
{{/lang}}
{{#lang swift}}
`AnalyticsAutoEventsConfig` exposes each toggle as a flat field: `dailyAuth`, `returnActive`, `minResumeMs` (ms before another `user_returned` will fire), `syncErrorsEnabled` + `syncErrorsMinIntervalMs`, `blobUploadsStart` / `blobUploadsSuccess` / `blobUploadsFailure`, and `sessionEnd`.
{{/lang}}

---

## Querying Analytics (CLI)

The `primitive` CLI provides commands for querying analytics. All accept `--json` for machine-readable output.

```bash
# DAU / WAU / MAU + growth (default --window-days 28)
primitive analytics overview
primitive analytics overview --window-days 28 --json

# Active users time series
primitive analytics daily-active --window-days 28
primitive analytics rolling-active --window-days 7   # default 7

# Cohort retention (no window flag — returns full matrix)
primitive analytics cohort-retention

# Top users (default --window-days 30, --limit 10)
primitive analytics top-users --window-days 7 --limit 20

# Search users (--query is required)
primitive analytics user-search --query user@example.com

# Per-user breakdown
primitive analytics user-detail <user-ulid>
primitive analytics user-snapshot <user-ulid>

# Raw event feed (default --window-days 7, --page 0)
primitive analytics events --window-days 7 --page 0

# Group by: action | feature | route | country | deviceType | plan | day
primitive analytics events-grouped --group-by feature --window-days 14

# Integration / workflow / prompt analytics (default --window-days 30)
primitive analytics integrations
primitive analytics workflows --limit 5
primitive analytics prompts --limit 5
```

---

## Querying Analytics (REST API)

All endpoints require `admin` permission on the app.

```text
# DAU / WAU / MAU / growth — separate endpoints (no combined `/overview`)
GET /app/{appId}/api/analytics/overview/dau?windowDays=28
GET /app/{appId}/api/analytics/overview/wau?windowDays=28
GET /app/{appId}/api/analytics/overview/mau?windowDays=28
GET /app/{appId}/api/analytics/overview/growth?windowDays=28

# Active-user series
GET /app/{appId}/api/analytics/daily-active?windowDays=28
GET /app/{appId}/api/analytics/rolling-active?windowDays=7
GET /app/{appId}/api/analytics/cohort-retention

# Users
GET /app/{appId}/api/analytics/users/top?windowDays=30&limit=10
GET /app/{appId}/api/analytics/users/search?q=...&limit=25
GET /app/{appId}/api/analytics/users/{userUlid}/detail
GET /app/{appId}/api/analytics/users/{userUlid}/snapshot

# Events
GET /app/{appId}/api/analytics/events?windowDays=7&page=0
GET /app/{appId}/api/analytics/events/grouped?windowDays=7&groupBy=action

# Integrations / workflows / prompts (admin-only top lists)
GET /app/{appId}/api/analytics/integrations?windowDays=30
GET /app/{appId}/api/analytics/workflows/top?windowDays=30&limit=10
GET /app/{appId}/api/analytics/prompts/top?windowDays=30&limit=10
```

> The REST API does **not** expose `users/{userUlid}/timeline`, `users/{userUlid}/events`, `workflows/overview`, `prompts/overview`, or a combined `overview` endpoint. Use the granular endpoints above.

### Filtering events / events-grouped

Both `/analytics/events` and `/analytics/events/grouped` accept up to 10 filter clauses via repeated query parameters of the form `filter[FIELD][OPERATOR]=value`. Values are capped at 200 chars and rejected if they contain control or binary characters (printable Unicode only).

Supported `FIELD`s and the `OPERATOR`s each accepts:

| Field | Operators |
| --- | --- |
| `user` | `is`, `contains`, `starts with` |
| `action` | `is`, `is not`, `contains` |
| `feature` | `is`, `is not`, `contains` |
| `route` | `is`, `contains`, `starts with` |
| `country` | `is`, `is not` |
| `region` | `is`, `contains` |
| `city` | `is`, `contains` |
| `plan` | `is`, `is not` |
| `deviceType` | `is`, `is not` |
| `os` | `is`, `is not`, `contains` |
| `browser` | `is`, `is not`, `contains` |
| `appVersion` | `is`, `is not`, `contains` |

Example: `?windowDays=7&filter[feature][is]=billing&filter[action][contains]=upgrade`. Unknown fields, unsupported operators, or rejected values are silently dropped.

### Event row shape

Each row from `/analytics/events` includes geographic and per-event metric fields beyond the basic `timestamp` / `user` / `action` / `feature`:

- `region_code`, `region`, `city`, `colo` — derived from the request edge
- `entity_key` — caller-provided correlation handle (e.g. workflow run id, prompt id)
- `duration_ms` — for `*_succeeded` / `*_failed` events that record latency
- `input_tokens`, `output_tokens`, `total_tokens` — populated for `prompt_succeeded` events

These fields are absent (or zero) when the event type doesn't produce them.

---

## Best Practices

1. **Use verb_noun action names** — `"photo_uploaded"`, `"report_generated"`, `"settings_changed"`.
2. **Group with `feature`** — set consistently to enable per-feature dashboards (`"gallery"`, `"settings"`, `"billing"`).
3. **Keep `context_json` small** — bounded at 1 KiB. Don't dump request bodies or full reports.
4. **Don't log high-frequency events** — rate limiter caps at 300/min with burst 60. Design around meaningful actions, not continuous telemetry.
5. **Use the override setters** instead of passing `plan` / `app_version` on every event.

{{#lang ts}}
6. **Don't add your own `beforeunload` flush** — the client already does this.
{{/lang}}

---

## Complete Example: Feature Usage Tracking

Configure auto events on the client, set the app-version override once, then log a per-feature action and a pre-auth landing event.

{{ example: analytics/feature-usage-tracking }}

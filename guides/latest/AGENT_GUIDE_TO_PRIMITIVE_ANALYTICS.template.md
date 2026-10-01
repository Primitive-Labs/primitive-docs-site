# Agent Guide to Primitive Analytics

Primitive records user activity and resource events automatically. Log custom events for app-specific actions, then query activity through the CLI, REST API, or `ctx.api.analytics` in a server function.

## Logging Custom Events

### Basic Event

Set `action` to the event name. Use `feature` to group related events; it defaults to `"unspecified"`.

{{ example: analytics/log-event }}

{{#lang ts}}
Pass `action` and `user_ulid` explicitly. Use `ANALYTICS_UNAUTHENTICATED_USER` on pre-auth screens.
{{/lang}}
{{#lang swift}}
`analytics.logEventAsync` takes an `AnalyticsEventInput`. `user_ulid` is optional on the struct: when omitted it is back-filled from the client's current user, or set to `AnalyticsEventInput.unauthenticatedUser` when no user is signed in.

### Gotchas when ordering events

Await analytics calls, including `logEventAsync` and `flushAsync`, when order matters. Awaiting the flush ensures the preceding event is included in that batch. `AnalyticsContext.logEventAsync` follows the same ordering; a custom context without an async callback uses its synchronous callback once.
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

### Gotchas when logging

- Supply `user_ulid` explicitly so the event has an identity.
- Keep `context_json` below 1 KiB; oversized context loses data.
- Log meaningful actions rather than mouse movement or other high-frequency activity. Excess events are dropped.
{{/lang}}

{{#lang swift}}
### AnalyticsEventInput Fields

`AnalyticsEventInput` carries the same fields as the event row (`action`, `feature`, `route`, `plan`, `tenant_id`, `user_ulid`, `device_type`, `os_name`, `os_version`, `browser_name`, `browser_version`, `app_version`, `context_json`). Only `action` is required at the initializer: `user_ulid` is back-filled from the signed-in user (or the unauthenticated-user constant), and `context_json` is a `JSONValue`.
{{/lang}}

---

## What's Tracked Automatically

The client and server record active users, sessions, document and permission changes, and function, prompt, and integration activity.

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
| `function.invoke` | `functions` | A server function's HTTP invoke or task start, attributed to the caller. Context: `functionKey`, `functionId`, `configId`, `contentHash`, `status` (the terminal status of an invoke; `started` for a task start, plus `executionMode: "durable"`). A trigger fire (webhook, cron) has no caller and emits none |
| `integration.invoke` | _(integration key)_ | Integration call |
| `created` | `token` | API token created |
| `revoked` | `token` | API token revoked |

Function and prompt events include `duration_ms`. Prompt events also include token counts and provider-reported cost when available. A missing cost is unknown, not zero.

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

## Logging Snapshots

`logSnapshot` records a state snapshot event. It auto-resolves the user; if no user is authenticated, the call is a no-op (no error).

{{ example: analytics/log-snapshot }}

This logs an event with `action: "_snapshot"`, `feature: "_state"`, and your context as `context_json`.

---

## Pre-Auth Events

For pre-auth screens, pass the unauthenticated-user constant as `user_ulid`. Its value is `"UNAUTHENTICATED"`. Use an identified user for signed-in activity.

{{ example: analytics/log-event-preauth }}

---

## Manual Flush

Events are buffered and flushed automatically — including when the WebSocket reconnects — so you rarely need to flush manually. Call `flush` to force a send (e.g. before an explicit teardown).

{{ example: analytics/manual-flush }}

{{#lang ts}}
The queue flushes every **100ms** or at **25 KiB**, and on reconnect, page hide, and page unload. Do not add a duplicate unload listener; the client also records `session_end`.
{{/lang}}
{{#lang swift}}
The queue auto-flushes every **100ms** (or earlier when batched). `await client.destroy()` cancels the flush timer and triggers a final flush before storage closes, so you don't need a manual flush on teardown.

`flushAsync()` returns once the batch has actually reached the socket, so awaiting it is how you know an event went out before whatever follows.
{{/lang}}

---

## Plan and App Version Overrides

If your app reports its plan/version dynamically (e.g. after an in-app upgrade), set them on the client. They flow into every subsequent event automatically. Pass `null`/`nil` to clear an override.

{{ example: analytics/analytics-overrides }}

---

## Writing Events from a Server Function

Use `ctx.api.analytics.writeForUser` to attribute an event to an app member. Scheduled and webhook-triggered functions can use it without a caller. No capability declaration is required.

```ts
await ctx.api.analytics.writeForUser({
  body: { userId: input.userId, action: "order_placed", feature: "orders" },
});
```

| Body field | Required | Notes |
|---|---|---|
| `userId` | Yes | App member whose activity to record. |
| `action` | Yes | verb_noun, as on the client. |
| `feature` | No | Always pass one, so per-feature queries find the event. |
| `route` | No | |
| `context` | No | Additional data; cannot override reserved `appId` or `userId`. |
| `durationMs` | No | Recorded in the event's context as `timings.totalMs`. |
| `metrics` | No | Recorded in the event's context as `metrics`. |

- Function-only route: `POST /app/{appId}/api/analytics/write-for-user`. Any member, admin or owner calling it directly gets `403 FUNCTION_ROUTE_FUNCTION_ONLY`.
- A refused write is a `400` carrying `code`: `ANALYTICS_SUBJECT_REQUIRED` (no `userId`), `ANALYTICS_ACTION_REQUIRED` (no `action`), `ANALYTICS_SUBJECT_UNKNOWN` (the subject is not a member of this app).

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
{{/lang}}
{{#lang swift}}
`AnalyticsAutoEventsConfig` exposes each toggle as a flat field: `dailyAuth`, `returnActive`, `minResume` (a `TimeInterval` — seconds before another `user_returned` will fire), `syncErrorsEnabled` + `syncErrorsMinInterval` (also seconds), `blobUploadsStart` / `blobUploadsSuccess` / `blobUploadsFailure`, and `sessionEnd`.
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

# Search users (give --query, or a signup filter)
primitive analytics user-search --query user@example.com

# Who joined the app on a given UTC day, or across a range of up to 90 days
primitive analytics user-search --signup-day 2026-09-06
primitive analytics user-search --signup-start-day 2026-09-01 --signup-end-day 2026-09-06

# A busy day holds more matches than one page: keep paging while the answer
# says it was truncated.
primitive analytics user-search --signup-day 2026-09-06 --limit 100 --json
primitive analytics user-search --signup-day 2026-09-06 --limit 100 --offset 100 --json

# Per-user breakdown
primitive analytics user-detail <user-ulid>
primitive analytics user-snapshot <user-ulid>

# Raw event feed (default --window-days 7, --page 0)
primitive analytics events --window-days 7 --page 0

# Group by: action | feature | route | country | deviceType | plan | day
primitive analytics events-grouped --group-by feature --window-days 14

# Error groups: failures grouped by fingerprint with per-day counts
# (default --window-days 7, --limit 50). Filter: --status-class 4xx|5xx|transport,
# --source workflow_run|workflow_step|integration|function_invocation|function_run
primitive analytics errors-groups --window-days 7 --status-class 5xx

# Integration / prompt analytics (default --window-days 30)
primitive analytics integrations
primitive analytics prompts --limit 5
```

Use `analytics prompts` and `analytics integrations` for their respective services. Query `function.invoke` events to count or inspect function invocations.

`--json` prints the endpoint's own payload ([Response shapes](#response-shapes)) for every command except two, which are shaped for the terminal: `analytics events --json` prints the shared inspection envelope `{ items, page, pageSize, totalRows }` with each row projected through the operator-facing allowlist, and `analytics overview --json` calls the four separate DAU/WAU/MAU/growth endpoints and prints them as one `{ dau, wau, mau, growth }` object.

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
GET /app/{appId}/api/analytics/users/search?signupDay=2026-09-06&limit=100&offset=0
GET /app/{appId}/api/analytics/users/search?signupStartDay=2026-09-01&signupEndDay=2026-09-06
GET /app/{appId}/api/analytics/users/{userUlid}/detail
GET /app/{appId}/api/analytics/users/{userUlid}/snapshot

# Events
GET /app/{appId}/api/analytics/events?windowDays=7&page=0
GET /app/{appId}/api/analytics/events/grouped?windowDays=7&groupBy=action

# Error groups — failures grouped by fingerprint, per-day count buckets
GET /app/{appId}/api/analytics/errors/groups?windowDays=7&limit=50

# Integrations / prompts (admin-only top lists)
GET /app/{appId}/api/analytics/integrations?windowDays=30
GET /app/{appId}/api/analytics/prompts/top?windowDays=30&limit=10
```

### Response shapes

Every endpoint returns a JSON object, and `ctx.api.analytics` in a server function answers the same body. The first column names each query.

| Query type — endpoint | Payload |
| --- | --- |
| `overview.dau` / `overview.wau` / `overview.mau` — `/overview/{dau,wau,mau}` | `{ value, previous, deltaPct }` |
| `overview.growth` — `/overview/growth` | `{ window_days, retained_users, new_users, reactivated_users, churned_users, current_active, previous_active, deltaPct }` |
| `daily-active` — `/daily-active` | `{ window_days, rows: [{ day_ts, day_label, active_users }] }` |
| `rolling-active` — `/rolling-active` | same `{ window_days, rows: [{ day_ts, day_label, active_users }] }` shape |
| `cohort-retention` — `/cohort-retention` | `{ weeks, rows: [{ signup_week, signup_week_label, cohort_size, retention }], averages }` |
| `users.top` — `/users/top` | `{ windowDays, limit, results: [{ userUlid, email, name, firstSeen, firstSeenInWindow, lastSeen, eventCount }] }` |
| `users.search` — `/users/search` | `{ query, limit, offset, truncated, signupStartDay, signupEndDay, results }` — rows have the `users.top` shape minus `firstSeenInWindow` (it searches the full retained range, so its `firstSeen` is already the all-time value) plus `signedUpAt` / `signupDay`. Optional params: `signupDay`, or `signupStartDay` + `signupEndDay` (≤ 90 days), and `offset` |
| `users.detail` — `/users/{userUlid}/detail` | `{ user: { user_ulid, email, name }, stats: { first_seen, last_active, total_events, days_active }, events_by_action: [{ action, event_count, last_occurred }], events_by_feature: [{ feature, event_count, pct }] }` |
| `users.snapshot` — `/users/{userUlid}/snapshot` | `{ snapshot: { timestamp, values } }`, or `{ snapshot: null }` when the user has none |
| `events` — `/events` | `{ page, page_size, total_rows, rows }` — row fields under [Event row shape](#event-row-shape) |
| `events.grouped` — `/events/grouped` | `{ group_by, rows: [{ group_value, raw_group_value, events, unique_users }] }` |
| `errors.groups` — `/errors/groups` | `{ window_days, rows: [{ fingerprint, normalized_title, source, status_class, total, daily: [{ day, count }], first_seen, last_seen, exemplar: { message, action, scope_key, step_id, run_id, at } \| null }] }` |
| `prompts.top` — `/prompts/top` | `{ windowDays, limit, prompts: [{ promptKey, executions, medianDurationMs, p95DurationMs, p10, p50, p95, avgInputTokens, avgOutputTokens, totalTokens, executionsWithCost, totalCost, avgCost }] }` — `totalCost` / `avgCost` are `null` when no execution in the window reported a cost |
| `integrations` — `/integrations` | `{ windowDays, integrations: [{ integrationKey, invocations, errorRate, medianDurationMs, p10DurationMs, p50DurationMs, p95DurationMs }] }` |

### Gotchas when calculating metrics

- **Current windows include a partial day.** `previous` covers complete preceding days. Avoid interpreting an early-day decline as a complete-day comparison.
- **`deltaPct` is fractional.** `0.25` means 25%. When `previous` is zero, render “no baseline”; the returned `1` or `0` is a sentinel.
- **Overview windows are fixed.** DAU, WAU, and MAU use 1, 7, and 28 days. Growth accepts `windowDays` from 1–90, default 28.
- **Activity is not a signup date.** `firstSeen` is the first retained event; `firstSeenInWindow` is the first event in the selected window. Use `signedUpAt` or `signupDay` for when the user joined the app.
- **Series have different windows.** `daily-active` returns one row per day, including zeros (7–90 days, default 28). `rolling-active` returns 28 points; its `windowDays` sets each point's trailing window (1–28, default 7).
- **Costs may be incomplete.** `totalCost` and `avgCost` cover `executionsWithCost`, not all executions. They are `null` when no price was reported. Label partial coverage and do not divide by all executions.
- **System activity is excluded from user metrics.** Query `sys:<appId>` explicitly through user search or events when inspecting system work.
- **Signup searches use activity records.** They omit users without activity in the retained 90 days; `signedUpAt` can be `null`. A signup range spans at most 90 days.
- **Page signup searches.** While `truncated` is true, increment `offset` by the returned `limit` (1–100). Re-read a completed day if new signups could have shifted pages during the search.
- **Retention uses percentages.** Values range from 0–100; `null` means the cohort has not reached that week. A rejoining member belongs to their latest cohort.
- **Error day buckets are sparse.** Divide totals by `window_days`, not by the number of returned buckets.

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

### Gotchas when filtering

Event, grouped-event, and error-group queries accept at most **10 filters**. More returns `400 FILTER_LIMIT_EXCEEDED`; match the code, not the message. Each supplied parameter counts, including repeated or invalid filters. Repeated parameters are ANDed.

### Error groups

`/analytics/errors/groups` groups similar failures by `fingerprint`. Use the returned title for a summary and `exemplar` for one concrete failure.

| Field | Use |
|---|---|
| `normalized_title` | Message with request-specific values removed |
| `source` | Kind of operation that failed |
| `total` | Failures in the selected window |
| `daily` | Per-day counts; days without failures are omitted |
| `first_seen` / `last_seen` | First and latest occurrence, as ISO-8601 timestamps |
| `exemplar` | Sample with message, scope, and run identifiers, or `null` |

For a function failure, the exemplar's `scope_key` names the function and `run_id` identifies its task run.

#### Gotchas when handling error groups

- `exemplar` may be `null`. Fall back to `normalized_title`.
- HTTP status class is reliable for integration failures. Other sources may have no HTTP status, so filtering by it can omit failures.
- An empty window returns `rows: []`. Sparse daily buckets do not imply missing data.

Only a bounded set of dimensions is filterable (the exact error code is inside the fingerprint, never a filter):

| Field | Operators | Values |
| --- | --- | --- |
| `fingerprint` | `is`, `contains`, `starts with` | any |
| `source` | `is`, `is not` | `workflow_run`, `workflow_step`, `integration`, `function_invocation`, `function_run` |
| `statusClass` | `is`, `is not` | `4xx`, `5xx`, `transport` (bounded enum; other values → 400) |

Query params: `windowDays` (1–90, default 7), `limit` (1–200 groups, default 50). Also available as the CLI `primitive analytics errors-groups` (add `--verbose` to print each group's exemplar and provenance).

### Event row shape

Each row from `/analytics/events` includes geographic and per-event metric fields beyond the basic `timestamp` / `user` / `action` / `feature`:

- `region_code`, `region`, `city`, `colo` — derived from the request edge
- `entity_key` — caller-provided correlation handle (e.g. a prompt id)
- `duration_ms` — for `*_succeeded` / `*_failed` events that record latency
- `input_tokens`, `output_tokens`, `total_tokens` — populated for `prompt_succeeded` events

These fields are absent (or zero) when the event type doesn't produce them.

---

## Complete Example: Feature Usage Tracking

Configure auto events on the client, set the app-version override once, then log a per-feature action and a pre-auth landing event.

{{ example: analytics/feature-usage-tracking }}

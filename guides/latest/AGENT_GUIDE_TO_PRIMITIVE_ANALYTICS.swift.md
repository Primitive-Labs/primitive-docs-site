# Agent Guide to Primitive Analytics

Guidelines for AI agents implementing analytics tracking in Primitive apps.

## Overview

Primitive provides built-in analytics. The platform tracks user activity and resource lifecycle automatically and stores events server-side for querying. The system handles offline persistence, rate limiting, and automatic lifecycle events out of the box.

Read aggregated analytics (DAU/WAU/MAU, retention, top users, event feeds) through the `primitive` CLI, the REST API, or a server function (`ctx.api.analytics`) — covered below.

---

## What's Tracked Automatically (Zero Developer Work)

Standing up an app gets you DAU/WAU/MAU tracking, session analytics, document/permission audit trails, and function/prompt/integration observability with no instrumentation.

**Key constraints for custom events:**
- Every analytics event requires an authenticated user. Events without a `user_ulid` are dropped silently. Use the unauthenticated-user constant for pre-auth screens.
- A tenant ID (resolved automatically from the client's `appId`) must also be present, or the event is dropped.


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

Function and prompt events also record `duration_ms`, and prompt events record LLM token counts (`input_tokens`, `output_tokens`, `total_tokens`) when available. A prompt run from a function emits `prompt.executed` attributed to the function's caller, and none on a trigger fire.


### Offline Persistence and Rate Limiting

Events are buffered on the device and persisted locally while offline; persisted events are flushed automatically when the WebSocket reconnects.
A rate limiter caps emission at **300 events per 60-second window, with no more than 60 events in the first 10 seconds** — events over the cap are dropped silently. No special code needed.

The offline buffer is persisted with a **~1 MiB** cap; when it exceeds the cap the **oldest** events are dropped.

---

## Logging Custom Events

### Basic Event

`action` and `user_ulid` are required; `feature` (defaults to `"unspecified"`) groups related events.

```swift
  await client.analytics.logEventAsync(AnalyticsEventInput(
    action: "photo_uploaded",
    feature: "gallery",
    user_ulid: currentUserUlid
  ))
```

`analytics.logEventAsync` takes an `AnalyticsEventInput`. `user_ulid` is optional on the struct: when omitted it is back-filled from the client's current user, or set to `AnalyticsEventInput.unauthenticatedUser` when no user is signed in.

**Every analytics call is `async`.** The queue is an `actor`, so `client.analytics` is used with `await`: `logEventAsync`, `logSnapshotAsync`, `flushAsync`, `setPlanOverrideAsync`, `setAppVersionOverrideAsync` — and `logAnalyticsEventAsync` / `flushAnalyticsAsync` / `setAnalyticsPlanOverrideAsync` / `setAnalyticsAppVersionOverrideAsync` on the client itself. Awaiting each call is what orders two consecutive events against each other, and what guarantees an event logged before `flushAsync()` goes out with *that* batch. `AnalyticsContext` — the logger handed to LLM calls, read from `client.llmAnalyticsContext` / `client.geminiAnalyticsContext` — follows the same shape: `await context.logEventAsync(_:)`, taking a `[String: JSONValue]`. A context you construct yourself supplies the synchronous `logEvent:` closure and, optionally, a `logEventAsync:` closure; without the latter, `logEventAsync` falls back to the synchronous one and calls it exactly once.

### Event with Context

Pass a `context_json` object for per-event debug data. The serialized payload is bounded at **1 KiB**, so keep it small — don't dump request bodies or full reports.

```swift
  await client.analytics.logEventAsync(AnalyticsEventInput(
    action: "search_executed",
    feature: "search",
    user_ulid: currentUserUlid,
    context_json: [
      "query": "quarterly report",
      "resultCount": 42,
    ]
  ))
```

`context_json` is a `JSONValue` — construct it with object/array/scalar literals. If its serialized size exceeds 1 KiB, the field is dropped from the event before sending.


### AnalyticsEventInput Fields

`AnalyticsEventInput` carries the same fields as the event row (`action`, `feature`, `route`, `plan`, `tenant_id`, `user_ulid`, `device_type`, `os_name`, `os_version`, `browser_name`, `browser_version`, `app_version`, `context_json`). Only `action` is required at the initializer: `user_ulid` is back-filled from the signed-in user (or the unauthenticated-user constant), and `context_json` is a `JSONValue`.

---

## Logging Snapshots

`logSnapshot` records a state snapshot event. It auto-resolves the user; if no user is authenticated, the call is a no-op (no error).

```swift
  await client.analytics.logSnapshotAsync(context: ["screen": "settings", "tab": "billing"])
```

This logs an event with `action: "_snapshot"`, `feature: "_state"`, and your context as `context_json`.

---

## Pre-Auth Events

Events with no authenticated user are dropped. To log on pre-auth screens (landing pages, sign-up flow), pass the unauthenticated-user constant as the `user_ulid`. Its value is `"UNAUTHENTICATED"`. Use sparingly — most analytics should be tied to real users.

```swift
  await client.analytics.logEventAsync(AnalyticsEventInput(
    action: "landing_page_view",
    feature: "onboarding",
    user_ulid: AnalyticsEventInput.unauthenticatedUser
  ))
```

---

## Manual Flush

Events are buffered and flushed automatically — including when the WebSocket reconnects — so you rarely need to flush manually. Call `flush` to force a send (e.g. before an explicit teardown).

```swift
  // Returns once the batch has reached the socket.
  await client.analytics.flushAsync()
```

The queue auto-flushes every **100ms** (or earlier when batched). `await client.destroy()` cancels the flush timer and triggers a final flush before storage closes, so you don't need a manual flush on teardown.

`flushAsync()` returns once the batch has actually reached the socket, so awaiting it is how you know an event went out before whatever follows.

---

## Plan and App Version Overrides

If your app reports its plan/version dynamically (e.g. after an in-app upgrade), set them on the client. They flow into every subsequent event automatically. Pass `null`/`nil` to clear an override.

```swift
  await client.analytics.setPlanOverrideAsync("pro")
  await client.analytics.setAppVersionOverrideAsync("2.1.4")

  // Pass nil to clear an override
  await client.analytics.setPlanOverrideAsync(nil)
```

---

## Writing Events from a Server Function

A server function writes an event with `ctx.api.analytics.writeForUser`, attributed to the **subject** user it names rather than to whoever is running — so a webhook- or cron-fired function (no caller, `ctx.user` is `null`) still records activity against the member it acted for. No capability line is needed.

```ts
await ctx.api.analytics.writeForUser({
  body: { userId: input.userId, action: "order_placed", feature: "orders" },
});
```

| Body field | Required | Notes |
|---|---|---|
| `userId` | Yes | The subject. Must be a member of this app; another app's user id is refused exactly as an id that never existed. |
| `action` | Yes | verb_noun, as on the client. |
| `feature` | No | Always pass one, so per-feature queries find the event. |
| `route` | No | |
| `context` | No | Your object. `appId` and `userId` are reserved and written last — a `context` cannot rewrite whose activity the event is. |
| `durationMs` | No | Recorded in the event's context as `timings.totalMs`. |
| `metrics` | No | Recorded in the event's context as `metrics`. |

- Function-only route: `POST /app/{appId}/api/analytics/write-for-user`. Any member, admin or owner calling it directly gets `403 FUNCTION_ROUTE_FUNCTION_ONLY`.
- A refused write is a `400` carrying `code`: `ANALYTICS_SUBJECT_REQUIRED` (no `userId`), `ANALYTICS_ACTION_REQUIRED` (no `action`), `ANALYTICS_SUBJECT_UNKNOWN` (the subject is not a member of this app).

---

## Configuring Auto Events

Pass `analyticsAutoEvents` to the constructor. All sub-options default to enabled.

```swift
  let client = JsBaoClient(options: JsBaoClientOptions(
    apiUrl: "https://primitiveapi.com",
    wsUrl: "wss://primitiveapi.com",
    appId: "YOUR_APP_ID",
    analyticsAutoEvents: AnalyticsAutoEventsConfig(
      dailyAuth: true,
      returnActive: true,
      minResume: 5 * 60, // seconds before another user_returned can fire
      syncErrorsEnabled: true,
      syncErrorsMinInterval: 30,
      blobUploadsStart: false,
      blobUploadsSuccess: true,
      blobUploadsFailure: true,
      sessionEnd: true
    )
  ))
```

`AnalyticsAutoEventsConfig` exposes each toggle as a flat field: `dailyAuth`, `returnActive`, `minResume` (a `TimeInterval` — seconds before another `user_returned` will fire), `syncErrorsEnabled` + `syncErrorsMinInterval` (also seconds), `blobUploadsStart` / `blobUploadsSuccess` / `blobUploadsFailure`, and `sessionEnd`.

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
# --source integration
primitive analytics errors-groups --window-days 7 --status-class 5xx

# Integration / prompt analytics (default --window-days 30)
primitive analytics integrations
primitive analytics prompts --limit 5
```

Per-subject analytics live under the top-level `analytics` noun — `analytics prompts` and `analytics integrations` are the homes for them; no subject noun carries its own analytics group. There is no per-function top list: function invocations are `function.invoke` events, counted with `events-grouped --group-by action` and listed by `events`.

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

> The REST API does **not** expose `users/{userUlid}/timeline`, `users/{userUlid}/events`, `prompts/overview`, or a combined `overview` endpoint. Use the granular endpoints above.

### Response shapes

Every endpoint returns a JSON object, and `ctx.api.analytics` in a server function answers the same body. The first column names each query.

Every payload also carries `_timing: { total_ms, wae_queries }` — diagnostics for the query itself, not data to consume. It is left out of the shapes below.

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
| `prompts.top` — `/prompts/top` | `{ windowDays, limit, prompts: [{ promptKey, executions, medianDurationMs, p95DurationMs, p10, p50, p95, avgInputTokens, avgOutputTokens, totalTokens }] }` |
| `integrations` — `/integrations` | `{ windowDays, integrations: [{ integrationKey, invocations, errorRate, medianDurationMs, p10DurationMs, p50DurationMs, p95DurationMs }] }` |

Read these before computing anything from a payload:

- **`value` / `previous` / `deltaPct`.** `value` is the distinct active users over the current window; `previous` is the immediately preceding, non-overlapping window. Both windows start on a UTC calendar-day boundary, but the current one ends at query time, so it holds a partial final day while `previous` is `windowDays` complete days. Expect `deltaPct` to read low early in the UTC day — most visibly for `overview.dau`, where "today so far" is compared against all of yesterday. A month-over-month comparison is still one `overview.mau` call, not a second series query. `deltaPct` is a **fraction, not a percentage** — `0.25` means +25%, and it is rounded to 4 decimal places. Against a zero base it is a fixed sentinel rather than a real ratio: `1` when `value > 0`, `0` when `value` is 0 too. Render "no baseline" instead of a percentage when `previous` is 0.
- **`overview.dau` / `wau` / `mau` windows are fixed** at 1, 7, and 28 days; they ignore `windowDays`. `overview.growth` honors it (1–90, default 28) and reports both halves as `current_active` / `previous_active`, with `deltaPct` on the same fraction scale. `overview.growth` builds its windows the same way, so `current_active` carries the same partial final day — and `churned_users` / `reactivated_users`, which are derived from the two counts, inherit it.
- **`firstSeen` is all-time; `firstSeenInWindow` is not.** On a `users.top` row, `firstSeen` is the user's first recorded event across the app's whole retained history — it does not move when you change `windowDays`, so `firstSeen` inside the last day is a genuinely new user. `firstSeenInWindow` is the first event *inside* the window, and it moves with `windowDays` by definition; the rest of the row (`eventCount`, `lastSeen`) is window-scoped too. Derive "new signups" from `firstSeen` (or from `overview.growth`'s `new_users`, which counts the same thing) — never from `firstSeenInWindow`, which over a short window marks every returning user as a new signup, day after day. Both are bounded by retention: a user whose first event has aged out reads as first seen at the oldest event still retained.
- **`daily-active` is dense.** It returns exactly `windowDays` rows (7–90, default 28), one per UTC day, zero-filled for days with no activity. `day_ts` is the day's UTC-midnight epoch second; `day_label` is `YYYY-MM-DD`.
- **`rolling-active` always returns 28 rows**, one per day. Its `windowDays` (1–28, default 7) is the length of the trailing window each point counts distinct users over — not the number of points.
- **Per-user surfaces count app users only.** `users.top`, `users.search`, `daily-active`, `rolling-active`, `overview.dau` / `wau` / `mau` and `overview.growth` exclude the app's synthetic system principal (`sys:<appId>`), so machine activity — a cron-fired function every night — never appears as a user and never marks anyone active. `events` still returns that principal's rows when you ask for it by id, and a trigger-fired function's own record is its run row (`primitive functions runs`).
- **Asking for the system principal by name still works.** The exclusion is on the unfiltered listings, not on a lookup: `users.search` with a `q` of `sys:<appId>` returns it, exactly as `events` with that `userId` does. Only `users.search` *without* a `q` — which lists the app's most recently active principals — drops it.
- **Signup means "joined this app".** Every user-attributed event the server writes carries the app-membership join time, so `users.search` can answer "who signed up on day D": pass `signupDay` (one UTC calendar day, `YYYY-MM-DD`) or `signupStartDay` + `signupEndDay` (an inclusive range of at most 90 days), and each row comes back with `signedUpAt` (ISO) and `signupDay` (UTC day). A user who belongs to two apps carries each app's own join date, so joining a second app makes them new on that app the day they joined it. Two limits worth knowing: the answer is drawn from activity, so a member who has not produced an event in the last 90 days does not appear; and `signedUpAt` is `null` for a user none of whose events carries a join time yet, which is why a range that includes `1970-01-01` returns nobody rather than everybody.
- **Page a signup day until it says it is done.** A filtered search returns at most `limit` rows (1–100) and sets `truncated: true` when more match. Repeat the call with `offset += limit`, stepping by the `limit` the response echoes rather than the one you asked for, until `truncated` is `false`; rows are ordered by signup time then user id, so pages do not overlap. `offset` is accepted only alongside a signup filter. A day whose signups are still arriving can shift a row between pages, so re-run the day's pages once it has closed if you need an exact set.
- **`cohort-retention` retention values are percentages** (0–100, one decimal place), with `null` for a week a cohort hasn't reached yet; week 0 is always 100. `averages` is the per-week mean across the returned cohorts. A cohort is keyed on the app-join time above, and a member who leaves and rejoins counts in exactly one cohort — their latest — for both the cohort size and every activity week, so no cell can exceed 100%.
- **`errors.groups` `daily` buckets are sparse** — see [Error groups](#error-groups).

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

**At most 10 filter params per request.** `/analytics/events`, `/analytics/events/grouped` and `/analytics/errors/groups` answer `400` — `Too many filters: N. Maximum is 10.` — for anything above that, rather than applying the first ten and dropping the rest. The body carries `code: "FILTER_LIMIT_EXCEEDED"`; **branch on that code, not the message**, which interpolates the submitted count and so cannot be matched literally. The count is over the `filter[…][…]` params you send, so a value that gets rejected by the allowlist still counts, and repeating the same field/operator with different values (each param is ANDed separately) counts once per param.

### Error groups

`/analytics/errors/groups` groups failure events (failed integration calls) by a stable **fingerprint** — a hash over the rules version, source, scope, step, normalized message, and status class. Messages that differ only in ids, numbers, URLs, quoted free-text values, or timestamps normalize to the same title and share a fingerprint.

Normalization preserves the tokens that tell two failures apart: JSON **keys** (an identifier-shaped quoted span that is followed by `:` *and* sits in JSON object position — after `{` or `,`; a quoted span in prose, like `Failed to parse "x.txt": ...` or `User "alice": not found`, is templated even though it precedes a colon), quoted `SCREAMING_SNAKE` error enums (`UNAVAILABLE`, `RESOURCE_EXHAUSTED` — but not an uppercase id such as `ABC123XYZ` or `DEADBEEFCAFE`, which are templated), a 3-digit status under a `code` / `status` / `statusCode` key, and an `HTTP <n>` code. A `Caused by:` chain folds to its head message, so a chained error groups with the unchained form of the same failure.

Each response row is one fingerprint: `fingerprint`, a representative `normalized_title`, `source`, `status_class`, a `total` over the window, `daily` — a per-day `{ day, count }` series (sampling-weighted, ascending, `day` a `YYYY-MM-DD` label) — `first_seen` / `last_seen` (ISO-8601), and an `exemplar`. One call covers the whole window, so "today versus the trailing baseline" needs no rolling store.

**The exemplar is how you identify a group.** `normalized_title` is a grouping key with the variable parts replaced by placeholders; `exemplar` is a raw sample of one real failure — `{ message, action, scope_key, step_id, run_id, at }` — so you can go from a spiking group straight to a concrete failure without paging `/analytics/events`. `step_id` and `run_id` are `null` when the failure had none. The whole object is `null` when no sampled row for that fingerprint carried a sample: a very noisy group can consume the sampling budget and leave a quieter one without one. Treat `exemplar: null` as a normal result, not an error.

`status_class` is the integration failure's real HTTP status class, including `transport`, which no message text can express.

**The `daily` series is sparse.** A day on which that fingerprint produced no events has no bucket at all, so `daily` is shorter than the window for any intermittent error. Divide by `window_days` (the response echoes it) to get a per-day baseline — dividing by `daily.length` averages over only the days the error fired, which overstates the baseline and suppresses exactly the spike an alert should catch. A window with no failures returns `rows: []`.

Only a bounded set of dimensions is filterable (the exact error code is inside the fingerprint, never a filter):

| Field | Operators | Values |
| --- | --- | --- |
| `fingerprint` | `is`, `contains`, `starts with` | any |
| `source` | `is`, `is not` | `integration` |
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

## Best Practices

1. **Use verb_noun action names** — `"photo_uploaded"`, `"report_generated"`, `"settings_changed"`.
2. **Group with `feature`** — set consistently to enable per-feature dashboards (`"gallery"`, `"settings"`, `"billing"`).
3. **Keep `context_json` small** — bounded at 1 KiB. Don't dump request bodies or full reports.
4. **Don't log high-frequency events** — rate limiter caps at 300/min with burst 60. Design around meaningful actions, not continuous telemetry.
5. **Use the override setters** instead of passing `plan` / `app_version` on every event.


---

## Complete Example: Feature Usage Tracking

Configure auto events on the client, set the app-version override once, then log a per-feature action and a pre-auth landing event.

```swift
  let client = JsBaoClient(options: JsBaoClientOptions(
    apiUrl: "https://primitiveapi.com",
    wsUrl: "wss://primitiveapi.com",
    appId: "YOUR_APP_ID",
    analyticsAutoEvents: AnalyticsAutoEventsConfig(
      blobUploadsStart: false,
      blobUploadsSuccess: true,
      blobUploadsFailure: true,
      sessionEnd: true
    )
  ))

  // Set version once after init (or after a deploy notification)
  await client.analytics.setAppVersionOverrideAsync("2.1.4")

  func trackFeatureUsed(
    _ userUlid: String,
    feature: String,
    action: String,
    context: JSONValue? = nil
  ) async {
    await client.analytics.logEventAsync(AnalyticsEventInput(
      action: action,
      feature: feature,
      user_ulid: userUlid,
      context_json: context
    ))
  }

  // Authenticated event
  await trackFeatureUsed(currentUserUlid, feature: "reports", action: "report_generated", context: [
    "reportType": "quarterly",
    "format": "pdf",
  ])

  // Pre-auth event (landing page)
  await client.analytics.logEventAsync(AnalyticsEventInput(
    action: "landing_page_view",
    feature: "onboarding",
    user_ulid: AnalyticsEventInput.unauthenticatedUser
  ))
```

# Agent Guide to Primitive Notifications

Multi-channel notifications: a durable in-app inbox, live WebSocket delivery while connected, and push (iOS/Android) once a device is registered. Client SDK surface is `client.notifications.*`; a workflow can send from the server with the `notification.send` step. **This capability is JS-client-only today** — there is no Swift client API for notifications.

## Client SDK Reference

| Call | Returns | Notes |
|---|---|---|
| `client.notifications.list(options?)` | `{ items: NotificationInfo[], unreadCount, cursor? }` | `options: { limit?, cursor? }`. Newest first. |
| `client.notifications.unreadCount()` | `{ unreadCount }` | Bounded scan (capped at 250 rows) — an approximation for very large inboxes, not a maintained counter. |
| `client.notifications.markRead(notificationId)` | `NotificationInfo` | Idempotent — marking an already-read row again just returns it. |
| `client.notifications.markAllRead()` | `{ updated }` | Scans up to 10 pages of 250 rows each. |
| `client.notifications.send(params)` | `{ results: NotificationSendResult[], deduplicated?, deduplicatedChannels? }` | **Requires app admin permission** — a member-level caller gets `403`. |
| `client.notifications.registerDevice(params)` | `PushDeviceInfo` | Upsert by token — re-registering the same token refreshes metadata instead of duplicating. |
| `client.notifications.listDevices()` | `{ items: PushDeviceInfo[] }` | Caller's own devices only. |
| `client.notifications.unregisterDevice(token)` | `{ deleted }` | Call on logout so a signed-out device stops receiving pushes for that account. |

`NotificationInfo`: `{ notificationId, title, body, iconUrl?, deepLink?, read, readAt?, expiresAt?, sourceRef?, createdAt }`.

`PushDeviceInfo`: `{ tokenId, tokenSuffix?, platform: "ios"|"macos"|"android", environment: "sandbox"|"production", bundleId?, deviceName?, appVersion?, lastSeenAt?, createdAt }` — `tokenSuffix` is the **last 8 characters only**; the full token is never echoed back once registered. A caller can hold up to 100 registered devices; the 101st registration for the same user fails.

### Sending

```typescript
const { results } = await client.notifications.send({
  title: "Your report is ready",
  body: "Tap to view this week's summary.",
  target: { userId },
  channels: ["in-app", "ios"],       // default ["in-app"] if omitted
  deepLink: "myapp://reports/latest",
  idempotencyKey: `weekly-report-${userId}-2026-07-14`,
});

for (const r of results) {
  console.log(r.channel, r.status); // e.g. "in-app" "delivered", "ios" "delivered"
}
```

`SendNotificationParams`: `{ title, body, target: { userId }, channels?: string[], iconUrl?, deepLink?, expiresAt?, sourceRef?, idempotencyKey? }`. Valid `channels` today: `"in-app" | "ios" | "android"` — an unrecognized channel name throws (`UnsupportedChannelError`, see Errors below). `expiresAt` (ISO date) auto-deletes the inbox row after that time; it does not affect push delivery.

`NotificationSendResult` (one per requested channel): `{ channel, status: "delivered"|"failed"|"skipped"|"invalidated", notificationId?, delivered?, failed?, invalidated?, tokenAttempts?, skipReason?, retryable? }`. `notificationId` is the durable inbox row id (in-app only). `delivered`/`failed`/`invalidated` are per-token tallies for push — one user can hold several device tokens, so a single "ios" entry can partially succeed. `tokenAttempts` is the per-token detail: `{ tokenSuffix, status, httpStatus?, reason?, retryable? }`. `status: "invalidated"` means every token for that channel was dead (evicted) — treat it the same as a failure for delivery purposes, though it is not itself a caller error.

### Reading and managing the inbox

```typescript
const { items, unreadCount } = await client.notifications.list({ limit: 20 });

for (const n of items) {
  console.log(n.title, n.read);
}

if (items[0] && !items[0].read) {
  await client.notifications.markRead(items[0].notificationId);
}
```

### Registering for push

```typescript
const device = await client.notifications.registerDevice({
  token: deviceToken,
  platform: "ios",
  environment: "production",
  bundleId: "com.example.myapp",
});

console.log(device.tokenSuffix); // last 8 chars only
```

`RegisterPushDeviceParams`: `{ token, platform: "ios"|"macos"|"android", environment: "sandbox"|"production", bundleId?, deviceName?, appVersion? }`. `environment` matters for `ios`/`macos` (APNs sandbox vs production certificates); FCM (`android`) has no sandbox tier — always pass `"production"` for Android tokens. Registering a token already owned by a different user **reassigns it** to the calling user (deliberate shared-device semantics: a token identifies a device, so when a different account signs in and re-registers, delivery must follow the signed-in user).

## Live WebSocket Event

```typescript
client.on("notification", (event) => {
  // event: { type: "notification", notificationId, title, body, iconUrl?, deepLink?, sourceRef?, createdAt }
  console.log(event.title, event.body);
});
```

This is a **best-effort real-time mirror**, not the source of truth — it fires only if the recipient is connected at send time. `client.notifications.list()` (the durable inbox row) is authoritative; a client that reconnects after a missed event simply sees the notification the next time it lists or checks `unreadCount()`. Don't build read/unread state purely off this event — always reconcile against `list()`/`unreadCount()`.

## The `notification.send` Workflow Step

```toml
[[steps]]
id = "notify"
kind = "notification.send"
toUserId = "{{ input.userId }}"      # required
title = "Your report is ready"       # required
body = "Tap to view this week's summary."  # required
channels = ["in-app", "ios"]         # optional, default ["in-app"]
iconUrl = "https://example.com/icon.png"    # optional
deepLink = "myapp://reports/latest"  # optional
expiresAt = "2026-08-01T00:00:00Z"   # optional, ISO date string
idempotencyKey = "{{ input.jobId }}" # optional — see Idempotency below
userInfo = { jobId = "{{ input.jobId }}" }   # optional, push-only — custom data delivered with the alert
collapseId = "report-ready"          # optional, push-only — provider collapse/coalesce id
threadId = "reports"                 # optional, push-only, APNs — Notification Center thread grouping
```

Output: same shape as the client's `send()` — `{ results: NotificationSendResult[], deduplicated?, deduplicatedChannels? }`.

**Verdict.** The step's `steps.<id>.ok` is `true` only when at least one requested channel actually delivered. A send that reaches nobody — every channel failed, was rate-limited, or the recipient had no registered device for a push channel — reports `ok: false` even though the step executed and returned a result. Branch on `steps.notify.ok`, not just the absence of a thrown error.

**Retry ownership.** Like every step, `notification.send` is single-attempt at the runner level — the engine's `step.do()` owns retries. The step's own error classification then decides whether a retry is worth attempting:

- A channel that failed for a transient reason — a provider 5xx/network error, or (for push) some device tokens failing retryably while others delivered — makes the step throw a **plain** error, so the engine retries the whole step.
- A channel that failed permanently — an unrecognized channel name, an unknown recipient, or every requested channel rate-limited — makes the step throw a **non-retryable** error, so the run doesn't spin on a failure a retry can't fix.

Set `idempotencyKey` so a retried step re-sends only what still needs it (see Idempotency below); without one, a retry re-sends every requested channel from scratch (at-least-once, possible duplicate in-app rows / push alerts).

## Idempotency and Deduplication

Pass `idempotencyKey` (a plain string, caller-chosen) on `send()` / the workflow step's `idempotencyKey` field to make a repeat call safe:

- **Full dedupe.** If every requested channel already has a terminal outcome under that key, the call returns immediately with `deduplicated: true` and the prior `results` — nothing is re-sent.
- **Partial dedupe.** If some channels are done and others still need a retry, the call re-sends only the channels that need it and reports `deduplicatedChannels: string[]` naming the ones served from the prior send.
- **Per-token granularity for push.** Dedupe tracks each device token separately within a channel: retrying a push send where one token delivered and another failed transiently re-sends only to the token that still needs it — the already-reached device does not get a duplicate alert.
- **Window.** Dedupe state is tracked for the same retention window as the audit trail (90 days) — an idempotency key reused after that window is treated as a brand-new send.

Without an `idempotencyKey`, every retry re-sends every requested channel from scratch (at-least-once, not exactly-once) — always set one on a send a caller or workflow step might retry.

## Rate Limiting

Sends are capped **per app, per hour, per channel**: 5,000 in-app notifications and 1,000 push dispatches. Each requested channel is checked independently, so a multi-channel send only blocks the channel(s) actually over their limit — the rest deliver normally. Only when **every** requested channel is blocked does the call throw `NotificationRateLimitError` (`limit`, `resetAt`); a workflow step turns this into a non-retryable step failure (see Retry ownership above), since waiting out the window is the caller's job, not something a step retry accomplishes.

## Errors

| Error | Thrown when | HTTP (REST) |
|---|---|---|
| `UnsupportedChannelError` | `channels` is empty, or names a channel outside `"in-app" \| "ios" \| "android"` | 400 |
| `NotificationTargetNotFoundError` | `target.userId` is not a member of the app | 404 |
| `NotificationRateLimitError` | Every requested channel was blocked by its hourly quota | 429 |

All three map to a non-retryable failure in the `notification.send` workflow step. A plain (retryable) error from the step means a transient per-channel/per-token delivery failure, not one of these three typed errors.

## Footguns

- **`send()` needs app admin permission**, not just app membership. A member-level caller gets `403` — this is deliberate (unrestricted member-level send would let any signed-in user spam the tenant's push/in-app quota).
- **Don't rely on the `notification` WS event as your only read path.** It's a best-effort live mirror; a disconnected recipient misses it entirely. Always back it with `list()` / `unreadCount()` for the authoritative state.
- **Set `idempotencyKey` on any send a caller or a workflow step might retry.** Without it, retries are at-least-once — expect duplicate inbox rows and duplicate push alerts, not a clean no-op.
- **A "delivered" push channel result can still need a retry.** When a user has multiple device tokens and only some fail retryably, the channel's overall `status` can read `"delivered"` while `tokenAttempts` shows a token that still needs a resend — the `notification.send` step accounts for this in its retry classification (see above); if you're driving `send()` directly from a client, check `tokenAttempts` yourself before treating a `"delivered"` status as fully done.
- **`registerDevice` reassigns tokens across users.** Registering a token already owned by a different account moves it to the new caller — correct for shared-device sign-out/sign-in, but don't assume a token uniquely and permanently identifies one user.
- **`environment` must match the build.** A production app build presenting a token issued under Apple's sandbox APNs environment (or vice versa) registers fine but silently fails to deliver — `environment` is not validated against the token itself.
- **Channels are additive, not implicit.** Omitting `channels` sends `"in-app"` only — a push notification never goes out unless `"ios"` / `"android"` is explicitly requested (and the user has a registered device for it).

## Tips for Coding Agents

1. Always set `idempotencyKey` when a send happens inside a workflow step or any other retryable call path.
2. Check `results[].status` per channel rather than assuming a resolved promise means full delivery — a partially-successful multi-channel send still resolves normally.
3. Register a device immediately after the user grants OS-level notification permission, and unregister on logout.
4. Treat the `notification` WS event as a UI nicety (badge/toast), never as the system of record — reconcile against `list()`/`unreadCount()`.
5. For a `notification.send` workflow step, branch on `steps.<id>.ok`, not just on whether the step threw — a fully-skipped/failed send still "runs".

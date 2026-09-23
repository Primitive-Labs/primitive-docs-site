# Agent Guide to Primitive Notifications

Multi-channel notifications: a durable in-app inbox, live WebSocket delivery while connected, and push (iOS/Android) once a device is registered. Client SDK surface is `client.notifications.*`, available on both the JS and Swift clients; a server function sends from the server with `ctx.api.notifications.send`.

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

{{ example: notifications/send }}

`SendNotificationParams`: `{ title, body, target: { userId }, channels?: string[], iconUrl?, deepLink?, expiresAt?, sourceRef?, idempotencyKey? }`. `channels` defaults to `["in-app"]` when omitted. Valid `channels` today: `"in-app" | "ios" | "android"` — an unrecognized channel name throws (`UnsupportedChannelError`, see Errors below). `expiresAt` (ISO date) auto-deletes the inbox row after that time; it does not affect push delivery.

`NotificationSendResult` (one per requested channel): `{ channel, status: "delivered"|"failed"|"skipped"|"invalidated", notificationId?, delivered?, failed?, invalidated?, tokenAttempts?, skipReason?, retryable? }`. `notificationId` is the durable inbox row id (in-app only). `delivered`/`failed`/`invalidated` are per-token tallies for push — one user can hold several device tokens, so a single "ios" entry can partially succeed. `tokenAttempts` is the per-token detail: `{ tokenSuffix, status, httpStatus?, reason?, retryable? }`. `status: "invalidated"` means every token for that channel was dead (evicted) — treat it the same as a failure for delivery purposes, though it is not itself a caller error.

### Reading and managing the inbox

{{ example: notifications/list-inbox }}

### Registering for push

{{ example: notifications/register-device }}

`RegisterPushDeviceParams`: `{ token, platform: "ios"|"macos"|"android", environment: "sandbox"|"production", bundleId?, deviceName?, appVersion? }`. `environment` matters for `ios`/`macos` (APNs sandbox vs production certificates); FCM (`android`) has no sandbox tier — always pass `"production"` for Android tokens. Registering a token already owned by a different user **reassigns it** to the calling user (deliberate shared-device semantics: a token identifies a device, so when a different account signs in and re-registers, delivery must follow the signed-in user).

## Live WebSocket Event

{{ example: notifications/live-events }}

{{#lang swift}}
Subscribe through `client.stream(for:)` — a `for await` loop in a `.task`, which unsubscribes when the loop ends — or through `client.observeOnMainActor(_:handler:)` when you need a main-actor callback, holding the returned `EventSubscription` for as long as you want the handler live.
{{/lang}}

This is a **best-effort real-time mirror**, not the source of truth — it fires only if the recipient is connected at send time. `client.notifications.list()` (the durable inbox row) is authoritative; a client that reconnects after a missed event simply sees the notification the next time it lists or checks `unreadCount()`. Don't build read/unread state purely off this event — always reconcile against `list()`/`unreadCount()`.

## Sending from a Server Function

A server function sends through `ctx.api.notifications.send({ body })`, where `body` is the same object the client's `send()` takes — `title`, `body`, `target: { userId }`, and the optional `channels`, `iconUrl`, `deepLink`, `expiresAt`, `sourceRef`, `idempotencyKey`. It returns the same `{ results, deduplicated?, deduplicatedChannels? }`. Function code acts as the system, so the admin-only restriction is met by construction and there is no capability line to declare; the function's own `access` gate decides who may cause the send.

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { userId: string; jobId: string }, ctx) => {
  const sent = await ctx.api.notifications.send({
    body: {
      title: "Your report is ready",
      body: "Tap to view this week's summary.",
      target: { userId: input.userId },
      channels: ["in-app", "ios"],
      idempotencyKey: `report-ready:${input.jobId}`,
    },
  });
  return { results: sent.results };
});
```

A resolved call is not full delivery: check `results[].status` per channel. Inside a task run, put the send inside a `step.do` so a replay does not repeat it, and still set `idempotencyKey` — a step retried after a partial failure then re-sends only the channels (and push tokens) that still need it.

## Idempotency and Deduplication

Pass `idempotencyKey` (a plain string, caller-chosen) on `send()` — from the client or from a function — to make a repeat call safe:

- **Full dedupe.** If every requested channel already has a terminal outcome under that key, the call returns immediately with `deduplicated: true` and the prior `results` — nothing is re-sent.
- **Partial dedupe.** If some channels are done and others still need a retry, the call re-sends only the channels that need it and reports `deduplicatedChannels: string[]` naming the ones served from the prior send.
- **Per-token granularity for push.** Dedupe tracks each device token separately within a channel: retrying a push send where one token delivered and another failed transiently re-sends only to the token that still needs it — the already-reached device does not get a duplicate alert.
- **Window.** Dedupe state is tracked for the same retention window as the audit trail (90 days) — an idempotency key reused after that window is treated as a brand-new send.

Without an `idempotencyKey`, every retry re-sends every requested channel from scratch (at-least-once, not exactly-once) — always set one on a send a caller or a task step might retry.

## Rate Limiting

Sends are capped **per app, per hour, per channel**: 5,000 in-app notifications and 1,000 push dispatches. Each requested channel is checked independently, so a multi-channel send only blocks the channel(s) actually over their limit — the rest deliver normally. Only when **every** requested channel is blocked does the call throw `NotificationRateLimitError` (`limit`, `resetAt`). Waiting out the window is the caller's job; retrying immediately cannot succeed.

## Errors

| Error | Thrown when | HTTP (REST) |
|---|---|---|
| `UnsupportedChannelError` | `channels` is empty, or names a channel outside `"in-app" \| "ios" \| "android"` | 400 |
| `NotificationTargetNotFoundError` | `target.userId` is not a member of the app | 404 |
| `NotificationRateLimitError` | Every requested channel was blocked by its hourly quota | 429 |

## Footguns

- **`send()` needs app admin permission**, not just app membership. A member-level caller gets `403` — this is deliberate (unrestricted member-level send would let any signed-in user spam the tenant's push/in-app quota).
- **Don't rely on the `notification` WS event as your only read path.** It's a best-effort live mirror; a disconnected recipient misses it entirely. Always back it with `list()` / `unreadCount()` for the authoritative state.
- **Set `idempotencyKey` on any send a caller or a task step might retry.** Without it, retries are at-least-once — expect duplicate inbox rows and duplicate push alerts, not a clean no-op.
- **A "delivered" push channel result can still need a retry.** When a user has multiple device tokens and only some fail retryably, the channel's overall `status` can read `"delivered"` while `tokenAttempts` shows a token that still needs a resend — check `tokenAttempts` yourself before treating a `"delivered"` status as fully done.
- **`registerDevice` reassigns tokens across users.** Registering a token already owned by a different account moves it to the new caller — correct for shared-device sign-out/sign-in, but don't assume a token uniquely and permanently identifies one user.
- **`environment` must match the build.** A production app build presenting a token issued under Apple's sandbox APNs environment (or vice versa) registers fine but silently fails to deliver — `environment` is not validated against the token itself.
- **Channels are additive, not implicit.** Omitting `channels` sends `"in-app"` only — a push notification never goes out unless `"ios"` / `"android"` is explicitly requested (and the user has a registered device for it).

## Tips for Coding Agents

1. Always set `idempotencyKey` when a send happens inside a task step or any other retryable call path.
2. Check `results[].status` per channel rather than assuming a resolved promise means full delivery — a partially-successful multi-channel send still resolves normally.
3. Register a device immediately after the user grants OS-level notification permission, and unregister on logout.
4. Treat the `notification` WS event as a UI nicety (badge/toast), never as the system of record — reconcile against `list()`/`unreadCount()`.
5. From a server function, branch on `results[].status`, not just on whether the call threw — a fully-skipped/failed send still resolves.

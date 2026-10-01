# Agent Guide to Primitive Notifications

Notifications create a durable inbox entry and can send push alerts to registered devices. Use `client.notifications` in the app and `ctx.api.notifications.send` in a server function.

## Sending

Sending requires app-admin permission. Use a server function for member-triggered notifications.

```swift
  let response = try await client.notifications.send(
    params: SendNotificationParams(
      title: "Your report is ready",
      body: "Tap to view this week's summary.",
      target: SendNotificationParams.Target(userId: userId),
      channels: ["in-app", "ios"],
      deepLink: "myapp://reports/latest",
      idempotencyKey: "weekly-report-\(userId)-2026-07-14"
    )
  )

  for result in response.results {
    print(result.channel, result.status)  // "in-app" "delivered", "ios" "delivered"
  }
```

Set `title`, `body`, and `target.userId`. `channels` defaults to `["in-app"]`; add `"ios"` or `"android"` for push. `expiresAt` removes the inbox entry after an ISO date without affecting push delivery.

Check `results` for each requested channel. A result can be delivered, failed, skipped, or invalidated. For push, inspect `tokenAttempts` when some devices need a retry.

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

See the generated client reference for complete response types.

Device lists return only the last eight token characters. A user can register up to 100 devices.

### Reading and managing the inbox

```swift
  let inbox = try await client.notifications.list(limit: 20)

  for notification in inbox.items {
    print(notification.title, notification.read)
  }

  if let first = inbox.items.first, !first.read {
    _ = try await client.notifications.markRead(notificationId: first.notificationId)
  }
```

### Registering for push

```swift
  let device = try await client.notifications.registerDevice(
    params: RegisterPushDeviceParams(
      token: deviceToken,
      platform: .ios,
      environment: .production,
      bundleId: "com.example.myapp"
    )
  )

  // Last 8 chars only — the full token is never echoed back
  print(device.tokenSuffix ?? "")
```

Register the device token with its platform and environment. Use the matching Apple sandbox or production environment; use `production` for Android. Re-registering a token transfers delivery to the currently signed-in user.

## Live WebSocket Event

```swift
  // Drop this in a SwiftUI `.task`: the loop runs for as long as the view is on
  // screen and unsubscribes when it goes away.
  for await event in client.stream(for: NotificationEvent.self) {
    // event.notificationId, event.title, event.body, event.iconUrl,
    // event.deepLink, event.sourceRef, event.createdAt
    print(event.title, event.body)
  }
```

Subscribe through `client.stream(for:)` — a `for await` loop in a `.task`, which unsubscribes when the loop ends — or through `client.observeOnMainActor(_:handler:)` when you need a main-actor callback, holding the returned `EventSubscription` for as long as you want the handler live.

Use live events for immediate UI updates. Read `list()` or `unreadCount()` after reconnecting to recover updates missed while disconnected.

## Sending from a Server Function

Call `ctx.api.notifications.send({ body })` with the same send fields. The function’s access rule controls who may trigger the send.

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

## Gotchas {#footguns}

- **`send()` needs app admin permission**, not just app membership. A member-level caller gets `403` — this is deliberate (unrestricted member-level send would let any signed-in user spam the tenant's push/in-app quota).
- **Don't rely on the `notification` WS event as your only read path.** It's a best-effort live mirror; a disconnected recipient misses it entirely. Always back it with `list()` / `unreadCount()` for the authoritative state.
- **Set `idempotencyKey` on any send a caller or a task step might retry.** Without it, retries are at-least-once — expect duplicate inbox rows and duplicate push alerts, not a clean no-op.
- **A "delivered" push channel result can still need a retry.** When a user has multiple device tokens and only some fail retryably, the channel's overall `status` can read `"delivered"` while `tokenAttempts` shows a token that still needs a resend — check `tokenAttempts` yourself before treating a `"delivered"` status as fully done.
- **`registerDevice` reassigns tokens across users.** Registering a token already owned by a different account moves it to the new caller — correct for shared-device sign-out/sign-in, but don't assume a token uniquely and permanently identifies one user.
- **`environment` must match the build.** A production app build presenting a token issued under Apple's sandbox APNs environment (or vice versa) registers fine but silently fails to deliver — `environment` is not validated against the token itself.
- **Channels are additive, not implicit.** Omitting `channels` sends `"in-app"` only — a push notification never goes out unless `"ios"` / `"android"` is explicitly requested (and the user has a registered device for it).

# Agent Guide to Primitive Locks

A **named lock** is a mutual-exclusion primitive keyed by an app-scoped, caller-chosen string. Every acquirer of a key — client code, background jobs, and workflows — is serialized against every other acquirer of that same key in the app. Each lock is a **lease**: acquire it for a `ttlMs` duration; if the holder crashes it never releases, the lease expires and the next acquirer takes over. Locks are cooperative coordination, not an access boundary. The client surface is `client.locks.*`; a workflow uses the `lock.*` steps. Keys are tenant-isolated — the same string in two apps is two independent locks.

## Client SDK Reference

| Call | Returns | Notes |
|---|---|---|
| `client.locks.acquire(key, { ttlMs, timeoutMs })` | `LockHandle` | Blocks (client-side poll loop) until acquired; throws `LockTimeoutError` when `timeoutMs` elapses first. |
| `client.locks.tryAcquire(key, { ttlMs })` | `LockHandle \| null` | Single non-blocking attempt; `null` when the key is held by another caller. |
| `client.locks.release(handle)` | `{ released: boolean, reason? }` | `reason`: `"not_holder"` (stale/wrong handle) or `"not_held"` (already free). The handle carries its own key. |
| `client.locks.renew(handle, { ttlMs })` | `{ renewed: boolean, leaseExpiresAt?, reason? }` | `reason: "lease_lost"` when the handle no longer matches — the lease already lapsed and the key was taken over. |
| `client.locks.status(key)` | `LockStatus` | `{ held: false }`, or `{ held: true, heldBy, holderKind, holderRunId, acquiredAt, leaseExpiresAt }`. Reports `held: false` once the lease has expired. |
| `client.locks.list()` | `{ locks: LockListEntry[] }` | Every currently-held lock in the app. **Requires app admin permission** — a member-level caller gets `403`. |

`LockHandle`: `{ key, handleId, leaseExpiresAt }`. `release` and `renew` require the `handleId`, so a caller can't free or extend a lock it no longer holds. `ttlMs` is required on every acquire and is capped at 24h server-side.

### Acquire and release

`acquire` blocks; `timeoutMs` bounds the wait, `ttlMs` sizes the lease. Release in a `finally`:

```typescript
import { LockTimeoutError } from "js-bao-wss-client";

const handle = await client.locks
  .acquire(`portfolio-import:${userId}`, { ttlMs: 60_000, timeoutMs: 10_000 })
  .catch((err) => {
    if (err instanceof LockTimeoutError) return null; // already running
    throw err;
  });
if (!handle) return; // couldn't get the key within timeoutMs

try {
  // ... exclusive work ...
} finally {
  await client.locks.release(handle);
}
```

### Try without waiting

```typescript
const handle = await client.locks.tryAcquire(`refresh:${userId}`, { ttlMs: 30_000 });
if (!handle) return; // someone else holds it
try {
  // ... exclusive work ...
} finally {
  await client.locks.release(handle);
}
```

### LockTimeoutError

Thrown only by the blocking `acquire()` when it reaches `timeoutMs` without winning the key. `code: "LOCK_TIMEOUT"`; carries `key` and `timeoutMs`. Branch on it to skip or reschedule rather than treating contention as a hard failure. `tryAcquire` never throws it — it returns `null`.

## Sizing the Lease

**The lease does not renew itself.** Size `ttlMs` to comfortably cover the work done while holding the lock. If the lease expires mid-operation, another acquirer can take the key and run concurrently — the exact overlap the lock exists to prevent. For long or variable-duration work, either set a generous `ttlMs` or call `renew(handle, { ttlMs })` before the current lease expires. `renew` returning `{ renewed: false, reason: "lease_lost" }` means the lease already lapsed and the key changed hands — stop and re-acquire.

## CLI

`primitive locks` inspects and scripts the same namespace.

| Command | Purpose |
|---|---|
| `primitive locks list [app-id] [--json]` | List every held lock in the app (**admin**). |
| `primitive locks status <key> [app-id] [--json]` | Show the current holder of a key. |
| `primitive locks acquire <key> [app-id] --ttl <ms> [--json]` | Single non-blocking attempt (`--ttl` default 60000); prints the handle. |
| `primitive locks release <key> [app-id] --handle <handleId> [--json]` | Release with the handle from `acquire`. |

```bash
primitive locks list
primitive locks status portfolio-import:user-123
primitive locks acquire portfolio-import:user-123 --ttl 60000
primitive locks release portfolio-import:user-123 --handle 01HXY...
```

## Workflow Steps

A workflow coordinates through the same keys with four steps. Each records the holder as `holderKind: "workflow"` with the run's id.

| Step | Purpose | Fields |
|---|---|---|
| `lock.acquire` | Acquire the key | `key`, `ttlMs` (default 30000), `timeoutMs` (default 30000), `blocking` (default `true`), `pollMs` (poll override) |
| `lock.release` | Release | `handle` (`{{ steps.<id>.handle }}`) — or flat `key` + `handleId` |
| `lock.renew` | Extend the lease | `handle` (or `key` + `handleId`), `ttlMs` |
| `lock.status` | Inspect the holder | `key` |

`lock.acquire` with `blocking = true` (the default) is a **durable** poll loop: the run suspends between attempts and resumes when the key frees, and on a replay it returns the handle from the attempt that won rather than re-acquiring. On timeout it fails the run rather than overlapping. `blocking = false` makes a single attempt and returns `{ acquired: false, ... }` on contention.

```toml
[[steps]]
id = "acquire"
kind = "lock.acquire"
key = "portfolio-import:{{ input.userId }}"
ttlMs = 60000
timeoutMs = 30000

# ... steps that must not overlap for this user ...

[[steps]]
id = "release"
kind = "lock.release"
handle = "{{ steps.acquire.handle }}"
```

The lease-sizing rule applies to the acquire/release span exactly as it does on the client: hold across long work only with a `ttlMs` that covers it, or `lock.renew` as you go. There is no automatic heartbeat.

## Run-Scoped Declarative Lock

A workflow can hold a lock for its **entire run** — acquired before the first step and released on both the success and failure branches — so two runs targeting the same key are serialized end to end with no explicit steps. Config: `key` (templated against the run's `input`/`user`/`meta`), `ttlMs` (default 5 minutes, capped at 24h), `timeoutMs` (default 30s), `onContention` (`"block"` — wait then fail — or `"fail"` — fail fast).

**Size `ttlMs` to cover the worst-case duration of the whole run.** The run holds the lease for its full lifetime with no periodic renewal; ownership is re-verified only when the durable engine replays. A run that executes continuously longer than `ttlMs` — many back-to-back compute or LLM steps with no durable pause to force a replay — can let its lease lapse while still running, at which point a second run can take the key over and both critical sections run concurrently. This is a deliberate tradeoff (the lease, not a heartbeat, is the safety bound), so the lease must be sized generously enough that a run never outlives it.

A run-scoped lock is honored **only on the durable execution path**. A `syncCallable` workflow cannot declare one (the acquire-around-run lifecycle does not run under run-sync, so the lock would be silently skipped while the run reported success), and a `workflow.call` into a lock-declaring workflow fails fast for the same reason (its child runs under run-sync). The server rejects the `syncCallable` + lock combination at save time. Honoring declarative locks under run-sync / `workflow.call` is tracked in [#1883](https://github.com/Primitive-Labs/js-bao-wss/issues/1883).

## Rate Limiting

Acquire attempts are capped at **600 per user per hour**. A blocking `acquire()` counts each poll against this limit and handles a rate-limit response internally — it keeps waiting within `timeoutMs` and raises `LockTimeoutError` if it never wins, rather than surfacing the limit. A single `tryAcquire()` that trips the limit surfaces the rate-limit error (`429`) to the caller.

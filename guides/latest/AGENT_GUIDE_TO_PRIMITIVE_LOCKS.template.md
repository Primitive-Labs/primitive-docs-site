# Agent Guide to Primitive Locks

A **named lock** is a mutual-exclusion primitive keyed by an app-scoped, caller-chosen string. Every acquirer of a key — client code and server functions — is serialized against every other acquirer of that same key in the app. Each lock is a **lease**: acquire it for a bounded TTL; if the holder crashes it never releases, the lease expires and the next acquirer takes over. Locks are cooperative — they coordinate willing participants, and holding one grants no rights over data. *Who* may take which key is a separate, opt-in question, answered by [Access Control](#access-control) below. The client surface is `client.locks.*`; a server function uses `ctx.api.locks.*`. Keys are tenant-isolated — the same string in two apps is two independent locks.

## When to Reach for a Lock

A lock is a last resort. Rule out cheaper serialization before acquiring one:

- **Partition the work** so concurrent workers cannot select the same rows — shard a batch by user, resource, or another stable key.
- **Make writes idempotent** so a retry or an overlapping run is harmless rather than something to serialize against.
- **Guard the write with a conditional write** instead of locking around it — a `condition` on the record write (a field-equality precondition, commonly a `version` field; in a server function, `ctx.db(id, "<type>").model("<Model>").patch(recordId, { data, condition })`) checked in the same transaction as the write. This is the actual integrity boundary in most "concurrent workers hit the same record" cases: the database, not a held lock, guarantees exactly one writer succeeds. See [Databases](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md).
- **For scheduled work, use a cron trigger's `overlapPolicy`** instead of locking inside the job body — `"skip"` (the default) already refuses to start a firing while the previous one is still running. A server function's cron trigger carries it: see [Server Functions — Triggers](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#triggers).

Reach for a lock only when none of these fit: a critical section spanning multiple independent writes, or non-database work (an external API call, a multi-step task) that must run exclusively. Even then, a held lease is not a correctness guarantee for a long-running critical section — see [Sizing the Lease](#sizing-the-lease): a lease that expires mid-operation lets a second run in.

## Client SDK Reference

{{#lang ts}}
| Call | Returns | Notes |
|---|---|---|
| `client.locks.acquire(key, { ttlMs, timeoutMs })` | `LockHandle` | Blocks (client-side poll loop) until acquired; throws `LockTimeoutError` when `timeoutMs` elapses first. |
| `client.locks.tryAcquire(key, { ttlMs })` | `LockHandle \| null` | Single non-blocking attempt; `null` when the key is held by another caller. |
| `client.locks.release(handle)` | `{ released: boolean, reason? }` | `reason`: `"not_holder"` (stale/wrong handle) or `"not_held"` (already free). The handle carries its own key. |
| `client.locks.renew(handle, { ttlMs })` | `{ renewed: boolean, leaseExpiresAt?, reason? }` | `reason: "lease_lost"` when the handle no longer matches — the lease already lapsed and the key was taken over. |
| `client.locks.status(key)` | `LockStatus` | `{ held: false }`, or `{ held: true, heldBy, holderKind, holderRunId, owner, acquiredAt, leaseExpiresAt }`. Reports `held: false` once the lease has expired. |
| `client.locks.list()` | `{ locks: LockListEntry[] }` | Every currently-held lock in the app. **Requires app admin permission** — a member-level caller gets `403`. |

`LockHandle`: `{ key, handleId, leaseExpiresAt }`. `release` and `renew` require the `handleId`, so a caller can't free or extend a lock it no longer holds. `ttlMs` is required on every acquire and is capped at 24h server-side.
{{/lang}}
{{#lang swift}}
| Call | Returns | Notes |
|---|---|---|
| `client.locks.acquire(key:ttl:timeout:)` | `LockHandle` | Blocks (client-side poll loop) until acquired; throws `JsBaoError` with `code == .lockTimeout` when `timeout` elapses first. |
| `client.locks.tryAcquire(key:ttl:)` | `LockHandle?` | Single non-blocking attempt; `nil` when the key is held by another caller. |
| `client.locks.release(_ handle:)` | `LockReleaseResult` | `released`, plus `reason`: `"not_holder"` (stale/wrong handle) or `"not_held"` (already free). The handle carries its own key. |
| `client.locks.renew(_ handle:ttl:)` | `LockRenewResult` | `renewed`, `leaseExpiresAt`, and `reason: "lease_lost"` when the handle no longer matches — the lease already lapsed and the key was taken over. |
| `client.locks.status(key:)` | `LockStatus` | `held`, plus `heldBy`, `holderKind`, `holderRunId`, `owner`, `acquiredAt`, `leaseExpiresAt` while held. Reports `held == false` once the lease has expired. |
| `client.locks.list()` | `LockListResult` | Every currently-held lock in the app. **Requires app admin permission** — a member-level caller gets `403`. |

`LockHandle`: `key`, `handleId`, `leaseExpiresAt`. `release` and `renew` require the `handleId`, so a caller can't free or extend a lock it no longer holds. `ttl` and `timeout` are `TimeInterval`s in **seconds**. `ttl` is required on every acquire and is capped at 24h server-side.
{{/lang}}

### Acquire and release

`acquire` blocks; the acquire timeout bounds the wait and the TTL sizes the lease. Always release, including on a thrown step:

{{ example: locks/acquire }}

### Try without waiting

{{ example: locks/try-acquire }}

### Inspecting a key

{{ example: locks/status }}

### The acquire timeout

{{#lang ts}}
`LockTimeoutError` is thrown only by the blocking `acquire()` when it reaches `timeoutMs` without winning the key. `code: "LOCK_TIMEOUT"`; carries `key` and `timeoutMs`. Branch on it to skip or reschedule rather than treating contention as a hard failure. `tryAcquire` never throws it — it returns `null`.
{{/lang}}
{{#lang swift}}
`JsBaoError` with `code == .lockTimeout` is thrown only by the blocking `acquire(key:ttl:timeout:)` when it reaches `timeout` without winning the key. Its `details` carry `key` and `timeoutMs` (the elapsed wait in milliseconds, as the server reports it). Branch on it to skip or reschedule rather than treating contention as a hard failure. `tryAcquire` never throws it — it returns `nil`.
{{/lang}}

## Sizing the Lease

**The lease does not renew itself.** Size the TTL to comfortably cover the work done while holding the lock. If the lease expires mid-operation, another acquirer can take the key and run concurrently — the exact overlap the lock exists to prevent. For long or variable-duration work, either set a generous TTL or call `renew` with a fresh one before the current lease expires. A `renew` that comes back not renewed, with `reason: "lease_lost"`, means the lease already lapsed and the key changed hands — stop and re-acquire.

## Re-taking Your Own Lease

A handle is the only thing that frees a lock, so a caller that loses its handle — a task run the platform resets, a process that restarts — can neither release its key nor acquire it. Name an `owner` on acquire and it can take its own lease back.

{{#lang ts}}
```ts
const handle = await client.locks.tryAcquire(key, { ttlMs: 60_000, owner: myRunId });
```
{{/lang}}
{{#lang swift}}
```swift
let handle = try await client.locks.tryAcquire(key: key, ttl: 60, owner: myRunId)
```
{{/lang}}

Presenting the owner that already holds the key succeeds with a **fresh handle** and a fresh lease; the handle the previous acquire minted is fenced — `release` on it answers `not_holder` and `renew` answers `lease_lost`. The guarantee is one current handle at every instant, with every replaced handle fenced, so the acquire that lost the key can neither release it nor renew it.

It is **not** a way to interrupt the previous holder. Nothing stops code that is already running; it runs on until it next talks to the lock and is told it is not the holder. Re-take when the earlier attempt is known to be gone — a run that was reset, naming itself with `owner: ctx.runId`.

Three things must match, not just the owner:

- **the same principal** — the same signed-in user, or the same function;
- **the same kind of caller** — a function's hold is re-taken by that function's run, a member's own hold by that member. The owner is readable off `status`, so without this anyone who could read it could take the hold over; a member who starts a task cannot rotate or release the lease their function holds;
- **the same owner string**, exactly.

Everything else is refused with the ordinary contention shape, and a hold made with **no** owner is never re-entered — omit `owner` and the lock is strictly non-reentrant.

**Choosing an owner.** Name the run (`ctx.runId` inside a server function), or the work a run key coalesces. Never a static string: two unrelated callers presenting one would re-enter each other's hold, which is the opposite of a lock.

`status` reports the owner, so a refused caller can tell its own hold from another's.

## Access Control

Lock keys share one app-wide namespace, so by default **any signed-in member may acquire or renew any key** — that is the shipped default and it stays until you opt out of it. Restrict it with a single **`lock` rule set**: one CEL rule per operation, matched against the key the caller asked for as `record.key`.

```toml
# primitive/dev/rule-sets/lock-policy.toml
[ruleSet]
name = "lock-policy"
resourceType = "lock"

[rules.lock]
acquire = "record.key.startsWith('user:' + user.userId + ':')"
renew = "record.key.startsWith('user:' + user.userId + ':')"
```

```bash
primitive config push --only rule-set/lock-policy
```

- **Context**: `user.userId` / `user.role` (the caller) and `record.key` (the exact requested key) — scope by prefix (`record.key.startsWith('jobs:')`), caller, or group (`isMemberOf('ops', 'core')`).
- App admins/owners always pass; rules govern regular members only.
- **No `lock` rule set installed → open** — any member may operate on any key. This is the explicit default; installing a rule set is the opt-in.
- **With a rule set installed, every operation it does not define is denied.** A set naming only `acquire` denies `renew` for members. Only `acquire` and `renew` are gated — `release` is authorized by the handle itself.
- At most **one** `lock` rule set per app — the policy is app-wide.
- Denial: `403 { errorCode: "LOCK_ACCESS_DENIED" }`. The response never echoes the rule.
- **`release` is never rule-gated.** The handle minted at acquire is itself proof of holding the key, so a caller can always free a key it holds — even if a policy change mid-hold has already revoked its `renew`. If it never releases, the lease reclaims the key on its own.
- **Server functions are unaffected**: a function's `ctx.api.locks.*` calls carry the app's authority and never pass through this rule — the function's own `access` gate governs who may run it. `locks/status` stays readable by any member; `locks list` remains admin-only.

## CLI

`primitive locks` inspects and scripts the same namespace.

| Command | Purpose |
|---|---|
| `primitive locks list [app-id] [--json]` | List every held lock in the app, with an `OWNER` column (**admin**). |
| `primitive locks status <key> [app-id] [--json]` | Show the current holder of a key, including its `Owner`. |
| `primitive locks acquire <key> [app-id] --ttl <ms> [--owner <owner>] [--json]` | Single non-blocking attempt (`--ttl` default 60000); prints the handle. `--owner` re-takes a lease that owner already holds. |
| `primitive locks release <key> [app-id] --handle <handleId> [--json]` | Release with the handle from `acquire`. |

```bash
primitive locks list
primitive locks status portfolio-import:user-123
primitive locks acquire portfolio-import:user-123 --ttl 60000
primitive locks acquire portfolio-import:user-123 --owner run-01M2H5EYQQ
primitive locks release portfolio-import:user-123 --handle 01HXY...
```

## From a server function

A server function takes a lock through `ctx.api.locks.tryAcquire` / `renew` / `release` / `status` — the same routes, with the same bodies (`{ key, ttlMs, owner }` to acquire, `{ key, handle: { handleId } }` to release). There is no blocking acquire and no declarative lock on the function side: the pattern is a single `tryAcquire` and a branch on `acquired`. A function's lock calls carry the app's authority, so a `lock` rule set never refuses them. Under the task runtime, acquire and release **live on every slice, outside `step.do`**, with `owner: ctx.runId` (or `ctx.trigger.runKey ?? ctx.runId` when a run key coalesces triggers) — see [Server Functions — Locks](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#locks) for the full pattern and Re-taking Your Own Lease above for the owner rule.

## Rate Limiting

Acquire attempts are capped at **600 per user per hour**. A blocking `acquire()` counts each poll against this limit and handles a rate-limit response internally — it keeps waiting within the acquire timeout and raises the acquire-timeout error if it never wins, rather than surfacing the limit. A single `tryAcquire()` that trips the limit surfaces the rate-limit error (`429`) to the caller.

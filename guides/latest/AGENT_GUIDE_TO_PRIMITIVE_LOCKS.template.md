# Agent Guide to Primitive Locks

A **named lock** is a mutual-exclusion primitive keyed by an app-scoped, caller-chosen string. Every acquirer of a key — client code, background jobs, and workflows — is serialized against every other acquirer of that same key in the app. Each lock is a **lease**: acquire it for a `ttlMs` duration; if the holder crashes it never releases, the lease expires and the next acquirer takes over. Locks are cooperative coordination, not an access boundary. The client surface is `client.locks.*`; a workflow uses the `lock.*` steps. Keys are tenant-isolated — the same string in two apps is two independent locks.

## Client SDK Reference

{{#lang ts}}
| Call | Returns | Notes |
|---|---|---|
| `client.locks.acquire(key, { ttlMs, timeoutMs })` | `LockHandle` | Blocks (client-side poll loop) until acquired; throws `LockTimeoutError` when `timeoutMs` elapses first. |
| `client.locks.tryAcquire(key, { ttlMs })` | `LockHandle \| null` | Single non-blocking attempt; `null` when the key is held by another caller. |
| `client.locks.release(handle)` | `{ released: boolean, reason? }` | `reason`: `"not_holder"` (stale/wrong handle) or `"not_held"` (already free). The handle carries its own key. |
| `client.locks.renew(handle, { ttlMs })` | `{ renewed: boolean, leaseExpiresAt?, reason? }` | `reason: "lease_lost"` when the handle no longer matches — the lease already lapsed and the key was taken over. |
| `client.locks.status(key)` | `LockStatus` | `{ held: false }`, or `{ held: true, heldBy, holderKind, holderRunId, acquiredAt, leaseExpiresAt }`. Reports `held: false` once the lease has expired. |
| `client.locks.list()` | `{ locks: LockListEntry[] }` | Every currently-held lock in the app. **Requires app admin permission** — a member-level caller gets `403`. |

`LockHandle`: `{ key, handleId, leaseExpiresAt }`. `release` and `renew` require the `handleId`, so a caller can't free or extend a lock it no longer holds. `ttlMs` is required on every acquire and is capped at 24h server-side.
{{/lang}}
{{#lang swift}}
| Call | Returns | Notes |
|---|---|---|
| `client.locks.acquire(key:ttlMs:timeoutMs:)` | `LockHandle` | Blocks (client-side poll loop) until acquired; throws `JsBaoError` with `code == .lockTimeout` when `timeoutMs` elapses first. |
| `client.locks.tryAcquire(key:ttlMs:)` | `LockHandle?` | Single non-blocking attempt; `nil` when the key is held by another caller. |
| `client.locks.release(_ handle:)` | `LockReleaseResult` | `released`, plus `reason`: `"not_holder"` (stale/wrong handle) or `"not_held"` (already free). The handle carries its own key. |
| `client.locks.renew(_ handle:ttlMs:)` | `LockRenewResult` | `renewed`, `leaseExpiresAt`, and `reason: "lease_lost"` when the handle no longer matches — the lease already lapsed and the key was taken over. |
| `client.locks.status(key:)` | `LockStatus` | `held`, plus `heldBy`, `holderKind`, `holderRunId`, `acquiredAt`, `leaseExpiresAt` while held. Reports `held == false` once the lease has expired. |
| `client.locks.list()` | `LockListResult` | Every currently-held lock in the app. **Requires app admin permission** — a member-level caller gets `403`. |

`LockHandle`: `key`, `handleId`, `leaseExpiresAt`. `release` and `renew` require the `handleId`, so a caller can't free or extend a lock it no longer holds. `ttlMs` is required on every acquire and is capped at 24h server-side.
{{/lang}}

### Acquire and release

`acquire` blocks; `timeoutMs` bounds the wait, `ttlMs` sizes the lease. Always release, including on a thrown step:

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
`JsBaoError` with `code == .lockTimeout` is thrown only by the blocking `acquire(key:ttlMs:timeoutMs:)` when it reaches `timeoutMs` without winning the key. Its `details` carry `key` and `timeoutMs`. Branch on it to skip or reschedule rather than treating contention as a hard failure. `tryAcquire` never throws it — it returns `nil`.
{{/lang}}

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

A workflow can hold a lock for its **entire run** — acquired before the first step and released on both the success and failure branches — so two runs targeting the same key are serialized end to end with no explicit steps. Config: `key` (templated against the run's `input`/`user`/`meta`), `ttlMs` (default 5 minutes, capped at 24h), `timeoutMs` (default 30s), `onContention` (`"block"` — wait then fail — or `"fail"` — fail fast). Declare it as a `[workflow.lock]` block; only `key` is required:

```toml
[workflow.lock]
key = "portfolio-bulk:{{ input.documentId }}"
ttlMs = 600000
timeoutMs = 30000
onContention = "fail"
```

The block is TOML-owned config that round-trips through `primitive sync pull`/`push`; removing it clears the lock.

**Size `ttlMs` to cover the worst-case duration of the whole run.** The run holds the lease for its full lifetime with no periodic renewal; ownership is re-verified only when the durable engine replays. A run that executes continuously longer than `ttlMs` — many back-to-back compute or LLM steps with no durable pause to force a replay — can let its lease lapse while still running, at which point a second run can take the key over and both critical sections run concurrently. This is a deliberate tradeoff (the lease, not a heartbeat, is the safety bound), so the lease must be sized generously enough that a run never outlives it.

A run-scoped lock is honored on **every** execution path: the durable path, `syncCallable` run-sync, and a `workflow.call` into a lock-declaring workflow (each acquires before the first step and releases on both the success and failure branches). A `syncCallable` workflow may therefore declare a `lock:`.

- **Cross-path serialization.** All paths share one app-scoped lock namespace, so a durable `start()` run and a `run-sync` run that declare the same key block each other — the intended end-to-end serialization.
- **Nested `workflow.call` re-entrancy.** A child that declares a key an ancestor in the same call chain already holds re-enters it: the child runs its body without re-acquiring and without releasing (the ancestor owns the lifecycle), so a chain never deadlocks on a lock it already holds.
- **Concurrent same-run siblings are not mutually excluded.** A `forEach { workflow.call }` running children in-process (concurrency > 1) that each declare the *same fresh* key share one run and re-enter without acquiring — a run does not serialize against itself. **The imperative `lock.*` steps are not an escape hatch for this.** A `workflow.call` child inherits its parent's run identity, and every workflow-held lock — declarative or imperative — is owned by that identity, so two `lock.acquire` steps on one key inside a single run both return `acquired: true` with the same handle. No lock serializes a run against itself. To serialize concurrent branches, set the `forEach` step's `concurrency = 1`; to let them run concurrently without contending, give each branch its own key (template the key on the iteration item).

## Rate Limiting

Acquire attempts are capped at **600 per user per hour**. A blocking `acquire()` counts each poll against this limit and handles a rate-limit response internally — it keeps waiting within `timeoutMs` and raises the acquire timeout if it never wins, rather than surfacing the limit. A single `tryAcquire()` that trips the limit surfaces the rate-limit error (`429`) to the caller.

# Agent Guide to Primitive Locks

A **named lock** is a mutual-exclusion primitive keyed by an app-scoped, caller-chosen string. Every acquirer of a key — client code, background jobs, and workflows — is serialized against every other acquirer of that same key in the app. Each lock is a **lease**: acquire it for a bounded TTL; if the holder crashes it never releases, the lease expires and the next acquirer takes over. Locks are cooperative — they coordinate willing participants, and holding one grants no rights over data. *Who* may take which key is a separate, opt-in question, answered by [Access Control](#access-control) below. The client surface is `client.locks.*`; a workflow uses the `lock.*` steps. Keys are tenant-isolated — the same string in two apps is two independent locks.

## Client SDK Reference

| Call | Returns | Notes |
|---|---|---|
| `client.locks.acquire(key:ttl:timeout:)` | `LockHandle` | Blocks (client-side poll loop) until acquired; throws `JsBaoError` with `code == .lockTimeout` when `timeout` elapses first. |
| `client.locks.tryAcquire(key:ttl:)` | `LockHandle?` | Single non-blocking attempt; `nil` when the key is held by another caller. |
| `client.locks.release(_ handle:)` | `LockReleaseResult` | `released`, plus `reason`: `"not_holder"` (stale/wrong handle) or `"not_held"` (already free). The handle carries its own key. |
| `client.locks.renew(_ handle:ttl:)` | `LockRenewResult` | `renewed`, `leaseExpiresAt`, and `reason: "lease_lost"` when the handle no longer matches — the lease already lapsed and the key was taken over. |
| `client.locks.status(key:)` | `LockStatus` | `held`, plus `heldBy`, `holderKind`, `holderRunId`, `acquiredAt`, `leaseExpiresAt` while held. Reports `held == false` once the lease has expired. |
| `client.locks.list()` | `LockListResult` | Every currently-held lock in the app. **Requires app admin permission** — a member-level caller gets `403`. |

`LockHandle`: `key`, `handleId`, `leaseExpiresAt`. `release` and `renew` require the `handleId`, so a caller can't free or extend a lock it no longer holds. `ttl` and `timeout` are `TimeInterval`s in **seconds**. `ttl` is required on every acquire and is capped at 24h server-side.

### Acquire and release

`acquire` blocks; the acquire timeout bounds the wait and the TTL sizes the lease. Always release, including on a thrown step:

```swift
  let handle: LockHandle
  do {
    handle = try await client.locks.acquire(
      key: "portfolio-import:\(userId)",
      ttl: 60,    // seconds — lease long enough to cover the work
      timeout: 10 // seconds — give up waiting after 10s
    )
  } catch let error as JsBaoError where error.code == .lockTimeout {
    return false // already running
  }

  do {
    try await work()
  } catch {
    _ = try? await client.locks.release(handle)
    throw error
  }
  _ = try? await client.locks.release(handle)
  return true
```

### Try without waiting

```swift
  guard let handle = try await client.locks.tryAcquire(
    key: "refresh:\(userId)",
    ttl: 30 // seconds
  ) else {
    return // someone else holds it — nothing to do
  }

  do {
    try await work()
  } catch {
    _ = try? await client.locks.release(handle)
    throw error
  }
  _ = try? await client.locks.release(handle)
```

### Inspecting a key

```swift
  let status = try await client.locks.status(key: key)
  if status.held {
    print("held by \(status.heldBy ?? "?"), lease expires \(status.leaseExpiresAt ?? "?")")
  } else {
    print("free")
  }
```

### The acquire timeout

`JsBaoError` with `code == .lockTimeout` is thrown only by the blocking `acquire(key:ttl:timeout:)` when it reaches `timeout` without winning the key. Its `details` carry `key` and `timeoutMs` (the elapsed wait in milliseconds, as the server reports it). Branch on it to skip or reschedule rather than treating contention as a hard failure. `tryAcquire` never throws it — it returns `nil`.

## Sizing the Lease

**The lease does not renew itself.** Size the TTL to comfortably cover the work done while holding the lock. If the lease expires mid-operation, another acquirer can take the key and run concurrently — the exact overlap the lock exists to prevent. For long or variable-duration work, either set a generous TTL or call `renew` with a fresh one before the current lease expires. A `renew` that comes back not renewed, with `reason: "lease_lost"`, means the lease already lapsed and the key changed hands — stop and re-acquire.

## Access Control

Lock keys share one app-wide namespace, so by default **any signed-in member may acquire or renew any key** — that is the shipped default and it stays until you opt out of it. Restrict it with a single **`lock` rule set**: one CEL rule per operation, matched against the key the caller asked for as `record.key`.

```toml
# config/rule-sets/lock-policy.toml
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
- **No `lock` rule set installed → open** — any member may operate on any key. This is the explicit, compatibility-preserving default; installing a rule set is the opt-in.
- **With a rule set installed, every operation it does not define is denied.** A set naming only `acquire` denies `renew` for members. Only `acquire` and `renew` are gated — `release` is authorized by the handle itself.
- At most **one** `lock` rule set per app — the policy is app-wide.
- Denial: `403 { errorCode: "LOCK_ACCESS_DENIED" }`. The response never echoes the rule.
- **`release` is never rule-gated.** The handle minted at acquire is itself proof of holding the key, so a caller can always free a key it holds — even if a policy change mid-hold has already revoked its `renew`. If it never releases, the lease reclaims the key on its own.
- **Workflows are unaffected**: the `lock.*` steps and a declarative `[workflow.lock]` run server-side and never pass through this rule — a workflow's own `accessRule` governs who may start it. `locks/status` stays readable by any member; `locks list` remains admin-only.

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

A workflow can hold a lock for its **entire run** — acquired before the first step and released on both the success and failure branches — so two runs targeting the same key are serialized end to end with no explicit steps. Config: `key` (templated against the run's `input`/`user`/`meta`), `ttlMs` (default 5 minutes, capped at 24h), `timeoutMs` (default 30s), `onContention` (`"block"` — wait then fail — `"fail"` — fail fast — or `"ignore"` — do not run and do not fail). Declare it as a `[workflow.lock]` block; only `key` is required:

```toml
[workflow.lock]
key = "portfolio-bulk:{{ input.documentId }}"
ttlMs = 600000
timeoutMs = 30000
onContention = "fail"
```

The block is TOML-owned config that round-trips through `primitive config pull`/`push`; removing it clears the lock.

**What the losing run does** is `onContention`'s whole job:

| Value | The losing run |
|---|---|
| `block` (default) | Waits up to `timeoutMs`, then fails with `errorCode: "LOCK_TIMEOUT"`. |
| `fail` | Fails immediately with `errorCode: "LOCK_CONTENTION"`. |
| `ignore` | Does not run and does not fail: it settles `status: "skipped"` with `skipReason: "LOCK_CONTENTION"`, carries no error, and emits no error events. |

Pick `fail` when two concurrent runs must never happen and you want the alert. Pick `ignore` when losing the race is expected — a client double-tap, a nightly job that overlaps itself — so it stops reading as a crash in error analytics. An elided run is still stored and listed: `primitive workflows runs list <workflow-id> --status skipped` is the contention-volume view.

Branch on the structured field, never on message text: `errorCode` is on every surface that carries `errorMessage` (the `run-sync` envelope, the run status endpoint, `listRuns`, the `workflowStatus` event) and `skipReason` sits next to `status`. Both are written by the platform from a closed set; an app cannot set or spoof either.

**`ignore` requires `js-bao-wss-client` >= 2.1.0** (or the Swift client at or after the release that ships it). `skipped` is a new value on an existing status enum, so an older client's `waitFor` does not treat it as terminal and waits out its own timeout (15 minutes by default) instead of returning. Upgrade the client before switching a workflow to `ignore`.

Inside a nested `workflow.call`, an elided child is a value rather than an error: the step reads `{ output: null, skipped: true, skipReason: "LOCK_CONTENTION", ok: false }`, the parent run continues, and a downstream step listing that step in `skipWhenSkipped` skips in turn. The child call is also recorded as its own run of the child workflow — an elided one lands with `status: "skipped"` and the calling run in `meta.parentRunId`, so `runs list <child-key> --status skipped` sees contention through `workflow.call` too.

**Size `ttlMs` to cover the worst-case duration of the whole run.** The run holds the lease for its full lifetime with no periodic renewal; ownership is re-verified only when the durable engine replays. A run that executes continuously longer than `ttlMs` — many back-to-back compute or LLM steps with no durable pause to force a replay — can let its lease lapse while still running, at which point a second run can take the key over and both critical sections run concurrently. This is a deliberate tradeoff (the lease, not a heartbeat, is the safety bound), so the lease must be sized generously enough that a run never outlives it.

A run-scoped lock is honored on **every** execution path: the durable path, `syncCallable` run-sync, and a `workflow.call` into a lock-declaring workflow (each acquires before the first step and releases on both the success and failure branches). A `syncCallable` workflow may therefore declare a `lock:`.

- **Cross-path serialization.** All paths share one app-scoped lock namespace, so a durable `start()` run and a `run-sync` run that declare the same key block each other — the intended end-to-end serialization.
- **Nested `workflow.call` re-entrancy.** A child that declares a key an ancestor in the same call chain already holds re-enters it: the child runs its body without re-acquiring and without releasing (the ancestor owns the lifecycle), so a chain never deadlocks on a lock it already holds.
- **Concurrent same-run siblings are not mutually excluded.** A `forEach { workflow.call }` running children in-process (concurrency > 1) that each declare the *same fresh* key share one run and re-enter without acquiring — a run does not serialize against itself. **The imperative `lock.*` steps are not an escape hatch for this.** A `workflow.call` child inherits its parent's run identity, and every workflow-held lock — declarative or imperative — is owned by that identity, so two `lock.acquire` steps on one key inside a single run both return `acquired: true` with the same handle. No lock serializes a run against itself. To serialize concurrent branches, set the `forEach` step's `concurrency = 1`; to let them run concurrently without contending, give each branch its own key (template the key on the iteration item).

## Rate Limiting

Acquire attempts are capped at **600 per user per hour**. A blocking `acquire()` counts each poll against this limit and handles a rate-limit response internally — it keeps waiting within the acquire timeout and raises the acquire-timeout error if it never wins, rather than surfacing the limit. A single `tryAcquire()` that trips the limit surfaces the rate-limit error (`429`) to the caller.

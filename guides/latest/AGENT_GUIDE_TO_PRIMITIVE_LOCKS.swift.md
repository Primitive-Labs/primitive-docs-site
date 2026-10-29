# Agent Guide to Primitive Locks

A **named lock** coordinates work on the same key within an app. Each lock is a lease that expires unless renewed. Locks are taken only in a server function, with `ctx.locks` (or the one-shot `ctx.api.locks`); the app invokes the function. A call to the lock routes from anything but a server function is refused `403 FUNCTION_ROUTE_FUNCTION_ONLY`.

All participating operations must take the same key. Holding a lock does not grant data access.

## Choosing a lock

A lock is a last resort. Rule out cheaper serialization before acquiring one:

- **Partition the work** so concurrent workers cannot select the same rows — shard a batch by user, resource, or another stable key.
- **Make writes idempotent** so a retry or an overlapping run is harmless rather than something to serialize against.
- **Guard the write with a conditional write** instead of locking around it — a `condition` on the record write (commonly on a `version` field; in a server function, `ctx.db3(db3Id, "<type>").model("<model>").patch(recordId, { data, condition })`, refused with `DB3_CONDITION_NOT_MET` when the record no longer matches) checked in the same transaction as the write. This is the actual integrity boundary in most "concurrent workers hit the same record" cases: the database, not a held lock, guarantees exactly one writer succeeds. See [Databases](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#writes).
- **For scheduled work, use a cron trigger's `overlapPolicy`** instead of locking inside the job body — `"skip"` (the default) already refuses to start a firing while the previous one is still running. A server function's cron trigger carries it: see [Server Functions — Triggers](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#triggers).

Reach for a lock only when none of these fit: a critical section spanning multiple independent writes, or non-database work (an external API call, a multi-step task) that must run exclusively. Even then, a held lease is not a correctness guarantee for a long-running critical section — see [lease expiry](#gotchas-lease-expiry): a lease that expires mid-operation lets a second run in.

## Locking in a function

`ctx.locks.withLock(key, { ttlMs, timeoutMs, owner? }, fn)` acquires the key, runs `fn(handle)` while it is held, and releases in `finally` — on a return and on a throw:

```ts
import { defineFunction, requireSignedIn } from "primitive-functions";

export default defineFunction("import-portfolio", {
  authorize: requireSignedIn(),
  handler: async (input: { portfolioId: string }, ctx) =>
    ctx.locks.withLock(`portfolio-import:${input.portfolioId}`, {
      ttlMs: 60_000,     // the lease
      timeoutMs: 5_000,  // how long to wait for a busy key
    }, async () => {
      // …work that must not overlap…
      return { imported: true };
    }),
});
```

- **Not acquired within `timeoutMs`:** `withLock` throws an error whose `name` and `code` are `LOCK_TIMEOUT` and never runs `fn`. Catch it to skip or reschedule.
- **To branch on contention**, call `ctx.locks.acquire(key, { ttlMs, timeoutMs })` and `ctx.locks.release(handle)` in a `try`/`finally`. `acquire` answers `{ acquired: true, handle }` or `{ acquired: false, timedOut: true, rateLimited, heldBy, holderKind, owner, leaseExpiresAt }`.
- **The wait is server-side:** each attempt asks the platform to hold the request for up to 30 seconds and answers the moment the key frees. A request invocation must fit the whole `timeoutMs` in its remaining budget, or the call fails `FUNCTION_RUNTIME_REFUSED`; a task's wait survives a reset. See [Server Functions — Locks](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#locks).

## Calling it from the app

The app invokes the function; it never calls a lock route itself.

```swift
  struct Greeting: Decodable, Sendable { let message: String }

  let result: FunctionResult<Greeting> = try await client.functions.invoke(
    "greet",
    input: ["name": "Ada"],   // travels as rootInput
    timeout: 10               // seconds; default 5, ceiling 30
  )

  if result.status == "completed" {
    print(result.output?.message ?? "") // "hello Ada"
  } else {
    // "failed" — the handler threw or its output failed outputSchema — or "timeout"
    print(result.status, result.error ?? "")
  }
```

## Who may lock

The function's `authorize` decides who may run it, and so who may take its keys. Inside the handler the function acts with the app's authority and can lock any key, with no rule set or per-key configuration to install. Derive the key from what `authorize` admitted:

```ts
import { defineFunction, requireMember } from "primitive-functions";

export default defineFunction("rebuild-team-report", {
  authorize: requireMember((input: { teamId: string }) => input.teamId),
  handler: async (input, ctx) =>
    ctx.locks.withLock(`team-report:${ctx.auth.groupId}`, { ttlMs: 120_000, timeoutMs: 5_000 }, async () => {
      return { rebuilt: true };
    }),
});
```

## Function SDK reference

| Call | Returns | Notes |
|---|---|---|
| `ctx.locks.withLock(key, { ttlMs, timeoutMs, owner? }, fn)` | `fn`'s value | Acquire, run, release in `finally`; throws `LOCK_TIMEOUT` without running `fn`. |
| `ctx.locks.acquire(key, { ttlMs, timeoutMs, owner? })` | `{ acquired, handle?, … }` | Waits up to `timeoutMs`; never throws for "not acquired". |
| `ctx.locks.release(handle)` | `{ released, reason? }` | `reason`: `"not_holder"` (stale handle) or `"not_held"` (already free). |
| `ctx.api.locks.tryAcquire({ body: { key, ttlMs, owner?, waitMs? } })` | `{ acquired: true, handle }` or the contention arm | One attempt; `waitMs` waits server-side as above. |
| `ctx.api.locks.renew({ body: { key, handle: { handleId }, ttlMs } })` | `{ renewed, leaseExpiresAt?, reason? }` | `reason: "lease_lost"` when the lease already lapsed and the key changed hands. |
| `ctx.api.locks.release({ body: { key, handle: { handleId } } })` | `{ released, reason? }` | As `ctx.locks.release`. |
| `ctx.api.locks.status({ body: { key } })` | `{ held, heldBy?, holderKind?, holderRunId?, owner?, acquiredAt?, leaseExpiresAt? }` | `held: false` once the lease has expired. |

`ttlMs` is required on every acquire and is capped at 24 hours.

## Gotchas: lease expiry

**The lease does not renew itself.** Size `ttlMs` to comfortably cover the work done while holding the lock. If the lease expires mid-operation, another acquirer can take the key and run concurrently — the exact overlap the lock exists to prevent. For long or variable-duration work, set a generous `ttlMs` or call `ctx.api.locks.renew` before the lease expires. A renew that answers `reason: "lease_lost"` means the key changed hands — stop and re-acquire.

## Re-taking Your Own Lease

`ctx.locks` names the run as the owner (`ctx.runId`), so a resumed slice, or a second acquire on a key the run already holds, re-takes it at once with a **fresh handle**; the previous handle is fenced (`release` answers `not_holder`, `renew` answers `lease_lost`). It does not interrupt code still running under the old handle.

- The match is the same caller identity, the same kind of caller and the same owner string. A hold made with no owner is never re-entered.
- When a run key coalesces several triggers into one run, pass `owner: ctx.trigger.runKey ?? ctx.runId`. Never a static string: two unrelated runs presenting one would re-enter each other's hold.
- `status.owner` tells a run its own hold from another's.

## CLI

App admins inspect and free locks with `primitive locks`.

| Command | Purpose |
|---|---|
| `primitive locks list [app-id] [--json]` | Every held lock in the app, with an `OWNER` column. |
| `primitive locks status <key> [app-id] [--json]` | The current holder of a key, including its `Owner`. |
| `primitive locks force-release <key> [app-id] [--json]` | Free a key whose lease is live, with no handle; the holder's handle is fenced. A free or expired key answers not held (exit 1). |

```bash
primitive locks list
primitive locks status portfolio-import:01JD7E8V3K9Q
primitive locks force-release portfolio-import:01JD7E8V3K9Q
```

Force-release only a key whose holder has stopped: it does not interrupt work still running under the old handle.

## Rate Limiting

Acquire attempts are capped at **600 per user per hour**, counted against the user the function runs for. Each attempt counts once however long the server waited on it, so a waiting `ctx.locks.acquire` spends one per half-minute. When the limit refuses an attempt, `acquire` keeps waiting within `timeoutMs` and answers `rateLimited: true` if it never wins; a single `ctx.api.locks.tryAcquire` that trips it throws a `429` platform error.

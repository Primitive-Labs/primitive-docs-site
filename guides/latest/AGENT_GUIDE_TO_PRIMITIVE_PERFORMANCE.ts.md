# Performance Guide for Primitive Apps

Audience: agents building apps on the Primitive platform.

How fast a Primitive app feels is decided by where its work runs, far more than by how well any piece of code is tuned. This guide starts with the four principles that decide it, then gives tips for databases and server functions, and ends with how to measure.

## Where the time goes

| Work | What it costs |
|---|---|
| A read or write on a synced database, on the device | Almost nothing. No network is involved. |
| A request from the device to the server | A network round trip, with a fixed overhead however small the request. |
| A platform call inside a server function | A fixed overhead per call, far more than the code around it. |
| Starting a task | A short startup delay, in exchange for durability and retries. |
| A call to a third party or a model | The slowest and least predictable step. |

Do each piece of work in the cheapest row that can do it. The four principles below say how.

## 1. Do it locally

A synced database keeps a full copy of its data on the device. Queries, counts, aggregations and writes against that copy run locally: they are effectively instant and work the same offline. This is the fastest thing on the platform, and it is the default way to build a screen.

```typescript
  const sub = client.db3.subscribe(db3Id);
  await sub.available; // a readable copy is on the device

  // A local read: no network, the same speed offline.
  const open = await Task.query(
    { completed: false },
    { db3: db3Id, sort: { priority: -1 } },
  );

  // A local write: applied to the device's copy at once, synced in the background.
  await new Task({ title: "Write the report" }).save({ db3: db3Id });
```

- **Render every screen you can from a synced database.** Filter, sort, count and aggregate with local queries.
- **Write locally.** A write applies to the device's copy at once and syncs in the background. The screen needs no spinner and no hand-written optimistic update.
- **Re-render from the model's change subscription, not by refetching.** Changes from other devices arrive the same way.
- **Show the screen at `available`.** Wait for `ready` only before a decision that must see the latest data.
- **Keep each synced database small.** The first open on a device downloads all of it. Partition per user, team or project.

Anti-patterns:

- A server function that returns records the device already holds in a synced database.
- A server call to compute a total, a filter or a sort that a local query answers.
- Copying synced records into a separate client-side cache and keeping it current by hand. The local copy is the cache.
- Holding the screen until the server confirms a local write.
- One large synced database for all of an app's data.

Some work cannot be local: data too large for a device, data a user may see only a slice of, secrets, third-party calls. That work goes to a server function, and the next three principles apply. See [Databases](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#the-decision-that-shapes-everything) for choosing between synced and sync off.

## 2. Make few requests, and run them together

Every request from the device pays a fixed overhead before any work happens. A chain of requests is slow however fast each one is, because the overheads add up.

- **One function call per screen.** Write one function that reads everything the screen needs and answers it together, instead of one call per model or per item.

```typescript
  const result = await client.functions.invoke<{
    groups: unknown[];
    accounts: unknown[];
    holdings: unknown[];
  }>("dashboard-bundle", { input: { db3Id } });

  const groups = result.output?.groups ?? [];
  const accounts = result.output?.accounts ?? [];
  const holdings = result.output?.holdings ?? [];
```

- **Run independent requests at the same time.** When no request needs another's answer, start them together.

```typescript
  const [bundle, memberships] = await Promise.all([
    client.functions.invoke("dashboard-bundle", { input: { db3Id } }),
    client.groups.listUserMemberships(userId),
  ]);
```

- **Block first paint only on what the screen renders.** Load what only some screens or actions need in the background, or on first use.
- **Never gate a screen on a third party.** Render the last stored value and refresh it behind the screen.
- **Keep what you fetched.** Hold answers the user will need again in a shared store, and invalidate them with a [channel](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#channels) message from the function that changes the data.
- **Return only the fields the screen shows.**

Anti-patterns:

- A request inside a loop: one call per item.
- Requests awaited one after another when none needs the previous answer.
- Startup that awaits config, memberships and preferences before showing anything.
- Reloading everything when one input changes. Recompute from the data already loaded.
- A request on every keystroke. Debounce it.

## 3. Requests for what a user waits on, tasks for everything else

A server function runs as a request or as a task. A request answers while the caller waits. A task runs in the background, survives failures and retries its steps.

- **Use a request when a person is waiting** and the work is small and bounded.
- **Use a task for background work:** imports, syncs with a third party, scheduled jobs, anything triggered by a data change, anything that grows with the data.
- **Don't worry about task startup time.** A task takes a moment to start. That buys durability and retries, and nobody is watching background work start.
- **Don't make a person watch a task.** When a user action starts one, update the screen from local data right away. The task's result arrives through sync or a channel message.
- **Fetch shared third-party data on a schedule.** When every user needs the same slowly changing value (prices, rates, weather), a scheduled function stores it and screens read the stored value.

Anti-patterns:

- Work that grows with the data inside a request, where it will eventually pass the request's deadline.
- Background work squeezed into a request to avoid a task's startup.
- A screen that blocks until a task completes.

## 4. Inside a function, count the platform calls

Inside a server function, every call to a database or another platform API has a fixed overhead that dwarfs the handler's own code. A function's running time is roughly the number of platform calls it makes in sequence.

1. **Count** the platform calls on the common path.
2. **Remove** the ones you can: a record read twice, a read made only to decide a write.
3. **Merge** the rest: one query for many ids, one batch for many writes.
4. **Run together** what remains, when the calls don't depend on each other.

Anti-patterns:

- An `await` on a platform call inside a loop.
- The same record read by a helper and again by its caller.
- Writing one row per call.
- An unbounded `Promise.all` over a page of rows.

The server function tips below give the technique for each.

## Database tips

- **Add an index only when profiling shows a read needs it.** Every index adds cost to every write, on the server and on each synced device, and most reads are fast without one. To find a read that needs one, rank the app's reads with `primitive analytics db3-queries`, explain the suspect (`--explain` on `primitive db3 records query`), and compare `cost.rowsRead` before and after. An index only server reads need can be registered with `on: "server"`, so devices don't pay for it. See [Indexes](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#indexes).
- **Count and aggregate instead of loading rows** when all you need is a number. This holds on the device and on the server.
- **Name a `limit` on every server query.** Without one a query answers 100 records, and code that ignores `hasMore` silently drops the rest. Page larger reads by cursor.
- **Fetch related records in the same read.** An `include` returns a record's related records with it, and `batchRead` answers several independent reads in one call.
- **Use a bulk load for thousands of records.** Imports and reference datasets go through a [bulk load](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#bulk-loading), not through saves or batches.
- **Read a sync-off database through one function call.** A screen that reads it record by record from the device pays a request per read.

## Server function tips

**Reading**

- **Read each record once per run and pass it down.** Helpers that take an id and re-read the record are the usual source of wasted calls. Have them take the record.
- **Reuse what `authorize` read.** Resolve the rows it loaded with `allow(resolved)` and read them from `ctx.auth` in the handler. See [Access Control](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#the-authorize-method).
- **One query for many ids.** Replace a query per item with one `$in` query and group the rows in memory. A list takes up to 1,000 values; chunk longer lists.

**Writing**

- **Let the write do the checking.** Don't read to decide whether to write. Use `upsertOn` to create or update, `ifNotExists` to create only when absent, and a `condition` to change a record only if it is unchanged.
- **Fill every batch.** A batch costs about the same with one operation as with a full load, so a write set costs its number of batches. A batch spans models: stage all of a phase's writes, then cut by the batch limits (100 operations, 1 MiB).
- **Never fall back to one write per call.** Send independent writes with `{ atomic: false }` and read each operation's result.

**Running calls together**

- **Bound every fan-out.** Use `pMap` from `primitive-functions` to keep a few calls in flight. An unbounded `Promise.all` can spend the invocation's subrequest budget in one line.
- **Keep ordered and conditional writes in sequence.**
- **Await an access check before a billed call.** Run beside a model or integration call, a refused caller has still cost you the call.

**Third parties and models**

- **Use the third party's bulk endpoint.** One integration call per id is the same loop anti-pattern. See [Integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md).
- **Send a model fewer rows.** Store each answer on the row it came from, skip rows already answered, and cap the calls in flight.

**Tasks**

- **Add a `step.do` only where a retry must not repeat what came before.** Every step has a cost. Put bookkeeping in the step of the work beside it, and use one step per unit of retry, not per item.
- **Keep a billed or external call in its own step,** apart from the write that uses its answer. A failed write then retries without calling out again.
- **Key every row a retried step writes** with a stable id or a unique field, so a second attempt rewrites the same rows.
- **Return ids and counts from a step,** not payloads.
- **Expect one trigger run per accepted batch.** A function that writes many batches to a watched database starts many runs. Keep the watcher cheap, skip the changes its own writes produced, and have a run that finds the work already under way return at once.
- **Lock only where two runs can write the same rows** and a `condition` can't catch it. Take one lock per run with `ctx.locks.withLock`, and never poll for a lock in a loop.

## Measuring

Measure before and after every change, and keep a change only if the numbers move. A change made on a guess often does nothing.

- **Run it at least five times.** A function starts cold: its first call after a push or a quiet spell is slower than the calls that follow. One run says little about real-world speed. Record the fastest, slowest and median of at least five runs on realistic data, and judge by the median.
- **Count before you time.** Requests per screen and platform calls per function are stable from run to run and easy to compare.
- **Time the phases of a request in its handler.** Take a timestamp around each platform call and log one line at the end. Every response also carries a `Server-Timing` header with the server's total.
- **Read a task's steps.** `primitive functions runs steps <function-id> <run-id>` lists each step with its duration.
- **Measure database work in rows.** `cost.rowsRead` and `cost.rowsWritten` on a server read or batch show what it cost the database.
- **Pin the call count in a unit test.** Run the handler against a fake database and assert the number of reads and batches it makes. An edit that adds a round trip then fails a test.

## Checklist

For a slow screen:

1. Can it render from a synced database on the device?
2. How many requests block first paint? Can they be one call, or run together?
3. Does it wait on anything it doesn't render: startup work, a third party, a task?

For a slow function:

1. Is someone waiting on it? If not, make it a task.
2. How many platform calls does the common path make, and which can be removed, merged or run together?
3. Is there an `await` on a platform call inside a loop?
4. Has it been measured over at least five runs?

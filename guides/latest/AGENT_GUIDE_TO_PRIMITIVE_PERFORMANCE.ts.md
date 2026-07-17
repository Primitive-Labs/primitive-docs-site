# Performance Guide for Primitive Apps

Audience: agents building apps on the Primitive platform.

Most slow Primitive apps aren't slow because Primitive is slow — they're slow because of patterns that quietly multiply round trips. Agents are particularly susceptible: the default approach to "fetch X for each Y" is to write a loop with an `await` inside, and every such loop becomes an N+1.

This guide catalogs the patterns that reliably cut cold-load time, the anti-patterns they replace, and how to measure both. Each pattern stands alone; the closing checklist orders them by impact for triaging a slow page.

## TL;DR

| Lever | Typical wins |
|---|---|
| Replace N+1 ops with a bulk op | one round trip instead of N |
| Replace several related ops with a `pipeline` op | one round trip instead of many |
| Use bulk third-party endpoints (e.g. `/v7/finance/quote?symbols=`) instead of one call per ID | one HTTP per page instead of N |
| Move third-party API calls to a server-side cron + cache | zero client calls per page |
| Parallelize independent awaits | wall-clock cut by serialization length |
| Defer non-critical work off the cold path (memberships, auth config, prefs init) | one or more sequential round trips removed from first paint |
| Render with stale/imported values immediately, refresh in background | first paint independent of slow upstream |
| Pure-compute over cached source data instead of reloading | next render is ~free |
| `client.databases.subscribe` for cache invalidation | warm navigation between screens can be 0 round trips |

The rest of this guide explains each in detail with the patterns and the anti-patterns they replace.

## Cold path vs. background

The single most useful frame for performance work on Primitive is: **what is on the critical path of first paint?**

If a value is read by the page being rendered, fetching it is on the critical path. Anything else is background work — and shouldn't be allowed to block first paint by accident.

Two recurring reasons "background" work ends up blocking:

1. It's unconditionally awaited inside an init function the page depends on (e.g. `userStore.initialize()` awaiting `getAuthConfig` even though only the login page reads `authConfig`).
2. It's awaited downstream because some other lever reads from it (e.g. caching a database ID in user prefs, then awaiting prefs to read the cached ID — re-serializing the load).

Both look correct in isolation. Both are bugs.

## Pattern 1 — Bundle related queries into a pipeline op

### Anti-pattern

A page needs several pieces of data from the same database, fetched one at a time:

```ts
// 5 sequential round trips to the SAME database
const groups = await db.executeOperation("listGroups");
const accounts = await db.executeOperation("listAccounts");
const holdings = await db.executeOperation("listHoldings");
const targets  = await db.executeOperation("listAllTargets");
const latest   = await db.executeOperation("listLatestSnapshot");
```

Each call has its own request/response overhead. They serialize even though none depend on each other.

### Pattern

Define a single pipeline operation in the database TOML:

```toml
[[operations]]
name = "dashboardBundle"
type = "pipeline"
modelName = "_pipeline"
access = "user.userId == database.metadata.ownerId"
[operations.definition]
return = "all"

[[operations.definition.steps]]
name = "groups"
type = "query"
modelName = "groups"
filter = { ownerId = "$user.userId" }
limit = 50

[[operations.definition.steps]]
name = "accounts"
type = "query"
modelName = "accounts"
filter = { ownerId = "$user.userId" }
limit = 100

[[operations.definition.steps]]
name = "holdings"
type = "query"
modelName = "holdings"
filter = { ownerId = "$user.userId" }
limit = 1000

[[operations.definition.steps]]
name = "targets"
type = "query"
modelName = "targets"
filter = {}
limit = 1000

[[operations.definition.steps]]
name = "latestSnapshot"
type = "query"
modelName = "snapshots"
filter = { ownerId = "$user.userId" }
sort = { createdAt = -1 }
limit = 1
```

Then execute it in one round trip and read each step's `data`:

```typescript
  const bundle = await client.databases.executeOperation(databaseId, "dashboardBundle");
  const groups = bundle.steps?.groups?.data ?? [];
  const accounts = bundle.steps?.accounts?.data ?? [];
  const holdings = bundle.steps?.holdings?.data ?? [];
```

### When to reach for it

Whenever a page reads three or more independent collections from the same database. Always when you're in a `for` loop calling the same op with different params (see Pattern 2).

### Gotchas

- Pipeline steps need an explicit `filter` field, even if empty (`"filter": {}`). Omitting it fails `primitive sync push` with a 400 (`query step requires a "filter" field`) attributed to that operation.
- Each step's `limit` matters — pipelines don't paginate. Per-step `limit` caps at 1000, a pipeline allows at most 10 steps, and step types are `query`/`count`/`aggregate` only. Pick limits that comfortably cover real data without blowing past the reasonable size of a single response.
- Pipelines only span one database. If the page also needs data from a *different* database, that's a second call (e.g. a per-user portfolio DB plus a shared securities-ref DB).

See [Databases — Pipeline operations](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#pipeline--multi-step-read-operations) for the full pipeline reference.

## Pattern 2 — Replace N+1 with a bulk op

### Anti-pattern

A per-item operation called once for every item in a collection:

```ts
for (const group of groups) {
  const targets = await db.executeOperation("listTargetsByGroup", {
    params: { groupId: group.id },
  });
  // ...use targets
}
```

With 5 groups, 5 sequential round trips. Doubles to 10 with 10 groups.

### Pattern

Add a `listAllTargets` op (no `groupId` filter), call it once, group client-side:

```toml
[[operations]]
name = "listAllTargets"
type = "query"
modelName = "targets"
access = "user.userId == database.metadata.ownerId"
[operations.definition]
filter = {}
sort = { groupId = 1 }
limit = 1000
```

Call it once, then group client-side:

```typescript
  const all = await client.databases.executeOperation(databaseId, "listAllTasks");
  const byCategory = new Map<string, TaskAttrs[]>();
  for (const task of all.data as TaskAttrs[]) {
    const key = task.category ?? "uncategorized";
    const list = byCategory.get(key) ?? [];
    list.push(task);
    byCategory.set(key, list);
  }
```

If you can fold the bulk op into a pipeline (Pattern 1), do that instead — same round-trip count, fewer named ops to maintain.

### When to reach for it

Any time you find an `await ...executeOperation(...)` inside a `for` loop. This is the most common N+1 in Primitive apps.

## Pattern 3 — Parallelize independent awaits

### Anti-pattern

Three independent reads awaited one after another:

```ts
const a = await db.executeOperation("opA");
const b = await db.executeOperation("opB");
const c = await db.executeOperation("opC");
```

Three sequential round trips even though none depends on the others.

### Pattern

```typescript
  const [groups, accounts, holdings] = await Promise.all([
    client.databases.executeOperation(databaseId, "listGroups"),
    client.databases.executeOperation(databaseId, "listAccounts"),
    client.databases.executeOperation(databaseId, "listHoldings"),
  ]);
```

One round trip's worth of wall-clock latency.

### When to reach for it

Any sequence of `await` calls where the second doesn't read the first's result. Trivial code change, real wins. Usually catch a few of these per non-trivial page.

### Gotcha

Don't parallelize blindly — if call 2 needs call 1's result (e.g. `getSecuritiesBySymbols(symbols)` where `symbols` comes from the holdings query), they have to stay sequential. Two reads of holdings (once for symbols, once for the page) is worse than one read followed by securities serial.

## Pattern 4 — Bulk-call third-party APIs through integrations

### Anti-pattern

One proxy call per ID — here, 21 calls for 21 symbols:

```ts
// 21 individual proxy calls for 21 symbols
for (const symbol of symbols) {
  await client.integrations.call({
    integrationKey: "yahoo-finance",
    path: `/v8/finance/chart/${symbol}`,
    // ...
  });
}
```

### Pattern

Use the third party's bulk endpoint (most have one) and a single proxy call:

```typescript
  const response = await client.integrations.call({
    integrationKey: "yahoo-finance",
    method: "GET",
    path: "/v7/finance/quote",
    query: { symbols: symbols.join(",") },
  });
```

Update the integration's TOML to allow the bulk path and forward the new query param:

```toml
[requestConfig]
allowedPaths = ["/v7/finance/quote", "..."]
forwardQueryParams = ["symbols", "..."]
```

Cap symbols per request at the provider's per-URL limit. Most apps don't hit it.

### Gotchas

- Read the upstream API's response shape. v8 chart and v7 quote have different field names (`chartPreviousClose` vs. `regularMarketPreviousClose`).
- Even better — see Pattern 8 (move third-party calls server-side).

See [Integrations — request config](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md) for `allowedPaths` and `forwardQueryParams`.

## Pattern 5 — Defer non-critical work off the cold path

The cleanest formulation: **fire-and-forget on init, await on use.**

### Anti-pattern

An init routine that eagerly awaits everything before signalling the app is ready:

```ts
async function initialize() {
  // Always loaded, even on pages that never consult it
  authConfig.value = await client.getAuthConfig();
  // Always loaded, even though only some pages care about memberships
  memberships.value = await client.groups.listUserMemberships(userId);
  // Always awaited before isAuthenticated flips true
  await initializeUserPrefs();
  isAuthenticated.value = true;
}
```

Every screen pays for things only some screens need.

### Pattern

Await only what first paint genuinely requires; kick off the rest in the background and expose a lazy, de-duped accessor each consumer can await on demand:

```ts
async function initialize() {
  await client.me.get();           // Genuinely required
  isAuthenticated.value = true;    // Page can render now

  // Background — pages that need them await whenXReady()
  if (!prefsReadyPromise) {
    prefsReadyPromise = initializeUserPrefs().catch(/* ... */);
  }
  void refreshGroupMemberships();
}

// Lazy: idempotent, de-duped
const loadAuthConfig = async (): Promise<void> => {
  if (authConfig.value !== null) return;
  if (authConfigPromise) return authConfigPromise;
  authConfigPromise = (async () => {
    const config = await client.getAuthConfig();
    authConfig.value = normalize(config);
  })();
  return authConfigPromise;
};

// Awaitable for callers that need it
const whenMembershipsReady = async (): Promise<void> => {
  if (isMembershipsReady.value) return;
  if (membershipsPromise) return membershipsPromise;
  return refreshGroupMemberships();
};
```

Screens that actually need the data call `void store.loadAuthConfig()` as the view appears (no await — the UI updates reactively when it lands), or `await store.whenMembershipsReady()` if they need a definitive value before proceeding (e.g. a navigation guard with a group check).

### Decision rule

- Used by a critical-path render: load eagerly, await it.
- Used by *some* pages, optional UI elsewhere: `void` it on init, lazy `loadX()` action that pages opt into.
- Used only by mutations or specific actions: don't load on init at all. Load on first use.

### Gotcha — the re-block trap

If you cache a value in `userPrefs` (which itself loads via the root doc) and then `await ensurePrefsReady()` before reading the cache, you just re-serialized the same work you were trying to background. Either:

- Use the cached value optimistically and fire a fallback in parallel (see Pattern 7), OR
- Cache somewhere that doesn't need to be loaded first (on-device key/value storage), OR
- Accept that this particular value isn't worth caching here.

## Pattern 6 — Render with stored values, refresh reactively

### Anti-pattern

First paint awaits a third-party round trip before rendering anything:

```ts
async function onMounted() {
  await livePriceStore.start();   // Awaits a third-party round trip
  data.value = await loadDashboard(dbStore);
  isReady.value = true;
}
```

First paint is now gated on a third-party API.

### Pattern

Render from the source data plus whatever values are already cached, then kick off the refresh in the background and recompute reactively when fresh values land:

```ts
async function onMounted() {
  // Load the source data; render with whatever prices are cached now
  // (typically the import-time price, or the previous refresh).
  sourceData = await loadSourceData(dbStore);
  data.value = compute(sourceData, livePriceStore.prices);
  isReady.value = true;

  // Kick off the price refresh in the background. The watch below
  // recomputes when fresh prices arrive.
  void livePriceStore.setSymbols(symbolsFromSource(sourceData));
}

watch(() => livePriceStore.lastFetchedAt, () => {
  if (!sourceData) return;
  data.value = compute(sourceData, livePriceStore.prices);
});
```

First paint stops waiting on the slowest upstream. The user sees something immediately; values quietly refresh when fresh data lands.

### When to reach for it

Any time you call a third-party API on cold load and you have *any* reasonable fallback value (most-recent-import price, last cached value, even just "—" with a spinner indicator).

### Pair with

Pattern 8 — moving the third-party call to a server-side cron means the "cached value" is always available and ≤refresh-interval stale, which closes the gap between this and a fully fresh first paint.

## Pattern 7 — Pure compute over cached source data

### Anti-pattern

A reactive observer that triggers a full re-render reloads *all* the underlying data every time:

```ts
watch(() => livePriceStore.lastFetchedAt, async () => {
  data.value = await loadDashboard(dbStore);  // Re-runs every DB query
});
```

Every 30-min price refresh re-fetches groups, accounts, holdings, targets, and securities. Dozens of round trips for a price update that should just recompute a few percentages.

### Pattern

Split load into:

1. `loadSourceData(dbStore)` — the I/O. Returns a plain struct.
2. `compute(source, prices)` — pure function. No I/O.

Cache the source. Recompute on price update; refetch only when source changed:

```ts
let sourceData: SourceData | null = null;

onMounted(async () => {
  sourceData = await loadSourceData(dbStore);
  data.value = compute(sourceData, livePriceStore.prices);
  isReady.value = true;
});

watch(() => livePriceStore.lastFetchedAt, () => {
  if (!sourceData) return;
  data.value = compute(sourceData, livePriceStore.prices);
});
```

Hold the source in a shared, observable store so the cache survives navigation between screens, and invalidate it via `client.databases.subscribe` when something actually changes upstream:

```ts
export const useSourceStore = defineStore("source", () => {
  const source = ref<SourceData | null>(null);
  const loadedAt = ref<number | null>(null);
  let inFlight: Promise<SourceData> | null = null;

  async function ensureLoaded(): Promise<SourceData> {
    if (source.value) return source.value;
    if (inFlight) return inFlight;
    inFlight = loadSourceData(dbStore).then((s) => {
      source.value = s;
      loadedAt.value = Date.now();
      return s;
    }).finally(() => { inFlight = null; });
    return inFlight;
  }
  function invalidate() { source.value = null; }

  // Wire up DB subscribe once on first use. `subscribe` takes the database id,
  // a server-registered subscription key, and an options object carrying the
  // `onChange` callback (plus any `params` forwarded to the subscription's
  // filter CEL).
  client.databases.subscribe(dbId, "source-changes", { onChange: invalidate });

  return { source, ensureLoaded, invalidate };
});
```

Navigating between screens then becomes ~zero network calls until something actually changes upstream. The DB-subscribe handler also catches changes from other devices / sessions for free.

See [Databases — real-time subscriptions](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#real-time-subscriptions) for the `databases.subscribe` API.

## Pattern 8 — Server-side cron for stale-tolerant third-party data

### When applicable

You're calling the same third-party API for the same set of values on behalf of every user. Examples: stock prices, exchange rates, weather, sports scores. The "right" answer changes slowly enough that minute- or hour-staleness is fine.

### Pattern

1. Add the cached fields to the relevant database row (no schema migration needed if you're using Primitive's schemaless model — the workflow just writes them).
2. Add a workflow that runs on a cron trigger, queries rows whose value is stale (`lastUpdatedAt < now() - refreshInterval`), calls the integration in bulk, and writes back via a bulk-update op.
3. Restrict the bulk-update op's `access` rule to the workflow's service identity (`access = "fromWorkflow('refresh-security-prices')"`).
4. The client-side fetch goes away entirely. The page reads the cached value from the same query it already runs.

### Wins

- Zero client-side third-party calls
- Same value across all users (no inter-user disagreement)
- Survives third-party rate limits (one app-wide budget, not N)
- Removes a whole subsystem of client code (price store, retry, status UI, watchers)

See [Workflows — cron triggers](AGENT_GUIDE_TO_PRIMITIVE_WORKFLOWS.md) and [Databases — `fromWorkflow()` access](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#cel-access-expressions).

## Pattern 9 — Bound how often you talk to your own server

The Primitive client persists auth tokens locally and caches local-first documents on the device. That makes it easy to forget that some things genuinely require a round trip.

### Useful questions before adding a new fetch on a hot path

- Does this value change often, or is it effectively static for this user (e.g. database ID, app config, user profile)?
- Can it be derived from data the screen already loaded?
- If it's per-user but stable, should it live in `userPrefs` (synced across the user's devices) or in on-device storage (per-device, no round trip to read)?
- Is there a single screen that needs it, or is it shared? Shared → a store computed once.

## Anti-patterns to grep for

When auditing an existing Primitive app, these usually find real wins:

| Pattern | What to grep |
|---|---|
| N+1 in a loop | `for.*await.*executeOperation` (regex) |
| Per-symbol third-party call | calls to `client.integrations.call` inside a loop |
| Eager full-reload on every state change | `watch(..., async () => { ... await load... })` |
| Awaiting non-critical init | `await client.getAuthConfig()`, `await listUserMemberships`, `await initializeUserPrefs()` inside `initialize()` / `completeAuthentication()` |
| Cached value behind a re-block | `await ensurePrefsReady(); const cached = getPref(...)` (yes — really) |
| Sequential awaits | three or more consecutive `await db.executeOperation(...)` lines |

## Measuring

Before optimizing, instrument so you can tell whether a change actually helped. The cheapest setup that's worked well:

```ts
// Inside the page that matters most (HomePage / dashboard equivalent)
onMounted(async () => {
  const t0 = performance.now();
  // ...load and render...
  isReady.value = true;
  console.log(`[PERF] first paint ${(performance.now() - t0).toFixed(0)}ms`);
  (window as any).__firstPaintMs = performance.now() - t0;
  (window as any).__dashboardReady = true;
});
```

Then drive the page from outside (e.g., browser automation) and read `window.__firstPaintMs` after each navigation. Run several times — the first cold load is always slower (dev-server warm-up, OS caches, etc.). Use the median of 3-5 runs.

For correctness verification across phases, hash a digest of the rendered data:

```ts
const json = JSON.stringify(normalize(data.value));   // round numbers, sort keys
const hash = await crypto.subtle.digest("SHA-256",
  new TextEncoder().encode(json));
```

If the hash changes between phases, you've broken something. If it doesn't, the change is at least correct on this dataset.

## Closing checklist

For a new Primitive page that feels slow, run through these in order:

1. **Count network requests on cold load** (filtered to your API host). Is it more than 5? Each one is a candidate for elimination.
2. **Identify the critical path** — which calls block first paint? Which fire in parallel?
3. **Bundle related queries** into a pipeline (Pattern 1).
4. **Replace N+1 ops with bulk ops** (Pattern 2).
5. **Parallelize the remaining independent awaits** (Pattern 3).
6. **Move third-party calls server-side** if practical (Pattern 8), otherwise switch to bulk endpoints (Pattern 4).
7. **Render with stale data, refresh reactively** for non-critical freshness (Pattern 6).
8. **Defer non-critical init** (Pattern 5) — but watch for the re-block trap.
9. **Cache source data in a shared store with DB-subscribe invalidation** (Pattern 7) so warm navigation between screens is ~free.
10. **Measure after each change.** Confirm the win is real and the output hash is stable.

# Performance Guide for Primitive Apps

Audience: agents building apps on the Primitive platform.

Most slow Primitive apps aren't slow because Primitive is slow — they're slow because of patterns that quietly multiply round trips. Agents are particularly susceptible: the default approach to "fetch X for each Y" is to write a loop with an `await` inside, and every such loop becomes an N+1.

This guide catalogs the patterns that reliably cut cold-load time, the anti-patterns they replace, and how to measure both. Each pattern stands alone; the closing checklist orders them by impact for triaging a slow page.

## TL;DR

| Lever | Typical wins |
|---|---|
| Return several models from one function call | one round trip instead of one per model |
| Replace N+1 calls with one bulk call — and one bulk query inside it | one round trip instead of N |
| Use bulk third-party endpoints (e.g. `/v7/finance/quote?symbols=`) instead of one call per ID | one HTTP per page instead of N |
| Move third-party API calls to a scheduled function + cache | zero client calls per page |
| Parallelize independent awaits — in app code and inside functions | wall-clock cut by serialization length |
| Defer non-critical work off the cold path (memberships, auth config, prefs init) | one or more sequential round trips removed from first paint |
| Render with stale/imported values immediately, refresh in background | first paint independent of slow upstream |
| Pure-compute over cached source data instead of reloading | next render is ~free |
| A channel message for cache invalidation | warm navigation between screens can be 0 round trips |

The rest of this guide explains each in detail with the patterns and the anti-patterns they replace.

## Cold path vs. background

The single most useful frame for performance work on Primitive is: **what is on the critical path of first paint?**

If a value is read by the page being rendered, fetching it is on the critical path. Anything else is background work — and shouldn't be allowed to block first paint by accident.

Two recurring reasons "background" work ends up blocking:

1. It's unconditionally awaited inside an init function the page depends on (e.g. `userStore.initialize()` awaiting `getAuthConfig` even though only the login page reads `authConfig`).
2. It's awaited downstream because some other lever reads from it (e.g. caching a database ID in user prefs, then awaiting prefs to read the cached ID — re-serializing the load).

Both look correct in isolation. Both are bugs.

## Pattern 1 — Return several models from one function call

### Anti-pattern

A page needs several collections from the same database and makes one function call per collection — `list-groups`, `list-accounts`, `list-holdings`, `list-targets`, `latest-snapshot` — one after another. Each call has its own request/response overhead, and they serialize even though none depends on another.

### Pattern

Write one function that reads every model the page needs, in parallel, and returns them together:

```ts
// primitive/dev/functions/dashboard-bundle/index.ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { databaseId: string }, ctx) => {
  const db = ctx.db(input.databaseId, "portfolio");
  const owner = ctx.user!.userId;
  const [groups, accounts, holdings, targets, latest] = await Promise.all([
    db.model("groups").query({ filter: { ownerId: owner }, options: { limit: 50 } }),
    db.model("accounts").query({ filter: { ownerId: owner }, options: { limit: 100 } }),
    db.model("holdings").query({ filter: { ownerId: owner }, options: { limit: 1000 } }),
    db.model("targets").query({ options: { limit: 1000 } }),
    db.model("snapshots").query({ filter: { ownerId: owner }, options: { sort: { createdAt: -1 }, limit: 1 } }),
  ]);
  return {
    groups: groups.items,
    accounts: accounts.items,
    holdings: holdings.items,
    targets: targets.items,
    latestSnapshot: latest.items[0] ?? null,
  };
});
```

Then call it once:

{{ example: performance/function-bundle }}

### When to reach for it

Whenever a page reads three or more independent collections. Always when you're in a `for` loop calling the same function with different input (see Pattern 2).

### Gotchas

- Each `query` answers one page. Set `limit` to comfortably cover real data, and page with `nextCursor` inside the function when a model can outgrow it — the client still pays one round trip.
- A function can open several databases (`ctx.db(otherId, "securities-ref")`) and read them in the same `Promise.all`, so a per-user database plus a shared reference database is still one call.
- A read that many callers repeat can be registered with `defineQuery` and `cache: { ttlMs }` — honored only when the run touched nothing but its declared `models` and wrote nothing, and capped at one minute. See [Registered queries](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#registered-queries).
- The function's response counts against its output ceiling; return the fields the page renders, not whole records it ignores (`options.projection`).

## Pattern 2 — Replace N+1 with a bulk query

### Anti-pattern

A per-item read for every item in a collection — either the client invoking a per-group function in a loop, or a function querying once per group:

```ts
// Inside a function: 5 groups → 5 sequential queries
for (const group of groups.items) {
  const targets = await db.model("targets").query({ filter: { groupId: group.id } });
  // ...use targets
}
```

With 5 groups, 5 sequential round trips. Doubles to 10 with 10 groups.

### Pattern

One query for every row, grouped in memory:

```ts
const ids = groups.items.map((g) => g.id);
const targets = await db.model("targets").query({
  filter: { groupId: { $in: ids } },           // up to 1,000 values per list
  options: { sort: { groupId: 1 }, limit: 1000 },
});
const byGroup = new Map<string, typeof targets.items>();
for (const t of targets.items) {
  byGroup.set(t.groupId, [...(byGroup.get(t.groupId) ?? []), t]);
}
```

From the client, the same shape is one call to a function that returns every row, grouped on the device:

{{ example: performance/bulk-then-group }}

If the rows belong in a page bundle anyway, fold the query into Pattern 1's function instead.

### When to reach for it

Any `await` inside a `for` loop — `client.functions.invoke(...)` in app code, or `.query(...)` in a function. This is the most common N+1 in Primitive apps. When per-item calls are genuinely unavoidable inside a function (a third-party call per item with no bulk endpoint), use `pMap` from `primitive-functions`, which bounds the concurrency (`PMAP_DEFAULT_CONCURRENCY` by default) instead of spending the invocation's subrequest budget in one line.

## Pattern 3 — Parallelize independent awaits

### Anti-pattern

Independent reads awaited one after another — three function calls in app code, or three queries in a function — even though none depends on the others.

### Pattern

In app code:

{{ example: performance/parallel-calls }}

Inside a function, the same `Promise.all` over the handle (Pattern 1).

One round trip's worth of wall-clock latency.

### When to reach for it

Any sequence of `await` calls where the second doesn't read the first's result. Trivial code change, real wins. Usually catch a few of these per non-trivial page.

### Gotcha

Don't parallelize blindly — if call 2 needs call 1's result (e.g. securities looked up by the symbols the holdings query returned), they have to stay sequential. Two reads of holdings (once for symbols, once for the page) is worse than one read followed by the dependent one.

## Pattern 4 — Bulk-call third-party APIs through integrations

### Anti-pattern

One integration call per ID from a function — here, 21 calls for 21 symbols:

```ts
for (const symbol of symbols) {
  await ctx.integrations.call("yahoo-finance", { method: "GET", path: `/v8/finance/chart/${symbol}` });
}
```

### Pattern

Use the third party's bulk endpoint (most have one) and a single call:

```ts
const quotes = await ctx.integrations.call("yahoo-finance", {
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

The function needs the integration's capability line (`capabilities = ["integration:yahoo-finance"]`). Cap symbols per request at the provider's per-URL limit. Most apps don't hit it.

### Gotchas

- Read the upstream API's response shape. v8 chart and v7 quote have different field names (`chartPreviousClose` vs. `regularMarketPreviousClose`).
- Even better — see Pattern 8 (fetch on a schedule, not per page).

See [Integrations — request config](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md) for `allowedPaths` and `forwardQueryParams`.

## Pattern 5 — Defer non-critical work off the cold path

The cleanest formulation: **fire-and-forget on init, await on use.**

### Anti-pattern

An init routine that eagerly awaits everything before signalling the app is ready:

{{#lang ts}}
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
{{/lang}}
{{#lang swift}}
```swift
func initialize() async throws {
  // Always loaded, even on screens that never consult it
  authConfig = try await client.auth.getAuthConfig()
  // Always loaded, even though only some screens care about memberships
  memberships = try await client.groups.listUserMemberships(userId: userId)
  // Always awaited before isAuthenticated flips true
  try await initializeUserPrefs()
  isAuthenticated = true
}
```
{{/lang}}

Every screen pays for things only some screens need.

### Pattern

Await only what first paint genuinely requires; kick off the rest in the background and expose a lazy, de-duped accessor each consumer can await on demand:

{{#lang ts}}
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
{{/lang}}
{{#lang swift}}
```swift
func initialize() async throws {
  try await client.me.get()        // Genuinely required
  isAuthenticated = true           // View can render now

  // Background — callers that need them await whenXReady()
  if prefsReadyTask == nil {
    prefsReadyTask = Task { try? await initializeUserPrefs() }
  }
  Task { await refreshGroupMemberships() }
}

// Lazy: idempotent, de-duped
func loadAuthConfig() async {
  if authConfig != nil { return }
  // getAuthConfig() is `throws`; catch inside the Task so the cached task —
  // and this loader — stay non-throwing (callers fire it without `try`).
  let task = authConfigTask ?? Task {
    (try? await client.auth.getAuthConfig()).map(normalize)
  }
  authConfigTask = task
  authConfig = await task.value
}

// Awaitable for callers that need it
func whenMembershipsReady() async {
  if isMembershipsReady { return }
  if let task = membershipsTask { return await task.value }
  await refreshGroupMemberships()
}
```

Screens that actually need the data kick off `loadAuthConfig()` in a detached `Task` as the view appears (the UI updates reactively when it lands), or `await whenMembershipsReady()` if they need a definitive value before proceeding (e.g. a navigation guard with a group check).
{{/lang}}

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

{{#lang ts}}
```ts
async function onMounted() {
  await livePriceStore.start();   // Awaits a third-party round trip
  data.value = await loadDashboard(dbStore);
  isReady.value = true;
}
```
{{/lang}}
{{#lang swift}}
```swift
func onAppear() async throws {
  try await livePriceStore.start()   // Awaits a third-party round trip
  data = try await loadDashboard(dbStore)
  isReady = true
}
```
{{/lang}}

First paint is now gated on a third-party API.

### Pattern

Render from the source data plus whatever values are already cached, then kick off the refresh in the background and recompute reactively when fresh values land:

{{#lang ts}}
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
{{/lang}}
{{#lang swift}}
```swift
func onAppear() async throws {
  // Load the source data; render with whatever prices are cached now
  // (typically the import-time price, or the previous refresh).
  sourceData = try await loadSourceData(dbStore)
  data = compute(sourceData, livePriceStore.prices)
  isReady = true

  // Kick off the price refresh in the background; the observer below
  // recomputes when fresh prices arrive.
  Task { await livePriceStore.setSymbols(symbolsFromSource(sourceData)) }
}

// React to the published lastFetchedAt on the price store
func onPricesChanged() {
  guard let sourceData else { return }
  data = compute(sourceData, livePriceStore.prices)
}
```
{{/lang}}

First paint stops waiting on the slowest upstream. The user sees something immediately; values quietly refresh when fresh data lands.

### When to reach for it

Any time you call a third-party API on cold load and you have *any* reasonable fallback value (most-recent-import price, last cached value, even just "—" with a spinner indicator).

### Pair with

Pattern 8 — moving the third-party call to a scheduled function means the "cached value" is always available and ≤refresh-interval stale, which closes the gap between this and a fully fresh first paint.

## Pattern 7 — Pure compute over cached source data

### Anti-pattern

A reactive observer that triggers a full re-render reloads *all* the underlying data every time:

{{#lang ts}}
```ts
watch(() => livePriceStore.lastFetchedAt, async () => {
  data.value = await loadDashboard(dbStore);  // Re-runs every DB query
});
```
{{/lang}}
{{#lang swift}}
```swift
func onPricesChanged() async throws {
  data = try await loadDashboard(dbStore)  // Re-runs every DB query
}
```
{{/lang}}

Every 30-min price refresh re-fetches groups, accounts, holdings, targets, and securities. Dozens of round trips for a price update that should just recompute a few percentages.

### Pattern

Split load into:

1. `loadSourceData(dbStore)` — the I/O. Returns a plain struct.
2. `compute(source, prices)` — pure function. No I/O.

Cache the source. Recompute on price update; refetch only when source changed:

{{#lang ts}}
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
{{/lang}}
{{#lang swift}}
```swift
var sourceData: SourceData?

func onAppear() async throws {
  sourceData = try await loadSourceData(dbStore)
  data = compute(sourceData!, livePriceStore.prices)
  isReady = true
}

func onPricesChanged() {
  guard let sourceData else { return }
  data = compute(sourceData, livePriceStore.prices)
}
```
{{/lang}}

Hold the source in a shared, observable store so the cache survives navigation between screens, and invalidate it when a channel message says something actually changed upstream. The functions that write the source data publish to a channel (`ctx.channels.publish("source:<userId>", …)`); the store subscribes with a grant from an authorizing function:

{{#lang ts}}
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

  // Join the channel once on first use: an authorizing function mints the
  // grant, the socket presents it, and any publish to it invalidates.
  async function observe(channel: string) {
    const res = await client.functions.invoke<{ grant: string }>("source-channel");
    if (res.status !== "completed" || !res.output) return;
    await client.subscribeToChannel(channel, res.output.grant);
    client.on("channelMessage", (event) => {
      if (event.channel === channel) invalidate();
    });
  }

  return { source, ensureLoaded, invalidate, observe };
});
```
{{/lang}}
{{#lang swift}}
```swift
@MainActor
final class SourceStore: ObservableObject {
  @Published private(set) var source: SourceData?
  private var loadedAt: Date?
  private var inFlight: Task<SourceData, Error>?
  private var listener: Task<Void, Never>?

  func ensureLoaded() async throws -> SourceData {
    if let source { return source }
    if let inFlight { return try await inFlight.value }
    let task = Task { try await loadSourceData(dbStore) }
    inFlight = task
    defer { inFlight = nil }
    let s = try await task.value
    source = s
    loadedAt = Date()
    return s
  }
  func invalidate() { source = nil }

  // Join the channel once on first use: an authorizing function mints the
  // grant, the socket presents it, and any publish to it invalidates.
  func observe(channel: String) async throws {
    struct Grant: Decodable, Sendable { let grant: String }
    let res: FunctionResult<Grant> = try await client.functions.invoke("source-channel", input: nil as JSONValue?)
    guard let grant = res.output?.grant else { return }
    _ = try await client.subscribeToChannel(channel, grant: grant)
    listener = Task { [weak self] in
      for await event in client.stream(for: ChannelMessageEvent.self) where event.channel == channel {
        self?.invalidate()
      }
    }
  }
}
```
{{/lang}}

Navigating between screens then becomes ~zero network calls until something actually changes upstream. Because the writing function publishes whoever made the change, the message also catches changes from other devices / sessions for free. A channel grant expires (300 s by default, 900 s at most) — renew it ahead of `expiresAt`.

See [Channels](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#channels) for authorizing, subscribing, publishing and renewal.

## Pattern 8 — A scheduled function for stale-tolerant third-party data

### When applicable

You're calling the same third-party API for the same set of values on behalf of every user. Examples: stock prices, exchange rates, weather, sports scores. The "right" answer changes slowly enough that minute- or hour-staleness is fine.

### Pattern

1. Add the cached fields to the relevant database rows (no schema change needed — the database is schemaless; the function just writes them).
2. Add a function with a cron trigger that queries rows whose value is stale, calls the integration in bulk (Pattern 4), and writes back with one `batch`:

```toml
# primitive/dev/functions/refresh-security-prices.toml
[function]
key = "refresh-security-prices"
entry = "functions/refresh-security-prices/index.ts"
access = "hasRole('admin')"                    # members cannot run it by hand
capabilities = ["integration:yahoo-finance"]

[[function.triggers.cron]]
name = "every-15-minutes"
cron = "*/15 * * * *"
```

3. The client-side fetch goes away entirely. The page reads the cached value from the same function call it already makes.

Every cron fire starts a task run (`primitive functions runs refresh-security-prices` lists them); see [Triggers](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#triggers).

### Wins

- Zero client-side third-party calls
- Same value across all users (no inter-user disagreement)
- Survives third-party rate limits (one app-wide budget, not N)
- Removes a whole subsystem of client code (price store, retry, status UI, watchers)

## Pattern 9 — Bound how often you talk to your own server

The Primitive client persists auth tokens locally and caches local-first documents on the device. That makes it easy to forget that some things genuinely require a round trip.

### Useful questions before adding a new fetch on a hot path

- Does this value change often, or is it effectively static for this user (e.g. database ID, app config, user profile)?
- Can it be derived from data the screen already loaded?
- If it's per-user but stable, should it live in `userPrefs` (synced across the user's devices) or in on-device storage (per-device, no round trip to read)?
- Is there a single screen that needs it, or is it shared? Shared → a store computed once.

## Anti-patterns to grep for

When auditing an existing Primitive app, these usually find real wins:

{{#lang ts}}
| Pattern | What to grep |
|---|---|
| N+1 in a loop | `for.*await.*functions\.invoke` (regex) in app code; `for.*await.*\.query\(` in function code |
| Per-symbol third-party call | `ctx.integrations.call` inside a loop in function code |
| Eager full-reload on every state change | `watch(..., async () => { ... await load... })` |
| Awaiting non-critical init | `await client.getAuthConfig()`, `await listUserMemberships`, `await initializeUserPrefs()` inside `initialize()` / `completeAuthentication()` |
| Cached value behind a re-block | `await ensurePrefsReady(); const cached = getPref(...)` (yes — really) |
| Sequential awaits | three or more consecutive `await client.functions.invoke(...)` lines, or `await ….query(...)` lines in a function |
{{/lang}}
{{#lang swift}}
| Pattern | What to grep |
|---|---|
| N+1 in a loop | `for .* in` followed by `try await .*functions.invoke` |
| Per-symbol third-party call | `ctx.integrations.call` inside a loop in function code |
| Eager full-reload on every state change | a change observer whose body does `try await load…` |
| Awaiting non-critical init | `try await client.auth.getAuthConfig()`, `try await listUserMemberships`, `try await initializeUserPrefs()` inside `initialize()` / `completeAuthentication()` |
| Cached value behind a re-block | `await ensurePrefsReady(); let cached = getPref(...)` (yes — really) |
| Sequential awaits | three or more consecutive `try await client.functions.invoke(...)` lines |
{{/lang}}

## Measuring

Before optimizing, instrument so you can tell whether a change actually helped. The cheapest setup that's worked well:

{{#lang ts}}
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
{{/lang}}
{{#lang swift}}
```swift
// Inside the view that matters most (HomeView / dashboard equivalent)
func onAppear() async throws {
  let t0 = Date()
  // ...load and render...
  isReady = true
  let ms = Date().timeIntervalSince(t0) * 1000
  print("[PERF] first paint \(Int(ms))ms")
}
```

Capture the first-paint timing across several launches — the first cold load is always slower (OS caches still warming, etc.). Use the median of 3-5 runs.

For correctness verification across phases, hash a digest of the rendered data:

```swift
let json = try JSONEncoder().encode(normalize(data))   // round numbers, sort keys
let hash = SHA256.hash(data: json)
```
{{/lang}}

If the hash changes between phases, you've broken something. If it doesn't, the change is at least correct on this dataset.

## Closing checklist

For a new Primitive page that feels slow, run through these in order:

1. **Count network requests on cold load** (filtered to your API host). Is it more than 5? Each one is a candidate for elimination.
2. **Identify the critical path** — which calls block first paint? Which fire in parallel?
3. **Bundle related reads** into one function call (Pattern 1).
4. **Replace N+1 calls with a bulk query** (Pattern 2).
5. **Parallelize the remaining independent awaits** (Pattern 3).
6. **Move third-party calls to a scheduled function** if practical (Pattern 8), otherwise switch to bulk endpoints (Pattern 4).
7. **Render with stale data, refresh reactively** for non-critical freshness (Pattern 6).
8. **Defer non-critical init** (Pattern 5) — but watch for the re-block trap.
9. **Cache source data in a shared store with channel invalidation** (Pattern 7) so warm navigation between screens is ~free.
10. **Measure after each change.** Confirm the win is real and the output hash is stable.

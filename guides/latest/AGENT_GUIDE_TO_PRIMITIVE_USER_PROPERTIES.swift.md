# Agent Guide to Primitive User Properties

Typed, app-declared fields on the platform user: `userId`/`email`/`name`/`avatarUrl`/`role`/`status` plus whatever your `user-properties.toml` adds. Declare once, read and write from server functions and the CLI, and optionally expose a field to clients.

## Declare the schema

```toml
# primitive/dev/user-properties.toml
[fields.plan]
type = "string"
enum = ["free", "pro", "team"]
default = "free"
client = "self"

[fields.theme]
type = "string"
default = "system"
client = "self-write"

[fields.stripeCustomerId]
type = "string"
unique = true
```

```bash
primitive config create user-properties   # scaffolds the file (no key — one per app)
primitive config push --only user-properties
primitive user-properties get             # active version, fields, unique constraints
```

Same field grammar as a [database type](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#types), no `[models.*]` level, plus `client` (see [The client table](#the-client-table)). Never on a property: `required` (absence is the unset state), `auto_assign`, `auto_stamp`, `vector`. A unique field never carries a `default` and a push refusing it a writable `client` (`self-write`/`self-write-members`). A field's **name, type and uniqueness are fixed at declaration** — changing any of them is refused at push, naming the field; declare a new field and copy the values into it with a [walk](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#walking-every-user) instead. Removing a field keeps stored values but stops serving them; redeclaring the same name with its original type picks them back up.

Caps (per app, raise with `primitive users index limits --max-fields/--max-unique/--max-indexed/--max-bytes`): 100 fields, 10 unique fields+constraints combined, 10 indexed fields, 16 KB stored per user.

## Read and write from a function

```ts
const member = await ctx.users.get(userId);          // null if not a member
member?.properties.plan;

await ctx.users.setProperties(userId, { plan: "pro", title: null });  // merge-patch; null clears to default
await ctx.users.addToSet(userId, "cohorts", "beta-2026");
await ctx.users.removeFromSet(userId, "cohorts", "beta-2026");

// Only write if the stored row still matches — same grammar a records query filter uses,
// over properties.* and userId/email/name/avatarUrl (role/status are refused: re-read membership instead).
await ctx.users.setProperties(userId, { plan: "pro" }, { condition: { "properties.plan": "free" } });
```

Two calls patching different fields of the same user both land, concurrently. A refused write throws `PrimitivePlatformError`; branch on its `errorCode`:

| Code | When |
|---|---|
| `USER_PROPERTY_UNKNOWN` | Name not declared |
| `USER_PROPERTY_INVALID` | Wrong type/`enum`/`max_length`/`max_count`, or `maxBytes` exceeded |
| `USER_PROPERTY_UNIQUE_CONFLICT` | Another user holds this value already |
| `USER_PROPERTY_CONDITION_FAILED` | `condition` false, or the row kept moving under repeated attempts |
| `USER_PROPERTY_NOT_CLIENT_WRITABLE` | A client session named a field not `self-write`/`self-write-members` |

`auth.userProperties()` and `auth.users.get`/`getBy` read the same way inside `authorize` — see the [Access Control guide](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#auth-context-read-only).

## Lookup by a unique value

```ts
const byField = await ctx.users.getBy("stripeCustomerId", "cus_123");
const byConstraint = await ctx.users.getBy("org_employeeNo", { orgId: "org-1", employeeNo: "E-42" }); // or [orgId, employeeNo] in declared order
```

`null` when nobody holds it; `409 USER_PROPERTY_UNIQUE_CONFLICT` (naming the field/constraint, never the holder) when a write collides with a live claim. CLI: `primitive users get-by <name> <value...>`.

**Watch out for claim timing:** a value's previous holder may have moved on (changed the value, left the app) without their old claim being released yet — that claim stays for up to ten minutes before anyone else can collide with it, so two users racing the same value in that window are still refused correctly. A collision past the ten minutes releases and takes the claim over right then, not before. A per-user background check also files a missing claim for each unique value a user holds, and logs (not refuses) a conflict when another claim already names the value. Removing `unique` from a field — or a path that was unique and no longer is — releases every one of its claims in one pass.

## The client table

| `client` | A client reads it | A client writes it (`me.update`) |
|---|---|---|
| `none` (default) | nobody | no |
| `self` | own row only | no |
| `self-write` | own row only | own row only |
| `members` | any user's row | no |
| `self-write-members` | any user's row | own row only |

Applies to every app-API route that returns a user (`me.get`/`update`, `users.getBasic`, `getProfiles`, `lookup`), app-admin and console-admin tokens included — `client` governs every session caller the same way. A function dispatch always sees every declared property; `client` doesn't apply there.

```swift
  let me = try await client.me.get()
  print(me?.properties["plan"]?.stringValue ?? "none")

  let other = try await client.users.getBasic(userId: userId)
  print(other.properties["plan"]?.stringValue ?? "none") // only if `plan`'s client is members/self-write-members

  let found = try await client.users.lookup(email: "ada@example.com")
  if found.exists, let user = found.user {
    print(user.properties)
  }
```

```swift
  let updated = try await client.me.update(
    params: UpdateMeParams(properties: ["theme": "dark", "bio": nil])
  )
```

Writing a field whose `client` isn't `self-write`/`self-write-members` refuses the **whole** call, `403 USER_PROPERTY_NOT_CLIENT_WRITABLE` — nothing in the request is written, not even the fields that would have been fine.


## Invitations

```bash
primitive users invite ada@example.com --initial-properties '{"plan":"pro"}'
primitive users create ada@example.com --initial-properties '{"plan":"pro"}'
```

```ts
await ctx.api.invitations.create({ body: { email, initialProperties: { plan: "pro" } } }); // functions only
const invited = await ctx.invitations.getBy("stripeCustomerId", "cus_123"); // pending only; null once accepted
```

`initialProperties` is admin-API/CLI/function-only — a client session sending it is refused `USER_PROPERTY_NOT_CLIENT_WRITABLE`. At acceptance, only properties the user has not already set are filled; a unique value someone else took meanwhile is dropped (logged `USER_PROPERTY_UNIQUE_CONFLICT`), not a failed signup. `primitive users create` fills the same way when it attaches an existing `User`.

## Reporting which user categories hold rows

For an app that keeps per-user values in resource metadata categories, `primitive metadata report user` (beta) starts a report over the app's current members and polls it to a table of each user category still holding rows, its exact row count, and whether its config still exists. The report is admin-only (the admin API's `apps/{appId}/users/metadata-report`, `write` to start and `read` to read).

```bash
primitive metadata report user --json             # { status, report: { members, membersWithRows, rows, categories: [{ category, rows, configured }] }, note }
primitive metadata report user --no-wait          # prints the run id
primitive metadata report user --run <run-id>     # read that run
```

- Rows held by users who are no longer members of the app are not counted.
- `configured: false` marks a category whose config was deleted: its rows exist but are unreachable until the config is pushed again.
- To move a category's values into typed properties, declare the fields and copy the values with a [walk](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#walking-every-user).
- Exit codes: `0` with a report, `1` for a failed run or a report whose totals could not be read back whole (`reportUnavailable` names why), `124` when `--timeout` (default 600 s) passes first.

## Consistency contract

| Reads | Freshness |
|---|---|
| `ctx.users.get`/`getBy`, a write's own `condition`, `me.get`/`update`, `users.getBasic`/`getProfiles`/`lookup` | Live — current the instant the call lands |
| `ctx.users.query`/`count`/`aggregate`, the admin search, `primitive users query`/`count`/`aggregate` | A user index, seconds behind a write |

See the [Server Functions guide](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#querying-users) for `ctx.users.query`'s filter grammar, paging and scan signals.

## Existing apps

An app whose user index has not been built yet starts with no index: `primitive users index status` answers a `null` generation and no shards. The index fills itself in the background — the first property write opens it holding only that one user, and revalidation adds the rest over time; `primitive users index rebuild` (after the step-up) builds a complete index now — so a non-`null` generation doesn't by itself mean the index is complete. Until it is, `query`/`count`/`aggregate` can answer an empty or a partial page, count or aggregate rather than erroring — a real filter can silently look like it matches none or only some of the real members. A [walk](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#walking-every-user) is stricter: `forEachBatch` refuses to start with `409 USER_INDEX_NOT_READY`.

```bash
primitive users index status            # Generation: none yet → no index
primitive login --step-up --scope admin # a plain login lacks the admin scope; run this first
primitive users index rebuild           # builds a complete index now, needs the admin scope
primitive users index reconcile --wait  # checks every member against the live record
```

See [Admin access](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md#admin-access-scopes-step-ups-and-ci-sessions) for the step-up prompt.

## Gotchas

- **A patch and null are different things.** `setProperties`/`me.update` merge: an omitted field is untouched, an explicit `null` clears it back to its default. Don't send `{}` meaning "no change" if you actually meant to clear something — name it with `null`.
- **A unique field's write refusal is a push-time rule, not a runtime check.** You cannot declare `unique = true` with `client = "self-write"` in the first place; the push itself is refused.
- **Changing a field's type or uniqueness isn't an edit.** Declare a new field under a new name and copy the values with a [walk](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#walking-every-user); there's no in-place change for either.
- **A condition can't see membership.** `role`, `status`, `addedAt` and friends live on the membership row, not `properties`; read them in the function instead of putting them in `condition`.
- **`getBy`/`get` are not what `query` reads.** A property you just wrote is visible immediately to `get`/`getBy`, but `query`/`count`/`aggregate` can lag by seconds — don't poll `query` right after a write expecting it there instantly.

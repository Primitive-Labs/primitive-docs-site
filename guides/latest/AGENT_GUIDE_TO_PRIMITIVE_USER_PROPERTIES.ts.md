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

Same field grammar as a [database type](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#types), no `[models.*]` level, plus `client` (see [The client table](#the-client-table)). Never on a property: `required` (absence is the unset state), `auto_assign`, `auto_stamp`, `vector`. A unique field never carries a `default` and a push refusing it a writable `client` (`self-write`/`self-write-members`). A field's **name, type and uniqueness are fixed at declaration** — changing any of them is refused at push, naming the field; declare a new field and [copy the values into it](#copy-one-property-into-a-new-one) instead. Removing a field keeps stored values but stops serving them; redeclaring the same name with its original type picks them back up.

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

```typescript
  const me = await client.me.get();
  console.log(me?.properties.plan);

  const other = await client.users.getBasic(userId);
  console.log(other.properties.plan); // only if `plan`'s client is members/self-write-members

  const { exists, user } = await client.users.lookup("ada@example.com");
  if (exists && user) console.log(user.properties);
```

```typescript
  const updated = await client.me.update({
    properties: { theme: "dark", bio: null },
  });
```

Writing a field whose `client` isn't `self-write`/`self-write-members` refuses the **whole** call, `403 USER_PROPERTY_NOT_CLIENT_WRITABLE` — nothing in the request is written, not even the fields that would have been fine.

`primitive functions codegen` writes `ClientUserProperties` (client-readable subset) and `UserProperties` (every field) into `user-properties.generated.ts` for your own client code, with a `UserProperties<Field>Values` (and `ClientUserProperties<Field>Values`) `as const` list for each enum field; cast to these instead of treating `properties` as a bare `Record<string, unknown>`.

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

## Copy a metadata category into properties

Declare and push the target fields, add a batch function (`[function]` with `key = "copy-user-metadata"`, `entry = "functions/copy-user-metadata.ts"`), and run it as a [walk](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#walking-every-user). `primitive metadata report user` lists the categories holding rows. The core of the function, per user in the batch:

```ts
// row: the user's category data, from ctx.api.resourceMetadata.getBatch (50 users per call)
const patch: Record<string, unknown> = {};
for (const [key, property] of Object.entries(input.fields)) {
  if (Object.hasOwn(row, key)) patch[property] = valueFor(key, row[key]); // valueFor returns raw unchanged
}
if (Object.keys(patch).length === 0) skipped++;
else {
  try {
    await ctx.users.setProperties(userId, patch);
    copied++;
  } catch (error) {
    const code = (error as PrimitivePlatformError).errorCode;
    if (code === "USER_PROPERTY_UNIQUE_CONFLICT" || code === "USER_PROPERTY_INVALID") refused.push({ userId, code });
    else throw code ? new Error(`${code}: ${(error as Error).message}`) : error; // fails the batch
  }
}
// the batch returns { copied, skipped, refused }
```

```bash
primitive users walk start --function copy-user-metadata --key copy-billing-1 --input '{"category":"billing","fields":{"customerId":"stripeCustomerId","plan":"plan"}}' --wait
```

- `authorize: allowCallers({ usersBatch: true })` — only a walk runs it, never a client with its own list of users.
- The walk's `input` is `{ category, fields: { <metadata key>: <property name> } }`. Throw at once when it's missing or malformed, so the walk fails with nothing written. From a function, start it with `ctx.users.forEachBatch({ function: "copy-user-metadata", key, input })`.
- A batch of up to 100 users costs one `getBatch` per 50 users plus one write per user with a row. A per-category answer with `ok: false` (for example, a deleted config) must throw `<code>: <message>`. Treating it as "no row" copies nothing silently.
- A unique conflict or a value that doesn't fit is one user's data problem: record it under `refused` and carry on. Any other error (for example `USER_PROPERTY_UNKNOWN`, an undeclared target) fails the batch, so the walk retries it and then fails `USER_WALK_BATCHES_FAILED`. Put the code in the thrown message, because the walk records only the message.
- A completed walk copied everyone not under `refused`. Sum the `refused` lists from the batch runs (`primitive functions runs <function-id> --json`) before treating the copy as done.
- Every write is a merge-patch of the row's current values, so a rerun under a new `key` is safe and picks up changed rows.

### Copy one property into a new one

To change a field's type or uniqueness, declare the new field and run the same function with two changes. Read the source from the user instead of the category: `const row = (await ctx.users.get(userId))?.properties ?? {}`. And convert in `valueFor`, because a field keeps its type and a copy without conversion refuses every user. For example, use `Number(raw)` for text into a number when `Number.isFinite` holds for the result (a non-finite number would be written as `null` and clear the field), or `[raw]` for a scalar into a stringset. Return what you can't convert unchanged, so the write refuses it into `refused`. A unique target lists each conflicting user under `refused`; the walk still completes.

## Consistency contract

| Reads | Freshness |
|---|---|
| `ctx.users.get`/`getBy`, a write's own `condition`, `me.get`/`update`, `users.getBasic`/`getProfiles`/`lookup` | Live — current the instant the call lands |
| `ctx.users.query`/`count`/`aggregate`, the admin search, `primitive users query`/`count`/`aggregate` | A user index, seconds behind a write |

See the [Server Functions guide](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#querying-users) for `ctx.users.query`'s filter grammar, paging and scan signals.

## Gotchas

- **A patch and null are different things.** `setProperties`/`me.update` merge: an omitted field is untouched, an explicit `null` clears it back to its default. Don't send `{}` meaning "no change" if you actually meant to clear something — name it with `null`.
- **A unique field's write refusal is a push-time rule, not a runtime check.** You cannot declare `unique = true` with `client = "self-write"` in the first place; the push itself is refused.
- **Changing a field's type or uniqueness isn't an edit.** Declare a new field under a new name and [copy the values with a walk](#copy-one-property-into-a-new-one); there's no in-place change for either.
- **A condition can't see membership.** `role`, `status`, `addedAt` and friends live on the membership row, not `properties`; read them in the function instead of putting them in `condition`.
- **`getBy`/`get` are not what `query` reads.** A property you just wrote is visible immediately to `get`/`getBy`, but `query`/`count`/`aggregate` can lag by seconds — don't poll `query` right after a write expecting it there instantly.

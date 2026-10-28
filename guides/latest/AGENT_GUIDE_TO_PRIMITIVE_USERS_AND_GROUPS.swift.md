# Users, Groups, and Access Control in Primitive

Primitive manages user identity and profiles. Use groups to model teams, roles, and relationships, then check membership in a function's `authorize`, a rule set, or a metadata rule.

## Core operations

Start with these client operations.

### Look up users

```swift
  // One user's basic profile
  let user = try await client.users.getBasic(userId: userId)

  // Batch-fetch several profiles in one round-trip
  let profiles = try await client.users.getProfiles(userIds: [userId, "user-456"])

  // Find a user by email
  let found = try await client.users.lookup(email: "alice@example.com")

  // The current signed-in user
  let me = try await client.me.get()
```

Per-user lookups use `client.users.getBasic(id)`, `client.users.getProfiles([ids])`, and `client.users.lookup(email)`; the current user lives on `client.me`. Enumerate all users via the CLI (`primitive users list`) or the admin REST surface.

### Manage group membership

```swift
  // Add a member by email (recommended for user-facing flows)
  let result = try await client.groups.addMember(
    groupId: "engineering",
    params: .email("alice@example.com")
  )

  // ...or by user id (internal / programmatic)
  _ = try await client.groups.addMember(
    groupId: "engineering",
    params: .userId("user-456")
  )

  // List a group's members
  let members = try await client.groups.listMembers(groupId: "engineering")

  // List the groups a user belongs to
  let memberships = try await client.groups.listUserMemberships(userId: userId)
```

### Share a database with a group

```swift
  let permissions = client.db3.permissions(db3Id)
  _ = try await permissions.grantGroup(
    GrantGroupPermissionParams(groupId: "engineering-team", permission: "read-write")
  )

  // Take the group's grant away
  _ = try await permissions.revokeGroup("engineering-team")
```

## Core Concept: Built-in User Model

Use the platform user as the source of truth for identity. Store app-specific fields separately and reference `userId`.

### What the platform provides

`client.users.getBasic(userId)` returns a `BasicUserInfo`:

| Field | Type | Description |
|-------|------|-------------|
| `userId` | string | Globally unique identifier (ULID) |
| `email` | string | Unique email address |
| `name` | string | Display name |
| `avatarUrl` | string \| undefined | Profile picture URL |
| `appRole` | string | Role in the current app: `"owner"`, `"admin"`, or `"member"` |
| `appId` | string | The app the user belongs to |
| `addedAt` | string \| undefined | When the user joined the app |

`getBasic` results are cached with a 5-minute default TTL. Override per call via `GetUserOptions`.

Its fields: `refreshNetwork` and `refreshIfOlderThan` (both refresh *behind* the cached value, so the call returns straight away and the *next* read is current), `waitForLoad` (a `WaitForLoadMode`: `.local`, `.network`, `.localIfAvailableElseNetwork`), `serverTimeout`. The two durations are `TimeInterval`s in seconds. Use `waitForLoad: .network` when the call must wait for fresh server data.

### Supplementing user data — not replacing it

For app-specific user data (preferences, settings, profile fields beyond name/email/avatar), extend the platform user rather than creating a parallel user model:

**Do this:**

Declare the `profile` type the example creates, and push it with `primitive config push` before the first create:

```toml
# primitive/dev/db3-types/profile.toml
[type]
name = "profile"

[models.appUsers]
class_name = "AppUser"

[models.appUsers.fields.id]
type = "id"
auto_assign = true
indexed = true
required = true

[models.appUsers.fields.email]
type = "string"
indexed = true
unique = true
required = true

[models.appUsers.fields.name]
type = "string"
```

```swift
  // The user's own database, created on first use and found again by its alias.
  let mine = try await client.db3.getOrCreateWithAlias(
    Db3GetOrCreateWithAliasOptions(
      alias: AliasRef(scope: .user, aliasKey: "profile"),
      title: "Profile",
      type: "profile"
    )
  )
  let sub = try client.db3.subscribe(mine.id)
  try await sub.available()

  // Key the record by the platform userId.
  let profile = AppUser(
    id: platformUserId,
    email: "alice@example.com",
    name: "Software engineer"
  )
  try profile.save(in: mine.id)
```

**Don't do this:**

```swift
// DON'T create a separate user model that duplicates platform fields
var user = AppUser()
user.email = "alice@example.com"  // Already managed by platform
user.name = "Alice"               // Already managed by platform
user.id = generateNewId()         // Use platform userId instead
```

### When to store additional user data

| Storage | Use case |
|---------|----------|
| **A per-user database** (`getOrCreateWithAlias` with `scope: "user"`) | Personal settings and preferences, private to the user |
| **A shared database** (a team's or project's) | Profile data the people sharing that database see |
| **A sync-off database read through a server function** | User data others see only in part (e.g., a public profile, a reputation score), each caller getting what the function returns |

See the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#partitioning-patterns).

### Looking up users

The user lookup surface — single fetch, batch fetch, and email lookup — is shown in [Look up users](#look-up-users) above. Result shapes:

- `getBasic(userId)` — cached `BasicUserInfo` (table above).
- `getProfiles([...])` — max 100; returns only users that exist + belong to the app: `[{ userId, email, name: string | null, avatarUrl: string | null }]`.
- `lookup(email)` — `{ exists: true, user: { userId, name, email } } | { exists: false }`.

There is no `list()` or `get()` method on `client.users`. The current authenticated user lives on a separate namespace: `client.me.get()` returns the current user's profile (cached, with the same `GetUserOptions` knobs). To enumerate or search users in the app, use the REST endpoint or the CLI:

```bash
# List app users
primitive users list

# --search: ULID → userId lookup; '@' → email lookup; otherwise substring
# name search (backed by a global search index on User.name).
primitive users list --search "ali"

# Block a user from signing in to this app, and restore them. Disabling signs
# them out of this app (sessions deleted, API tokens revoked, open connections
# dropped) and is reversible: memberships and ownership are untouched, and
# after `enable` they sign in again and their group access resumes. The app's
# only owner cannot be disabled.
primitive users disable <user-id> [-y]
primitive users enable <user-id>

# Set a user's display name and avatar, at creation or afterwards. Each
# set-profile flag changes only its field; the user can still change their own
# profile afterwards.
primitive users create user@example.com --name "Ada Lovelace" --avatar-url https://example.com/ada.png
primitive users set-profile <user-id> [--name <name>] [--avatar-url <url> | --clear-avatar]
```

An app owner or admin sets the same profile over REST with `PATCH /app/{appId}/api/users/{userId}/profile` and a body of `{ name?, avatarUrl? }` (`avatarUrl` is an `http:` or `https:` URL; `null` clears it); a server function calls `ctx.api.users.setProfile({ userId, body: { name } })`. An invalid value answers 400 with code `INVALID_PROFILE`.

For in-app user pickers, call the REST endpoint directly:

```
GET /app/{appId}/api/users?name=ali&limit=20
Authorization: Bearer <token>
```

It returns `{ items, nextCursor }`, where each row is `{ userId, email, name, avatarUrl, role, addedAt }` filtered to members of the current app.

### App roles

Every user has one of three built-in roles: `"owner"`, `"admin"`, or `"member"` (default). **Prefer groups over app roles** for application-level role modeling — groups are multi-tenant by context (a user can be `editor` in one project, `viewer` in another) while app roles are global per-app.

Behavior:
- **`owner`** and **`admin`** users **bypass all rule-set evaluation** for groups and collections, and can read and write every database in the app. Do not try to restrict them via rules.
- **`member`** is the default — access is determined by direct permissions and group memberships.
- In CEL rule contexts the field is `user.role` (NOT `user.appRole`). See the [Access Control guide's identity context](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#identity-context).

## Current User: `client.me`

The current authenticated user has its own namespace. Use it for "me"-scoped reads and writes — profile and avatar.

### Profile

```swift
  // Current profile (cache-backed). Nil when signed out.
  let profile = try await client.me.get(
    options: FetchCachedOptions(
      waitForLoad: .localIfAvailableElseNetwork,  // | .local | .network
      refreshNetwork: true,  // refresh behind the cached value
      refreshIfOlderThan: 60,  // seconds; default is 5 minutes
      serverTimeout: 5  // seconds
    )
  )

  // Update name and/or avatar (pass .clear to remove the avatar).
  _ = try await client.me.update(
    params: UpdateMeParams(
      name: "Alice Reyes",
      avatarUrl: .value("https://cdn.example.com/u/alice.png")
    )
  )

  // Upload an image directly; the server hosts it and returns a URL.
  let uploaded = try await client.me.uploadAvatar(imageData: avatar, contentType: .png)
  let avatarUrl = uploaded.avatarUrl

  // update() and uploadAvatar() clear the cache automatically; reach for
  // these only when you need to inspect or force a refresh yourself.
  let info = await client.me.cacheInfo()  // MeCacheInfo(updatedAt?, ageMs?)
  await client.me.clearCache()  // next get() hits the network
```

`update()` accepts only an `http:` or `https:` avatar URL; any other scheme answers 400. `get()` is cache-backed; `update()` and `uploadAvatar()` clear that cache automatically. `get()` returns `{ userId, email, name, appRole, appId, avatarUrl? }` (null when signed out).

### Databases the user can open

`client.db3.listMine()` lists the databases the signed-in user can open. See the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#access).

## Core Concept: Groups

Groups organize users and control access to databases and collections. Instead of granting permissions to individual users, grant access to a group and all members inherit it.

Groups are identified by `groupId` alone, unique across the app — no type, no per-app-type taxonomy. The server assigns the id, a ULID, when the group is created and returns it: store it with what the group belongs to (the project's record, the student's row) and look it up from there — see [Common Patterns](#common-patterns).

### Quick start

1. Create a group from a server function (`ctx.api.groups.create({ body: { name } })`) or the admin API (`primitive groups create`), and keep the `groupId` it returns; a client session's create is refused. See [Creating Groups](#creating-groups).
2. Add members with `client.groups.addMember(groupId, ...)` (by `userId` or `email`), each a `manager` or `member` (the default).
3. Share a database with the group with `client.db3.permissions(db3Id).grantGroup({ groupId, permission })`.
4. For a sync-off database read through a server function, check membership in the function: in its `access` gate (`isMemberOf('<groupId>')`, the id the create returned) or, when the group depends on the input, in its code.

The full list/get/update/delete surface is in [Managing Groups](#managing-groups); membership in [Managing Members](#managing-members); roles, custody and transfer in [Group Roles](#group-roles).

## Creating Groups

A group is created only by a server function (`ctx.api.groups.create`) or through the admin API — the CLI's `primitive groups create`, or `POST /admin/api/apps/{appId}/groups`. A client session's `client.groups.create()` is refused with `403 FUNCTION_ROUTE_FUNCTION_ONLY`.

```bash
primitive groups create --name "Engineering" --owner <user-id>   # that user holds the owner row; the app holds it without --owner
```

```ts
import { defineFunction, requireSignedIn } from "primitive-functions";

export default defineFunction({
  authorize: requireSignedIn(),
  handler: async (input, ctx) => {
    return ctx.api.groups.create({
      body: {
        name: "Engineering",
        custodian: `user:${ctx.user!.userId}`,   // defaults to the calling user; "app" for the app
        initialMembers: [{ userId: "user-456", role: "manager" }],
      },
    });
  },
});
```

The server assigns the group's id, a ULID, and returns it as `groupId` in the response — the same way databases and collections get their ids. A create names no id: a body that sends `groupId` (any value, `null` included) is refused with `400 GROUP_ID_NOT_ACCEPTED`. The creator does not become a member unless the create names them (directly, or via `custodian`/`initialMembers`) — there is no `autoAddCreator`.

`groups.create()` also takes an optional `initialMetadata` — category name → that category's values — stamped on the group in the same call: each category is schema-validated before the group is created, the category's `writeRule` is waived for this stamp, at most 10 categories, and an invalid entry fails the whole create (neither the group nor any metadata is left).

```bash
primitive groups create --name "Reading group" \
  --initial-metadata '{"classLink":{"classId":"class-A"}}'
```

## Managing Groups

### List / Get / Update / Delete

```swift
  // List — returns { items, nextCursor }.
  let page1 = try await client.groups.list(options: ListGroupsOptions(limit: 10))
  let page2 = try await client.groups.list(
    options: ListGroupsOptions(limit: 10, cursor: page1.nextCursor)
  )

  // Get a single group — includes its custodian and settings.
  let group = try await client.groups.get(groupId: groupId)

  // Update name and/or description (both optional).
  _ = try await client.groups.update(
    groupId: groupId,
    params: UpdateGroupParams(name: "Platform Engineering", description: "Owns the platform stack")
  )

  // Cascade-deletes all memberships and group permissions.
  _ = try await client.groups.delete(groupId: groupId)
```

`groups.list()` returns a paginated `{ items: GroupInfo[], nextCursor?, hasMore? }`. `ListGroupsOptions` supports `limit`, `cursor`, and `includeSystem: true` to include platform-managed internal groups (e.g. `_col-reader`/`_col-writer` backing collection sharing) — filtered out by default; only set it for admin tooling. Its `type` filter is deprecated and matched against the group's hidden stored type, not a live taxonomy.

`groups.get()` returns `{ appId, groupId, name, description?, memberCount, custodian, settings, createdAt, createdBy, modifiedAt }` (`custodian` is `"user:<userId>"` or `"app"`, `null` on a group from before custody). `update()` takes optional `name`, `description` and/or `settings`. `delete()` cascade-deletes all memberships and group permissions.

## Managing Members

### Add members

Add a member by `userId` (always direct) or by `email` (direct if the email maps to an app user, deferred otherwise). Pass **either** `userId` or `email`, never both — the server rejects requests carrying both. `role` is `manager` or `member` (the default); `owner` is the custodian's row and only a server function or the admin API writes it.

`addMember` returns a result you branch on by `status` (`DirectGroupAdd | DeferredGroupAdd`):

```swift
  let result = try await client.groups.addMember(
    groupId: "engineering",
    params: .email(email)
  )

  switch result {
  case let .direct(add) where add.status == "added":
    // New membership: userId, role, userName?, userEmail?, addedAt, addedBy
    print("added", add.userId)
  case .direct:
    // status == "already_member": idempotent no-op (replaces the old HTTP 409).
    print("already a member")
  case let .deferred(deferred):
    // Email isn't an app user yet. The server created an AppInvitation +
    // DeferredGroupAdd: email, appInvitationCreated, deferredId, expiresAt,
    // role, invitationId, inviteToken. Use inviteToken to build an accept
    // URL; cancel via revokeDeferredGrant.
    _ = try await client.invitations.revokeDeferredGrant(
      deferredId: deferred.deferredId, type: .group
    )
  }
```

Status meanings:
- `"added"` — new membership created. `{ userId, role, userName?, userEmail?, addedAt, addedBy }`.
- `"already_member"` — idempotent no-op. `role`/`addedAt`/`addedBy` reflect the pre-existing row.
- `"pending_signup"` — email not yet an app user. Server created an `AppInvitation` + `DeferredGroupAdd`: `{ email, appInvitationCreated, deferredId, expiresAt, role, invitationId, inviteToken }`. Use `inviteToken` to build your own accept URL; the platform's default email is sent unless the underlying invitation was created with `sendEmail: false`.

Until a deferred add resolves, `isMemberOf` returns false for that email's user — do not assume membership before sign-up/accept.

**Don't do this:**

- **Don't pass both `userId` and `email`** in one call — it's rejected.
- **Don't assume the result is always direct.** When the email isn't an app user yet, `status` is `"pending_signup"` and there is no `userId` on the result — always branch on `status` before reading direct-add fields.

See the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#deferred-grants) for the full deferred-grant lifecycle and the token-based acceptance path.

### List members

```swift
  let page = try await client.groups.listMembers(groupId: "engineering")
  // page.items: [GroupMemberInfo(userId, role, userName?, userEmail?, addedAt, addedBy)]

  let next = try await client.groups.listMembers(
    groupId: "engineering",
    options: PaginationOptions(limit: 50, cursor: page.nextCursor)
  )

  // Join profile data in the same call with `include: .profiles`.
  let withProfiles = try await client.groups.listMembers(
    groupId: "engineering",
    include: .profiles
  )
  // withProfiles.items: [GroupMemberInfo(userId, role, userName?, userEmail?, avatarUrl?, addedAt, addedBy)]
  // avatarUrl is a URL, or nil if the user has no avatar or the membership
  // is orphaned (deleted user).
```

Pass `include: .profiles` to join each member's profile in the same call. Without it, `GroupMemberInfo.avatarUrl` is `nil` — the field is absent from the default response. With `include: .profiles`, `userName`/`userEmail` become reliably populated and `avatarUrl` is a resolved URL to the uploaded avatar, or `nil` when the user has no avatar or the membership is orphaned. Orphaned memberships (the user was deleted) still appear in the list either way — only the profile fields go nil. `include` is opt-in and purely additive: omitting it returns exactly the sparse shape above. The server rejects unknown `include` values with HTTP 400.

### Remove members

```swift
  // By user ID
  _ = try await client.groups.removeMember(groupId: "engineering", userId: "user-456")

  // By email — removes a direct membership if one exists, otherwise cancels
  // the pending DeferredGroupAdd for that email.
  _ = try await client.groups.removeMemberByEmail(
    groupId: "engineering", email: "alice@example.com"
  )
```

The email form is the right call when you don't know — and don't want to branch on — whether the target has signed up yet. If a direct membership exists, it's removed; otherwise the pending `DeferredGroupAdd` for that email is canceled. Revoke the whole `AppInvitation` only when you want to cancel **every** grant attached to it (group add, database share, app-join right) at once.

### List pending invitations for a group

```swift
  let pending = try await client.groups.listPendingInvitations(groupId: "engineering")
  // [["email", "role", "invitationId", "deferredId", "createdAt", "expiresAt", "addedBy"?]]
  // deferredId → invitations.revokeDeferredGrant(deferredId:type: .group) cancels it
```

Use this to render the "pending members" section of a group sharing UI without having to filter the lower-level `client.invitations.listDeferredGrants()` surface.

Pending invitations are visible to whoever can list members — the owner and managers always, other members when `settings.memberList` is `"members"` (the default) — plus app admins and owners.

Each result includes a `deferredId`. Cancel it with`client.invitations.revokeDeferredGrant(deferredId: entry.deferredId, type: .group)`. Cancellation requires app-admin authority, being the one who created the invitation, or the owner/manager role on the group.

### List a user's memberships

```swift
  let memberships = try await client.groups.listUserMemberships(userId: userId)
  // [["groupId", "name", "description"?, "role", "addedAt", "addedBy"]]
```

## Group Roles

Every group has the same fixed roles, enforced by the platform — no types, no rule sets, no per-type configuration:

| Role | Can |
|---|---|
| `owner` | Everything a manager can, plus delete the group and transfer ownership. One per group: the custodian. |
| `manager` | Add, remove and re-role members up to manager; edit name, description and settings. |
| `member` | Read the group; list members when `settings.memberList` is `"members"` (the default). |

The owner is the user named by `custodian`, or nobody while `custodian` is `"app"`. `POST /groups/{groupId}/transfer` with `{ to: "user:<userId>" | "app" }` moves the custodian and the owner row together (the previous owner becomes a manager); a session may move a group only between users, and a server function or the admin API also to and from the app. While a transfer is in progress, another transfer and any owner-row change answer `409 TRANSFER_PENDING`. A server function lists one custodian's groups with `ctx.api.groups.listByCustodian({ custodian, cursor })`. Removing a user from the app passes each group they hold to the app when anyone else is a member, and deletes it otherwise. `group-type-configs/` is no longer part of a config tree: `primitive config push` refuses a tree holding a file there, before any request, naming each file and the replacement:

- A rule the roles do not express → a server function whose `authorize` uses `requireMember(groupIdOf, role)` (see the [Access Control guide](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md)).
- A group metadata manifest (`[metadata.self]`, `[metadata.paths.*]`, `secrets`) → a metadata category config (see the [Resource Metadata guide](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md)).

## Groups and databases

Share a synced database with a group, and every member gets the permission; access follows membership as members join and leave. The grant call is shown in [Share a database with a group](#share-a-database-with-a-group) above (permission is `"reader"` or `"read-write"` — NOT `"write"` or `"view"`). A server function grants the same way with `ctx.api.db3.grantGroupPermission({ db3Id, body: { groupId, permission } })`.

**Permission resolution:** A user's effective permission on a database is the **highest** across their direct grant and all their groups' grants. If a user has `reader` directly but their team has `read-write`, they get `read-write`.

A sync-off database can't be shared, with a group or anyone: members reach its records through **server functions**, so group membership gates the data in two places — see the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#sync-off):

- **The function's `authorize`** — checks the caller's identity with helpers (`requireRole`, `requireMember`) or the `auth` context directly (`auth.hasRole`, `auth.isMemberOf`, `auth.memberGroups`, `auth.hasGroupRole` — the shared identity context, documented in the [Access Control guide](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#identity-context)). Unlike the legacy `access` rule, `authorize` also sees the function's input, so a group that depends on it can be checked here directly — and gives owners and admins no implicit bypass.
- **The function's code** — for a check `authorize`'s helpers don't express, read the caller's memberships in the handler and refuse before touching the records.

**Authorize patterns:**

```ts
// Any member of one fixed group (the groupId its create returned)
authorize: requireMember(() => "01JD7E8V3K9Q2N6R4T5W0X1Y2Z"),

// Either of two fixed groups
authorize: async (input, auth) =>
  auth.isMemberOf("01JD7E8V3K9Q2N6R4T5W0X1Y2Z") || auth.isMemberOf("01JD7E9A4M2P8S6V3B7C1D5F9G")
    ? allow()
    : deny("FORBIDDEN"),
```

**Code pattern — membership that depends on the input:**

```ts
import { defineFunction, requireSignedIn } from "primitive-functions";

export default defineFunction({
  authorize: requireSignedIn(),
  handler: async (input: { db3Id: string; teamId: string }, ctx) => {
    // A bare array of { groupId, name, role, … }.
    const teams = await ctx.api.groups.listUserMemberships({ userId: ctx.user!.userId });
    if (!teams.some((m: { groupId: string }) => m.groupId === input.teamId)) {
      throw new Error("Not a member of this team");
    }
    return (await ctx.db3(input.db3Id, "team").model("tasks").query({})).items;
  },
});
```

**Don't do this — common CEL footguns**, in a rule set or metadata rule:

```
// BAD — `user.appRole` does not exist in CEL. Use `user.role` or hasRole().
read = "user.appRole == 'admin'"

// GOOD — checks the caller's app role.
read = "hasRole('admin')"

// BAD — isMemberOf returns bool, not the group. `==` is meaningless.
read = "isMemberOf('01JD7E8V3K9Q2N6R4T5W0X1Y2Z') == '01JD7E8V3K9Q2N6R4T5W0X1Y2Z'"

// GOOD
read = "isMemberOf('01JD7E8V3K9Q2N6R4T5W0X1Y2Z')"

// BAD — memberGroups returns an array, can't compare with ==.
read = "memberGroups() == '01JD7E8V3K9Q2N6R4T5W0X1Y2Z'"

// GOOD — use `in` for containment.
read = "'01JD7E8V3K9Q2N6R4T5W0X1Y2Z' in memberGroups()"

// BAD — referencing fields not in the context (silently denies; runtime
// errors are caught and turned into "deny").
read = "user.email == 'admin@example.com'"   // user.email is not in context
```

## Common Patterns

For full app architectures showing how groups fit alongside databases, see the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#partitioning-patterns).

### Team-based workspace access

A function creates a team and gives the creator its owner row; the team's database is shared with the team's group, so every member reaches it.

```ts
import { defineFunction, requireSignedIn } from "primitive-functions";

export default defineFunction({
  authorize: requireSignedIn(),
  handler: async (input: { name: string }, ctx) => {
    return ctx.api.groups.create({ body: { name: input.name } });   // custodian defaults to ctx.user, who gets the owner row
  },
});
```

**Runtime** (in app code, once the team's `groupId` is known):

```swift
  _ = try await client.db3.permissions(db3Id).grantGroup(
    GrantGroupPermissionParams(groupId: teamId, permission: "read-write")
  )
```

When the team's database has sync off, the function that reads it checks the caller's membership in the team in code — see [Groups and databases](#groups-and-databases).

### Role-based access (reviewer, editor, viewer)

Use one group per role per context: the function that creates a project creates its editors and viewers groups and stores their `groupId`s on the project (its record, or its metadata).

```swift
  let permissions = client.db3.permissions(db3Id)
  _ = try await permissions.grantGroup(
    GrantGroupPermissionParams(groupId: "project-1-editors", permission: "read-write")
  )
  _ = try await permissions.grantGroup(
    GrantGroupPermissionParams(groupId: "project-1-viewers", permission: "reader")
  )
```

The project comes from the input, so the function checks role membership in code: a write function reads the project's editors `groupId` from where it was stored and requires a membership in it; a read function accepts either role's group.

### Relationship modeling (parent-child, mentor-mentee)

Use one group per relationship, its `groupId` stored on the relationship's subject (the student's record carries `parentsGroupId`).

**Server side** — the function that reads a student's grades checks the relationship before it reads:

```ts
import { defineFunction, requireSignedIn } from "primitive-functions";

export default defineFunction({
  authorize: requireSignedIn(),
  handler: async (input: { db3Id: string; studentId: string }, ctx) => {
    const classroom = ctx.db3(input.db3Id, "classroom");
    const student = await classroom.model("students").get(input.studentId);
    const children = await ctx.api.groups.listUserMemberships({ userId: ctx.user!.userId });
    if (!student || !children.some((m: { groupId: string }) => m.groupId === student.parentsGroupId)) {
      throw new Error("Not this student's parent");
    }
    const grades = classroom.model("grades");
    return (await grades.query({ filter: { studentId: input.studentId }, sort: { date: -1 } })).items;
  },
});
```

The membership check is what ensures parents can only view their own children's grades; the caller never supplies whose relationship is checked.

**Runtime** (in app code, once the group is created by a server function):

```swift
  _ = try await client.groups.addMember(groupId: parentsGroupId, params: .userId(parentUserId))

  // Parent asks for their child's grades — the function enforces access.
  let grades: FunctionResult<JSONValue> = try await client.functions.invoke(
    "view-grades",
    input: ["db3Id": db3Id, "studentId": studentId]
  )
  _ = grades
```

### Organization hierarchy

Model nested organizational structure as distinct groups — an org, its departments, their teams — each `groupId` stored on the record of the unit it stands for.

```swift
  _ = try await client.groups.addMember(groupId: orgGroupId, params: .userId(userId))
  _ = try await client.groups.addMember(groupId: deptGroupId, params: .userId(userId))
  _ = try await client.groups.addMember(groupId: teamGroupId, params: .userId(userId))
```

A function can check any level: a fixed one in its `authorize` (`requireMember(() => "<org groupId>")`), and one named by the input in its code, reading that unit's `groupId` and checking it against `listUserMemberships({ userId })`.

## Gotchas

- Keep each group's `groupId` where the app looks it up (the record or metadata of what the group stands for): the server assigns it, so it cannot be derived from a name.
- Do not duplicate platform identity fields in an app model.
- Check `addMember.status`: an email may create a pending invitation rather than an immediate membership.
- App owners and admins bypass CEL rules — rule sets, metadata rules, a legacy function's `access`. A function's `authorize` gives them no such pass. Test denials as a regular member.

## Error handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| `addMember` returns `status: "pending_signup"` | Email isn't an app user yet | Expected. Membership resolves when they sign up or accept via `inviteToken`. Render a pending-members UI; cancel via `removeMemberByEmail(groupId:email:)` or `invitations.revokeDeferredGrant(deferredId:type: .group)`. |
| `addMember` returns `status: "already_member"` | User already in the group | Idempotent — no error. |
| 400 `GROUP_ID_NOT_ACCEPTED` on `groups.create` | The body sent `groupId` | Omit it; read `groupId` from the response. |
| 400 on `groups.create` with `initialMetadata` | A staged category is unknown, reserved (`attrs`), fails its schema, or there are more than 10 | Fix the payload; nothing was created. |
| 409 `METADATA_EXISTS` on `groups.create` | Metadata for a staged category is already stored under the new `groupId` | Delete the stored category (`resourceMetadata.delete`, gated by the category's `writeRule`) or leave it out of `initialMetadata`. Nothing was created. |
| 409 `UNIQUE_VALUE_CONFLICT` on `groups.create` | A staged category's unique field holds a value another resource already holds | Change the conflicting value. Nothing was created. |
| 403 `FUNCTION_ROUTE_FUNCTION_ONLY` on `groups.create` | A client session called it | Create from a server function (`ctx.api.groups.create`) or the admin API / CLI instead. |
| CEL `isMemberOf` returns false unexpectedly | Wrong `groupId`, user not yet added (deferred), or membership cache stale | Verify with `listUserMemberships(userId)`; remember email-based adds defer until sign-up. |
| Can't restrict admin/owner access via rules | By design — app owner/admin bypass all rule evaluation | Use group memberships and roles for fine-grained access; don't try to gate admins. |

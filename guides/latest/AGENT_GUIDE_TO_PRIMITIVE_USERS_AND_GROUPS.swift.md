# Users, Groups, and Access Control in Primitive

Guidelines for modeling user relationships and managing access control with Primitive's built-in user system and groups.

## Core operations

The key client calls, compiled against the real clients as part of the docs build. The dense reference below adds the result shapes, result-status branching, and CEL access-control detail.

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
    groupType: "team", groupId: "engineering",
    params: .email("alice@example.com")
  )

  // ...or by user id (internal / programmatic)
  _ = try await client.groups.addMember(
    groupType: "team", groupId: "engineering",
    params: .userId("user-456")
  )

  // List a group's members
  let members = try await client.groups.listMembers(groupType: "team", groupId: "engineering")

  // List the groups a user belongs to
  let memberships = try await client.groups.listUserMemberships(userId: userId)
```

### Grant document access to a group

```swift
  _ = try await client.documents.grantGroupPermission(
    documentId: documentId,
    params: GrantGroupPermissionParams(
      groupType: "team",
      groupId: "engineering-team",
      permission: "read-write"
    )
  )
```

## Core Concept: Built-in User Model

Primitive provides a built-in user model that every app should leverage. **Do not reinvent user identity.** The platform manages user accounts, authentication, and basic profile information — your app builds on top of this.

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

`getBasic` results are cached with a 5-minute default TTL. Override per call via `GetUserOptions`: `refreshNetwork` (bypass cache once), `refreshIfOlderThanMs`, `waitForLoad` (`"local" | "network" | "localIfAvailableElseNetwork"`), `serverTimeoutMs`.

### Supplementing user data — not replacing it

For app-specific user data (preferences, settings, profile fields beyond name/email/avatar), extend the platform user rather than creating a parallel user model:

**Do this:**

```swift
  // Store additional user data in a document, keyed by the platform userId.
  let profile = AppUser(
    id: platformUserId, // reference the platform user
    email: "alice@example.com",
    name: "Software engineer"
  )
  try profile.save(in: userDocumentId)

  // Or in a database via a registered operation. The operation uses
  // $user.userId server-side — no need to pass the userId yourself.
  _ = try await client.databases.executeOperation(
    databaseId: dbId, name: "updateProfile",
    options: ExecuteOperationOptions(params: ["bio": "Software engineer", "theme": "dark"])
  )
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
| **Root document** | Personal settings, preferences (auto-created per user, never shared) |
| **User's document** | App-specific profile data that follows document sharing rules |
| **Database** | User metadata visible to other users (e.g., public profile, reputation score) accessed via operations with `$user.userId` |

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
```

For in-app user pickers, call the REST endpoint directly:

```
GET /app/{appId}/api/users?name=ali&limit=20
Authorization: Bearer <token>
```

It returns `{ items, nextCursor }`, where each row is `{ userId, email, name, avatarUrl, role, addedAt }` filtered to members of the current app.

### App roles

Every user has one of three built-in roles: `"owner"`, `"admin"`, or `"member"` (default). **Prefer groups over app roles** for application-level role modeling — groups are multi-tenant by context (a user can be `editor` in one project, `viewer` in another) while app roles are global per-app.

Behavior:
- **`owner`** and **`admin`** users **bypass all rule-set evaluation** for groups, collections, and database type rules. Do not try to restrict them via rules.
- **`member`** is the default — access is determined by direct permissions and group memberships.
- In CEL rule contexts the field is `user.role` (NOT `user.appRole`). See [Rule CEL context](#rule-cel-context) below.

## Current User: `client.me`

The current authenticated user has its own namespace. Use it for "me"-scoped reads and writes — profile, avatar, the user's owned and shared documents, and pending document invitations.

### Profile

```swift
  // Current profile (cache-backed). Nil when signed out.
  let profile = try await client.me.get(
    options: FetchCachedOptions(
      waitForLoad: .localIfAvailableElseNetwork,  // | .local | .network
      refreshNetwork: true,  // bypass the cache once
      refreshIfOlderThanMs: 60_000,  // default is 5 minutes
      serverTimeoutMs: 5_000
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

`get()` is cache-backed; `update()` and `uploadAvatar()` clear that cache automatically. `get()` returns `{ userId, email, name, appRole, appId, avatarUrl? }` (null when signed out).

### Documents view

```swift
  // Documents the user owns (created, or had ownership transferred to).
  let owned = try await client.me.ownedDocuments(limit: 50, tag: "draft")

  // Documents shared directly with the user (non-owner permission rows +
  // pending invitations). Group/collection shares do NOT appear here.
  let shared = try await client.me.sharedDocuments(limit: 50, tag: "shared")
  let items = shared.items
  let cursor = shared.cursor

  // Document invitations the user can accept — an inbox view.
  let pending = try await client.me.pendingDocumentInvitations()
```

- `ownedDocuments()` is cache-backed and offline-aware and returns `[DocumentInfo]`. Use `ownedDocumentsPage(...)` when you need a paginated `DocumentListPage`; the root document is excluded by default (`includeRoot: false`).
- `sharedDocuments()` returns the unified `{ items, cursor }` envelope (raw-JSON cursor, NOT base64url). Group- and collection-scoped shares do NOT appear here — those are accessed via the group or collection. Each `SharedDocument` extends `DocumentInfo`, so rows carry the base document fields plus the share extras (`permission`, `source`, `grantedBy`, `invitationId`).
- `pendingDocumentInvitations()` returns `[{ invitationId, documentId, title?, email, permission, invitedAt, invitedBy, expiresAt?, accepted, document?: {...} }, ...]`.

Together, `me.ownedDocuments()` + `me.sharedDocuments()` give the two halves of "documents the user has direct access to." For group- or collection-scoped access, iterate `groups.listUserMemberships(...)` / `client.collections.list()` and call `groups.listDocuments` / `collections.listDocuments`.

## Core Concept: Groups

Groups organize users and control access to documents and databases. Instead of granting permissions to individual users, grant access to a group and all members inherit it.

Groups are identified by a `(groupType, groupId)` pair, allowing multiple taxonomies:
- `team/engineering` — team membership
- `department/sales` — organizational unit
- `role/reviewer` — functional role
- `class/math-101` — context-specific grouping
- `parent-of/student-123` — relationship modeling

### Quick start

1. Create a group with `client.groups.create({ groupType, groupId, name })` — `groupId` is optional; omit it (or pass `null`) and the server assigns a ULID, returned in the response.
2. Add members with `client.groups.addMember(...)` (by `userId` or `email`).
3. Grant group access to a document with `client.documents.grantGroupPermission(...)`.
4. Gate database operations in CEL: `access: "isMemberOf('team', database.metadata.teamId)"`.

The full create/list/get/update/delete surface is in [Managing Groups](#managing-groups); membership in [Managing Members](#managing-members).

## Managing Groups

### Create / List / Get / Update / Delete

```swift
  // Create. If the group type has autoAddCreator (default), the creator is
  // added as a member.
  _ = try await client.groups.create(params: CreateGroupParams(
    groupType: "team",
    groupId: "engineering",
    name: "Engineering Team",
    description: "Platform engineering team"  // optional
  ))

  // List — returns { items, cursor }. Filter by type and page through.
  let page1 = try await client.groups.list(options: ListGroupsOptions(type: "team", limit: 10))
  let page2 = try await client.groups.list(
    options: ListGroupsOptions(type: "team", limit: 10, cursor: page1.cursor)
  )

  // Get a single group.
  let group = try await client.groups.get(groupType: "team", groupId: "engineering")

  // Update name and/or description (both optional).
  _ = try await client.groups.update(
    groupType: "team", groupId: "engineering",
    params: UpdateGroupParams(name: "Platform Engineering", description: "Owns the platform stack")
  )

  // Cascade-deletes all memberships and group permissions.
  _ = try await client.groups.delete(groupType: "team", groupId: "engineering")
```

If the group type has `autoAddCreator: true` (default), the creator is automatically added as a member.

`groupId` on create is optional — a supplied id keeps its existing validation (`#` banned, no `_`-prefixed reserved type, `409` on a duplicate); a non-string supplied value is rejected with `400 "groupId must be a string"`, an empty string with `400 "groupId cannot be empty"`. Omit it (or send `null`) to have the server assign a ULID, returned as `groupId` in the response — the same pattern documents, databases, and collections use for server-assigned ids.

`groups.list()` returns a paginated `{ items: GroupInfo[], cursor? }`. `ListGroupsOptions` supports `type`, `limit`, `cursor`, and `includeSystem: true` to include platform-managed internal groups whose `groupType` is prefixed with `_` (e.g. `_col-reader`/`_col-writer` backing collection sharing). These are filtered out by default — only set `includeSystem` for admin tooling.

`groups.get()` returns `{ appId, groupType, groupId, name, description?, memberCount, createdAt, createdBy, modifiedAt }`. `update()` takes optional `name` and/or `description`. `delete()` cascade-deletes all memberships and group permissions.

## Managing Members

### Add members

Add a member by `userId` (always direct) or by `email` (direct if the email maps to an app user, deferred otherwise). Pass **either** `userId` or `email`, never both — the server rejects requests carrying both.

`addMember` returns a result you branch on by `status` (`DirectGroupAdd | DeferredGroupAdd`):

```swift
  let result = try await client.groups.addMember(
    groupType: "team", groupId: "engineering",
    params: .email(email)
  )

  switch result {
  case let .direct(add) where add.status == "added":
    // New membership: userId, userName?, userEmail?, addedAt, addedBy
    print("added", add.userId)
  case .direct:
    // status == "already_member": idempotent no-op (replaces the old HTTP 409).
    print("already a member")
  case let .deferred(deferred):
    // Email isn't an app user yet. The server created an AppInvitation +
    // DeferredGroupAdd: email, appInvitationCreated, deferredId, expiresAt,
    // groupType, groupId, invitationId, inviteToken. Use inviteToken to build
    // an accept URL; cancel via revokeDeferredGrant.
    _ = try await client.invitations.revokeDeferredGrant(
      deferredId: deferred.deferredId, type: .group
    )
  }
```

Status meanings:
- `"added"` — new membership created. `{ userId, userName?, userEmail?, addedAt, addedBy }`.
- `"already_member"` — idempotent no-op. `addedAt`/`addedBy` reflect the pre-existing row.
- `"pending_signup"` — email not yet an app user. Server created an `AppInvitation` + `DeferredGroupAdd`: `{ email, appInvitationCreated, deferredId, expiresAt, groupType, groupId, invitationId, inviteToken }`. Use `inviteToken` to build your own accept URL; the platform's default email is sent unless the underlying invitation was created with `sendEmail: false`.

Until a deferred add resolves, `isMemberOf` returns false for that email's user — do not assume membership before sign-up/accept.

**Don't do this:**

- **Don't pass both `userId` and `email`** in one call — it's rejected.
- **Don't assume the result is always direct.** When the email isn't an app user yet, `status` is `"pending_signup"` and there is no `userId` on the result — always branch on `status` before reading direct-add fields.

See the [Invitations guide](AGENT_GUIDE_TO_PRIMITIVE_INVITATIONS.md#deferred-grants) for the full deferred-grant lifecycle and the token-based acceptance path.

### List members

```swift
  let page = try await client.groups.listMembers(groupType: "team", groupId: "engineering")
  // page.items: [GroupMemberInfo(userId, userName?, userEmail?, addedAt, addedBy)]

  let next = try await client.groups.listMembers(
    groupType: "team", groupId: "engineering",
    options: PaginationOptions(limit: 50, cursor: page.cursor)
  )
```


### Remove members

```swift
  // By user ID
  _ = try await client.groups.removeMember(
    groupType: "team", groupId: "engineering", userId: "user-456"
  )

  // By email — removes a direct membership if one exists, otherwise cancels
  // the pending DeferredGroupAdd for that email.
  _ = try await client.groups.removeMemberByEmail(
    groupType: "team", groupId: "engineering", email: "alice@example.com"
  )
```

The email form is the right call when you don't know — and don't want to branch on — whether the target has signed up yet. If a direct membership exists, it's removed; otherwise the pending `DeferredGroupAdd` for that email is canceled. Revoke the whole `AppInvitation` only when you want to cancel **every** grant attached to it (group add, document share, app-join right) at once.

### List pending invitations for a group

```swift
  let pending = try await client.groups.listPendingInvitations(
    groupType: "team", groupId: "engineering"
  )
  // [["email", "role", "invitationId", "deferredId", "createdAt", "expiresAt", "addedBy"?]]
  // deferredId → invitations.revokeDeferredGrant(deferredId:type: .group) cancels it
```

Use this to render the "pending members" section of a group sharing UI without having to filter the lower-level `client.invitations.listDeferredGrants()` surface.

Authorization is the same gate as `listMembers`: the group's `member.list` rule, with app admins/owners always allowed and direct members of the group allowed as a fallback even under a stricter custom rule. A cross-group manager whose rules let them list a group's members can therefore also see its pending invitations — being a member of the group is not required.
Each entry carries a `deferredId` — cancel that invitation by revoking the deferred grant: `client.invitations.revokeDeferredGrant(deferredId: entry.deferredId, type: .group)`. Cancelling an invitation another caller created is allowed for app admins/owners, the invitation's creator, or a caller who passes the group's `member.delete` rule (evaluated with `target.email` set to the invitee's email, so email-scoped rules work on the pending path).

### List a user's memberships

```swift
  let memberships = try await client.groups.listUserMemberships(userId: userId)
  // [["groupType", "groupId", "name", "description"?, "addedAt", "addedBy"]]
```

## Group Type Configuration

Group types are configured via TOML config files and the `primitive sync` command (version-controlled alongside your code).

**File:** `config/group-type-configs/team.toml`

```toml
[groupTypeConfig]
groupType = "team"
ruleSetName = "team-rules"     # optional — name of an attached rule set
autoAddCreator = true          # auto-add creator as member when the config exists (default: true)

[metadata.self]
categories = ["config"]        # optional — self-reads like md.self.config.* are
                                # inferred from the rule set; declare here only to
                                # load a category no rule names directly
```

A group type config reads `md.self.<category>.<key>` in its rule set with **no declaration** — a category a rule names is inferred and loaded automatically — plus a reserved, schema-less `attrs` category (`groupType`, `groupId`, `name`, `createdBy`). An explicit metadata manifest (`[metadata.self]`/`[metadata.paths.*]`/top-level `secrets`) is also supported and unions with the inferred set — declare a category no rule names, a traversal path, or a secret. Collection type configs take the identical `[metadata]` block (collection `attrs` adds `contextId`). See the [Resource Metadata guide](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md).

Push to the server:

```bash
primitive sync push
```

**Defaults & gotchas:**
- `autoAddCreator` defaults to `true` **only when a group type config exists** for the type. With **no config at all**, no auto-add happens.
- A group type with **no config** falls back to built-in default rules. Per-op fallback also applies: when a configured rule set leaves a `(category, op)` pair undefined, that op resolves against the defaults too. The defaults: `group.create = "true"` (any signed-in member); `group.edit/delete` and `member.create/edit/delete` are creator-only (`user.userId == group.createdBy`); `group.get` and `member.list` allow the creator OR any direct group member (`isMemberOf(group.groupType, group.groupId)`).
- A group type config with no `ruleSetName` (`ruleSetId: null`) is an **explicit opt-out** and denies everything except admin/owner; it does NOT fall through to defaults. To re-enable defaults, delete the config entirely (`primitive group-type-configs delete <group-type>`, or `client.groupTypeConfigs.delete(groupType)`).

See the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#configuring-with-the-cli) for the full sync workflow (`init`, `pull`, `diff`, `push`).

## Groups and Documents

Grant a group access to a document. All members inherit the permission.

The grant call is shown in [Grant document access to a group](#grant-document-access-to-a-group) above (permission is `"read-write"` or `"reader"` — NOT `"write"` or `"view"`). To inspect what a group reaches and to manage a document's group grants:

```swift
  // What the group can access
  let docs = try await client.groups.listDocuments(groupType: "team", groupId: "engineering")
  let dbs = try await client.groups.listDatabases(groupType: "team", groupId: "engineering")

  // A document's group grants
  let groupPerms = try await client.documents.listGroupPermissions(documentId: documentId)
  _ = try await client.documents.revokeGroupPermission(
    documentId: documentId, groupType: "team", groupId: "engineering"
  )
```

**Permission resolution:** A user's effective permission on a document is the **highest** across their direct permission and all group memberships. If a user has `reader` direct access but their team has `read-write`, they get `read-write`.

## Groups and Databases

Databases support a coarse-grained group grant that gives every member of the group `manager`-level access (the database's admin permission, the only level supported for groups):

```swift
  _ = try await client.databases.grantGroupPermission(
    databaseId: databaseId,
    params: GrantDatabaseGroupPermissionParams(
      groupType: "team", groupId: "engineering", permission: "manager"
    )
  )
```

For everything else — gating individual queries and mutations — group memberships are checked in **CEL access expressions** on registered operations. See the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md) for how to register operations.

The shared identity context every rule can use (`user.*`, `isMemberOf`, `memberGroups`, `hasRole`, `isAnonymous`) is documented in the [Access Control guide](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#identity-context-available-everywhere); the database-operation `access` context (`database.celContext`, `params.*`, `secrets.*`, `workflow`) is documented in the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#cel-access-expressions). This section covers only how group membership factors into those rules.

**Common CEL patterns for database operations:**

```
// Team members of the database's team
access: "isMemberOf('team', database.metadata.teamId)"

// Team ID passed as parameter — caller must belong to that team
access: "isMemberOf('team', params.teamId)"

// User can only query groups they belong to (containment check)
access: "params.teamId in memberGroups('team')"

// Per-parameter access: parent can only view their own child's data
params: {
  studentId: {
    type: "string",
    required: true,
    access: "value in memberGroups('parent-of')"
  }
}

// Allow only when called from a specific workflow
access: "fromWorkflow('grade-import')"
```

**Don't do this — common CEL footguns:**

```
// BAD — `user.appRole` does not exist in CEL. Use `user.role` or hasRole().
access: "user.appRole == 'admin'"

// GOOD — checks the caller's app role.
access: "hasRole('admin') || hasRole('owner')"
// (Note: admins/owners bypass *rule-set* evaluation for groups/collections,
// but database operation `access` CEL is NOT bypassed. You must allow them
// explicitly if you want them in.)

// BAD — isMemberOf returns bool, not the group. `==` is meaningless.
access: "isMemberOf('team') == params.teamId"

// GOOD
access: "isMemberOf('team', params.teamId)"

// BAD — memberGroups returns an array, can't compare with ==.
access: "memberGroups('team') == params.teamId"

// GOOD — use `in` for containment.
access: "params.teamId in memberGroups('team')"

// BAD — referencing fields not in the context (silently denies; runtime
// errors are caught and turned into "deny").
access: "user.email == 'admin@example.com'"   // user.email is not in context

// BAD — assuming database.metadata.teamId exists when it doesn't.
// Missing fields evaluate to null; `isMemberOf('team', null)` returns false.
// Either guarantee metadata is set at create time, or guard:
access: "has(database.metadata.teamId) && isMemberOf('team', database.metadata.teamId)"
```

## Rule Sets for Group Management

Rule sets control who can manage groups (`category: "group"`) and group members (`category: "member"`). Each `(category, operation)` pair holds a CEL expression evaluated against the requesting user and the target group.

**Valid operations:**
- `category: "group"` — `create`, `edit`, `delete`, `get` (the read op; use `get` in TOML configs).
- `category: "member"` — `create`, `edit`, `delete`, `list`.

There is no `read` or `update` — use `edit`. There is no `add` — use `create` (for `member.create` = "add member").

**Default rule set:** A group type with no group type config falls back to built-in defaults:

| Op | Default | Meaning |
|----|---------|---------|
| `group.create` | `"true"` | Any signed-in member |
| `group.edit` | `user.userId == group.createdBy` | Creator only |
| `group.delete` | `user.userId == group.createdBy` | Creator only |
| `group.get` | `user.userId == group.createdBy \|\| isMemberOf(group.groupType, group.groupId)` | Creator or direct member |
| `member.create` | `user.userId == group.createdBy` | Creator only |
| `member.edit` | `user.userId == group.createdBy` | Creator only |
| `member.delete` | `user.userId == group.createdBy` | Creator only |
| `member.list` | `user.userId == group.createdBy \|\| isMemberOf(group.groupType, group.groupId)` | Creator or direct member |

Per-op fallback applies — when a configured rule set defines some ops but leaves others undefined, the missing ops still resolve against this table. Apps that need stricter (or looser) policy install a custom rule set whose defined ops always win. A group type config with no rule set attached (`ruleSetId: null`) is an explicit opt-out and denies everything except admin/owner — it does NOT fall through to these defaults.

### Defining a rule set

Rule sets are defined in TOML config files:

**File:** `config/rule-sets/team-rules.toml`

```toml
[ruleSet]
name = "team-rules"
resourceType = "group"
description = "Controls who can manage team groups and members"

[rules.group]
create = "true"                                               # any signed-in member can create a team
edit   = "isMemberOf(group.groupType, group.groupId)"         # only members can rename
delete = "user.userId == group.createdBy"                     # only the creator can delete
get    = "isMemberOf(group.groupType, group.groupId)"         # only members see the team in list

[rules.member]
create = "isMemberOf(group.groupType, group.groupId)"         # any member can invite
delete = "user.userId == target.userId || user.userId == group.createdBy"  # self-leave or creator-kick
list   = "isMemberOf(group.groupType, group.groupId)"
```

### Attaching to a group type

Reference the rule set by name in the group type config:

**File:** `config/group-type-configs/team.toml`

```toml
[groupTypeConfig]
groupType = "team"
ruleSetName = "team-rules"
autoAddCreator = true
```

Push both configs:

```bash
primitive sync push
```

### Rule CEL context

| Variable | Always present? | Description |
|----------|-----------------|-------------|
| `user.userId` | yes | Requesting user's ID |
| `user.role` | yes | Requesting user's app role (`"owner"` / `"admin"` / `"member"`). NOT `user.appRole`. |
| `group.groupType` | yes | Target group's type |
| `group.groupId` | yes | Target group's ID (also present at `create` time — server passes the requested ID) |
| `group.contextId` | yes | Read-alias of `group.groupId` — the same value under the `contextId` name collection rules use, so group and collection rule sets can share expressions |
| `group.name` | yes | Target group's display name |
| `group.createdBy` | yes (after create) | userId of the group creator |
| `target.userId` | only `category: "member"`, ops `create`/`edit`/`delete` | Target user being added/removed. Absent for `member.list`. |

`group.description` is **NOT** in the rule context. Don't reference it.

Plus all CEL functions: `isMemberOf`, `memberGroups`, `hasRole`, `now()`, `fromWorkflow()`.

**Owners and admins always bypass rule evaluation.** Don't try to restrict them via rules.

**Don't do this — rule CEL footguns:**

```toml novalidate
# BAD — `user.appRole` does not exist. Use user.role.
create = "user.appRole == 'admin'"

# BAD — admins bypass anyway, so this is dead code in practice.
create = "hasRole('admin')"

# BAD — referencing target on member.list (target is undefined there).
list = "target.userId != user.userId"

# BAD — referencing group.description (not in the context).
edit = "group.description != ''"

# BAD — typo: group ops are `create/edit/delete/get`, not `update/read`.
[rules.group]
update = "true"
read   = "true"

# GOOD — same intent with valid operations.
[rules.group]
edit = "true"
get  = "true"
```

### Testing rules

Test a rule set with a simulated request, or debug against a real user's live memberships (full trace). `test()` returns `{ allowed, expression?, context?, trace?, error? }`; `debug()` returns `{ allowed, expression?, reason?, ruleSetId?, ruleSetName?, user?, memberships?, context?, trace? }`. `debug()` **requires console-admin auth** — regular app callers get 403.

```swift
  // Simulated request — no live data needed.
  let result = try await client.ruleSets.test(
    ruleSetId: ruleSetId,
    data: TestRuleSetParams(
      category: "group",  // "group" or "member" for resourceType "group"
      operation: "create",  // group: create|edit|delete|get; member: create|edit|delete|list
      user: TestRuleSetUser(userId: "user-123", role: "member"),
      memberships: [RuleSetMembership(groupType: "team", groupId: "engineering")],  // optional
      group: TestRuleSetGroup(
        groupType: "team", groupId: "engineering",
        name: "Eng", createdBy: "user-456"
      ),
      target: TestRuleSetTarget(userId: "user-456")  // for member.create/edit/delete
    )
  )
  // result: allowed, expression?, context?, trace?, error?

  // Debug against a real user (live memberships, full trace).
  // Requires console-admin auth — regular app callers get 403.
  let debug = try await client.ruleSets.debug(data: DebugRuleSetParams(
    userId: "user-123",
    groupType: "team",
    category: "member",
    operation: "create",
    groupId: "engineering",  // optional — omit for create
    targetUserId: "user-456"  // optional
  ))
```

`result.trace` shows every `isMemberOf`/`memberGroups`/`hasRole` call and its result. To update rule sets, edit the TOML file and run `primitive sync push`.

## Rule Sets for Collection Management

Collections (see the [Documents guide](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#collections)) use the same rule-set pipeline as groups — a `CollectionTypeConfig` row binds a `collectionType` to a rule set, and per-op fallback resolves missing ops against the built-in defaults. The CEL namespace is `collection.*` (separate from `group.*`), and an extra helper `hasCollectionAccess(collectionId)` is available only inside collection rule sets.

**Resource type:** `collection`. **Categories and operations:**

- `category: "collection"` — `create`, `edit`, `delete`, `get` (the read op, parallel to `group.get`; use `get` in TOML configs).
- `category: "document"` — `add`, `remove`, `delete`, `list` (controls which documents the collection can hold, plus authorization for deleting a member document outright — see [Deleting Documents](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#deleting-documents)).
- `category: "member"` — `add`, `remove`, `list`.

**Default rule set:** A collection type with no `CollectionTypeConfig` row falls back to:

| Op | Default | Meaning |
|----|---------|---------|
| `collection.create` | `"true"` | Any signed-in member |
| `collection.edit` / `delete` | `user.userId == collection.createdBy` | Creator only |
| `collection.get` | `user.userId == collection.createdBy \|\| hasCollectionAccess(collection.collectionId)` | Creator or collection member (direct or via `CollectionGroupPermission`) |
| `document.add` / `remove` | `user.userId == collection.createdBy` | Creator only |
| `document.delete` | `"false"` | Denied. Unlike every other write op above, NOT creator-only — an app must explicitly configure this op to let a non-owner/non-app-owner delete a member document at all |
| `document.list` | `user.userId == collection.createdBy \|\| hasCollectionAccess(collection.collectionId)` | Creator or collection member |
| `member.add` / `remove` | `user.userId == collection.createdBy` | Creator only |
| `member.list` | `user.userId == collection.createdBy \|\| hasCollectionAccess(collection.collectionId)` | Creator or collection member |

Per-op fallback applies — configured ops always win, missing ops resolve against this table. A `CollectionTypeConfig` row whose `ruleSetId` is null is an explicit opt-out and denies everything except admin/owner. A non-creator reader/writer removing their own membership via `member.remove` is denied (403) unless the rule set grants it.

`document.delete` is distinct from `document.remove`: `remove` only detaches a document from this collection, while `delete` authorizes destroying the whole document (`client.documents.delete`) when the caller isn't the document's owner or the app owner — the delete endpoint checks every collection containing the document and allows the delete if any one collection's `document.delete` rule passes. Because that check runs per collection, granting `document.add` on a collection can extend who is able to delete documents placed in it — configure `document.add` and `document.delete` together with that reach in mind.

### CEL context for collection rule sets

| Variable | Always present? | Description |
|----------|-----------------|-------------|
| `user.userId` | yes | Requesting user's ID |
| `user.role` | yes | App role (`"owner"` / `"admin"` / `"member"`) |
| `collection.collectionType` | yes | Collection's type (matches the `CollectionTypeConfig` this rule set is bound to) |
| `collection.collectionId` | yes (after create) | Collection's ID |
| `collection.contextId` | yes | Per-instance identifier — parallels a group's `groupId`. Set at create time and immutable. `null` for collections with no context. Expresses "caller belongs to the group this collection represents." Prefer storing that external id in a [resource metadata](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md) category and reading it as `md.self.<category>.<key>` — see [Migrating `contextId` to a metadata category](#migrating-contextid-to-a-metadata-category). |
| `collection.name` | yes | Display name |
| `collection.createdBy` | yes (after create) | userId of the collection's creator |
| `target.userId` | only `category: "member"`, ops `add` / `remove` | The user being added or removed. Absent for `member.list`. |

Plus the standard CEL functions (`isMemberOf`, `memberGroups`, `hasRole`, `now()`, `fromWorkflow()`) and the collection-only helper:

- `hasCollectionAccess(collectionId)` — true when the caller has direct collection membership (the platform-managed `_col-reader` / `_col-writer` system groups) OR membership in a non-system user-group that holds a `CollectionGroupPermission` of `reader` or `read-write` on the collection. Resolves to `false` outside collection rule sets, and to `false` on `collection.create` (no `collectionId` in scope yet).

### Defining a collection rule set

```toml
# config/rule-sets/class-reports-rules.toml
[ruleSet]
name = "class-reports-rules"
resourceType = "collection"
description = "Per-class collections — only members of the class can see/list reports"

[rules.collection]
create = "isMemberOf('class', collection.contextId)"  # only class members can create their class's collection
edit   = "user.userId == collection.createdBy"
delete = "user.userId == collection.createdBy"
get    = "isMemberOf('class', collection.contextId) || hasCollectionAccess(collection.collectionId)"

[rules.document]
add    = "isMemberOf('class', collection.contextId)"  # any class member can add reports
remove = "user.userId == collection.createdBy"
list   = "isMemberOf('class', collection.contextId) || hasCollectionAccess(collection.collectionId)"
```

Bind the rule set to a collection type the same way as groups:

```toml
# config/collection-type-configs/class-reports.toml
[collectionTypeConfig]
collectionType = "class-reports"
ruleSetName    = "class-reports-rules"
```

Then push with `primitive sync push`. The SDK equivalents are `client.collectionTypeConfigs.{ list, get, create, update, delete }` (parallel to `client.groupTypeConfigs.*`).

### Migrating `contextId` to a metadata category

Prefer storing a collection's external-entity id in a [resource metadata](AGENT_GUIDE_TO_PRIMITIVE_RESOURCE_METADATA.md) category and reading it in the rule set as `md.self.<category>.<key>`, rather than in the built-in `collection.contextId` field. **The move is not 1:1** — a rule can read `md.self.<category>.<key>` only for a category the collection type's manifest declares, and only after a value has been stored, so migrating means declaring a manifest and stamping the value, not just renaming a field.

**1. Define a category** for the link, with separate read/write rules:

```toml
# config/metadata-category-configs/collection.classLink.toml
[metadataCategoryConfig]
resourceType = "collection"
category = "classLink"
readRule = "true"
writeRule = "user.userId == resource.attrs.createdBy"

[metadataCategoryConfig.schema.fields.classId]
type = "string"
required = true
```

**2. Declare the category on the collection type's manifest** — the prerequisite for the rule set to read it:

```toml
# config/collection-type-configs/class-reports.toml
[collectionTypeConfig]
collectionType = "class-reports"
ruleSetName    = "class-reports-rules"

[metadata.self]
categories = ["classLink"]
```

**3. Replace `collection.contextId` with `md.self.classLink.classId`** in the rule set, and stamp `classId` when the collection is created (via `initialMetadata` on `collections.create()`) so the value exists when these ops evaluate:

```toml
[rules.collection]
get    = "isMemberOf('class', md.self.classLink.classId) || hasCollectionAccess(collection.collectionId)"

[rules.document]
add    = "isMemberOf('class', md.self.classLink.classId)"
list   = "isMemberOf('class', md.self.classLink.classId) || hasCollectionAccess(collection.collectionId)"
```

The `collection.create` rule is the exception: the collection and its metadata don't exist yet when it evaluates, so `md.self.classLink.classId` reads `null` there. Gate `create` on caller identity or membership another way (for example a group the caller must already belong to), and let the post-create ops above carry the `md.self` check.

## Common Patterns

For full app architecture examples showing how groups fit alongside documents and databases, see the [Data Modeling guide](AGENT_GUIDE_TO_PRIMITIVE_DATA_MODELING.md#worked-architectures).

### Team-based workspace access

Users create teams. Team members get access to team documents and databases.

**Setup** (via CLI config):

**File:** `config/group-type-configs/team.toml`

```toml
[groupTypeConfig]
groupType = "team"
autoAddCreator = true
```

**Runtime** (in app code):

```swift
  // User creates a team.
  _ = try await client.groups.create(params: CreateGroupParams(
    groupType: "team",
    groupId: "alpha-team",
    name: "Alpha Team"
  ))

  // Share a document with the team — every member inherits read-write.
  _ = try await client.documents.grantGroupPermission(
    documentId: docId,
    params: GrantGroupPermissionParams(
      groupType: "team",
      groupId: "alpha-team",
      permission: "read-write"
    )
  )
```

Gate the team's database operations on membership in CEL: `access: "isMemberOf('team', database.metadata.teamId)"`.

### Role-based access (reviewer, editor, viewer)

Use group types as roles within a context.

```swift
  // Create role groups for a project.
  _ = try await client.groups.create(params: CreateGroupParams(
    groupType: "editor", groupId: "project-1", name: "Project 1 Editors"))
  _ = try await client.groups.create(params: CreateGroupParams(
    groupType: "viewer", groupId: "project-1", name: "Project 1 Viewers"))

  // Grant different document permissions per role.
  _ = try await client.documents.grantGroupPermission(
    documentId: docId,
    params: GrantGroupPermissionParams(
      groupType: "editor", groupId: "project-1", permission: "read-write"))
  _ = try await client.documents.grantGroupPermission(
    documentId: docId,
    params: GrantGroupPermissionParams(
      groupType: "viewer", groupId: "project-1", permission: "reader"))
```

Gate the operations on role membership in CEL — editors can modify (`access: "isMemberOf('editor', params.projectId)"`); viewers can read (`access: "isMemberOf('viewer', params.projectId) || isMemberOf('editor', params.projectId)"`).

### Relationship modeling (parent-child, mentor-mentee)

Use groups to model relationships between users.

**Setup** (via CLI config):

**File:** `config/database-types/classroom.toml` (excerpt)

```toml
[[operations]]
name = "viewGrades"
type = "query"
modelName = "grades"
access = "true"
[operations.definition]
filter = { studentId = "$params.studentId" }
sort = { date = -1 }

[[operations.params]]
name = "studentId"
type = "string"
required = true
access = "value in memberGroups('parent-of')"
```

The per-parameter `access` expression ensures parents can only view their own children's grades.

**Runtime** (in app code):

```swift
  // A "parent-of" group per student.
  _ = try await client.groups.create(params: CreateGroupParams(
    groupType: "parent-of",
    groupId: "student-123",
    name: "Parents of Student 123"
  ))
  _ = try await client.groups.addMember(
    groupType: "parent-of", groupId: "student-123",
    params: .userId(parentUserId)
  )

  // Parent queries their child's grades — server enforces access.
  let grades = try await client.databases.executeOperation(
    databaseId: dbId, name: "viewGrades",
    options: ExecuteOperationOptions(params: ["studentId": "student-123"])
  )
  _ = grades
```

### Organization hierarchy

Model nested organizational structure with multiple group types.

```swift
  // Organization-level group.
  _ = try await client.groups.create(params: CreateGroupParams(
    groupType: "org", groupId: "acme-corp", name: "Acme Corp"))

  // Department-level groups.
  _ = try await client.groups.create(params: CreateGroupParams(
    groupType: "dept", groupId: "engineering", name: "Engineering"))
  _ = try await client.groups.create(params: CreateGroupParams(
    groupType: "dept", groupId: "marketing", name: "Marketing"))

  // Team-level group.
  _ = try await client.groups.create(params: CreateGroupParams(
    groupType: "team", groupId: "backend", name: "Backend Team"))

  // A user can be in multiple groups at different levels.
  _ = try await client.groups.addMember(
    groupType: "org", groupId: "acme-corp", params: .userId(userId))
  _ = try await client.groups.addMember(
    groupType: "dept", groupId: "engineering", params: .userId(userId))
  _ = try await client.groups.addMember(
    groupType: "team", groupId: "backend", params: .userId(userId))
```

CEL can check any level: `"isMemberOf('org', database.metadata.orgId)"`, `"isMemberOf('dept', params.deptId)"`, `"isMemberOf('team', database.metadata.teamId)"`.

## Best Practices

### Users

- **Always use the platform user model** for identity. Reference `userId` from the platform, don't generate your own user IDs.
- **Use `client.users.getBasic()`** to display user info (name, avatar, email). It caches results automatically.
- **Store supplemental user data** in the root document (personal settings) or in a database (public profile data accessible via operations).
- **Don't duplicate platform fields.** Name, email, and avatar are managed by the platform — read them from there.

### Groups

- **Choose meaningful group types.** Use types that map to your domain: `team`, `class`, `department`, `parent-of`. The `groupType` is the taxonomy, the `groupId` is the instance.
- **Use `autoAddCreator: true`** (default) for groups where the creator should be a member (teams, clubs). Set to `false` for groups managed by admins (classes, departments).
- **Prefer groups over per-user grants** for document access. Easier to manage and audit.
- **Use CEL `isMemberOf()` for database access** rather than trying to grant database permissions to individual users.

### Access control

- **App owners and admins bypass all group/collection rule-set evaluation.** Design rules for regular members — don't try to restrict owners/admins there.
- **Database operation `access` CEL is NOT bypassed for admins.** If admins should be able to call an operation, include them explicitly: `hasRole('admin') || hasRole('owner') || isMemberOf(...)`.
- **Use per-parameter access** for sensitive relationships (parent-child, manager-report).
- **Keep rule sets simple.** Complex nested CEL expressions are hard to debug. Prefer multiple focused operations over one operation with complex access logic.
- **Test rules** with `client.ruleSets.test()` before deploying, and use `client.ruleSets.debug()` to trace evaluation for real users.

## Common Errors

| Symptom | Cause | Fix |
|---------|-------|-----|
| `addMember` returns `status: "pending_signup"` | Email isn't an app user yet | Expected. Membership resolves when they sign up or accept via `inviteToken`. Render a pending-members UI; cancel via `removeMemberByEmail(groupType:groupId:email:)` or `invitations.revokeDeferredGrant(deferredId:type: .group)`. |
| `addMember` returns `status: "already_member"` | User already in the group | Idempotent — no error. |
| 409 on `groups.create` | A group with that `(groupType, groupId)` already exists | Use a different `groupId` or call `groups.get` first. |
| 403 on `groups.create` (member role) | The group type has a config with no rule set attached (explicit opt-out), OR a configured `group.create` rule denied the caller | Either delete the group type config to fall back to the permissive default (`group.create = "true"`), attach a rule set with a permissive `group.create`, or call as an owner/admin. |
| 403 on `groups.create` with `groupType` starting with `_` | Reserved system group type | Pick a different prefix. |
| Group permission not taking effect on document | User hasn't reopened the document | Close and reopen the document to pick up new group permissions. |
| CEL `isMemberOf` returns false unexpectedly | Wrong `groupType`/`groupId` casing, user not yet added (deferred), or membership cache stale | Verify with `listUserMemberships(userId)`; remember email-based adds defer until sign-up. |
| Rule evaluation always denies, no obvious reason | Expression references a field not in the context (e.g. `user.appRole`, `group.description`) — runtime errors are silently turned into deny | Run `client.ruleSets.debug({...})` and inspect `trace`/`expression`/`context`. |
| Can't restrict admin/owner access via rules | By design — app owner/admin bypass all rule evaluation | Use group memberships for fine-grained access; don't try to gate admins. |

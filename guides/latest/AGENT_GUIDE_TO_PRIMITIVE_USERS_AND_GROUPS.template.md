# Users, Groups, and Access Control in Primitive

Guidelines for modeling user relationships and managing access control with Primitive's built-in user system and groups.

## Core operations

The key client calls, compiled against the real clients as part of the docs build. The dense reference below adds the result shapes, result-status branching, and CEL access-control detail.

### Look up users

{{ example: users-and-groups/users-lookup }}

Per-user lookups use `client.users.getBasic(id)`, `client.users.getProfiles([ids])`, and `client.users.lookup(email)`; the current user lives on `client.me`. Enumerate all users via the CLI (`primitive users list`) or the admin REST surface.

### Manage group membership

{{ example: users-and-groups/group-membership }}

### Grant document access to a group

{{ example: users-and-groups/grant-group-document }}

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

`getBasic` results are cached with a 5-minute default TTL. Override per call via `GetUserOptions`.

{{#lang ts}}
Its fields: `refreshNetwork` and `refreshIfOlderThanMs` (both refresh *behind* the cached value, so the call returns straight away and the *next* read is current), `waitForLoad` (`"local" | "network" | "localIfAvailableElseNetwork"`), `serverTimeoutMs`. Use `waitForLoad: "network"` when the call must wait for fresh server data.
{{/lang}}
{{#lang swift}}
Its fields: `refreshNetwork` and `refreshIfOlderThan` (both refresh *behind* the cached value, so the call returns straight away and the *next* read is current), `waitForLoad` (a `WaitForLoadMode`: `.local`, `.network`, `.localIfAvailableElseNetwork`), `serverTimeout`. The two durations are `TimeInterval`s in seconds. Use `waitForLoad: .network` when the call must wait for fresh server data.
{{/lang}}

### Supplementing user data — not replacing it

For app-specific user data (preferences, settings, profile fields beyond name/email/avatar), extend the platform user rather than creating a parallel user model:

**Do this:**

{{ example: users-and-groups/supplement-user-data }}

**Don't do this:**

{{#lang ts}}
```typescript
// DON'T create a separate user model that duplicates platform fields
const user = new AppUser();
user.email = "alice@example.com"; // Already managed by platform
user.name = "Alice";              // Already managed by platform
user.id = generateNewId();        // Use platform userId instead
```
{{/lang}}
{{#lang swift}}
```swift
// DON'T create a separate user model that duplicates platform fields
var user = AppUser()
user.email = "alice@example.com"  // Already managed by platform
user.name = "Alice"               // Already managed by platform
user.id = generateNewId()         // Use platform userId instead
```
{{/lang}}

### When to store additional user data

| Storage | Use case |
|---------|----------|
| **Root document** | Personal settings, preferences (auto-created per user, never shared) |
| **User's document** | App-specific profile data that follows document sharing rules |
| **Database** | User metadata visible to other users (e.g., public profile, reputation score), read and written through a server function |

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
# only owner cannot be disabled. `users remove` detaches the user instead.
primitive users disable <user-id> [-y]
primitive users enable <user-id>
```

Console admin accounts have their own pair, `primitive admins disable <admin-id>` / `primitive admins enable <admin-id>` — super-admin only, like every `primitive admins` verb.

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
- In CEL rule contexts the field is `user.role` (NOT `user.appRole`). See the [Access Control guide's identity context](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#identity-context-available-everywhere).

## Current User: `client.me`

The current authenticated user has its own namespace. Use it for "me"-scoped reads and writes — profile, avatar, the user's owned and shared documents, and pending document invitations.

### Profile

{{ example: users-and-groups/me-profile }}

`get()` is cache-backed; `update()` and `uploadAvatar()` clear that cache automatically. `get()` returns `{ userId, email, name, appRole, appId, avatarUrl? }` (null when signed out).

### Documents view

{{ example: users-and-groups/me-documents }}

{{#lang ts}}
- `ownedDocuments()` is cache-backed and offline-aware. Pass `returnPage: true` to get a paginated `DocumentListPage`; the root document is excluded by default (`includeRoot: false`).
{{/lang}}
{{#lang swift}}
- `ownedDocuments()` is cache-backed and offline-aware and returns `[DocumentInfo]`. Use `ownedDocumentsPage(...)` when you need a paginated `DocumentListPage`; the root document is excluded by default (`includeRoot: false`).
{{/lang}}
- `sharedDocuments()` returns the unified `{ items, cursor }` envelope (raw-JSON cursor, NOT base64url). Group- and collection-scoped shares do NOT appear here — those are accessed via the group or collection. Each `SharedDocument` extends `DocumentInfo`, so rows carry the base document fields plus the share extras (`permission`, `source`, `grantedBy`).

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
4. Check membership where database data is read: in a server function's `access` gate (`isMemberOf('team', 'engineering')`) or, when the group depends on the input, in its code.

The full create/list/get/update/delete surface is in [Managing Groups](#managing-groups); membership in [Managing Members](#managing-members).

## Managing Groups

### Create / List / Get / Update / Delete

{{ example: users-and-groups/group-crud }}

If the group type has `autoAddCreator: true` (default), the creator is automatically added as a member.

`groupId` on create is optional — a supplied id keeps its existing validation (`#` banned, no `_`-prefixed reserved type, `409` on a duplicate); a non-string supplied value is rejected with `400 "groupId must be a string"`, an empty string with `400 "groupId cannot be empty"`. Omit it (or send `null`) to have the server assign a ULID, returned as `groupId` in the response — the same pattern documents, databases, and collections use for server-assigned ids.

`groups.list()` returns a paginated `{ items: GroupInfo[], cursor? }`. `ListGroupsOptions` supports `type`, `limit`, `cursor`, and `includeSystem: true` to include platform-managed internal groups whose `groupType` is prefixed with `_` (e.g. `_col-reader`/`_col-writer` backing collection sharing). These are filtered out by default — only set `includeSystem` for admin tooling.

`groups.get()` returns `{ appId, groupType, groupId, name, description?, memberCount, createdAt, createdBy, modifiedAt }`. `update()` takes optional `name` and/or `description`. `delete()` cascade-deletes all memberships and group permissions.

## Managing Members

### Add members

Add a member by `userId` (always direct) or by `email` (direct if the email maps to an app user, deferred otherwise). Pass **either** `userId` or `email`, never both — the server rejects requests carrying both.
{{#lang ts}}
`AddGroupMemberParams` is a discriminated union and the types enforce the either/or at compile time.
{{/lang}}

`addMember` returns a result you branch on by `status` (`DirectGroupAdd | DeferredGroupAdd`):

{{ example: users-and-groups/add-member-branch }}

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

{{ example: users-and-groups/list-members }}

{{#lang ts}}
Pass `{ include: "profiles" }` to join each member's profile in the same call. Without it, `GroupMemberInfo` is the sparse shape: `{ userId, addedAt, addedBy, userName?, userEmail? }` — no `avatarUrl` key at all. With `include: "profiles"`, `userName`/`userEmail` become reliably populated and a new `avatarUrl?: string | null` field is always present: a resolved URL to the uploaded avatar, or `null` when the user has no avatar or the membership is orphaned. Orphaned memberships (the user was deleted) still appear in the list either way — only the profile fields go null. `include` is opt-in and purely additive: omitting it returns exactly the sparse shape above.
{{/lang}}
{{#lang swift}}
Pass `include: .profiles` to join each member's profile in the same call. Without it, `GroupMemberInfo.avatarUrl` is `nil` — the field is absent from the default response. With `include: .profiles`, `userName`/`userEmail` become reliably populated and `avatarUrl` is a resolved URL to the uploaded avatar, or `nil` when the user has no avatar or the membership is orphaned. Orphaned memberships (the user was deleted) still appear in the list either way — only the profile fields go nil. `include` is opt-in and purely additive: omitting it returns exactly the sparse shape above. The server rejects unknown `include` values with HTTP 400.
{{/lang}}

### Remove members

{{ example: users-and-groups/remove-member }}

The email form is the right call when you don't know — and don't want to branch on — whether the target has signed up yet. If a direct membership exists, it's removed; otherwise the pending `DeferredGroupAdd` for that email is canceled. Revoke the whole `AppInvitation` only when you want to cancel **every** grant attached to it (group add, document share, app-join right) at once.

### List pending invitations for a group

{{ example: users-and-groups/group-pending-invitations }}

Use this to render the "pending members" section of a group sharing UI without having to filter the lower-level `client.invitations.listDeferredGrants()` surface.

Authorization is the same gate as `listMembers`: the group's `member.list` rule, with app admins/owners always allowed and direct members of the group allowed as a fallback even under a stricter custom rule. A cross-group manager whose rules let them list a group's members can therefore also see its pending invitations — being a member of the group is not required.
Each entry carries a `deferredId` — cancel that invitation by revoking the deferred grant{{#lang ts}}: `client.invitations.revokeDeferredGrant(deferredId, "group")`{{/lang}}{{#lang swift}}: `client.invitations.revokeDeferredGrant(deferredId: entry.deferredId, type: .group)`{{/lang}}. Cancelling an invitation another caller created is allowed for app admins/owners, the invitation's creator, or a caller who passes the group's `member.delete` rule (evaluated with `target.email` set to the invitee's email, so email-scoped rules work on the pending path).

### List a user's memberships

{{ example: users-and-groups/user-memberships }}

## Group Type Configuration

Group types are configured via TOML config files and the `primitive config` command (version-controlled alongside your code).

**File:** `primitive/dev/group-type-configs/team.toml`

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
primitive config push
```

**Defaults & gotchas:**
- `autoAddCreator` defaults to `true` **only when a group type config exists** for the type. With **no config at all**, no auto-add happens.
- A group type with **no config** falls back to built-in default rules. Per-op fallback also applies: when a configured rule set leaves a `(category, op)` pair undefined, that op resolves against the defaults too. The defaults: `group.create = "true"` (any signed-in member); `group.edit/delete` and `member.create/edit/delete` are creator-only (`user.userId == group.createdBy`); `group.get` and `member.list` allow the creator OR any direct group member (`isMemberOf(group.groupType, group.groupId)`).
- A group type config with no `ruleSetName` (`ruleSetId: null`) is an **explicit opt-out** and denies everything except admin/owner; it does NOT fall through to defaults. To re-enable defaults, delete the config entirely — remove `group-type-configs/<group-type>.toml` and run `primitive config push --prune`, or call `client.groupTypeConfigs.delete(groupType)`.

See the [Configuration guide](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md#the-sync-loop) for the full sync loop (`init`, `pull`, `diff`, `push`).

## Groups and Documents

Grant a group access to a document. All members inherit the permission.

The grant call is shown in [Grant document access to a group](#grant-document-access-to-a-group) above (permission is `"read-write"` or `"reader"` — NOT `"write"` or `"view"`). To inspect what a group reaches and to manage a document's group grants:

{{ example: users-and-groups/group-resources }}

**Permission resolution:** A user's effective permission on a document is the **highest** across their direct permission and all group memberships. If a user has `reader` direct access but their team has `read-write`, they get `read-write`.

## Groups and Databases

A database can carry a group grant that records every member of the group as a `manager` — an administrative permission record (the only level a group can hold), not a way into the data: the database routes refuse a non-admin caller, so the grant gives members nothing directly. App admins manage it, and a function reads it with `ctx.api.databases.listGroupPermissions` to make its own decisions. A function makes it, declaring `capabilities = ["databases:grantGroupPermission"]`:

```ts
await ctx.api.databases.grantGroupPermission({
  databaseId: input.databaseId,
  body: { groupType: "team", groupId: "ops", permission: "manager" },
});
```

End users reach database records through **server functions**, so group membership gates data in two places — see the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#access):

- **The function's `access` gate** — CEL over the caller's identity: `user.userId`, `user.role`, `isMemberOf`, `memberGroups`, `hasRole` (the shared identity context, documented in the [Access Control guide](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#identity-context-available-everywhere)). The gate does not see the function's input, so it checks fixed groups and roles. App owners and admins bypass it.
- **The function's code** — for a group that depends on the input (the team that owns `input.databaseId`, the student in `input.studentId`), read the caller's memberships and refuse before touching the rows.

**Gate patterns:**

```
// Any member of one fixed group
access = "isMemberOf('staff', 'all')"

// Either of two fixed groups
access = "isMemberOf('staff', 'all') || isMemberOf('support', 'tier-2')"
```

**Code pattern — membership that depends on the input:**

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { databaseId: string }, ctx) => {
  // A bare array of { groupType, groupId, name, … }.
  const teams = await ctx.api.groups.listUserMemberships({ userId: ctx.user!.userId, type: "team" });
  if (!teams.some((m: { groupId: string }) => m.groupId === input.databaseId)) {
    throw new Error("Not a member of this team");
  }
  return await ctx.db(input.databaseId, "team").model("tasks").query({});
});
```

**Don't do this — common CEL footguns:**

```
// BAD — `user.appRole` does not exist in CEL. Use `user.role` or hasRole().
access = "user.appRole == 'admin'"

// GOOD — checks the caller's app role.
access = "hasRole('editor-in-chief')"

// BAD — isMemberOf returns bool, not the group. `==` is meaningless.
access = "isMemberOf('team') == 'engineering'"

// GOOD
access = "isMemberOf('team', 'engineering')"

// BAD — memberGroups returns an array, can't compare with ==.
access = "memberGroups('team') == 'engineering'"

// GOOD — use `in` for containment.
access = "'engineering' in memberGroups('team')"

// BAD — referencing fields not in the context (silently denies; runtime
// errors are caught and turned into "deny").
access = "user.email == 'admin@example.com'"   // user.email is not in context
```

## Rule Sets for Groups

Group management operations (create/edit/delete, member add/remove) are gated by **rule sets** — a named bundle of CEL rules per `(category, operation)` pair, defined in `primitive/dev/rule-sets/*.toml` and bound to a group type (`ruleSetName` in the type config — see [Group Type Configuration](#group-type-configuration) above for the binding and per-op fallback rule). The mechanism itself — defining and binding a rule set, `memberGroupsOf` for subject-form membership, owner/admin bypass, `test()`/`debug()` — is documented once in the [Access Control guide's rule sets section](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#rule-sets-management-operations); read that first. This section covers only what's specific to groups: which operations exist, the CEL context each adds, and the built-in defaults. Collections use the same rule-set mechanism under a separate `collection.*` namespace — see [Collection Rule Sets](AGENT_GUIDE_TO_PRIMITIVE_DOCUMENTS.md#collection-rule-sets) in the Documents guide.

**Resource type:** `group`. **Categories and operations:**
- `category: "group"` — `create`, `edit`, `delete`, `get` (the read op; use `get` in TOML configs — there is no `read`/`update`).
- `category: "member"` — `create`, `edit`, `delete`, `list` (there is no `add` — use `create` for "add member").

**Default rule set** — applies to any group type with no group type config:

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

**CEL context**, beyond the [identity context](AGENT_GUIDE_TO_PRIMITIVE_ACCESS_CONTROL.md#identity-context-available-everywhere):

| Variable | Always present? | Description |
|----------|-----------------|-------------|
| `group.groupType` | yes | Target group's type |
| `group.groupId` | yes | Target group's ID (also present at `create` time — server passes the requested ID) |
| `group.contextId` | yes | Read-alias of `group.groupId` — the same value under the `contextId` name collection rules use, so group and collection rule sets can share expressions |
| `group.name` | yes | Target group's display name |
| `group.createdBy` | yes (after create) | userId of the group creator |
| `target.userId` | only `category: "member"`, ops `create`/`edit`/`delete` | Target user being added/removed. Absent for `member.list`. |

`group.description` is **NOT** in the rule context — don't reference it.

{{ example: users-and-groups/rule-sets-test }}

## Common Patterns

For full app architecture examples showing how groups fit alongside documents and databases, see the [Data Modeling guide](AGENT_GUIDE_TO_PRIMITIVE_DATA_MODELING.md#worked-architectures).

### Team-based workspace access

Users create teams. Team members get access to team documents and databases.

**Setup** (via CLI config):

**File:** `primitive/dev/group-type-configs/team.toml`

```toml
[groupTypeConfig]
groupType = "team"
autoAddCreator = true
```

**Runtime** (in app code):

{{ example: users-and-groups/team-workspace }}

When each team has its own database (the database id reused as the team's group id), the function that reads it checks the caller's `team` membership against `input.databaseId` in code — see [Groups and Databases](#groups-and-databases).

### Role-based access (reviewer, editor, viewer)

Use group types as roles within a context.

{{ example: users-and-groups/role-groups }}

The project comes from the input, so the function checks role membership in code: a write function requires an `editor` membership whose `groupId` is `input.projectId`; a read function accepts `viewer` or `editor`.

### Relationship modeling (parent-child, mentor-mentee)

Use groups to model relationships between users.

**Server side** — the function that reads a student's grades checks the relationship before it reads:

```ts
import { defineFunction } from "primitive-functions";

export default defineFunction(async (input: { databaseId: string; studentId: string }, ctx) => {
  const children = await ctx.api.groups.listUserMemberships({ userId: ctx.user!.userId, type: "parent-of" });
  if (!children.some((m: { groupId: string }) => m.groupId === input.studentId)) {
    throw new Error("Not this student's parent");
  }
  return await ctx.db(input.databaseId, "classroom").model("grades").query({
    filter: { studentId: input.studentId },
    options: { sort: { date: -1 } },
  });
});
```

The membership check is what ensures parents can only view their own children's grades; the caller never supplies whose relationship is checked.

**Runtime** (in app code):

{{ example: users-and-groups/relationship-groups }}

### Organization hierarchy

Model nested organizational structure with multiple group types.

{{ example: users-and-groups/org-hierarchy }}

A function can check any level: a fixed one in its `access` gate (`isMemberOf('org', 'acme')`), and one named by the input in its code, with `listUserMemberships({ userId, type: "dept" })`.

## Best Practices

### Users

- **Always use the platform user model** for identity. Reference `userId` from the platform, don't generate your own user IDs.
- **Use `client.users.getBasic()`** to display user info (name, avatar, email). It caches results automatically.
- **Store supplemental user data** in the root document (personal settings) or in a database (public profile data, read through a server function).
- **Don't duplicate platform fields.** Name, email, and avatar are managed by the platform — read them from there.

### Groups

- **Choose meaningful group types.** Use types that map to your domain: `team`, `class`, `department`, `parent-of`. The `groupType` is the taxonomy, the `groupId` is the instance.
- **Use `autoAddCreator: true`** (default) for groups where the creator should be a member (teams, clubs). Set to `false` for groups managed by admins (classes, departments).
- **Prefer groups over per-user grants** for document access. Easier to manage and audit.
- **Check group membership in the function** that reads a database, rather than granting database permissions to individual users.

### Access control

- **App owners and admins bypass all group/collection rule-set evaluation.** Design rules for regular members — don't try to restrict owners/admins there.
- **A function's `access` gate is bypassed by app owners and admins too.** A check in the function's code is not — write it to let them through if they should be.
- **Check sensitive relationships in code** (parent-child, manager-report): read the caller's memberships and compare against the input.
- **Keep rule sets simple.** Complex nested CEL expressions are hard to debug. Prefer several focused functions over one function with complex gate logic.
- **Test rules** with `client.ruleSets.test()` before deploying, and use `client.ruleSets.debug()` to trace evaluation for real users.

## Common Errors

| Symptom | Cause | Fix |
|---------|-------|-----|
{{#lang ts}}
| `addMember` returns `status: "pending_signup"` | Email isn't an app user yet | Expected. Membership resolves when they sign up or accept via `inviteToken`. Render a pending-members UI; cancel via `removeMember({ email })` or `invitations.revokeDeferredGrant(deferredId, "group")`. |
{{/lang}}
{{#lang swift}}
| `addMember` returns `status: "pending_signup"` | Email isn't an app user yet | Expected. Membership resolves when they sign up or accept via `inviteToken`. Render a pending-members UI; cancel via `removeMemberByEmail(groupType:groupId:email:)` or `invitations.revokeDeferredGrant(deferredId:type: .group)`. |
{{/lang}}
| `addMember` returns `status: "already_member"` | User already in the group | Idempotent — no error. |
| 409 on `groups.create` | A group with that `(groupType, groupId)` already exists | Use a different `groupId` or call `groups.get` first. |
| 403 on `groups.create` (member role) | The group type has a config with no rule set attached (explicit opt-out), OR a configured `group.create` rule denied the caller | Either delete the group type config to fall back to the permissive default (`group.create = "true"`), attach a rule set with a permissive `group.create`, or call as an owner/admin. |
| 403 on `groups.create` with `groupType` starting with `_` | Reserved system group type | Pick a different prefix. |
| Group permission not taking effect on document | User hasn't reopened the document | Close and reopen the document to pick up new group permissions. |
| CEL `isMemberOf` returns false unexpectedly | Wrong `groupType`/`groupId` casing, user not yet added (deferred), or membership cache stale | Verify with `listUserMemberships(userId)`; remember email-based adds defer until sign-up. |
| Rule evaluation always denies, no obvious reason | Expression references a field not in the context (e.g. `user.appRole`, `group.description`) — runtime errors are silently turned into deny | Run `client.ruleSets.debug({...})` and inspect `trace`/`expression`/`context`. |
| Can't restrict admin/owner access via rules | By design — app owner/admin bypass all rule evaluation | Use group memberships for fine-grained access; don't try to gate admins. |

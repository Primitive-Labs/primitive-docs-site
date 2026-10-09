# Agent Guide to Primitive Configuration

Manage service configuration in `primitive/<env>/` as TOML, then apply it with
the CLI. Use the Admin Console for inspection and interactive testing.

## The sync loop

```bash
primitive config init
primitive config pull
# Edit the configuration files.
primitive config diff
primitive config push
```

Confirm the target with `primitive env show`. Each environment binds an API URL
and app ID in `primitive/config.json`. Select it with `--env <name>` or
`PRIMITIVE_ENV`; `primitive env use <name>` saves a machine-local selection.

## Child apps

A child app is a copy of the current environment's app — its `mode`, settings
and (unless `--no-secrets`) secrets — owned by whoever creates it and deleted
automatically once idle (7–30 days, 30 by default; it's warned by email
first). It runs your branch's configuration in isolation, so a push to it
reaches nobody else.

### When to create one

Recommend a child app whenever a change needs `config push` to be tested and
the environment you'd otherwise push to is used by anyone else: other
developers, live users, or you yourself from another agent or git worktree.
A push lands immediately, so pushing an unfinished branch to a shared
environment changes the app for everyone on it and gets overwritten by the
next push from another branch.

Pick the shape that matches how the work is done:

- **One developer, one line of work at a time:** one child per developer,
  used as that developer's own integration environment and re-pointed at each
  branch.
- **Parallel work in several worktrees or agents:** one child per worktree,
  each tracking that worktree's branch. Every git worktree keeps its own
  environment selection, so the children never cross.

Push to the shared environment directly only when the change is already
merged, or when nobody else uses that environment.

### The workflow

1. **Create the child from the branch's checkout.** Confirm the header names
   the parent environment first (`primitive env show`).

   ```bash
   primitive env use dev
   primitive apps children create feature-x
   ```

   `create` copies the parent's settings and secrets, registers the child as
   this checkout's machine-local environment `feature-x`, selects it, and
   pushes the checkout's tree into it. `--branch` defaults to the current git
   branch.

2. **Set up what isn't copied.** Users, documents, database records, blobs
   and other app data are not copied: the child starts empty. Sign in to it
   to create your user, then seed the data the change needs to be tested
   against. Every admin of the parent can already administer the child. Secrets the branch
   adds are set on the child with `primitive secrets set`.

3. **Iterate.** With the child selected, `config push`, `config diff` and
   `config pull` act on the child, and the app template's dev, test and
   codegen scripts resolve to it with no other setup. Edit the TOML, push, test,
   repeat. Before declaring the work done, merge the base branch in and push
   once more so the child is tested against the parent's current
   configuration.

4. **Merge, then apply to the parent.** After the branch merges, switch back
   and push the merged tree to the parent. Set any secret the branch added on
   the parent first, so the pushed configuration never references a secret
   the parent lacks.

   ```bash
   primitive env use dev
   primitive secrets set NEW_KEY --value <value>   # only secrets the branch added
   primitive config diff
   primitive config push
   ```

5. **Delete the child** once the parent has the change, or when the work is
   abandoned. Deleting removes its data and this checkout's registration.

   ```bash
   primitive apps children delete feature-x --yes
   ```

A teammate, or another of your worktrees, attaches an existing child with
`primitive env use feature-x`.

### Reference

```bash
primitive apps children create feature-x   # create, register, select, push the tree
primitive apps children list                # this app's children
primitive apps children get feature-x       # one child, and its local status
primitive apps children delete feature-x --yes
```

`create` takes `--branch <name>` (default: the checkout's current git
branch), `--idle-days <n>` (7–30), `--no-secrets`, and `--no-use`. It fails
closed: a create that errors on the server leaves no child behind.

A child shares its parent's `primitive/<env>/` tree but keeps its own
sync-state baseline and snapshot backups — `config pull`, `push`, and `diff`
on a child never touch the parent's committed state. With a child selected,
those three commands act on the child itself, but every `apps children`
command acts on its **parent**, so running `create` again makes a sibling,
not a grandchild. A pull on a child never writes `[app].name`, so a branch
merged back into the parent's tree keeps the parent's name.

`primitive env use <name>` selects a committed environment or an existing
local entry as usual; otherwise it looks the name up as a child slug of the
current environment's app, and if found, registers it, copies the login, and
selects it in one step. It refuses an unknown slug, and refuses a known one
only when that child's app id is already registered here under a different
name.

`primitive apps list` shows parent apps only; `--all` includes children.
Deleting a parent with children is refused, naming them;
`primitive apps delete <app-id> --with-children` deletes each child, then the
parent, stopping before the parent on the first child it can't delete.

See the deploying guide's [Preview a branch against a child app](AGENT_GUIDE_TO_PRIMITIVE_DEPLOYING.md#preview-a-branch-against-a-child-app)
for previewing a child's front end.

## Authoring

```bash
primitive config fields function
primitive config create function order-intake
primitive config set function/order-intake function.description="Order intake"
primitive config push --only function/order-intake
```

`fields` lists accepted keys, types, and defaults. `create` scaffolds a file;
`set` changes scalars locally. Edit arrays and tables directly. Use native
TOML tables for schemas and other structured values.

`--only` accepts `<type>/<key>`, `app`, or `var/<key>`. Secrets and resource
creation use their own commands; service configuration uses files and push.

## Directory map

```
primitive/<env>/
  app.toml                        # App settings
  vars.toml                       # Config vars for this environment
  prompts/*.toml                  # Managed prompts
  prompts/{key}.tests/*.toml      # Prompt test cases
  integrations/*.toml             # External API integrations
  integrations/{key}.tests/*.toml # Integration test cases (fixture files in integrations/{key}.tests/{case}/)
  functions/*.toml                # Server functions ([function] + [function.triggers.*])
  functions/{key}/**              # Function sources (the `entry` its TOML names)
  functions/*.d.ts                # Generated by config push (see Functions directory below)
  functions/generated/*.generated.ts # Generated: typed client invokers, one per function
  functions/tsconfig.json         # Scaffolded once: wires sources + declarations together
  database-type-configs/*.toml    # Database types: [models.*] schema, rule set, triggers, [metadata]
  blob-buckets/*.toml             # Blob bucket configs
  email-templates/*.toml          # Email template overrides
  rule-sets/*.toml                # Access rule sets
  group-type-configs/*.toml       # Group type configs
  collection-type-configs/*.toml  # Collection type configs
  metadata-category-configs/*.toml # Resource metadata category configs (schema + readRule/writeRule)
```

### Functions directory

`functions/<key>.toml` and the TypeScript its `entry` names are yours. Everything else under `functions/` is written by `config push` (and by `primitive functions codegen`, which writes the same files without pushing; `--check` exits non-zero when they are stale — the CI form):

- `primitive-db-types.d.ts` — the app's database types from `database-type-configs/*.toml`; types `ctx.db(databaseId, "<type>").model("<Model>")`.
- `primitive-functions.d.ts` — the `primitive-functions` module's own types.
- `primitive-function-types.d.ts` — `<Key>Input` / `<Key>Output` from each function's `inputSchema` / `outputSchema`.
- `primitive-document-types.d.ts` — document models from `models/models.toml`; types `ctx.doc(documentId).model("<Model>")`.
- `primitive-prompt-types.d.ts` — `<Key>PromptOutput` for each prompt with a `[prompt.outputSchema]`; types `ctx.prompts.run("<key>")`.
- `primitive-document-schema.generated.toml` — a copy of `models/models.toml` carried in each function version.
- `generated/<key>.generated.ts` — typed client invokers (both `invoke` and `start`); `primitive functions codegen -o <dir>` writes them elsewhere, `--lang swift` emits Swift.
- `tsconfig.json` — scaffolded once, then yours; wires the sources and declarations into one program.

Generated on every push, and none of it is sent to the server; see [Server Functions — Codegen](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#the-typed-handle) for the rewrite rule, the tsconfig repair, and `codegen --check`.

See [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md) for
function declarations and trigger configuration.

## App settings

Edit `app.toml` and use `primitive config push --only app`. Its sections are
`[app]`, `[auth]`, `[cors]`, and `[invitations]`. `[app].name` and `[app].mode`
are required. See the authentication guide for sign-in settings.

`[app].protected = true` marks an app as live with real users. Only the app's
console owner can set or clear it; a push from anyone else that would change
it fails with `PROTECTED_FLAG_OWNER_ONLY`. On a protected app, a scoped login
not pinned to it can only read; see
[Admin access](#admin-access-scopes-step-ups-and-ci-sessions).

Removing a managed setting clears it or restores its default. Keep explicit
values when a feature must stay disabled, especially
`emailSignInEnabled = false`.

## Gotchas

### Push failures

Preflight validates files before changes are applied. A `function` check
includes entry, capabilities, and trigger settings. Any local validation
failure aborts the push.

Apply is incremental: missing secrets, app limits, or conflicts can fail after
other changes succeed. There is no rollback. Resolve the reported problem and
push again; applying successful changes again is safe.

### Drift and conflicts

Server-only edits are reported as drift; edits on both sides produce conflicts.
Pull takes the server’s values. Use `--force` only to deliberately overwrite
remote changes. `diff --json` is available for automation.

Keep `.sync-state.json` committed with configuration. If an external deletion
leaves an entry pointing to a missing object, remove that object’s state entry
before pushing to recreate it. Pull also reconciles state and rewrites every
file that differs from the server; preserve local edits first. A file `diff`
reports as in sync keeps its bytes, comments and key order included.

### Deletion and availability

- Remove a managed field to clear it or restore its default on push.
- Remove a file and push with `--prune` to permanently delete its managed
  server object. Ordinary push leaves it in place.
- Pull removes previously managed files whose objects were deleted remotely.
  It preserves uncommitted changes; `--no-prune` preserves all local files.
- Use `disable` and `enable` for availability. Do not put object `status` in
  TOML. A prompt configuration’s `[[configs]] status` remains TOML-owned.
- Archive preserves a function, integration, or prompt and its key. To reuse
  the key, pull, prune the archived object, then add it again.
- Deleting a metadata category leaves its values unreachable. Delete values
  first if they must be removed.

Review prune output before confirming. `--yes` skips confirmation for an
already-reviewed automated operation.

### Restore a pull

Pull saves a local snapshot retained for 28 days. Revert replaces the config
directory, so inspect local edits first:

```bash
primitive config revert --list
primitive config revert --snapshot <id>
primitive config diff
```

### Test cases

A `<key>.tests/<case>.toml` filename identifies the case. Reference the
configuration and judge prompt by name — `configName`, `evaluatorPromptKey`,
`evaluatorConfigName` — so the file works unchanged in every app. Push changes
before running tests: runners use registered server cases, not local files.
Delete a case file and push with `--prune` to remove it.

## Secrets and email

Store credentials with `primitive secrets set` and reference
`{{secrets.KEY}}` in TOML. See the App Secrets guide.

Email overrides live in `email-templates/<emailType>.toml`. A `[template]`
requires `emailType` and `subject`; `htmlBody` and `textBody` are optional.
Use `email-sign-in` for sign-in email, retaining its code and expiry variables.
Delete an override and prune it to restore the built-in template.

## Admin access: scopes, step-ups and CI sessions

Every CLI login and token is an admin session with a scope, optionally pinned
to some apps. A plain `primitive login` holds the default scope, unpinned:
`read`, `write`, `data:read` and `data:write`. `primitive whoami` shows the
current session's kind, scope and pins.

The scopes are `read`, `write`, `admin`, `data:read` and `data:write`:

| Scope | Allows |
|---|---|
| `read` | Listing and reading configuration |
| `write` | Changing configuration, setting and deleting secrets, setting `protected`, deploying and activating functions and prompts |
| `data:read` | Reading documents and database records (admin API, app API, WebSocket) |
| `data:write` | Changing documents and database records |
| `admin` | Deleting apps, databases and users, removing collaborators, changing roles, hard deletes, clearing `protected` |

None implies another; a command needs every scope its route declares. The
data scopes guard direct access only: a token holding `write` can deploy and
activate code, and that code reads and writes the app's data whatever the
token's data scopes are.

**Protected apps.** On a protected app, anything beyond `read` needs a login
pinned to that app. Clearing `protected` also needs `admin`.

```bash
primitive login --scope read,write,data:read,data:write --app <app-id>
```

A pin also reaches the pinned app's child apps; a pin on a child app does not
reach its parent. Child apps cannot be protected.

**Step-up.** Deleting an app or a database, removing a collaborator, changing a
user's role and clearing `protected` need `admin`. Ask the user to run
`primitive login --scope admin` for a 15-minute step-up: the login's scopes
plus `admin`, pinned to the environment's app (`--app <id>` pins another).
Google re-prompts and the console asks for Approve. A command refused for
lacking `admin` retries once with the step-up; `primitive logout` ends both.

**A narrower token** for a script or an agent, at most 1 hour:

```bash
primitive token --scope read --app <app-id> --ttl 15m
```

It prints with no browser when it is no wider than the login, and opens the
approval page otherwise.

**CI sessions.** Create one from your own login, then run the CLI with the
printed token in `PRIMITIVE_TOKEN`; no credentials file is needed:

```bash
primitive auth sessions create --name deploys --scope read,write --app <app-id>
PRIMITIVE_TOKEN=<token> primitive config push
```

A session from `primitive auth sessions create` lasts 90 days, at most 1 year.
It is always pinned, never holds `admin`, and cannot be refreshed or derive
tokens. To log in as yourself without a browser, pipe a refresh token on stdin:

```bash
primitive token --refresh | primitive -e <env> login --token-stdin
```

**Sessions.** Every login, step-up and CI session is listed and revocable:

```bash
primitive auth sessions list
primitive auth sessions revoke <session-id>
```

**Refusals.** A refused command answers 403 with a `code` and, for a missing
scope or pin, prints the `primitive login …` command that grants it. A login
that can no longer be refreshed answers 401 instead:

| `code` | Means | Do |
|---|---|---|
| `INSUFFICIENT_SCOPE` | The login lacks a scope the route declares (`details.missing`) | Ask the user to run the printed `primitive login --scope …` command |
| `APP_NOT_PINNED` | The login's pins do not reach the app or route named | Confirm the command targets the intended app (`primitive whoami` shows the pins), then the printed command |
| `PROTECTED_APP_PIN_REQUIRED` | The app is protected and the login is not pinned to it | Ask the user to run the printed `--app` command |
| `PROTECTED_FLAG_OWNER_ONLY` | Only the app's console owner changes `protected` | `primitive config pull --only app` if `app.toml` is stale; otherwise the owner |
| `SESSION_UNSCOPED` | A 401 on refresh: the login predates scoped sessions | Ask the user to run `primitive login` |

### Gotchas

- **`write` deploys code that reaches data.** A `read,write` CI session cannot
  read data itself but can deploy a function that does. Grant `write` only
  where deploying is intended.
- **A parent pin reaches its child apps.** A token pinned to an app can change
  that app's children with the scopes it holds.
- **A printed `--app` does not mean the app is protected.** `APP_NOT_PINNED`
  and `INSUFFICIENT_SCOPE` print it too; act on the `code`.
- While `PRIMITIVE_TOKEN` is set it takes precedence over the stored login,
  and `primitive auth sessions create` is refused. Unset it to create sessions.

Target the environment explicitly in automated pushes. Keep tokens out of
configuration files and logs.

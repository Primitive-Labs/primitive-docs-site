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
  environment selection, so the children never cross. A linked worktree uses
  the main checkout's login, so it needs no `primitive login` (git 2.31 or
  newer); `primitive logout` in any worktree signs them all out.

Push to the shared environment directly only when the change is already
merged, or when nobody else uses that environment.

### The workflow

In an app made from the Vue or Swift template, [the template
scripts](#one-command-per-worktree-the-template-scripts) run steps 1–3 and
the delete in one command each. The steps by hand:

1. **Create the child from the branch's checkout.** Confirm the header names
   the parent environment first (`primitive env show`).

   ```bash
   primitive env use dev
   primitive apps children create feature-x
   ```

   `create` copies the parent's settings and secrets, registers the child as
   this checkout's machine-local environment `feature-x`, and selects it. It
   pushes nothing: its last line names the push to run once the child is set
   up (step 3). `--branch` defaults to the current git branch. Exit code 2
   means the child was created but isn't registered on this machine: run
   `primitive env use feature-x` to attach it, and don't create it again.
   Exit code 1 means no child was created.

2. **Set up what isn't copied.** Users, documents, database records, blobs
   and other app data are not copied: the child starts empty. An
   `invite-only` parent's `mode` carries over, so the child admits nobody —
   even a whitelisted `testAccountBaseEmails` address is stopped at the
   email step (waitlisted or rejected, exactly like a normal signup) until
   it's added directly with `primitive users create`; see [Test User
   Sign-In](AGENT_GUIDE_TO_PRIMITIVE_AUTHENTICATION.md#test-user-sign-in-per-app-whitelist).
   Otherwise, sign in to the child to create your user, then seed the data
   the change needs to be tested against — `databases create` makes an empty
   database, and `databases import` always mints a new database id on the
   child, so any var or document that names the parent's database id still
   needs updating to the child's. Every admin of the parent can already
   administer the child. Secrets the branch adds are set on the child with
   `primitive secrets set`. Config vars whose value belongs to the child —
   its database ids, URLs containing its app id, its origins — go in its
   override file (see Reference), set before the first push:

   ```bash
   primitive --env feature-x config set vars DB_ID=01M4VFW6B2WANJXA58FG810HDX
   ```

3. **Push the tree into the child**, with the command `create` printed:

   ```bash
   primitive --env feature-x config push --yes
   ```

4. **Iterate.** With the child selected, `config push`, `config diff` and
   `config pull` act on the child, and the app template's dev, test and
   codegen scripts resolve to it with no other setup. Edit the TOML, push, test,
   repeat. Before declaring the work done, merge the base branch in and push
   once more so the child is tested against the parent's current
   configuration.

5. **Merge, then apply to the parent.** After the branch merges, switch back
   and push the merged tree to the parent. Set any secret the branch added on
   the parent first, so the pushed configuration never references a secret
   the parent lacks.

   ```bash
   primitive env use dev
   primitive secrets set NEW_KEY --value <value>   # only secrets the branch added
   primitive config diff
   primitive config push
   ```

6. **Delete the child** once the parent has the change, or when the work is
   abandoned. Deleting removes its data and this checkout's registration.
   An `internal` app is deleted with `write`, so `apps children delete`
   needs no step-up: every child app is `internal`.

   ```bash
   primitive apps children delete feature-x --yes
   ```

A teammate, or another of your worktrees, attaches an existing child with
`primitive env use feature-x`.

### One command per worktree: the template scripts

Apps made from the Vue and Swift templates bring a child up for a branch, and
tear it down, with one command each. Run them with the parent environment
selected (or a sibling child, whose parent is then used):

```bash
pnpm child:up feature-x --port 5174   # Vue; --port defaults to 5173
pnpm child:down feature-x
scripts/child-app.sh up feature-x     # Swift
scripts/child-app.sh down feature-x
```

`up` creates the child, or attaches or resumes one that exists, checks that
`--env feature-x` resolves to that child, runs the app's
`scripts/child-app-init`, pushes the tree, allows the Vue dev server's
`http://localhost:<port>` origin, and selects the child. Every step is safe
to repeat, so rerunning `up` resumes an interrupted run. A Swift app is then
rebuilt so `primitive.json` names the child. `down` checks the child's
environment the same way, runs `scripts/child-app-teardown`, then deletes the
child and its local state.

The two hooks are the app's own code; the templates ship neither. Add one as
an executable file when the app needs it:

- **`scripts/child-app-init`** seeds what a fresh child needs: users,
  databases, the child's own config vars (`primitive config set vars
  KEY=value`). It runs before the first push because a function caches the
  config vars it reads, and on every `up`, so it must be safe to repeat.
- **`scripts/child-app-teardown`** revokes or deletes anything the child
  created outside the platform — third-party connections, webhooks, external
  buckets. That cleanup is the app's responsibility: deleting the child
  doesn't reach those services. `down --skip-teardown` deletes the child
  without running the hook and leaves them behind.

Each hook runs from the app root with `PRIMITIVE_ENV` (the child's local
environment name), `CHILD_APP_ID`, `CHILD_APP_SLUG` and `PARENT_ENV` set.
Address the child through `PRIMITIVE_ENV` or `--env`, never through the
selected environment: a hook may change the selection, and the scripts don't
rely on it afterwards. A hook that exits non-zero stops the script before the
push (`up`) or the delete (`down`); a hook that exists but isn't executable
is an error.

**Gotchas**

- A child's slug must not be the name of a committed environment in
  `primitive/config.json`: `up` refuses it before anything runs.
- `down` refuses a child that isn't registered on this machine, since its
  teardown hook couldn't act on it; attach it with `primitive env use <slug>`
  first, or pass `--skip-teardown`.
- An exported `PRIMITIVE_ENV` outranks the selection `up` writes; `up` says
  so when it's set.

### Reference

```bash
primitive apps children create feature-x   # create, register and select it
primitive apps children list                # this app's children
primitive apps children get feature-x       # one child, and its local status
primitive apps children delete feature-x --yes
```

`create` takes `--branch <name>` (default: the checkout's current git
branch), `--idle-days <n>` (7–30), `--no-secrets`, `--no-use` and `--json`.
It fails closed: a create that errors on the server leaves no child behind.
`--json` prints one document whose `nextStep` is the command to run next.

A child's own var values live in `primitive/<env>/vars.<slug>.override.toml`,
a flat `KEY = "value"` table beside `vars.toml`:

```toml
# primitive/dev/vars.feature-x.override.toml
REFERENCE_DB_ID = "01M4VFW6B2WANJXA58FG810HDX"
WEB_ORIGIN = "https://feature-x.example.com"
```

With the child selected, `config push` sends `vars.toml` with the override's
keys replacing or adding to it, `config diff` compares against those values,
`config pull` writes differing server values into the override (never into
`vars.toml`), and `config set vars KEY=value` edits the override. The parent
and other children never read it. It is git-ignored
(`primitive/*/*.override.toml`), and `apps children delete` removes it.

**Gotchas**

- The override cannot remove a var: every `vars.toml` key reaches the child.
- A running function version caches the config vars it reads. Adding an
  override to a child whose functions are already pushed can leave them
  reading the old values until each one is pushed as a new version (any
  change to its code or TOML) or its running instance is recycled on its
  own; `config push` prints a note when a changed var left a function's
  version unchanged. Set the child's vars with `config set vars` before its
  first push to avoid it.
- `databases delete` needs the `admin` scope on a child too. Only the child
  app itself is deleted with `write`; deleting a database is never scoped
  down because the app is a child.

A child shares its parent's `primitive/<env>/` tree but keeps its own
sync-state baseline and snapshot backups — `config pull`, `push`, and `diff`
on a child never touch the parent's committed state. With a child selected,
those three commands act on the child itself, but every `apps children`
command acts on its **parent**, so running `create` again makes a sibling,
not a grandchild. A pull on a child never writes `[app].name`, so a branch
merged back into the parent's tree keeps the parent's name.

`primitive env use <name>` selects a committed environment or an existing
local entry as usual; otherwise it looks the name up as a child slug of the
current environment's app, and if found, registers it and selects it in one
step. It refuses an unknown slug, and refuses a known one only when that
child's app id is already registered here under a different name.

A child uses its parent environment's login and step-up; it has none of its
own. `login` or `logout` in the parent or any child signs in or out all of
them, and names them.

`primitive apps list` shows parent apps only; `--all` includes children.
Deleting a parent with children is refused, naming them;
`primitive apps delete <app-id> --with-children` deletes each child, then the
parent, stopping before the parent on the first child it can't delete. A
plain login deletes the children; the parent needs an `admin` step-up unless
it is `internal`.

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
  vars.<child>.override.toml      # A child app's own var values (git-ignored)
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

`[app].protection` is the app's protection level, one of
`default`, `protected` and `internal`: `protected` for an app with real users,
`internal` for a throwaway app (every child app is `internal`). Only the app's
console owner changes it; a push from anyone else
that would change it fails with `PROTECTION_OWNER_ONLY`, and a child's level
cannot change (`PROTECTION_FIXED_ON_CHILD`). A file without the line pushes
`default`. The deprecated `protected = true` reads as
`protection = "protected"` until the first breaking release after 2026-12-15;
`primitive config pull --only app` rewrites it. On a `protected` app a login's
writes need a step-up; see
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
| `write` | Changing configuration, setting and deleting secrets, raising the protection level, deleting an `internal` app, deploying and activating functions and prompts |
| `data:read` | Reading documents and database records (admin API, app API, WebSocket) |
| `data:write` | Changing documents and database records |
| `admin` | Deleting `default` and `protected` apps, databases and users, removing collaborators, changing roles, hard deletes, lowering the protection level from `protected` |

None implies another; a command needs every scope its route declares. The
data scopes guard direct access only: a token holding `write` can deploy and
activate code, and that code reads and writes the app's data whatever the
token's data scopes are.

**Protected apps.** On an app whose level is `protected`, anything beyond
`read` and `data:read` needs a step-up pinned to that app, even from a login
pinned to it; a pinned CI session needs none. Only the app's own level counts:
a `default` or `internal` app, including the `internal` child of a protected
app, never needs a step-up. Lowering the level from `protected` also needs
`admin`. An `internal` app is deleted with `write`, with no step-up.

A pin also reaches the pinned app's child apps: a login pinned to an app
creates its children (with no step-up, even under a `protected` parent) and
lists them with it. A pin on a child app does not reach its parent.

**Step-up.** A step-up is a 15-minute session holding exactly the scopes the
command needs, pinned to the environment's app (`--app <id>` pins another).
Ask the user to run the command a refusal prints:

```bash
primitive login --step-up --scope write --app <app-id>
primitive login --step-up --scope admin
```

Deleting a `default` or `protected` app or a database, removing a
collaborator, changing a user's role and lowering the protection level need
`admin`; a step-up holds it only when asked for. The console asks for Approve, and Google re-prompts for `admin`. A
command refused for lacking `admin`, or by a protected app, retries once with
the stored step-up; `primitive logout` ends both.

**A narrower token** for a script or an agent, at most 1 hour:

```bash
primitive token --scope read --app <app-id> --ttl 15m
```

It prints with no browser when it is no wider than the login, and opens the
approval page otherwise. `primitive token --step-up --scope <list>` prints a
step-up, always after approval.

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
| `INSUFFICIENT_SCOPE` | The login lacks a scope the route declares (`details.missing`) | Ask the user to run the printed `primitive login …` command |
| `APP_NOT_PINNED` | The login's pins do not reach the app or route named | Confirm the command targets the intended app (`primitive whoami` shows the pins), then the printed command |
| `PROTECTED_APP_STEP_UP_REQUIRED` | The app is protected; the login is neither a step-up pinned to it nor a CI session | Ask the user to run the printed `primitive login --step-up …` command, then retry |
| `PROTECTION_OWNER_ONLY` | Only the app's console owner changes the protection level | `primitive config pull --only app` if `app.toml` is stale; otherwise the owner |
| `PROTECTION_FIXED_ON_CHILD` | A child app's level is always `internal` | Change the level on the parent |
| `SESSION_UNSCOPED` | A 401 on refresh: the login predates scoped sessions | Ask the user to run `primitive login` |

### Gotchas

- **`write` deploys code that reaches data.** A `read,write` CI session cannot
  read data itself but can deploy a function that does. Grant `write` only
  where deploying is intended.
- **A parent pin reaches its child apps.** A token pinned to an app can create
  that app's children and change them with the scopes it holds.
- **A printed `--app` does not mean the app is protected.** `APP_NOT_PINNED`
  and `INSUFFICIENT_SCOPE` print it too; act on the `code`.
- **A pushing CI session needs `read` too.** `config push` reads the app's
  settings before it writes them; give a pushing session `read,write`.
- While `PRIMITIVE_TOKEN` is set it takes precedence over the stored login,
  and `primitive auth sessions create` is refused. Unset it to create sessions.

Target the environment explicitly in automated pushes. Keep tokens out of
configuration files and logs.

# Agent Guide to Primitive Configuration

Guidelines for AI agents configuring Primitive services. Everything Primitive does for an app — auth settings, workflows, prompts, integrations, database types/operations, webhooks, cron triggers, blob buckets, email templates, rule sets — is server-side configuration with two equivalent interfaces: the web Admin Console (interactive) and **TOML files synced via the CLI** (configuration as code). Agents should use the TOML + CLI path.

## TOML is the only write path

Server-side configuration is authored in TOML and applied with `primitive config push`. The CLI has **no** `create`/`update`/`delete` commands and no configuration-setting flags for these types — if a field belongs to a config object, the file is how you set it. Authoring commands, all local except the push:

```bash
primitive config fields <type>                   # keys, types, required, defaults for a type (--json for tooling)
primitive config create <type> <key>             # scaffold <dir>/<key>.toml from the type's defaults
primitive config set <object> <path>=<value>...  # set scalars in place, comments and key order preserved
primitive config push --only <selector>[,...]    # apply just these objects
```

- `<type>` is one of `integration`, `webhook`, `cron-trigger`, `blob-bucket`, `prompt`, `workflow`, `transform`, `database-type-config`, `rule-set`, `group-type-config`, `collection-type-config`, `metadata-category-config`, `email-template`, `app`, `var`.
- `<object>` on `set` is `<type>/<key>`, `app`, or `vars`; `--only` selectors are `<type>/<key>`, `app`, or `var/<key>`.
- `set` writes scalars only — edit a table or an array in the file. `true`/`false` and numerics are written with those types; double-quote a value to force a string.
- `fields`, `create` and `set` make no API call. Nothing reaches the server until a push.

The scripted form of a one-field change is two commands, no here-document:

```bash
primitive config set workflow/order-intake workflow.name="Order intake"
primitive config push --only workflow/order-intake
```

Deleting a config object = delete its file + `primitive config push --prune`. Every type, no exceptions — there is no per-type delete verb. (`primitive <noun> archive` is not that: it retires the object and keeps the row, its key and its capped slot.)

Not configuration, so still direct commands: secrets (`primitive secrets set`), resource creation that mints an id (`apps create`, `databases create`, `collections create`, `groups create`), data operations, and every read/test command.

Availability is runtime state, not configuration. `<noun> disable`/`enable` — for `workflows`, `cron-triggers`, `webhooks`, `integrations` and `prompts` alike — are its only writers, along with the matching web-admin action and the delete flow, which writes the third value `archived` (a retired object, kept so its history still resolves; `enable` refuses it). The delete flow has a CLI spelling on those same five nouns: `primitive <noun> archive <id>` (confirms first, `--yes`/`-y` skips it, `--json` prints the server envelope). It is a SOFT delete — the row keeps its key, and for webhooks and cron triggers its slot against the per-app cap — so it is not how you free a key; deleting the file and running a confirmed `primitive config push --prune` is. There is no un-archive: recover by hard-deleting the holder, then re-adding the file and pushing. `status` is server-owned and is deliberately not a TOML key: `config pull` does not emit it, a file that still carries the line fails `config push` with guidance naming the verb, and `config diff` reports "inactive; configuration unchanged" rather than drift. Anything created or pushed is active; there is no `draft` state on any object. Read the current value back with `<noun> get` or `<noun> list`. Per-VERSION status is a different key and stays in TOML: `[[configs]] status = "archived"` retires one named config, and says nothing about whether the object is serving. For a prompt's `[[configs]]`, pull writes that line only for a config that IS retired — an omitted one means active, so an ordinary pulled prompt file carries no `status` at all. A workflow's named-config sidecar (`workflows/<key>.configs/<name>.toml`) still states its own `[config] status` either way.

## The sync loop

```bash
primitive config init  # create directory structure
primitive config pull  # download server config as TOML
primitive config diff  # preview changes
primitive config push  # apply local TOML to the server
```

The config directory is `primitive/<env>/` — one isolated slot per environment, so a `pull --env staging` never touches production state. There is exactly one way to reach it: the environment named by `primitive/config.json`. No flag relocates it, and no flag points a read or write in it at an app other than the environment's. The resolution walks up from the working directory to the nearest project config, so in an app with several clients the config lives once at the repository root and every client directory resolves it — see the `multi-client` guide for that layout.

Structured values — an operation's `definition`/`params`, a subscription's `params`, a workflow step's object fields — are written as nested TOML tables. A file may instead hold them as single-line JSON strings (the older encoding); the server treats the two identically, `config pull` keeps whichever form a file already uses, and `primitive config migrate-toml` rewrites a file from JSON strings to TOML tables in place. Author new files in table form.

## Pull snapshots and revert

Every `config pull` snapshots the config directory before writing anything; if the snapshot can't be written, the pull aborts with no files changed. Snapshots older than 28 days are pruned automatically. They live at `<projectRoot>/.primitive/backups/<env>/<appId>/` — gitignored machine-local state, partitioned by app so an environment rebound to a different app cannot restore the former one's snapshot.

```bash
primitive config revert --list           # enumerate snapshots, newest first
primitive config revert                  # restore the most recent
primitive config revert --snapshot <id>  # timestamp dirname, or a unique >=8-char prefix
primitive config revert -y               # skip the confirmation prompt
```

`revert` replaces the entire config directory with the snapshot, including `.sync-state.json` (the committed sync-state baseline). It warns when the config directory has uncommitted git changes (the restore overwrites them), and refuses to restore a partial/corrupted snapshot. After reverting, run `primitive config diff` to inspect the restored state versus the server.

## Pull pruning (deleting local files)

After writing the current server entities, `config pull` reconciles deletions: it removes a local file **iff** its key was managed by a prior pull (present in `.sync-state.json` sync state) **and** absent from this pull's server entities. This keeps the sync tree a faithful mirror — because a `config push` without `--prune` treats every local TOML as create-or-update, a stale file for a server-deleted entity would re-create it on the next push. (To reconcile deletions the other direction — delete a server entity when its local file is gone — see [Push pruning](#push-pruning-deleting-server-entities).) Presence is read from the **server listing**, never from what this pull just wrote (several types legitimately skip writing a present entity). Pruning is **on by default**; the pull summary reports a `Pruned` count. Removed files are recoverable — the pre-pull snapshot is taken first, so `config revert` restores anything pruned in error.

A file is pruned only when it clears every safety rule; otherwise it stays and keeps its managed marker (its prior state entry is carried forward, so the discriminator survives to the next pull):

- **`--no-prune`** — the operator opted out; no file is removed this pull.
- **Never-pulled file** — a key absent from prior sync state was hand-authored; it is left alone and `config diff` reports it as *new*.
- **Uncommitted git edits** — a file with uncommitted changes (tracked edits; untracked files do not count) is kept and reported, not deleted — the operator may be mid-edit. Remove it yourself or re-create the entity on the server.
- **Failed listing/detail fetch** — a type whose listing or per-entity fetch failed is skipped whole with a warning; an error that reads as an empty list must never delete every local file of a type.
- **Incomplete listing** — a type whose server listing is not paginated (so an entity missing from it may still be live) is skipped; the omitted files are reported so you can delete them yourself if the entity is really gone.
- **Unsafe filename** — a sync-state key that does not map to a safe filename is skipped with a warning.

When a block type (workflow, prompt) is pruned, its sidecar `<key>.tests/` directory and the block's test-case state are removed with it.

## Push pruning (deleting server entities)

`config push` creates and updates only — by default it never deletes, so removing a local TOML leaves its server entity in place. Pass **`--prune`** to reconcile deletions the other direction: after the create/update pass, push deletes managed entities whose local declaration is gone. A candidate is an entity whose key was recorded in `.sync-state.json` by a prior pull **and** which no local file in its directory still declares — keys are read from each file's declared identity (the in-file key or, for keyless types, the basename), mirroring how push derives them, so a file renamed away from `<key>.toml` but still declaring the key keeps its entity out of the prune set. A key never in sync state was authored server-side and is never a prune candidate. A push **without** `--prune` deletes nothing and makes no extra server calls.

Prune is fail-closed on every candidate:

- **Point read + drift check** — each candidate is confirmed by a detail fetch by id/key (immune to listing truncation), and deleted only when the live `modifiedAt` matches the value recorded at last sync. A mismatch (edited out-of-band) is skipped as *drift*; **`--force`** deletes regardless.
- **404** — the entity is already gone server-side; its stale sync-state entry is dropped and nothing is deleted.
- **Any other fetch failure** — presence could not be confirmed, so the entity and its state are left untouched.
- **Still referenced (409)** — the delete is blocked and reported, its state kept, and the remaining deletes proceed (partial success — remove the referrer and re-run).
- **Orphaned metadata values** — deleting a **metadata-category-config** deletes the definition only; stored metadata values are NOT deleted and become unreachable (no query path from a category to its rows). The prune plan states this caveat under the delete line, in `--dry-run` too. Delete the values first (`primitive metadata delete`) if you need them gone.

One batch confirmation gates the destructive deletes (`Delete N remote entity(ies)? This cannot be undone.`); **`-y`/`--yes`** skips it for CI. **`--dry-run`** reports the full plan — will delete / already absent / skipped, with the skip reason — and mutates nothing. When a pruned block type (workflow, prompt) is deleted, its sidecar `<key>.tests/` directory and test-case state are cleaned up with it.

A prune always hard-deletes. Every type that carries `archived` is soft-archived by a plain API `DELETE` (or the console's Archive action) and **hard-deleted** by `config push --prune`, so its `key` — and, for webhooks and cron triggers, its slot against the per-app cap — is freed. That is deliberate: an archiving prune would keep the key with no way to reclaim it from `config`. It is also irreversible, and the object's history (webhook deliveries, workflow runs, prompt executions) stops resolving what it points at, so archive first if that history still matters.

## Directory map

```
config/
  app.toml                        # App settings
  workflows/*.toml                # Workflow definitions
  workflows/{key}.configs/*.toml  # Named workflow config bodies (activeConfigName in the workflow file names the live one)
  workflows/{key}.tests/*.toml    # Workflow test cases
  workflow-fragments/*.toml       # Reusable [[steps]] blocks (include = ["<name>"])
  transforms/*.rhai               # Rhai scripts for workflow script steps
  transforms/{name}.tests/*.toml  # Script test cases
  prompts/*.toml                  # Managed prompts
  prompts/{key}.tests/*.toml      # Prompt test cases
  integrations/*.toml             # External API integrations
  integrations/{key}.tests/*.toml # Integration test cases (fixture files in integrations/{key}.tests/{case}/)
  database-type-configs/*.toml    # Database type configs + operations + subscriptions
  webhooks/*.toml                 # Inbound webhook configs
  cron-triggers/*.toml            # Cron trigger configs
  blob-buckets/*.toml             # Blob bucket configs
  email-templates/*.toml          # Email template overrides
  rule-sets/*.toml                # Access rule sets
  group-type-configs/*.toml       # Group type configs
  collection-type-configs/*.toml  # Collection type configs
  metadata-category-configs/*.toml # Resource metadata category configs (schema + readRule/writeRule)
```

**A test case's file name is its identity.** The basename of a
`<key>.tests/<case>.toml` file is the case's key on the server, unique per block
and case-insensitive. That is what lets a case authored by hand — which has no
`.sync-state.json` entry until its first push or pull — reconcile the committed
tree instead of creating a duplicate of every case. `config pull` writes a case back
under that same name (never the slug of its `[test] name`), and renaming the file
renames the case on the next push, updating the same test case rather than
creating a second one. Keep the name path-safe: no `/` or `\`, no leading or
trailing spaces, no trailing dot, 200 characters at most. A case created outside
the CLI (the console, the raw API) has no key until a `config push` stamps one;
where a push cannot tell which server case an unkeyed file belongs to, it says
so and creates nothing — run `config pull` first, or push once from the machine
that has the sync state.

### A case file is local until a push registers it

Writing or editing a `.tests/<case>.toml` file only changes the local file.
`primitive <noun> tests run-all` / `tests list` (the workflows/scripts/prompts/
integrations test runners) act on the **registered** set on the server, not
the sync tree — a new case is silently absent from `run-all` results, and an
edit to a registered case's file is not what runs, until `config push` sends
it. `primitive config diff` names exactly this gap, per case, with four
counters:

- **Test Cases (local only)** — file exists, no registered case yet; `run-all` will not run it until a push.
- **Test Cases (remote only)** — a registered case whose file is gone (deleted, renamed away, or never pulled); it keeps running until `config push --prune` removes it — deleting the file alone does not stop it.
- **Test Cases (modified)** — both exist but differ; `run-all` still executes the last-pushed version, not what's on disk.
- **Test Cases (synced)** — file and registered case match; the only state where "authored" and "running" are the same thing.
- **Test Cases (push would refuse)** — a case file that fails validation, printed only when there is one. This status is *separate* from "local only", not a subset of it: a malformed new case counts here and nowhere else, so "local only: 0" does **not** mean every case file made it to the server. Fix the file and push again.

## App settings (`app.toml`)

App-level settings sync from `app.toml`. Edit the TOML and apply it with `primitive config push` (or `config push --only app` for the settings alone); `primitive config pull --only app` writes current server settings into it, `primitive config diff --only app` shows per-field differences, and `primitive apps get` renders the server-effective settings without touching any file. There is no command that writes a setting — `app.toml` plus a push is the only way to change one. TOML-syncable settings:

- `[app]` — `name`, `mode`, `baseUrl`, `waitlistEnabled`, `waitlistNotifyAdmins`, `directLlmEnabled` (boolean; opts into the deprecated direct LLM/Gemini proxy routes, off by default), `allowedDomains` (string array), `testAccountBaseEmails` (string array), `largeDocumentWindowDays` (1–14, default 7)
- `[auth]` — `googleOAuthEnabled`, `emailSignInEnabled`, `passkeyEnabled`, `appleSignInEnabled`, `appleAudiences` (string array), `emailRedirectUris` (string array — the sign-in-link allow-list, see below), `passkeyUserVerification` (`"preferred"` | `"required"`, see below), `[auth.google.clients.<type>]` Google client entries (see below), `[auth.passkeys]` relying-party config
- `[cors]` — `mode`, `allowedOrigins`, `allowCredentials`, `allowedMethods`, `allowedHeaders`, `exposedHeaders`, `maxAge` (the `[cors]` table is always emitted, in every mode)
- `[invitations]` — `enabled`, `limit` (whether role `member` users may send invitations, and the per-member cap; `0` = unlimited)

`app.toml` is the **whole truth**, the same source-of-truth rule every other configuration file follows (see [Owned scalar fields](#owned-scalar-fields-clear-on-absence)): push sends a value for every setting listed above, not just the keys present, so a deleted line **clears** the setting or **resets it to its declared default** — `emailSignInEnabled` / `waitlistNotifyAdmins` to `true`, `[cors] mode` to `"universal"`, `[invitations] limit` to `5`, the remaining booleans to `false`, everything else cleared. `[app].name` and `[app].mode` are **required** (no default to reset to, and `mode` decides who can sign up), so their absence is a validation error from `config push` and `config diff` alike. `config pull` still omits a setting the server does not hold, so a pull → push round trip is a no-op. An explicit `false` is forwarded and `appleAudiences = []` clears the audiences. An unrecognized or retired key is rejected by name before anything is applied — never ignored.

### Google sign-in and email sign-in redirect URIs

`[auth.google.clients.<type>]` (one entry per platform) and `[auth].emailRedirectUris` follow the same TOML rules as every other setting above — see [Google Client Configuration](AGENT_GUIDE_TO_PRIMITIVE_AUTHENTICATION.md#google-client-configuration) and [Email Sign-In](AGENT_GUIDE_TO_PRIMITIVE_AUTHENTICATION.md#email-sign-in-one-email-both-credentials) in the authentication guide for the per-type `clientSecret` rules and error codes, the redirect-URI selection and matching semantics, and the `emailSignInEnabled` switch.

### Passkey user verification

`passkeyUserVerification` is the policy BOTH halves of a passkey ceremony state
— the options ask the authenticator for it, and the server enforces exactly that
when verifying. Flat in `[auth]`, beside the `[auth.passkeys]` relying-party map:

```toml
[auth]
passkeyEnabled          = true
passkeyUserVerification = "required"   # default: "preferred"
```

- `"preferred"` (the default, and what an app that omits the key gets) — the
  authenticator MAY skip Face ID / a PIN, and the server accepts the credential
  it returns. iOS legitimately skips it; so do password managers counting a
  vault unlock, and PIN-less security keys.
- `"required"` — the authenticator must verify the user before returning
  anything, and a credential it did not verify is rejected with
  `PASSKEY_USER_VERIFICATION_FAILED` (401 signing in, 400 registering).

Only `"preferred"`, `"required"` and no key at all are accepted; any other value
is a 400 naming the field.

### Which relying party a ceremony runs against

`[auth.passkeys]` (`passkeyRpConfig`) is the set of relying parties an app HAS;
each ceremony still has to select one of them. A browser is matched by its
`Origin` header. A native request carries none, so it names the relying party
itself — `AuthConfig(passkeyRpId:)` at client construction, or a per-call
`rpId:` — and an app that names one that is not a key of `passkeyRpConfig` is
rejected with `PASSKEY_RP_NOT_CONFIGURED` rather than served another.

A Swift app built on `PrimitiveAppState` needs no code for this: `initialize()`
names the host of the selected environment's `webUrl`, which is the relying
party a browser at that origin would be matched to — so keep that host among
the `[auth.passkeys]` keys. Override `passkeyRpId(for:)` on the app state to
name a different one (return `nil` to leave the choice to the server), or pass
`rpId:` to `PrimitiveAuthManager.signInWithPasskey` / `enrollPasskey` for a
single ceremony. An IP-literal `webUrl` host derives nothing, since an IP
address is never a valid relying-party id.

### Retired keys

`googleClientId`, `googleClientSecret`, `redirectUris`, `passkeyRpId`,
`passkeyRpName`, `magicLinkEnabled` and `otpEnabled` are no longer authored. A
file that still carries one is rejected by name, with the replacement:

| retired key | replacement |
|---|---|
| `googleClientId` | `[auth.google.clients.<type>].clientId` |
| `googleClientSecret` | `[auth.google.clients.<type>].clientSecret` |
| `redirectUris` | `[auth.google.clients.<type>].redirectUris` for Google callbacks; `emailRedirectUris` for magic links — a stored list may legitimately have mixed the two, so split it |
| `passkeyRpId` / `passkeyRpName` | `[auth.passkeys]` (`passkeyRpConfig`) |
| `magicLinkEnabled` / `otpEnabled` | `emailSignInEnabled` — one email flow, one setting |

## Owned scalar fields (clear on absence)

For a set of **TOML-owned scalar fields** on synced entities, the local file is the source of truth: `config push` always sends these fields as value-or-`null`, so removing a field's line from the TOML clears it on the server rather than leaving the prior value in place. This is field-level and separate from removing a whole TOML file — a default push never deletes an entity (whole-file reconciliation is [Pull pruning](#pull-pruning-deleting-local-files), or opt-in [Push pruning](#push-pruning-deleting-server-entities)). Owned fields by entity type:

- **database-type-configs** — `ruleSetName` (wire `ruleSetId`), `triggers`, `celContextAccess` (wire `metadataAccess`; `celContextAccess = ""` is honored as an explicit clear), `defaultAccess`, `autoPopulatedFields`, `timestamps`, and the `[metadata]`/`secrets` manifest. The `[models.*]` schema is owned too but tracked separately — removing every `[models.*]` block forwards `schema: null` and clears the schema (blocked while operations are still registered on the type — delete or migrate them first). `push` prints `Cleared <field> on <type>` on a genuine non-null → null transition.
- **group-type-configs** — `ruleSetName` (wire `ruleSetId`).
- **integrations, webhooks, cron-triggers, blob-buckets** — `description`.
- **prompts** — `description`, `inputSchema`, and each `[[config]]` block's `description`.
- **workflows** — the whole `[workflow]` table (see below); the file, not a field list, is what push sends.

Enum and defaulted fields (`status`, `timeoutMs`, `timezone`, `overlapPolicy`, `state`, …) are **not** cleared this way — `null` is not a valid value for them, so omitting the line leaves the current server value unchanged. To change one, set it explicitly in the TOML.

**Workflows go further: every `[workflow]` field converges, defaulted ones included.** A push to an existing workflow sends the whole table, so removing `description`, `accessRule`, `runAs`, `capabilities`, `inputSchema`, `outputSchema`, `[workflow.lock]` or `syncCallable` clears it, and removing `perUserMaxRunning`, `perUserMaxQueued`, `perAppMaxRunning`, `perAppMaxQueued`, `queueTtlSeconds`, `dequeueOrder` or `requiresClientApply` resets it to its default (4, 100, 25, 10000, 43200, `fifo`, `true`). `status` is NOT among them: availability is server-owned, so a push neither sends nor converges it. `name` is required and cannot be cleared: a `[workflow]` table with no non-empty `name` fails the push locally, naming the file. A workflow edited server-side since your last sync is reported as a push conflict — a modified-timestamp check, so it catches an edit that landed before the push began; `--force` overrides it. The check rides the update request, so it only fires for a file the push sends; an unchanged file is skipped, and its drift shows in `primitive config diff`.

## Environment resolution

Every command resolves its target Primitive environment in order: `--env <name>` flag → `PRIMITIVE_ENV` env var → this machine's selection in `.primitive/local.json` → `defaultEnvironment` in `primitive/config.json` → the only defined environment → error. Manage environments with `primitive env add|list|show|use|remove`.

`primitive env use <name>` writes the selection to `.primitive/local.json`, NOT to the committed config: which backend this machine is pointed at is per-machine state, so switching backends never modifies a tracked file. `defaultEnvironment` stays the committed team default a fresh clone resolves. Tokens (`.primitive/credentials.json`) and the selection (`.primitive/local.json`) are gitignored; `primitive/config.json` is committed and is the single place a backend URL and app ID are typed — app templates read it rather than repeating those values in their own config. A corrupt or dangling selection is reported by `env list`/`env show` rather than silently falling back.

A web app resolves the same way through the `primitiveEnv()` Vite plugin, and its Vite mode (`--mode`, i.e. which `.env.<mode>` supplies app-behavior keys) is a SEPARATE axis — `PRIMITIVE_ENV=dev pnpm build --mode alpha` is a legitimate combination. When a mode's keys are only correct against one backend, pin the pair by declaring `VITE_EXPECTED_PRIMITIVE_ENV=<name>` in that mode's `.env` file: every entry point that resolves an environment (dev server, build, deploy, headless vitest run) then fails at startup on a mismatch. Opt-in — absent, the axes stay independent. Details in the deploying guide.

`primitive env add` adds an environment to an existing project (`primitive init` creates one; outside a project `env add` fails like every other app-scoped command). It writes only the environment entry into `primitive/config.json` — it seeds no credentials, and project mode never borrows credentials from `~/.primitive/credentials.json`. A freshly-added environment therefore starts logged-out: commands report "not logged in" for it until you run `primitive login` for that environment, even when another environment in the same project is signed in. (The `dev` environment scaffolded by `primitive init` is the exception — init seeds it with the session it authenticated during setup.) Agents and CI can log in without a browser by piping a refresh token — `primitive token --refresh | primitive -e <env> login --token-stdin` (see [Headless auth](#headless-auth-ci)).

## Previewing a push

`primitive config diff` lists entities that would be created, changed, or removed; `primitive config push --dry-run` reports the full push without applying it. A local file with no matching server entity is reported in one of two buckets, counted separately in the summary: **Local only (new)** — a file no prior pull managed, hand-authored, which push will create — and **Local only (absent from export)** — a file a prior pull wrote whose entity is now absent from the server (deleted server-side); push would RE-create it unless you remove the file, and a default `config pull` would prune it (see [Pull pruning](#pull-pruning-deleting-local-files)). The discriminator is the same prior-sync-state check the pull-side prune uses. A server entity with **no** local file is reported under **Remote only**, split by the same check: **Remote only (managed)** — a prior pull recorded it, so `push --prune` deletes it (see [Push pruning](#push-pruning-deleting-server-entities)) — and **Remote only (unmanaged)** — never synced, authored server-side, left alone. **`diff` covers every resource type `config push` handles** — the config types (database types, rule sets, group- and collection-type configs, metadata-category configs), email templates, webhooks, prompts, integrations, cron triggers, blob buckets, test-case sidecars and the app settings in `app.toml` (compared per field) are all compared for CONTENT, not just presence, so an edit to any of them is reported as **Modified** rather than silently reading as in-sync until push applies it. A **Modified** row names the field it differs in ahead of the `config pull` remedy — for a database type, the operation or subscription and the sub-field (`operations["getInstitution"].params`), or the model field inside the `[models.*]` schema — so a difference you cannot see in the file (an operation block the file declares twice, for instance) is diagnosable from the row itself. Any type whose current server state `diff` could not fetch is listed under an explicit **not compared** line instead of being omitted, so the summary is never mistaken for exhaustive. Both run the **same local preflight** a real push runs — the identical checks, producing the identical messages — and both add the server comparisons `diff` performs. What neither can report is what only the apply stage reaches: a function's build, and a server refusal. See [Push failures](#push-failures) for the two stages and where the line falls. Both commands decide from the same comparison against live server state, for every type push handles and with no per-type exception list: a comment-only or formatting-only edit is a change to neither, and a difference `diff` reports is one `push` acts on — it applies the edit, or reports it as server drift (**DRIFT**, not overwritten) or a conflict (**CONFLICT**, cleared by `config pull` or `--force`), or names an immutable-field difference with its remedy. A file `push` would refuse — an unrecognized or retired key, or a value whose spelling is not its declared type — is reported by `diff` with the identical message under **Invalid**, never as in-sync. When a type's live state cannot be read, that type degrades to the last-sync content hash with a named warning, and fails closed when there is no stored hash to gate on: nothing is written, and the run tells you to `config pull` or `--force`. One prompt difference push cannot apply is named instead of being reported as a clean update: push updates and creates the `[[configs]]` entries the file lists, but deletes none and clears no activation, so a file omitting an entry the server holds — or naming no `active` configuration — is reported as still differing after the push (`Prompt <key> still differs from the server after this push`), which `config diff` will keep reporting until you `config pull` or remove the configuration server-side. Schema-gate rejections surface identically in the preview and in a real push: an operation whose database type has no schema set, an unresolved `$params.X`/reference, or a schema change that would break an existing registered operation. A previewed or blocked entity records no content hash in sync state, so it stays pending on the next `config diff` rather than reading "in sync" — a server-rejected change cannot silently disappear from the diff. `primitive config diff --json` emits machine-readable output on stdout.

## Push failures

**The preflight is the only all-or-nothing stage.** `config push` runs a local pass over the tree before the first mutating call, and any error in it aborts the run with nothing applied (`Aborting push: N TOML validation error(s) — no changes were applied.`). The two stages are a registry classification, not a description: `src/config-surface/`'s `RuleStage` (`"preflight" | "apply"`) is the `stage` of every cross-field rule (`CrossFieldRule.stage`) and of every field's own validation (`ConfigField["validation"].stage`), and `crossFieldRulesForStage(surface, stage)` returns the declared set for either. `preflight` means decidable from the authored files alone AND enforced before the first mutating call on that surface's push path — which is why a `preflight` handler's module lives under `src/config-surface/`, vendored into the CLI artifact (`cli/src/lib/generated-config-surfaces.ts`, held fresh by a build check), so a preflight refusal is the same sentence the server would have produced. `apply` means the rule needs the app's persisted state, or has path semantics only the server can decide. Today `crossFieldRulesForStage` returns `preflight` for the `function` and `workflow` surfaces' declared rules — the trigger and webhook block shapes, a webhook trigger's signing-secret requirement and reference shape, a cron entry's expression and timezone, and a workflow's `runAs`/`accessRule` conflict — plus every field whose own `validation.stage` is `preflight`. The rest of the pass is the same contract, run ahead of a per-rule declaration: unrecognized keys and declared types on every config type, the workflow lints, a function's grant grammar and `entry`, and a database operation's `$params` reference against its declared `[[operations.params]]` — file-decidable and refused before anything applies, whether or not that check's surface has migrated onto the registry (most have not; the shrinking list is `CROSS_FIELD_RULES_UNMIGRATED`). Cross-field constraints inside one object are decidable from the file alone because the state a push writes is file-derived: a workflow update sends the whole `[workflow]` table, and a function's version envelope carries the whole authored TOML.

**The apply stage is incremental and convergent, and there is no rollback.** Once the preflight passes, entities are applied in order. A rejection may stop the remaining work — a rule set's or an integration's non-conflict server error ends the run — while a refused FUNCTION is recorded and the loop continues, so the rest of the tree still lands. Whatever was already applied stays applied; the exit status is non-zero; re-running converges, because every applied change is idempotent.

**What the preflight cannot decide, and why.** Whether a referenced secret exists, whether a webhook scheme's `config` is valid under this environment's JWKS policy, per-app caps, key-namespace collisions, archived tombstones: each needs the app's state, so each is `apply`, surfacing at apply time rather than in preflight or preview. Nothing decidable from the file alone is on that list — a local guess at any of these would be a preflight that gives a false all-clear. Which stage a refusal came from says whether re-running could ever help without touching the file: an `apply` refusal may depend on the app's state, so fixing that state — creating the missing secret, freeing the cap — and re-running can turn it into a success; a `preflight` refusal cannot, because it is decided by the tree in front of you and nothing server-side changes the answer. A function whose webhook trigger names `{{secrets.MISSING}}`, for instance, is well formed: the push starts, that one function fails naming the secret, and its siblings land. When validation passes but the server rejects an entity, the error names the entity and file. A workflow, cron trigger, blob bucket, rule set, integration, webhook, or prompt that already exists on the server but is missing from local sync state is adopted by key and updated in place rather than failing the create (a 409 or a "must be unique" conflict) — re-running a failed push converges. For the types push reconciles against live server state (rule sets, transform scripts, prompts, and the group-, collection- and metadata-type configs) the adoption happens one step earlier, in the reconcile read: an out-of-manifest entity the server already holds is compared rather than overwritten, so one that MATCHES the file is simply recorded in sync state, and one that differs is reported as a conflict — with no recorded baseline nothing says which side moved, so the remedy is `config pull` (take the server's version) or `push --force` (make the local file win). Database types and transform scripts converge the same way through their own conflict recovery, as do group- and collection-type configs — a config created out-of-band (whose "already exists" 409 would otherwise fail every later push) is adopted by its type-name key and updated in place. Adoption matches only on the entity's key, and a few immutability guards fail the adopt loudly rather than letting sync state diverge: a blob bucket whose local `ttlTier` differs from the server's fails the adopt with both values named (`ttlTier` is immutable — reconcile the local file to the server's tier, or recreate the bucket); a rule set whose local `resourceType` differs from the server's is rejected the same way (`resourceType` is immutable — align the TOML, or rename/recreate the rule set). Prompt adoption covers only the create itself: post-create config work (extra prompt configs, activation) runs outside the adopt path, so a duplicate-config conflict surfaces as the real error instead of being mis-adopted, and sync state is stamped only after all create work succeeds. Diagnostics go to stderr; `--json` data goes to stdout (pipes stay clean).

## Out-of-band changes and stale sync state

Push decides create-vs-update per entity from the server id stored in `.sync-state.json` (`entities.<entityType>.<entityKey>.id`, at the root of the config directory; database types and the group/collection/metadata type-config families track the entry without an id — the decision there is on the entry's presence). The types push reconciles against live server state (rule sets, transform scripts, prompts, and the group-, collection- and metadata-type configs) decide it from that read instead, and fall back to the stored id only when the server could not be read. Changes made outside the sync loop — console edits — leave that state out of date, and the two directions behave differently:

- **Created out-of-band** (a console edit) — converges for workflows, cron triggers, database types, blob buckets, transform scripts, rule sets, integrations, webhooks, prompts, and group- and collection-type configs: push adopts the existing entity by key and updates it in place instead of failing the create (see [Push failures](#push-failures)).
- **Deleted out-of-band** — does not self-heal, except for the live-reconciled types above: their reconcile read finds no entity, so the next push creates it fresh. For the rest the stored id goes stale, and the next push of a changed TOML for that entity issues an update against the missing entity, failing the push with a "not found" error for that entity.

Symptom → fix: `config push` fails with `Failed to update <entity> <key>: … not found` ⇒ the entity was deleted on the server while its sync-state entry survived. Delete `entities.<entityType>.<entityKey>` from `.sync-state.json` and push again — the entity is created fresh (or adopted, for the kinds listed above). A full `config pull` also rebuilds sync state from the server, but it overwrites every server-backed file with server state (workflow files authored with `[[include]]` fragments are the one exception — pull keeps the fragment form while the server's steps still match its expansion; see the workflows guide) — hand-removing the stale entry is the fix that preserves local TOML edits.

Keep everything in the sync tree — including test, probe, and dev workflows. An entity created in the console shows as "Remote only" in `config diff` until a `config pull` folds it (and its sync-state entry) into the tree; from then on it's subject to the staleness above if anything deletes it server-side. Retire an entity by deleting its TOML file and running `config push --prune`, which drops its `.sync-state.json` entry with it.

## Every `[workflow]` field round-trips

`syncCallable`, `capabilities` and `requiresClientApply` are ordinary `[workflow]` TOML fields — `config push` forwards each on create **and** on update, so a field can be flipped on a workflow that already exists:

```bash
primitive config set workflow/order-intake workflow.syncCallable=true
primitive config push --only workflow/order-intake
```

`capabilities` is an array, so edit it in the file rather than with `config set`; an empty array revokes every capability.

## Secrets

App-level secrets are server-side values referenced from workflow/integration TOML as `{{secrets.KEY}}` — never inline credentials in TOML:

```bash
primitive secrets set OPENAI_API_KEY --value sk-...
primitive secrets list
primitive secrets delete OPENAI_API_KEY
```

## Email templates

Primitive sends transactional email on the app's behalf; each type has a built-in default you can override, and workflows can register custom types. Built-in types: `email-sign-in`, `document-share`, `document-share-deferred`, `collection-share`, `collection-share-deferred`, `waitlist-invite`, `waitlist-signup-notification`, `admin-invite`, `app-invite`, `access-request-created`, `access-request-resolved`. Custom types are any kebab-case name, registered by pushing an override and triggered from an `email.send` workflow step. Each type exposes template variables (`{{code}}`, `{{expiryMinutes}}`, the optional `{{magicLink}}`, …) substituted at send time. `magic-link` and `otp` are RETIRED: a stored override for either is kept and listed as retired but never rendered, and create/update/preview/test on one is refused with migration guidance — re-author it into `email-sign-in` (`{{code}}` for the code, the `{{#if magicLink}}` block for the link) and delete the retired override.

An override is `email-templates/<emailType>.toml` — `[template]` with `emailType` and `subject` required, `htmlBody` and `textBody` optional:

```bash
primitive config create email-template email-sign-in   # scaffold the file
primitive config push --only email-template/email-sign-in

primitive email-templates list                     # all types + override status
primitive email-templates get email-sign-in        # current subject + body + variables
primitive email-templates variables email-sign-in  # available {{vars}}
primitive email-templates test email-sign-in       # send a test email
```

Revert to the built-in default by deleting the file and running `primitive config push --prune`.

## Headless auth (CI)

Browser-based `primitive login` doesn't work in CI. Either log in non-interactively by piping a refresh token — `primitive token --refresh | primitive -e <env> login --token-stdin` — or create a long-lived API token and target an environment explicitly:

```bash
primitive tokens create --name "CI deploys" --ttl 90d    # m/h/d/w/mo/y units; omit for non-expiring
PRIMITIVE_ENV=prod primitive config push
```

`login` resolves its server from the active environment's `apiUrl` (inside a project) or the default server (outside one). Set `PRIMITIVE_SERVER_URL` to point an outside-a-project `login` at a custom server (`PRIMITIVE_SERVER_URL=http://localhost:8787 primitive login`). `login` and `init` are the only commands that work outside a project (with `logout` and `bootstrap`); everything else stops with an error naming the missing `primitive/config.json`, because there is no global credentials fallback to run against. An unresolvable environment — multiple environments with no `-e <env>`, no `primitive env use` selection and no `defaultEnvironment` — fails loudly rather than defaulting to production.

`primitive token` (singular) prints the current access token, auto-refreshing it — usable for `curl -H "Authorization: Bearer $(primitive token)"`.

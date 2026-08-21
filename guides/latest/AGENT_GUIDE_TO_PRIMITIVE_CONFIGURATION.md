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

Deleting a config object = delete its file + `primitive config push --prune`. Every type, no exceptions — there is no per-type delete verb.

Not configuration, so still direct commands: secrets (`primitive secrets set`), resource creation that mints an id (`apps create`, `databases create`, `collections create`, `groups create`), data operations, and every read/test command.

Availability is runtime state, not configuration. `<noun> disable`/`enable` — for `workflows`, `cron-triggers`, `webhooks`, `integrations` and `prompts` alike — are its only writers, along with the matching web-admin action. `status` is server-owned and is deliberately not a TOML key: `config pull` does not emit it, a file that still carries the line fails `config push` with guidance naming the verb, and `config diff` reports "inactive; configuration unchanged" rather than drift. Anything created or pushed is active; there is no `draft` state on any object. Read the current value back with `<noun> get` or `<noun> list`. Per-VERSION status is a different key and stays in TOML: `[[configs]] status = "archived"` retires one named config, and says nothing about whether the object is serving.

## The sync loop

```bash
primitive config init  # create directory structure
primitive config pull  # download server config as TOML
primitive config diff  # preview changes
primitive config push  # apply local TOML to the server
```

With project-scoped environments (`.primitive/config.json`), the config directory auto-resolves to `.primitive/sync/<env>/<appId>/` — one isolated slot per environment, so a `pull --env staging` never touches production state. Pass `--dir <path>` only to override that location with a fixed directory shared across environments.

Structured values — an operation's `definition`/`params`, a subscription's `params`, a workflow step's object fields — are written as nested TOML tables. A file may instead hold them as single-line JSON strings (the older encoding); the server treats the two identically, `config pull` keeps whichever form a file already uses, and `primitive config migrate-toml` rewrites a file from JSON strings to TOML tables in place. Author new files in table form.

## Pull snapshots and revert

Every `config pull` snapshots the config directory before writing anything; if the snapshot can't be written, the pull aborts with no files changed. Snapshots older than 28 days are pruned automatically. Location follows the sync-dir resolution: in project mode `<projectRoot>/.primitive/sync-backups/<env>/<appId>/` (gitignored local state); with an explicit `--dir`, `<dir>/.snapshots/`.

```bash
primitive config revert --list           # enumerate snapshots, newest first
primitive config revert                  # restore the most recent
primitive config revert --snapshot <id>  # timestamp dirname, or a unique >=8-char prefix
primitive config revert -y               # skip the confirmation prompt
```

`revert` replaces the entire config directory with the snapshot, including `.primitive-sync.json` (the local sync state). It warns when the config directory has uncommitted git changes (the restore overwrites them), and refuses to restore a partial/corrupted snapshot. After reverting, run `primitive config diff` to inspect the restored state versus the server.

## Pull pruning (deleting local files)

After writing the current server entities, `config pull` reconciles deletions: it removes a local file **iff** its key was managed by a prior pull (present in `.primitive-sync.json` sync state) **and** absent from this pull's server entities. This keeps the sync tree a faithful mirror — because a `config push` without `--prune` treats every local TOML as create-or-update, a stale file for a server-deleted entity would re-create it on the next push. (To reconcile deletions the other direction — delete a server entity when its local file is gone — see [Push pruning](#push-pruning-deleting-server-entities).) Presence is read from the **server listing**, never from what this pull just wrote (several types legitimately skip writing a present entity). Pruning is **on by default**; the pull summary reports a `Pruned` count. Removed files are recoverable — the pre-pull snapshot is taken first, so `config revert` restores anything pruned in error.

A file is pruned only when it clears every safety rule; otherwise it stays and keeps its managed marker (its prior state entry is carried forward, so the discriminator survives to the next pull):

- **`--no-prune`** — the operator opted out; no file is removed this pull.
- **Never-pulled file** — a key absent from prior sync state was hand-authored; it is left alone and `config diff` reports it as *new*.
- **Uncommitted git edits** — a file with uncommitted changes (tracked edits; untracked files do not count) is kept and reported, not deleted — the operator may be mid-edit. Remove it yourself or re-create the entity on the server.
- **Failed listing/detail fetch** — a type whose listing or per-entity fetch failed is skipped whole with a warning; an error that reads as an empty list must never delete every local file of a type.
- **Incomplete listing** — a type whose server listing is not paginated (so an entity missing from it may still be live) is skipped; the omitted files are reported so you can delete them yourself if the entity is really gone.
- **Unsafe filename** — a sync-state key that does not map to a safe filename is skipped with a warning.

When a block type (workflow, prompt) is pruned, its sidecar `<key>.tests/` directory and the block's test-case state are removed with it.

## Push pruning (deleting server entities)

`config push` creates and updates only — by default it never deletes, so removing a local TOML leaves its server entity in place. Pass **`--prune`** to reconcile deletions the other direction: after the create/update pass, push deletes managed entities whose local declaration is gone. A candidate is an entity whose key was recorded in `.primitive-sync.json` by a prior pull **and** which no local file in its directory still declares — keys are read from each file's declared identity (the in-file key or, for keyless types, the basename), mirroring how push derives them, so a file renamed away from `<key>.toml` but still declaring the key keeps its entity out of the prune set. A key never in sync state was authored server-side and is never a prune candidate. A push **without** `--prune` deletes nothing and makes no extra server calls.

Prune is fail-closed on every candidate:

- **Point read + drift check** — each candidate is confirmed by a detail fetch by id/key (immune to listing truncation), and deleted only when the live `modifiedAt` matches the value recorded at last sync. A mismatch (edited out-of-band) is skipped as *drift*; **`--force`** deletes regardless.
- **404** — the entity is already gone server-side; its stale sync-state entry is dropped and nothing is deleted.
- **Any other fetch failure** — presence could not be confirmed, so the entity and its state are left untouched.
- **Still referenced (409)** — the delete is blocked and reported, its state kept, and the remaining deletes proceed (partial success — remove the referrer and re-run).
- **Orphaned metadata values** — deleting a **metadata-category-config** deletes the definition only; stored metadata values are NOT deleted and become unreachable (no query path from a category to its rows). The prune plan states this caveat under the delete line, in `--dry-run` too. Delete the values first (`primitive metadata delete`) if you need them gone.

One batch confirmation gates the destructive deletes (`Delete N remote entity(ies)? This cannot be undone.`); **`-y`/`--yes`** skips it for CI. **`--dry-run`** reports the full plan — will delete / already absent / skipped, with the skip reason — and mutates nothing. When a pruned block type (workflow, prompt) is deleted, its sidecar `<key>.tests/` directory and test-case state are cleaned up with it.

Webhooks are the one exception to what a prune deletion means. Every other archive-semantics type is archived by a prune; a pruned **webhook** is hard-deleted, so its slot against the 50-webhook-per-app cap and its `key` are both freed. That is deliberate — an archiving prune would burn a slot every cycle with no way to reclaim it from `sync` — but it is irreversible, and the webhook's delivery history stops resolving.

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

## App settings (`app.toml`)

App-level settings sync from `app.toml`. Edit the TOML and apply it with `primitive config push` (or `config push --only app` for the settings alone); `primitive config pull --only app` writes current server settings into it, `primitive config diff --only app` shows per-field differences, and `primitive apps get` renders the server-effective settings without touching any file. There is no command that writes a setting — `app.toml` plus a push is the only way to change one. TOML-syncable settings:

- `[app]` — `name`, `mode`, `baseUrl`, `waitlistEnabled`, `waitlistNotifyAdmins`, `directLlmEnabled` (boolean; opts into the deprecated direct LLM/Gemini proxy routes, off by default), `allowedDomains` (string array), `testAccountBaseEmails` (string array)
- `[auth]` — `googleOAuthEnabled`, `googleClientId`, `googleClientSecret` (a `{{secrets.KEY}}` reference — see below), `magicLinkEnabled`, `passkeyEnabled`, `appleSignInEnabled`, `otpEnabled`, `appleAudiences` (string array), `redirectUris` (string array), `[auth.passkeys]` relying-party config
- `[cors]` — `mode`, `allowedOrigins`, `allowCredentials`, `allowedMethods`, `allowedHeaders`, `exposedHeaders`, `maxAge` (the `[cors]` table is always emitted, in every mode)
- `[invitations]` — `enabled`, `limit` (whether role `member` users may send invitations, and the per-member cap; `0` = unlimited)

Push forwards only recognized keys and only those present: an omitted key is left untouched on the server (not cleared), an explicit `false` is forwarded, and `appleAudiences = []` clears the audiences. This omit-preserves rule is specific to app settings — a synced entity's owned scalar fields are cleared when their line is removed (see [Owned scalar fields](#owned-scalar-fields-clear-on-absence)). An unrecognized key is ignored with a warning (`Unrecognized [<section>] key "<key>" in app.toml — ignored. Recognized keys: …`).

`googleClientSecret` holds a whole `{{secrets.KEY}}` reference, never the secret itself, so it round-trips through `app.toml` like any other string setting. Store the value as an app secret first:

```bash
primitive secrets set GOOGLE_CLIENT_SECRET --value <client-secret>
```

```toml
[auth]
googleClientId = "1234.apps.googleusercontent.com"
googleClientSecret = "{{secrets.GOOGLE_CLIENT_SECRET}}"
```

A literal value is rejected with `GOOGLE_CLIENT_SECRET_MUST_BE_SECRET_REF`, and a reference naming a key that doesn't exist with `MISSING_GOOGLE_CLIENT_SECRET_REF`. Clearing the setting (`""`) leaves the app secret in place.

A plain value stored before this rule shipped is a literal, and it keeps working — Google sign-in on an app that never migrated still uses the stored secret. It is deprecated, and it blocks configuration: while the stored value is a literal, **any** app-settings write is rejected with `GOOGLE_CLIENT_SECRET_MIGRATION_REQUIRED` unless that same write migrates the field — sets a valid `{{secrets.KEY}}` reference, or clears it. Store the value as an app secret and re-point the setting in one push.

`GOOGLE_OAUTH_MISCONFIGURED` is the code for a stored value that can't be resolved to a client secret — a *reference* naming a secret that doesn't exist, or a malformed reference (below): the **web** sign-in flow fails closed with it before the request reaches Google. Native (PKCE) sign-in is deliberately exempt from that guard — a PKCE exchange can prove possession of the auth code with the code verifier alone, so Primitive lets it through rather than rejecting it outright. Exempt is not unaffected: when the reference resolves the server does send `client_secret` alongside `code_verifier`, and Google requires it for the "Web application" client type the setup guide provisions — so on a dangling reference the native exchange fails at Google instead, as a generic `INVALID_TOKEN`.

The server only ever sends `googleClientSecret` back when it holds a reference. A pre-existing literal is real credential material, so it is withheld from every read — `apps get`, `config pull`, the API — and reported as `googleClientSecretStatus = "legacy-literal"` instead. Google sign-in keeps working on such an app; the setting is simply absent from `app.toml`. `apps get` and `config pull` each warn when they read that status, naming the remediation — a pulled `app.toml` cannot be pushed back until the field is migrated. Re-point it at a secret reference and push.

`googleClientSecretStatus = "malformed-reference"` is the case that is **not** working: the stored value carries reference syntax that no `{{secrets.KEY}}` reference accounts for (`{{secrets.foo}}`, `{secrets.KEY}`, or an otherwise-valid reference carrying an invisible character such as a zero-width space), so it is neither a pointer nor the secret Google issued, and sign-in fails with `GOOGLE_OAUTH_MISCONFIGURED`. It is withheld from reads for the same reason a literal is. Store the real client secret and point the setting at it.

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

Every command resolves its target environment in order: `--env <name>` flag → `PRIMITIVE_ENV` env var → `defaultEnvironment` in `.primitive/config.json` → the only defined environment → error. Manage environments with `primitive env add|list|show|use|remove`. Tokens are stored per-environment in `.primitive/credentials.json` (gitignored); `.primitive/config.json` is committed.

`primitive env add` writes only the environment entry into `.primitive/config.json` — it seeds no credentials, and project mode does **not** fall back to the global `~/.primitive/credentials.json`. A freshly-added environment therefore starts logged-out: project-scoped commands report "not logged in" until you run `primitive login` for that environment, even when a global `primitive whoami` succeeds. (The `dev` environment scaffolded by `primitive init` is the exception — init seeds it with the session it authenticated during setup.) Agents and CI can log in without a browser by piping a refresh token — `primitive token --refresh | primitive -e <env> login --token-stdin` (see [Headless auth](#headless-auth-ci)).

## Previewing a push

`primitive config diff` lists entities that would be created, changed, or removed; `primitive config push --dry-run` reports the full push without applying it. A local file with no matching server entity is reported in one of two buckets, counted separately in the summary: **Local only (new)** — a file no prior pull managed, hand-authored, which push will create — and **Local only (absent from export)** — a file a prior pull wrote whose entity is now absent from the server (deleted server-side); push would RE-create it unless you remove the file, and a default `config pull` would prune it (see [Pull pruning](#pull-pruning-deleting-local-files)). The discriminator is the same prior-sync-state check the pull-side prune uses. A server entity with **no** local file is reported under **Remote only**, split by the same check: **Remote only (managed)** — a prior pull recorded it, so `push --prune` deletes it (see [Push pruning](#push-pruning-deleting-server-entities)) — and **Remote only (unmanaged)** — never synced, authored server-side, left alone. **`diff` covers every resource type `config push` handles** — the config types (database types, rule sets, group- and collection-type configs, metadata-category configs), email templates, webhooks and prompts are compared for content, not just presence, so an edit to one of them is reported as **Modified** rather than silently reading as in-sync until push applies it. Any type whose current server state `diff` could not fetch is listed under an explicit **not compared** line instead of being omitted, so the summary is never mistaken for exhaustive. **`diff` counts entities only** (workflows, prompts, database types, integrations, webhooks, …) — an `app.toml` / app-settings-only edit is invisible to it, reporting every count as zero, which reads as "nothing to push" when there is. App-settings changes surface only in `push --dry-run`, which walks the full push. Both run the **same** validate-first gate as a real push — local TOML validation followed by the server-side checks via the validate-first pass — so the preview is faithful: what it reports is what the push applies. For the types push reconciles against live server state — the config types, email templates, transforms and prompts — both commands decide from the **same** comparison, so a comment-only or formatting-only edit is a change to neither, and a difference `diff` reports is one `push` acts on: it applies the edit, or reports it as server drift or a conflict rather than overwriting. Integrations, webhooks, cron triggers and blob buckets still gate their push on the file's byte hash, so for those a cosmetic edit can still appear as a push update. One prompt difference push cannot apply is named instead of being reported as a clean update: push updates and creates the `[[configs]]` entries the file lists, but deletes none and clears no activation, so a file omitting an entry the server holds — or naming no `active` configuration — is reported as still differing after the push (`Prompt <key> still differs from the server after this push`), which `config diff` will keep reporting until you `config pull` or remove the configuration server-side. Schema-gate rejections surface identically in the preview and in a real push: an operation whose database type has no schema set, an unresolved `$params.X`/reference, or a schema change that would break an existing registered operation. A previewed or blocked entity records no content hash in sync state, so it stays pending on the next `config diff` rather than reading "in sync" — a server-rejected change cannot silently disappear from the diff. `primitive config diff --json` emits machine-readable output on stdout.

## Push failures

`config push` validates every TOML file before applying anything — any validation error aborts the push with no changes applied (`Aborting push: N TOML validation error(s) — no changes were applied.`). For workflows it validates `$params.X` references against declared `[[operations.params]]` entries at push time, naming the file and line of a bad reference. When validation passes but the server rejects an entity, the error names the entity and file. A workflow, cron trigger, blob bucket, rule set, integration, webhook, or prompt that already exists on the server but is missing from local sync state is adopted by key and updated in place rather than failing the create (a 409 or a "must be unique" conflict) — re-running a failed push converges. For the types push reconciles against live server state (rule sets, transform scripts, prompts, and the group-, collection- and metadata-type configs) the adoption happens one step earlier, in the reconcile read: an out-of-manifest entity the server already holds is compared rather than overwritten, so one that MATCHES the file is simply recorded in sync state, and one that differs is reported as a conflict — with no recorded baseline nothing says which side moved, so the remedy is `config pull` (take the server's version) or `push --force` (make the local file win). Database types and transform scripts converge the same way through their own conflict recovery, as do group- and collection-type configs — a config created out-of-band (whose "already exists" 409 would otherwise fail every later push) is adopted by its type-name key and updated in place. Adoption matches only on the entity's key, and a few immutability guards fail the adopt loudly rather than letting sync state diverge: a blob bucket whose local `ttlTier` differs from the server's fails the adopt with both values named (`ttlTier` is immutable — reconcile the local file to the server's tier, or recreate the bucket); a rule set whose local `resourceType` differs from the server's is rejected the same way (`resourceType` is immutable — align the TOML, or rename/recreate the rule set). Prompt adoption covers only the create itself: post-create config work (extra prompt configs, activation) runs outside the adopt path, so a duplicate-config conflict surfaces as the real error instead of being mis-adopted, and sync state is stamped only after all create work succeeds. Diagnostics go to stderr; `--json` data goes to stdout (pipes stay clean).

## Out-of-band changes and stale sync state

Push decides create-vs-update per entity from the server id stored in `.primitive-sync.json` (`entities.<entityType>.<entityKey>.id`, at the root of the config directory; database types and the group/collection/metadata type-config families track the entry without an id — the decision there is on the entry's presence). The types push reconciles against live server state (rule sets, transform scripts, prompts, and the group-, collection- and metadata-type configs) decide it from that read instead, and fall back to the stored id only when the server could not be read. Changes made outside the sync loop — console edits — leave that state out of date, and the two directions behave differently:

- **Created out-of-band** (a console edit) — converges for workflows, cron triggers, database types, blob buckets, transform scripts, rule sets, integrations, webhooks, prompts, and group- and collection-type configs: push adopts the existing entity by key and updates it in place instead of failing the create (see [Push failures](#push-failures)).
- **Deleted out-of-band** — does not self-heal, except for the live-reconciled types above: their reconcile read finds no entity, so the next push creates it fresh. For the rest the stored id goes stale, and the next push of a changed TOML for that entity issues an update against the missing entity, failing the push with a "not found" error for that entity.

Symptom → fix: `config push` fails with `Failed to update <entity> <key>: … not found` ⇒ the entity was deleted on the server while its sync-state entry survived. Delete `entities.<entityType>.<entityKey>` from `.primitive-sync.json` and push again — the entity is created fresh (or adopted, for the kinds listed above). A full `config pull` also rebuilds sync state from the server, but it overwrites every server-backed file with server state (workflow files authored with `[[include]]` fragments are the one exception — pull keeps the fragment form while the server's steps still match its expansion; see the workflows guide) — hand-removing the stale entry is the fix that preserves local TOML edits.

Keep everything in the sync tree — including test, probe, and dev workflows. An entity created in the console shows as "Remote only" in `config diff` until a `config pull` folds it (and its sync-state entry) into the tree; from then on it's subject to the staleness above if anything deletes it server-side. Retire an entity by deleting its TOML file and running `config push --prune`, which drops its `.primitive-sync.json` entry with it.

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

Primitive sends transactional email on the app's behalf; each type has a built-in default you can override, and workflows can register custom types. Built-in types: `magic-link`, `otp`, `document-share`, `document-share-deferred`, `collection-share`, `collection-share-deferred`, `waitlist-invite`, `waitlist-signup-notification`, `admin-invite`, `app-invite`, `access-request-created`, `access-request-resolved`. Custom types are any kebab-case name, registered by pushing an override and triggered from an `email.send` workflow step. Each type exposes template variables (`{{magicLinkUrl}}`, `{{otpCode}}`, …) substituted at send time.

An override is `email-templates/<emailType>.toml` — `[template]` with `emailType` and `subject` required, `htmlBody` and `textBody` optional:

```bash
primitive config create email-template magic-link     # scaffold the file
primitive config push --only email-template/magic-link

primitive email-templates list                   # all types + override status
primitive email-templates get magic-link         # current subject + body + variables
primitive email-templates variables magic-link   # available {{vars}}
primitive email-templates test magic-link        # send a test email
```

Revert to the built-in default by deleting the file and running `primitive config push --prune`.

## Headless auth (CI)

Browser-based `primitive login` doesn't work in CI. Either log in non-interactively by piping a refresh token — `primitive token --refresh | primitive -e <env> login --token-stdin` — or create a long-lived API token and target an environment explicitly:

```bash
primitive tokens create --name "CI deploys" --ttl 90d    # m/h/d/w/mo/y units; omit for non-expiring
PRIMITIVE_ENV=prod primitive config push
```

`login` resolves its server from the active environment's `apiUrl` (project mode) or the default server (legacy/no project config). Set `PRIMITIVE_SERVER_URL` to point legacy-mode login and scripts at a custom server (`PRIMITIVE_SERVER_URL=http://localhost:8787 primitive login`). An unresolvable environment — multiple environments with no `-e <env>` and no `defaultEnvironment` — fails loudly rather than defaulting to production.

`primitive token` (singular) prints the current access token, auto-refreshing it — usable for `curl -H "Authorization: Bearer $(primitive token)"`.

# Agent Guide to Primitive Configuration

Guidelines for AI agents configuring Primitive services. Everything Primitive does for an app — auth settings, workflows, prompts, integrations, database types/operations, webhooks, cron triggers, blob buckets, email templates, rule sets — is server-side configuration with two equivalent interfaces: the web Admin Console (interactive) and **TOML files synced via the CLI** (configuration as code). Agents should use the TOML + CLI path.

## The sync loop

```bash
primitive sync init     # create directory structure
primitive sync pull     # download server config as TOML
primitive sync diff     # preview changes
primitive sync push     # apply local TOML to the server
```

With project-scoped environments (`.primitive/config.json`), the sync directory auto-resolves to `.primitive/sync/<env>/<appId>/` — one isolated slot per environment, so a `pull --env staging` never touches production state. Pass `--dir <path>` only to override that location with a fixed directory shared across environments.

## Pull snapshots and revert

Every `sync pull` snapshots the sync directory before writing anything; if the snapshot can't be written, the pull aborts with no files changed. Snapshots older than 28 days are pruned automatically. Location follows the sync-dir resolution: in project mode `<projectRoot>/.primitive/sync-backups/<env>/<appId>/` (gitignored local state); with an explicit `--dir`, `<dir>/.snapshots/`.

```bash
primitive sync revert --list            # enumerate snapshots, newest first
primitive sync revert                   # restore the most recent
primitive sync revert --snapshot <id>   # timestamp dirname, or a unique >=8-char prefix
primitive sync revert -y                # skip the confirmation prompt
```

`revert` replaces the entire sync directory with the snapshot, including `.primitive-sync.json` (the local sync state). It warns when the config directory has uncommitted git changes (the restore overwrites them), and refuses to restore a partial/corrupted snapshot. After reverting, run `primitive sync diff` to inspect the restored state versus the server.

## Pull pruning (deleting local files)

After writing the current server entities, `sync pull` reconciles deletions: it removes a local file **iff** its key was managed by a prior pull (present in `.primitive-sync.json` sync state) **and** absent from this pull's server entities. This keeps the sync tree a faithful mirror — because a `sync push` without `--prune` treats every local TOML as create-or-update, a stale file for a server-deleted entity would re-create it on the next push. (To reconcile deletions the other direction — delete a server entity when its local file is gone — see [Push pruning](#push-pruning-deleting-server-entities).) Presence is read from the **server listing**, never from what this pull just wrote (several types legitimately skip writing a present entity). Pruning is **on by default**; the pull summary reports a `Pruned` count. Removed files are recoverable — the pre-pull snapshot is taken first, so `sync revert` restores anything pruned in error.

A file is pruned only when it clears every safety rule; otherwise it stays and keeps its managed marker (its prior state entry is carried forward, so the discriminator survives to the next pull):

- **`--no-prune`** — the operator opted out; no file is removed this pull.
- **Never-pulled file** — a key absent from prior sync state was hand-authored; it is left alone and `sync diff` reports it as *new*.
- **Uncommitted git edits** — a file with uncommitted changes (tracked edits; untracked files do not count) is kept and reported, not deleted — the operator may be mid-edit. Remove it yourself or re-create the entity on the server.
- **Failed listing/detail fetch** — a type whose listing or per-entity fetch failed is skipped whole with a warning; an error that reads as an empty list must never delete every local file of a type.
- **Incomplete listing** — a type whose server listing is not paginated (so an entity missing from it may still be live) is skipped; the omitted files are reported so you can delete them yourself if the entity is really gone.
- **Unsafe filename** — a sync-state key that does not map to a safe filename is skipped with a warning.

When a block type (workflow, prompt) is pruned, its sidecar `<key>.tests/` directory and the block's test-case state are removed with it.

## Push pruning (deleting server entities)

`sync push` creates and updates only — by default it never deletes, so removing a local TOML leaves its server entity in place. Pass **`--prune`** to reconcile deletions the other direction: after the create/update pass, push deletes managed entities whose local declaration is gone. A candidate is an entity whose key was recorded in `.primitive-sync.json` by a prior pull **and** which no local file in its directory still declares — keys are read from each file's declared identity (the in-file key or, for keyless types, the basename), mirroring how push derives them, so a file renamed away from `<key>.toml` but still declaring the key keeps its entity out of the prune set. A key never in sync state was authored server-side and is never a prune candidate. A push **without** `--prune` deletes nothing and makes no extra server calls.

Prune is fail-closed on every candidate:

- **Point read + drift check** — each candidate is confirmed by a detail fetch by id/key (immune to listing truncation), and deleted only when the live `modifiedAt` matches the value recorded at last sync. A mismatch (edited out-of-band) is skipped as *drift*; **`--force`** deletes regardless.
- **404** — the entity is already gone server-side; its stale sync-state entry is dropped and nothing is deleted.
- **Any other fetch failure** — presence could not be confirmed, so the entity and its state are left untouched.
- **Still referenced (409)** — the delete is blocked and reported, its state kept, and the remaining deletes proceed (partial success — remove the referrer and re-run).
- **No delete surface** — metadata-category-configs have no delete path; a candidate of that type is reported and never deleted.

One batch confirmation gates the destructive deletes (`Delete N remote entity(ies)? This cannot be undone.`); **`-y`/`--yes`** skips it for CI. **`--dry-run`** reports the full plan — will delete / already absent / skipped, with the skip reason — and mutates nothing. When a pruned block type (workflow, prompt) is deleted, its sidecar `<key>.tests/` directory and test-case state are cleaned up with it.

## Directory map

```
config/
  app.toml                        # App settings
  workflows/*.toml                # Workflow definitions
  workflows/{key}.tests/*.toml    # Workflow test cases
  workflow-fragments/*.toml       # Reusable [[steps]] blocks (include = ["<name>"])
  transforms/*.rhai               # Rhai scripts for workflow script steps
  prompts/*.toml                  # Managed prompts
  prompts/{key}.tests/*.toml      # Prompt test cases
  integrations/*.toml             # External API integrations
  database-types/*.toml           # Database type configs + operations + subscriptions
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

App-level settings sync from `app.toml`. Edit the TOML and apply with `primitive settings push` (or a whole-config `sync push` — both operate on the same `app.toml`); `primitive settings pull` writes current server settings into it, `primitive settings diff` shows per-field differences, and `primitive settings show` renders the server-effective settings without touching any file. Change these settings through the TOML path, not with imperative `primitive apps update --flag` calls: those mutate the server directly and drift from the checked-in TOML, so the next push reverts them unless mirrored back. TOML-syncable settings:

- `[app]` — `name`, `mode`, `baseUrl`, `waitlistEnabled`, `waitlistNotifyAdmins`, `allowedDomains` (string array), `testAccountBaseEmails` (string array)
- `[auth]` — `googleOAuthEnabled`, `googleClientId`, `magicLinkEnabled`, `passkeyEnabled`, `appleSignInEnabled`, `otpEnabled`, `appleAudiences` (string array), `redirectUris` (string array), `[auth.passkeys]` relying-party config
- `[cors]` — `mode`, `allowedOrigins`, `allowCredentials`, `allowedMethods`, `allowedHeaders`, `exposedHeaders`, `maxAge` (the `[cors]` table is always emitted, in every mode)
- `[invitations]` — `enabled`, `limit` (whether role `member` users may send invitations, and the per-member cap; `0` = unlimited)

Push forwards only recognized keys and only those present: an omitted key is left untouched on the server (not cleared), an explicit `false` is forwarded, and `appleAudiences = []` clears the audiences. This omit-preserves rule is specific to app settings — a synced entity's owned scalar fields are cleared when their line is removed (see [Owned scalar fields](#owned-scalar-fields-clear-on-absence)). An unrecognized key is ignored with a warning (`Unrecognized [<section>] key "<key>" in app.toml — ignored. Recognized keys: …`). The Google OAuth **client secret** is the one auth setting excluded from `app.toml`: `settings pull` never writes it, and a hand-added `googleClientSecret` key under `[auth]` aborts the push before any write. Set it with `primitive apps update <app-id> --google-client-secret <secret>` (`''` clears it) — this flag is the exception, since the secret never lives in `app.toml`.

## Owned scalar fields (clear on absence)

For a set of **TOML-owned scalar fields** on synced entities, the local file is the source of truth: `sync push` always sends these fields as value-or-`null`, so removing a field's line from the TOML clears it on the server rather than leaving the prior value in place. This is field-level and separate from removing a whole TOML file — a default push never deletes an entity (whole-file reconciliation is [Pull pruning](#pull-pruning-deleting-local-files), or opt-in [Push pruning](#push-pruning-deleting-server-entities)). Owned fields by entity type:

- **database-types** — `ruleSetName` (wire `ruleSetId`), `triggers`, `celContextAccess` (wire `metadataAccess`; `celContextAccess = ""` is honored as an explicit clear), `defaultAccess`, `autoPopulatedFields`, `timestamps`, and the `[metadata]`/`secrets` manifest. The `[models.*]` schema is owned too but tracked separately — removing every `[models.*]` block forwards `schema: null` and clears the schema (blocked while operations are still registered on the type — delete or migrate them first). `push` prints `Cleared <field> on <type>` on a genuine non-null → null transition.
- **group-type-configs** — `ruleSetName` (wire `ruleSetId`).
- **integrations, webhooks, cron-triggers, blob-buckets** — `description`.
- **prompts** — `description`, `inputSchema`, and each `[[config]]` block's `description`.

Enum and defaulted fields (`status`, `timeoutMs`, `timezone`, `overlapPolicy`, `state`, …) are **not** cleared this way — `null` is not a valid value for them, so omitting the line leaves the current server value unchanged. To change one, set it explicitly in the TOML.

## Environment resolution

Every command resolves its target environment in order: `--env <name>` flag → `PRIMITIVE_ENV` env var → `defaultEnvironment` in `.primitive/config.json` → the only defined environment → error. Manage environments with `primitive env add|list|show|use|remove`. Tokens are stored per-environment in `.primitive/credentials.json` (gitignored); `.primitive/config.json` is committed.

`primitive env add` writes only the environment entry into `.primitive/config.json` — it seeds no credentials, and project mode does **not** fall back to the global `~/.primitive/credentials.json`. A freshly-added environment therefore starts logged-out: project-scoped commands report "not logged in" until you run `primitive login` for that environment, even when a global `primitive whoami` succeeds. Agents and CI can log in without a browser by piping a refresh token — `primitive token --refresh | primitive -e <env> login --token-stdin` (see [Headless auth](#headless-auth-ci)).

## Previewing a push

`primitive sync diff` lists entities that would be created, changed, or removed; `primitive sync push --dry-run` reports the full push without applying it. A local file with no matching server entity is reported in one of two buckets, counted separately in the summary: **Local only (new)** — a file no prior pull managed, hand-authored, which push will create — and **Local only (absent from export)** — a file a prior pull wrote whose entity is now absent from the server (deleted server-side); push would RE-create it unless you remove the file, and a default `sync pull` would prune it (see [Pull pruning](#pull-pruning-deleting-local-files)). The discriminator is the same prior-sync-state check the pull-side prune uses. A server entity with **no** local file is reported under **Remote only**, split by the same check: **Remote only (managed)** — a prior pull recorded it, so `push --prune` deletes it (see [Push pruning](#push-pruning-deleting-server-entities)) — and **Remote only (unmanaged)** — never synced, authored server-side, left alone. **`diff` covers every resource type `sync push` handles** — the config types (database types, rule sets, group- and collection-type configs, metadata-category configs) are compared for content, not just presence, so an edit to one of them is reported as **Modified** rather than silently reading as in-sync until push applies it. Any type whose current server state `diff` could not fetch is listed under an explicit **not compared** line instead of being omitted, so the summary is never mistaken for exhaustive. **`diff` counts entities only** (workflows, prompts, database types, integrations, webhooks, …) — an `app.toml` / app-settings-only edit is invisible to it, reporting every count as zero, which reads as "nothing to push" when there is. App-settings changes surface only in `push --dry-run`, which walks the full push. Both run the **same** validate-first gate as a real push — local TOML validation followed by the server-side checks via the validate-first pass — so the preview is faithful: what it reports is what the push applies. Schema-gate rejections surface identically in the preview and in a real push: an operation whose database type has no schema set, an unresolved `$params.X`/reference, or a schema change that would break an existing registered operation. A previewed or blocked entity records no content hash in sync state, so it stays pending on the next `sync diff` rather than reading "in sync" — a server-rejected change cannot silently disappear from the diff. `primitive sync diff --json` emits machine-readable output on stdout.

## Push failures

`sync push` validates every TOML file before applying anything — any validation error aborts the push with no changes applied (`Aborting push: N TOML validation error(s) — no changes were applied.`). For workflows it validates `$params.X` references against declared `[[operations.params]]` entries at push time, naming the file and line of a bad reference. When validation passes but the server rejects an entity, the error names the entity and file. A workflow, cron trigger, blob bucket, rule set, integration, webhook, or prompt that already exists on the server but is missing from local sync state is adopted by key and updated in place rather than failing the create (a 409 or a "must be unique" conflict) — re-running a failed push converges. Database types and transform scripts converge the same way through their own conflict recovery. Adoption matches only on the entity's key, and a few immutability guards fail the adopt loudly rather than letting sync state diverge: a blob bucket whose local `ttlTier` differs from the server's fails the adopt with both values named (`ttlTier` is immutable — reconcile the local file to the server's tier, or recreate the bucket); a rule set whose local `resourceType` differs from the server's is rejected the same way (`resourceType` is immutable — align the TOML, or rename/recreate the rule set). Prompt adoption covers only the create itself: post-create config work (extra prompt configs, activation) runs outside the adopt path, so a duplicate-config conflict surfaces as the real error instead of being mis-adopted, and sync state is stamped only after all create work succeeds. Diagnostics go to stderr; `--json` data goes to stdout (pipes stay clean).

## Out-of-band changes and stale sync state

Push decides create-vs-update per entity from the server id stored in `.primitive-sync.json` (`entities.<entityType>.<entityKey>.id`, at the root of the sync directory; database types and the group/collection/metadata type-config families track the entry without an id — the decision there is on the entry's presence). Changes made outside the sync loop — console edits, ad-hoc CLI creates/deletes — leave that state out of date, and the two directions behave differently:

- **Created out-of-band** — converges for workflows, cron triggers, database types, blob buckets, transform scripts, rule sets, integrations, webhooks, and prompts: push adopts the existing entity by key and updates it in place instead of failing the create (see [Push failures](#push-failures)).
- **Deleted out-of-band** — does not self-heal. The stored id goes stale, and the next push of a changed TOML for that entity issues an update against the missing entity, failing the push with a "not found" error for that entity.

Symptom → fix: `sync push` fails with `Failed to update <entity> <key>: … not found` ⇒ the entity was deleted on the server while its sync-state entry survived. Delete `entities.<entityType>.<entityKey>` from `.primitive-sync.json` and push again — the entity is created fresh (or adopted, for the kinds listed above). A full `sync pull` also rebuilds sync state from the server, but it overwrites every server-backed file with server state — hand-removing the stale entry is the fix that preserves local TOML edits.

Keep everything in the sync tree — including test, probe, and dev workflows. An entity created ad-hoc shows as "Remote only" in `sync diff` until a `sync pull` folds it (and its sync-state entry) into the tree; from then on it's subject to the staleness above if anything deletes it server-side. Create through TOML + push; when retiring an entity, delete it explicitly (CLI or console), then remove its TOML file *and* its `.primitive-sync.json` entry together.

## What sync does NOT carry

A few settings are set with dedicated commands rather than TOML (app-level settings that *do* sync are listed under [App settings](#app-settings-app-toml)). The Google OAuth **client secret** is the app-level exception — it is never written to `app.toml`, and a hand-added `googleClientSecret` key aborts the push; set it with `primitive apps update <id> --google-client-secret <secret>`. Two workflow-level fields are the exception to "TOML round-trips everything": `sync push` sets them only when it **creates** a workflow, never when it updates one already on the server — flip them on an existing workflow with a direct update instead:

```bash
primitive workflows update <id> --sync-callable true     # existing workflow only; the server re-validates against the currently-active steps
primitive workflows update <id> --capabilities membership   # existing workflow only; comma-separated, "" revokes all
primitive apps update <id> --google-client-secret <secret>  # secret, excluded from app.toml; "" clears
```

`requiresClientApply` is an ordinary `[workflow]` TOML field — `sync push` forwards it on both create and update — so `primitive workflows update <id> --requires-client-apply false` is a convenience, not the only path.

## Secrets

App-level secrets are server-side values referenced from workflow/integration TOML as `{{secrets.KEY}}` — never inline credentials in TOML:

```bash
primitive secrets set OPENAI_API_KEY --value sk-...
primitive secrets list
primitive secrets delete OPENAI_API_KEY
```

## Headless auth (CI)

Browser-based `primitive login` doesn't work in CI. Either log in non-interactively by piping a refresh token — `primitive token --refresh | primitive -e <env> login --token-stdin` — or create a long-lived API token and target an environment explicitly:

```bash
primitive tokens create --name "CI deploys" --ttl 90d    # m/h/d/w/mo/y units; omit for non-expiring
PRIMITIVE_ENV=prod primitive sync push
```

`login` resolves its server from the active environment's `apiUrl` (project mode) or the default server (legacy/no project config). Set `PRIMITIVE_SERVER_URL` to point legacy-mode login and scripts at a custom server (`PRIMITIVE_SERVER_URL=http://localhost:8787 primitive login`). An unresolvable environment — multiple environments with no `-e <env>` and no `defaultEnvironment` — fails loudly rather than defaulting to production.

`primitive token` (singular) prints the current access token, auto-refreshing it — usable for `curl -H "Authorization: Bearer $(primitive token)"`.

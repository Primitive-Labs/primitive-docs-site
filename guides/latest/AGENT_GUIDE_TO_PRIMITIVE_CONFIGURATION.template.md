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

App-level settings sync from `app.toml`; editing the TOML and `sync push` is the default path. The equivalent `primitive apps update --flag` calls mutate the server imperatively and drift from the checked-in TOML — the next `sync push` reverts them unless mirrored back. TOML-syncable settings:

- `[app]` — `name`, `mode`, `waitlistEnabled`, `baseUrl`
- `[auth]` — `googleOAuthEnabled`, `magicLinkEnabled`, `passkeyEnabled`, `appleSignInEnabled`, `otpEnabled`, `appleAudiences` (string array), `[auth.passkeys]` relying-party config
- `[cors]` (serialized only when `mode = "custom"`) — `allowedOrigins`, `allowCredentials`, `allowedMethods`, `maxAge`

Push forwards only recognized `[auth]` keys and only those present: an omitted key is left untouched on the server (not cleared), an explicit `false` is forwarded, and `appleAudiences = []` clears the audiences. An unrecognized `[auth]` key is ignored with a warning (`Unrecognized [auth] key "<key>" in app.toml — ignored. Recognized keys: …`). Not in `app.toml` (see [What sync does NOT carry](#what-sync-does-not-carry)): `redirectUris` (set via `--redirect-uris "<uri1>,<uri2>"` or the Admin Console — no TOML key).

## Environment resolution

Every command resolves its target environment in order: `--env <name>` flag → `PRIMITIVE_ENV` env var → `defaultEnvironment` in `.primitive/config.json` → the only defined environment → error. Manage environments with `primitive env add|list|show|use|remove`. Tokens are stored per-environment in `.primitive/credentials.json` (gitignored); `.primitive/config.json` is committed.

`primitive env add` writes only the environment entry into `.primitive/config.json` — it seeds no credentials, and project mode does **not** fall back to the global `~/.primitive/credentials.json`. A freshly-added environment therefore starts logged-out: project-scoped commands report "not logged in" until you run `primitive login` for that environment, even when a global `primitive whoami` succeeds. Agents and CI can log in without a browser by piping a refresh token — `primitive token --refresh | primitive -e <env> login --token-stdin` (see [Headless auth](#headless-auth-ci)).

## Previewing a push

`primitive sync diff` lists entities that would be created, changed, or removed; `primitive sync push --dry-run` reports the full push without applying it. **`diff` counts entities only** (workflows, prompts, database types, integrations, webhooks, …) — an `app.toml` / app-settings-only edit is invisible to it, reporting `Local only: 0 / Remote only: 0 / Synced: 0`, which reads as "nothing to push" when there is. App-settings changes surface only in `push --dry-run`, which walks the full push. Both run the **same** validate-first gate as a real push — local TOML validation followed by the server-side checks via the validate-first pass — so the preview is faithful: what it reports is what the push applies. Schema-gate rejections surface identically in the preview and in a real push: an operation whose database type has no schema set, an unresolved `$params.X`/reference, or a schema change that would break an existing registered operation. A previewed or blocked entity records no content hash in sync state, so it stays pending on the next `sync diff` rather than reading "in sync" — a server-rejected change cannot silently disappear from the diff. `primitive sync diff --json` emits machine-readable output on stdout.

## Push failures

`sync push` validates every TOML file before applying anything — any validation error aborts the push with no changes applied (`Aborting push: N TOML validation error(s) — no changes were applied.`). For workflows it validates `$params.X` references against declared `[[operations.params]]` entries at push time, naming the file and line of a bad reference. When validation passes but the server rejects an entity, the error names the entity and file. A cron trigger, workflow, or blob bucket that already exists on the server but is missing from local sync state is adopted by key and updated in place rather than failing on the 409 — re-running a failed push converges. One adoption guard: a blob bucket whose local `ttlTier` differs from the server's fails the adopt with both values named — `ttlTier` is immutable, so reconcile the local file to the server's tier (or recreate the bucket) rather than expecting push to change it. Diagnostics go to stderr; `--json` data goes to stdout (pipes stay clean).

## Out-of-band changes and stale sync state

Push decides create-vs-update per entity from the server id stored in `.primitive-sync.json` (`entities.<entityType>.<entityKey>.id`, at the root of the sync directory; database types and the group/collection/metadata type-config families track the entry without an id — the decision there is on the entry's presence). Changes made outside the sync loop — console edits, ad-hoc CLI creates/deletes — leave that state out of date, and the two directions behave differently:

- **Created out-of-band** — converges for workflows, cron triggers, database types, blob buckets, and transform scripts: push adopts the existing entity by key and updates it in place instead of failing the create (see [Push failures](#push-failures)).
- **Deleted out-of-band** — does not self-heal. The stored id goes stale, and the next push of a changed TOML for that entity issues an update against the missing entity, failing the push with a "not found" error for that entity.

Symptom → fix: `sync push` fails with `Failed to update <entity> <key>: … not found` ⇒ the entity was deleted on the server while its sync-state entry survived. Delete `entities.<entityType>.<entityKey>` from `.primitive-sync.json` and push again — the entity is created fresh (or adopted, for the kinds listed above). A full `sync pull` also rebuilds sync state from the server, but it overwrites every server-backed file with server state — hand-removing the stale entry is the fix that preserves local TOML edits.

Keep everything in the sync tree — including test, probe, and dev workflows. An entity created ad-hoc shows as "Remote only" in `sync diff` until a `sync pull` folds it (and its sync-state entry) into the tree; from then on it's subject to the staleness above if anything deletes it server-side. Create through TOML + push; when retiring an entity, delete it explicitly (CLI or console), then remove its TOML file *and* its `.primitive-sync.json` entry together.

## What sync does NOT carry

Some settings are set with dedicated update commands rather than TOML (app-level settings that *do* sync are listed under [App settings](#app-settings-app-toml)). Two workflow-level fields are the exception to "TOML round-trips everything": `sync push` sets them only when it **creates** a workflow, never when it updates one already on the server — flip them on an existing workflow with a direct update instead:

```bash
primitive workflows update <id> --sync-callable true     # existing workflow only; the server re-validates against the currently-active steps
primitive workflows update <id> --capabilities membership   # existing workflow only; comma-separated, "" revokes all
primitive apps update --member-invitations-enabled true --member-invitation-limit 5
primitive apps update --test-account-bases alice@example.com
primitive apps update --redirect-uris "https://app.example.com/auth/callback"  # no TOML key
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

# Agent Guide to Primitive App Secrets

Guidelines for AI agents managing credentials in Primitive apps. App secrets are the platform's server-side store for API keys, tokens, and other credentials referenced from backend config as `{{secrets.KEY}}`. Values resolve only on the server — they never appear in the repo, client code, or anything shipped to users. **Config vars** (below) are the non-secret twin of the same mechanism.

## CLI

```bash
primitive secrets set OPENAI_API_KEY --value sk-... --summary "OpenAI production key"
primitive secrets set OPENAI_API_KEY --value sk-new...    # same key = update in place
primitive secrets list                                     # keys + summaries; values never shown
primitive secrets delete OPENAI_API_KEY
```

- **Write-only**: a value can be overwritten or deleted but never read back. Always pass `--summary` so keys stay identifiable.
- **Per-app scope**: secrets belong to one app, so each environment (dev/staging/prod) keeps its own values — target with the global `--env <name>` flag or `--app <app-id>`.

## Where `{{secrets.KEY}}` resolves

| Surface | Fields | When resolved |
|---|---|---|
| Integrations | `requestConfig.defaultHeaders`, `requestConfig.staticQuery` | Per request, when the proxy executes the call |
| Workflows | Any step-config template string; CEL contexts (`runIf`, `switch` `when`) expose `secrets.*` | Just before the step runs |
| Inbound webhooks | A webhook's `signingSecret` | Server-side, immediately before HMAC verification of an incoming event. Referenced key must exist at create/update; **fails closed** with a `401` (`rejectionReason: secret_unresolved`) if unresolvable at delivery |
| Databases | Operation `access` / per-param `access` CEL; trigger stamp `value` CEL | When the operation executes (secrets load only when the expression references `secrets.`) |

Write the reference without inner spaces — `{{secrets.KEY}}`, uppercase key, max 64 chars. Integration and webhook fields resolve exactly that form (a spaced `{{ secrets.KEY }}` is left as literal text there); workflow step templates additionally accept the spaced form.

**The CEL `secrets.*` variable is declared-only.** In a CEL expression (workflow `runIf` / `switch` `when`, a database operation or per-param `access` rule, a trigger stamp `value`), `secrets.*` binds **only** the keys the owning config declares in a top-level `secrets = ["KEY", ...]` manifest — on the database or collection type config, or the workflow definition. An undeclared `secrets.KEY` is absent at evaluation, so a rule that reads it denies closed. The `{{secrets.KEY}}` template form (the rows above) is unaffected: it resolves any set key.

There is no client-side surface: apps cannot read secret values through any API.

Integration templates only match uppercase keys — `{{secrets.MY_KEY}}` with `[A-Z][A-Z0-9_]`, max 64 chars. Stick to `UPPER_SNAKE_CASE` keys everywhere.

## Rules

1. **Never inline a credential in TOML** — config files are committed. Reference `{{secrets.KEY}}` and set the value with the CLI.
2. **Resolve secrets in integration config, not workflow step config.** A workflow step's resolved config is recorded in the run's step output snapshots — a secret templated into `request.headers` becomes readable in run detail. Secrets in the integration's `defaultHeaders`/`staticQuery` resolve after that snapshot and stay invisible. (See the integrations guide.)
3. **Rotation is an overwrite**: `primitive secrets set KEY --value <new>` takes effect on the next resolution — no config push needed, since config references the key, not the value.
4. After changing which keys exist, re-check references: an unresolved `{{secrets.MISSING}}` in a workflow renders the missing-path sentinel (or fails the step under `strict = true`); an integration header referencing a deleted key sends the unresolved placeholder upstream.

## Config Vars

The non-secret twin of app secrets: same key format (`^[A-Z][A-Z0-9_]{0,63}$`), same 2 KB value cap, same 100-per-app limit, same declared-only CEL binding, plus a template-resolution form — minus masking and minus save-time validation. Use a var for a plaintext app-wide scalar a rule needs to compare against (a platform-assigned group ID, say); use a secret for anything that grants access to an external system.

```bash
primitive vars set ADMIN_GROUP_ID --value grp_01ABC --summary "Admins group id"
primitive vars list          # values ARE shown — vars are not secret
primitive vars delete ADMIN_GROUP_ID
```

### Where `{{ vars.KEY }}` resolves

`{{ vars.KEY }}` resolves everywhere `{{secrets.KEY}}` resolves — the same table as above: integration `requestConfig.defaultHeaders`/`requestConfig.staticQuery` (per request, at proxy time) and any workflow step-config template string (including forEach/compensate/durable-batch contexts), just before the step runs. Same grammar as secrets: uppercase key, `{{vars.KEY}}` ≡ `{{ vars.KEY }}` (whitespace-tolerant), `[A-Z][A-Z0-9_]`, max 64 chars.

Two deliberate divergences from secrets, both worth tracking explicitly:

1. **Never redacted.** A var resolved into a header/query value is not marked sensitive and is not masked in admin logs or the test-mode request preview — only a value substituted from `{{secrets.*}}` becomes `[redacted]`. Vars are non-secret and stay visible everywhere.
2. **Not validated at save time.** Saving/updating an integration whose `defaultHeaders`/`staticQuery` reference a nonexistent `{{secrets.KEY}}` fails with a 400 naming the missing key. The same integration referencing a nonexistent `{{vars.KEY}}` saves successfully — the reference simply resolves to the literal `{{vars.KEY}}` placeholder at call time (the same behavior secrets fall back to only if the key is deleted *after* save).

### Declared-only in CEL

Declare a var for CEL access the same way as a secret, in the owning config's top-level manifest:

```toml
vars = ["ADMIN_GROUP_ID"]
```

A rule reads it as `vars.<KEY>`, bound only to declared keys — an undeclared `vars.KEY` is absent at evaluation (denies closed), exactly like `secrets.*`. Vars bind in workflow CEL contexts (`runIf`, `switch` `when`, predicate expressions on batch/collect steps), database operation `access`/per-param `access`, database trigger CEL, and trigger stamp `value` expressions. This CEL path is independent of the template-resolution form above — a config can use either or both.

### Syncing vars: `vars.toml`

Unlike secrets — which never appear in TOML, by design — config vars round-trip through `primitive sync` as a flat `vars.toml` at the sync directory root (sibling to `app.toml`, **not** a subdirectory): a plain top-level `KEY = "value"` table, no `[vars]` header, keys sorted:

```toml
# Per-environment non-secret config vars.
# Bind as {{ vars.KEY }} in workflow/integration config and vars.* in CEL rules.
# Values are checked into the repo and NOT secret — never put a credential
# here; use `primitive secrets` for that.

ADMIN_GROUP_ID = "grp_01ABC"
API_HOST = "https://api.example.com"
```

- `sync pull` writes it; a failed fetch leaves the existing `vars.toml` untouched rather than clobbering it with an empty file.
- `sync push` upserts changed keys and hard-deletes keys removed from the file, printing `Unsetting var KEY` for each; a missing/failed `vars.toml` read never mass-deletes.
- `sync diff` reports var add/remove/modified rows like any other synced entity.
- **Concurrent-edit guard**: if the server's value drifted since the last local sync (edited from the Admin Console, say), push reports `CONFLICT var: KEY` — with `Local last sync:` / `Server modified:` lines, the same pattern used for every other synced entity type — and exits non-zero. `--force` bypasses the check and overwrites/deletes unconditionally.

There is no client-side read API for vars — `primitive vars list`, the admin API, `vars.toml`, and the Admin Console's Config Vars view are the only read surfaces.

## Related guides

- **integrations** — the most common secret and var consumer (auth headers, static query params)
- **workflows** — `secrets.*` / `vars.*` in the template/CEL context
- **databases** — `secrets.*` / `vars.*` in operation access rules and trigger stamps
- **configuration** — the sync loop that carries secret-referencing TOML and the var-carrying `vars.toml`

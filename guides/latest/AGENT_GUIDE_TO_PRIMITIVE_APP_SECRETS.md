# Agent Guide to Primitive App Secrets

Guidelines for AI agents managing credentials in Primitive apps. App secrets are the platform's server-side store for API keys, tokens, and other credentials referenced from backend config as `{{secrets.KEY}}` or read by a server function with `ctx.secret`. Values resolve only on the server — they never appear in the repo, client code, or anything shipped to users. **Config vars** (below) are the non-secret twin of the same mechanism.

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
| Integrations | `requestConfig.defaultHeaders`, `requestConfig.staticQuery` | Per request, when the platform makes the outbound call (including a function's `ctx.integrations.call` — the value never enters the sandbox) |
| Server functions | `ctx.secret("KEY")` — not a template; requires `secret:KEY` in the function's `capabilities` | At call time. Undeclared → `FUNCTION_SECRET_GRANT_MISSING`; declared but unset in this environment → `FUNCTION_SECRET_NOT_FOUND`. Read through a short cache, so a just-rotated value may serve the old one for about a minute |
| Function webhook triggers | `[function.triggers.webhook] signingSecret` — a **whole** reference, never a literal (refused at push) | Server-side, immediately before verification of an incoming event. The referenced key must exist when the push applies the trigger; **fails closed** with a `401` (`rejectionReason: secret_unresolved`) if unresolvable at delivery. `primitive webhooks rotate-secret` moves the trigger onto a new reference and keeps the previous one verifying through the grace window |
| Databases | Server-stamped field trigger `value` CEL | When the trigger evaluates (secrets load only when the expression references `secrets.`) |

`{{secrets.KEY}}` ≡ `{{ secrets.KEY }}` — whitespace around the reference is tolerated everywhere it resolves (integration and webhook fields alike), but not inside `secrets.KEY` itself (no space around the dot). Uppercase key, max 64 chars. Type the reference rather than pasting it out of a document: a reference-only field (a webhook `signingSecret`, a Google client's `[auth.google.clients.<type>].clientSecret`) rejects a reference carrying an invisible character — a zero-width space or a byte-order mark — instead of resolving it with the stray character inside the credential.

**The CEL `secrets.*` variable is declared-only.** In a CEL expression (a database trigger stamp `value`, a collection rule), `secrets.*` binds **only** the keys the owning config declares in a top-level `secrets = ["KEY", ...]` manifest — on the database or collection type config. An undeclared `secrets.KEY` is absent at evaluation, so a rule that reads it denies closed. The `{{secrets.KEY}}` template form (the rows above) is unaffected: it resolves any set key.

There is no client-side surface: apps cannot read secret values through any API.

Integration templates only match uppercase keys — `{{secrets.MY_KEY}}` with `[A-Z][A-Z0-9_]`, max 64 chars. Stick to `UPPER_SNAKE_CASE` keys everywhere.

## Rules

1. **Never inline a credential in TOML** — config files are committed. Reference `{{secrets.KEY}}` and set the value with the CLI.
2. **Resolve credentials in integration config, not function code.** A secret in the integration's `defaultHeaders`/`staticQuery` is resolved by the platform into the outbound request and never enters the function. Use `ctx.secret` only when the function itself must hold the value (e.g. to compute a signature), and never print it: invocation logs redact a `ctx.secret` value best-effort only (`[REDACTED:<NAME>]`), and a transformed value is not detectable. (See the integrations and server-functions guides.)
3. **Rotation is an overwrite**: `primitive secrets set KEY --value <new>` takes effect once the platform's short read cache refreshes (up to about a minute) — no config push needed, since config references the key, not the value. A key holds exactly one value, so this is a sharp cutover; where a provider gives an overlap window (a webhook signing secret), create a SECOND key and rotate the reference onto it instead.
4. After changing which keys exist, re-check references: a function reading a deleted key gets `FUNCTION_SECRET_NOT_FOUND`, naming it; an integration header referencing a deleted key sends the unresolved placeholder upstream; a webhook trigger whose referenced key is gone rejects every signed delivery `401`.
5. **Budget the 100-key limit.** Every credential the server side uses is one key — an integration's API key, each webhook trigger's signing secret (plus a second key while a rotation window is open), a token a function reads. An app with many signed webhook triggers needs one key per trigger.

## Config Vars

The non-secret twin of app secrets: same key format (`^[A-Z][A-Z0-9_]{0,63}$`), same 2 KB value cap, same 100-per-app limit, same declared-only CEL binding, plus a template-resolution form — minus masking and minus save-time validation. Use a var for a plaintext app-wide scalar a rule needs to compare against (a platform-assigned group ID, say); use a secret for anything that grants access to an external system.

Vars are configuration, so they are authored in `vars.toml` and applied with `config push` — there is no command that writes one directly:

```bash
primitive config set vars ADMIN_GROUP_ID=grp_01ABC
primitive config push --only var/ADMIN_GROUP_ID
primitive vars list                # values ARE shown — vars are not secret
primitive vars get ADMIN_GROUP_ID  # one var, as the server has it
```

Delete a var by removing its line from `vars.toml` and pushing — a key the file no longer declares is deleted server-side, no `--prune` needed.

Var writes classify their `409` by code: `VAR_KEY_EXISTS` (a create targets a key the app already holds, or a concurrent create won the key mid-upsert), `VAR_LIMIT_REACHED` (the app is at the 100-var cap), or `CONFLICT` (the by-key upsert's optimistic-concurrency precondition failed — what `config push` surfaces as `CONFLICT var: KEY` when a var changed on the server since the last pull). A full store is never reported as a duplicate key. A push upserts by key: an existing key is replaced, not refused.

### Where `{{ vars.KEY }}` resolves

`{{ vars.KEY }}` resolves in integration `requestConfig.defaultHeaders`/`requestConfig.staticQuery` (per request, when the outbound call is made). A server function reads a var with `ctx.configVar("KEY")` — no capability line; a missing var answers `FUNCTION_VAR_NOT_FOUND`; each name is read once per config version and cached in the sandbox, so a changed value reaches a function on its next pushed version. Same grammar as secrets: uppercase key, `{{vars.KEY}}` ≡ `{{ vars.KEY }}` (whitespace-tolerant), `[A-Z][A-Z0-9_]`, max 64 chars.

Two deliberate divergences from secrets, both worth tracking explicitly:

1. **Never redacted.** A var resolved into a header/query value is not marked sensitive and is not masked in admin logs or the test-mode request preview — only a value substituted from `{{secrets.*}}` becomes `[redacted]`. Vars are non-secret and stay visible everywhere.
2. **Not validated at save time.** Saving/updating an integration whose `defaultHeaders`/`staticQuery` reference a nonexistent `{{secrets.KEY}}` fails with a 400 naming the missing key. The same integration referencing a nonexistent `{{vars.KEY}}` saves successfully — the reference simply resolves to the literal `{{vars.KEY}}` placeholder at call time (the same behavior secrets fall back to only if the key is deleted *after* save).

### Declared-only in CEL

Declare a var for CEL access in the owning config's top-level manifest:

```toml
vars = ["ADMIN_GROUP_ID"]
```

A rule reads it as `vars.<KEY>`, bound only to declared keys — in every CEL context: collection rules, database trigger CEL and trigger stamp `value` expressions alike. An undeclared key is unbound, so the rule errors and access is refused.

Test presence first when a key may be missing: `'KEY' in vars` is false rather than throwing, and `vars.?KEY` returns an optional. This CEL path is independent of the template-resolution form above — a config can use either or both.

### Syncing vars: `vars.toml`

Unlike secrets — which never appear in TOML, by design — config vars round-trip through `primitive config` as a flat `vars.toml` at the config directory root (sibling to `app.toml`, **not** a subdirectory): a plain top-level `KEY = "value"` table, no `[vars]` header, keys sorted:

```toml
# Per-environment non-secret config vars.
# Bind as {{ vars.KEY }} in integration config and vars.* in CEL rules.
# Values are checked into the repo and NOT secret — never put a credential
# here; use `primitive secrets` for that.

ADMIN_GROUP_ID = "grp_01ABC"
API_HOST = "https://api.example.com"
```

- `config pull` writes it; a failed fetch leaves the existing `vars.toml` untouched rather than clobbering it with an empty file.
- `config push` upserts changed keys and hard-deletes keys removed from the file, printing `Unsetting var KEY` for each; a missing/failed `vars.toml` read never mass-deletes.
- `config diff` reports var add/remove/modified rows like any other synced entity.
- **Concurrent-edit guard**: a var edited outside the sync loop is reported rather than overwritten, classified by which side moved — the same pattern used for every other synced entity type. Server only (the value changed in the Admin Console, your `vars.toml` still holds what the last sync left): push reports `DRIFT var: KEY` under "Server drift — not pushed", writes nothing, and the exit code is unchanged. Both sides (your file changed too, or the var was deleted on the server after you edited it locally): push reports `CONFLICT var: KEY` — with `Local last sync:` / `Server modified:` lines — and exits non-zero. Either way `config pull` takes the server's value and `--force` bypasses the check, overwriting or deleting unconditionally.

There is no client-side read API for vars — `primitive vars list` / `primitive vars get`, the admin API, `vars.toml`, the Admin Console's Config Vars view, and a server function's `ctx.configVar` are the only read surfaces.

## Related guides

- **integrations** — the most common secret and var consumer (auth headers, static query params)
- **server-functions** — `ctx.secret`, `ctx.configVar`, the `secret:` capability, webhook trigger signing secrets
- **databases** — `secrets.*` / `vars.*` in trigger stamps
- **configuration** — the sync loop that carries secret-referencing TOML and the var-carrying `vars.toml`

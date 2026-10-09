# Agent Guide to Primitive App Secrets

Use app secrets for credentials and config vars for non-secret settings.
Values belong to one app/environment, and deleting the app deletes its
secrets and vars. Secrets are set separately from the configuration tree; vars are
stored in `vars.toml`.

## Set and reference a secret

```bash
primitive secrets set WEATHER_API_KEY --value <value> --summary "Weather API"
primitive secrets list
```

```toml
[requestConfig.defaultHeaders]
X-API-Key = "{{secrets.WEATHER_API_KEY}}"
```

Keys use uppercase letters, digits, and underscores, starting with a letter,
up to 64 characters. Each app can hold 100 secrets with values up to 2 KB.
Secret management lists names and summaries, never values.

## Where values are used

| Surface | Usage |
| --- | --- |
| Integration | `{{secrets.KEY}}` in `defaultHeaders` or `staticQuery`; resolved into the outgoing request. |
| Function | `ctx.secret("KEY")`, with `secret:KEY` in its capabilities. |
| Webhook | Whole `{{secrets.KEY}}` reference in `signingSecret`. |
| CEL | `secrets.KEY`, after declaring `secrets = ["KEY"]` on the owning configuration. |

Prefer integration references when the secret is an API credential. Read it
in function code only when the function needs the value itself, such as when
computing a signature.

## Config vars

```bash
primitive config set vars ADMIN_GROUP_ID=grp_01ABC
primitive config push --only var/ADMIN_GROUP_ID
primitive vars get ADMIN_GROUP_ID
```

Vars use the same key format, value-size limit, and per-app count as secrets.
Store them in a flat `vars.toml`, without a `[vars]` table:

```toml
ADMIN_GROUP_ID = "grp_01ABC"
API_HOST = "https://api.example.com"
```

Reference `{{vars.KEY}}` in integration headers or query parameters, or read
it with `ctx.configVar("KEY")`. For CEL, declare `vars = ["KEY"]` on the
owning configuration before reading `vars.KEY`.

## Gotchas

- Never put credentials in TOML, vars, client code, or logs. Secret redaction
  is best-effort and cannot recognize transformed values.
- Integration secret references must exist when saved. Var references are not
  validated at save time: a missing value remains a literal placeholder.
- A function must declare `secret:KEY` before reading a secret. Missing values
  return `FUNCTION_SECRET_NOT_FOUND`; missing vars return
  `FUNCTION_VAR_NOT_FOUND`.
- Functions cache config vars per version. Push a new function version after
  changing a var. Secrets refresh through a short cache, usually within about
  a minute, without a push.
- Updating an existing secret replaces its value. For overlapping webhook
  rotation, create a second secret and use `primitive webhooks rotate-secret`
  to change the reference while retaining the old one for the grace period.
- Removing a var from `vars.toml` deletes it on the next push without `--prune`.
  Pull takes server values; `--force` deliberately overwrites server changes.
- CEL can read only declared keys. For optional vars, check `'KEY' in vars`
  before reading, or use `vars.?KEY`.

## Related guides

- [Integrations](AGENT_GUIDE_TO_PRIMITIVE_INTEGRATIONS.md)
- [Server Functions](AGENT_GUIDE_TO_PRIMITIVE_SERVER_FUNCTIONS.md#secrets-and-config-vars)
- [Configuration](AGENT_GUIDE_TO_PRIMITIVE_CONFIGURATION.md)

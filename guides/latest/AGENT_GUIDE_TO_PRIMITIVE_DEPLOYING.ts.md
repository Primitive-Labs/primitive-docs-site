# Deploying Primitive apps to production

This guide is the reference for shipping a Primitive app to production through its native distribution channel.

## Web (Cloudflare Workers)

The web template deploys to Cloudflare Workers via `pnpm cf-deploy <environment>`. You need a Cloudflare account with Workers deploy access.

### 1. Configure `wrangler.toml`

Set the worker name and per-environment overrides:

```toml
name = "my-app"

[env.production]
name = "my-app-prod"
```

By default the app deploys to a `*.workers.dev` URL. For a custom domain, add a route under the environment:

```toml
[[env.production.routes]]
pattern = "your-domain.com"
custom_domain = true
```

### 2. Configure `.env.production`

`cf-deploy` reads `.env.{environment}`. For production, `.env.production`:

```bash
# Primitive App ID — same as dev, or a separate production app.
VITE_APP_ID=your_production_app_id

# OAuth redirect URI for the production origin (must match the
# server-side OAuth config; mismatches fail the callback exchange).
VITE_OAUTH_REDIRECT_URI=https://my-app-prod.your-subdomain.workers.dev/oauth/callback
```

### 3. Deploy

```bash
pnpm cf-deploy production
```

The script reads `.env.production`, builds, and deploys to the `[env.production]` worker. Pass extra wrangler flags after `--`:

```bash
pnpm cf-deploy production -- --dry-run
```

### Adding environments

`cf-deploy <name>` generalizes to any environment:

1. Add `[env.<name>]` (and any `[env.<name>.vars]`) to `wrangler.toml`.
2. Create `.env.<name>`.
3. `pnpm cf-deploy <name>`.

```toml
[env.test]
name = "my-app-test"

[env.test.vars]
REFRESH_PROXY_COOKIE_MAX_AGE = "604800"
REFRESH_PROXY_COOKIE_PATH = "/proxy/"
```

The deploy reads `.env.{environment}` and the matching `[env.{environment}]` block.


# Agent Guide to the Primitive Changelog

User-visible changes in each production release of the Primitive platform, newest first. Use this when upgrading an app's platform libraries or CLI: scan the entries newer than the app's current versions for new capabilities to adopt, changed behavior to accommodate, and deprecations to migrate off. Every deprecated client member is marked in the typings, so a TypeScript app lists its uses with the `@typescript-eslint/no-deprecated` lint rule (the Vue starter template's `pnpm lint` runs it) and a Swift app reads them as compiler warnings; a deprecated configuration key is reported at `config push`. Only user-visible changes are listed — new features, client API and CLI additions and changes, and behavior-changing fixes. Each release is grouped by feature area; bullets tagged **Breaking** need an app change on upgrade, and a client is named only when a change is specific to it. An `## Unreleased` section, when present, lists changes already merged since the last production release; it is date-stamped when that release ships.

<!-- changelog:insert — the nightly docs sweep maintains the `## Unreleased` section directly below this line; the production deploy date-stamps it. Keep this comment in place. -->

## Unreleased

### Locks

- A server function waits for a named lock with `ctx.locks.acquire`, or runs code under it with `ctx.locks.withLock`, and the JavaScript and Swift clients' blocking `acquire` waits on the server through the acquire route's new `waitMs`.

### Server functions

- **Breaking:** `ctx.api.documents.blobs.uploadWithoutId` is removed from `primitive-functions`; it only ever answered 405. Upload with `ctx.api.documents.blobs.upload` and a `blobId` you choose. The server no longer serves `PUT /documents/{documentId}/blobs` without an id.

## 2026-09-30

### Server functions

- Server functions are TypeScript you keep in your config tree, ship with `primitive config push`, and call from the JavaScript client, the Swift client or the CLI with `functions.invoke`.
- Any function can also run as a task with `functions.start`, pausing with `step.sleep` for hours or days and tracked with `getStatus` and `waitFor`.
- A function works on the app's authority through `ctx`: typed database and document handles, prompts, integrations, secrets, config vars, the app's users, and starting other functions.
- `ctx.channels` authorizes and publishes to realtime channels that both clients join with `subscribeToChannel`, and `ctx.users.send` delivers direct messages to a user's live connections.
- A function can declare its own webhook and cron triggers, and a workflow's `workflow.call` step can call a function by key.
- `primitive functions codegen` generates typed invokers for TypeScript and Swift, and `config push` typechecks every function before shipping it.
- Every invocation leaves a seven-day log (`primitive functions logs`), and a failed run carries a typed `errorCode`.

### Large documents

- A document created with `documentFormat: 2` (CLI: `primitive documents create --large`) holds up to 2 GB of records and opens through the same document, model and query APIs in the JavaScript and Swift clients.
- Clients keep a local copy that syncs incrementally, can load only some models on a constrained device, and query or aggregate across all of a user's open large documents in one call.
- Offline writes are reconciled on reconnect and reported through `documentOfflineWritesResolved`, and a client offline longer than `largeDocumentWindowDays` (default 7) is read-only until it syncs.
- `primitive documents ingest` and `client.documents.ingests` bulk-load records from NDJSON files, and connected clients pick up the load without reloading.
- `primitive documents snapshots` lists a document's snapshot builds and builds one on demand.

### Documents

- `validateAccess` takes a `userId` to answer what another user may do with a document.
- `createWithAlias` and `getOrCreateWithAlias` take `documentFormat`, `tags` and `metadata`.
- A client that expects the wrong format for a document is refused with `DOCUMENT_FORMAT_MISMATCH`, and records requests may state the format they expect.
- `client.registerModel` adds a model to a running JavaScript client.
- **Breaking:** JavaScript client 3.0.0 removes `documents.list()` and per-document invitations, replaced by `me.ownedDocuments()`, `me.sharedDocuments()` and sharing by email with `documents.updatePermissions`.
- **Breaking:** The Swift client's `documents.create` no longer opens the new document, so open it before writing.
- **Breaking:** Document create routes refuse body keys they do not read with `400 VALIDATION_FAILED`.
- **Fixed:** A document whose sync stalls reports `documentSyncStateChanged` with `state: "error"`, and the client reconnects on its own.
- **Fixed:** An awaited `documents.open()` in the JavaScript client resolves only once the synced records are queryable.
- **Fixed:** Records queries handle `$or` with many branches, sorting on a field some records lack, and a unique key whose earlier record was deleted.
- **Fixed:** A records request to a document that is being deleted answers `404` instead of `500`.
- **Fixed:** Large updates sent inline, as the Swift client sends them, are stored durably.
- **Fixed:** The Swift client handles numbers at or above 2^63, evicts `localOnly` and revoked documents correctly, and sends a `transactAndSync` transaction once.

### Models and queries

- **Breaking:** `$ne` and `$nin` match records that never wrote the field, so `{ deleted: { $ne: true } }` means "false or not set".
- **Breaking:** A unique constraint on a `stringset` field is refused where it is declared (js-bao 0.11.0, codegen and `config push`), so remove it and regenerate.
- **Breaking:** Model fields named `type` or starting with `_` are refused at `config push` and codegen.
- **Breaking:** A grouped `Model.aggregate` with a single `sum`, `avg`, `min` or `max` returns the bare value per group (js-bao 0.10.0).
- **Breaking:** Each `initJsBao` call creates its own isolated instance, and `js-bao-wss-client` requires `js-bao` 0.6.0 or newer.
- **Fixed:** Database `save` keeps a field written as `null`, and a unique constraint no longer fires for an index entry left by a deleted record.

### API and clients

- Every API error response is JSON with a machine-readable `code`, exposed as `JsBaoApiError.code`, `HttpError.serverCode` in Swift and `ApiError.code` in the CLI (see the Error Handling guide).
- App API responses are `Cache-Control: no-store`, and the Swift client never caches HTTP responses.
- **Breaking:** Error bodies that were plain text are now JSON, so code calling the API with a raw `fetch` reads `body.error` instead of the text.
- **Deprecated:** The CEL context surfaces, the `cursor` alias of `nextCursor`, and removing a user (use `users disable`) are marked deprecated on every surface.

### Prompts

- A `kind = "decisions"` prompt answers typed questions about a state with probabilities, with each run able to supply its own options.
- `strictOutput = true` on a chat config has the provider enforce the prompt's `outputSchema`.
- **Breaking:** The direct `llm` and `gemini` APIs are removed from both clients, server functions and `app.toml`, so run a prompt with `ctx.prompts.run` instead.
- **Fixed:** A failed run reports the provider's `upstreamStatus` and `PROMPT_UPSTREAM_TIMEOUT`, and a template's trailing newline survives `config push`.

### Collections

- `primitive collections export` and `import` move collections with their owners, members and grants between apps.
- `primitive collections create --owner` creates a collection on behalf of a user, and `listCollectionsForDocument` pages.
- **Fixed:** Collection group grants are recomputed on every change, and access listings return every entry instead of the first 100.

### Locks

- An `owner` passed when acquiring a lock lets that owner re-take its own lease.

### Sign-in and sessions

- The `passkeyUserVerification` app setting (`preferred` or `required`) sets one verification policy for both passkey ceremonies.
- Every admin sign-in is a revocable session (`primitive auth sessions list` and `revoke`), and `primitive logout` ends it on the server.
- Environments accept an `iosAppId`, and `pnpm cf-deploy` generates the matching `apple-app-site-association` file.
- `PrimitiveAuthManager.authFailure` in the Swift app layer exposes the typed code of the last failed sign-in.
- **Fixed:** Passkey sign-in from iOS respects preferred verification, and the Swift sign-in flow forwards `rpId` and reports the server's error codes.

### Analytics

- `users.search` filters by signup day, and prompt analytics report per-prompt cost.
- Server-function failures are grouped in error analytics by function key.
- Cron and system workflow runs no longer count as user activity.
- **Fixed:** `users.top` reports each user's all-time `firstSeen`, so a returning user no longer reads as new.

### Workflows, webhooks and integrations

- `primitive analytics workflow-usage` reports which step kinds an app's workflows use and how often they ran.
- Workflows, server functions, scripts and webhooks share one key namespace, enforced at `config push`.
- `primitive webhooks test --deliver` sends the signed preview to the real endpoint so the handler actually runs.
- `primitive integrations logs --run` shows the calls a server-function run made.
- The Rhai script step's operation ceiling is 250,000.
- **Fixed:** Model-typed operation parameters are coerced like workflow input, a `switch` step's outputs resolve, and creates and starts no longer fail on a delayed read-back.

### CLI

- **Breaking:** The configuration tree lives in `primitive/<env>/` beside `primitive/config.json`, `--env` selects another environment, and `--dir`, `--sync-dir`, `primitive use` and `primitive context` are removed.
- **Breaking:** Every `list` prints one page, and `--json` prints the `{ items, hasMore, nextCursor }` envelope.
- `config push` validates the whole tree before applying anything and reports a var changed only on the server as drift.
- Destructive collection and membership verbs take `--json`, and admin document listings show tags.
- **Fixed:** `config diff` no longer reports identical types as modified, test verbs accept keys, and `documents export` and `import` keep tags and root documents.

### Admin API

- A user's document and database inventories page with `{ items, hasMore, nextCursor }` and reach every item.
- **Breaking:** Listing every app (`GET /admin/api/apps`) is super-admin only, so use `GET /admin/api/admins/me/apps`.

### Starter templates

- `pnpm lint` in the Vue template reports uses of deprecated platform APIs.
- Generated sources are committed and regenerated on every build.
- The Swift template's App Store lanes sign entirely from the App Store Connect API key.
- **Fixed:** A new Swift app passes App Store validation, archives on the first try, and builds under Xcode 27 and SwiftBuild.

## 2026-08-28

### Sign-in

- Passkey calls can name their relying party (`rpId` in JavaScript, `AuthConfig(passkeyRpId:)` in Swift).
- **Fixed:** A passkey request naming an unconfigured relying party is refused with `PASSKEY_RP_NOT_CONFIGURED`, and a failed OAuth exchange throws a typed error in JavaScript.

### Documents

- **Fixed:** Shares granted by email before the recipient signs up resolve reliably at signup.

### CLI and templates

- Environments accept a `webUrl`, and `primitive init` scaffolds universal-link sign-in for an app with both iOS and web clients.
- The Swift template's `run-ios.sh` boots a simulator dedicated to the app.
- **Fixed:** `integrations tests` resolves by key or ID, `scripts tests` names the cases it skips, and the Swift template finds its App Store Connect key from any directory.

## 2026-08-27

### Sign-in

- An app with both iOS and web clients emails one sign-in link that opens the iOS app where it is installed and the web app everywhere else.
- **Breaking:** Email sign-in sends one email carrying both a code and a link, controlled by `[auth].emailSignInEnabled`, and the `magic-link` and `otp` templates are retired in favor of `email-sign-in`.
- **Breaking:** Google sign-in is configured per client type under `[auth.google.clients.<type>]`, `redirectUris` becomes `emailRedirectUris`, and older clients hide Google sign-in until upgraded.
- **Fixed:** Swift client sessions restore on cold start, refresh reliably, and clear fully on sign-out.

### Access and credentials

- An app can install a `lock` rule set controlling which keys members may acquire.
- **Breaking:** Prompts, workflows and integrations require an `accessRule`, and a resource without one refuses callers.
- **Breaking:** Webhook signing secrets and Google client secrets accept only secret references, and app secrets are the only credential store.
- **Breaking:** The direct LLM and Gemini proxy routes are off unless `directLlmEnabled = true`.

### Workflows

- Templates can read whether an upstream step succeeded, failed or was skipped.
- The database write step accepts several records per call, and the iterate-users step takes a `metadataFilter`.
- `forEach` iterates up to 500 items by default and fails, naming the limit, instead of truncating.
- Declarative locks apply to synchronous runs and `workflow.call`, and guards read every app config var through `vars.KEY`.
- Multi-step runs keep sub-second step gaps wherever they run.
- **Breaking:** An unresolved template reference fails the step, and `config push` validates `{{ }}` expressions.
- **Breaking:** `prompt.execute` parses JSON only with `expect = "json"`.
- **Breaking:** Workflow push sends the full definition, so a field omitted from TOML is cleared.
- **Breaking:** Workflow document writes enforce the model's field types.
- **Fixed:** Runs are visible while in flight, `forEach` and template fallbacks behave, status is consistent across clients, and database reads return real booleans.

### Configuration and CLI

- `primitive <noun> archive` retires an integration, webhook, cron trigger, workflow or prompt, and deleting a workflow or prompt now archives it.
- `primitive documents` gains inspection, permission, create and delete verbs, and `databases records` gains get, count, aggregate, save and patch.
- Log commands follow new entries with `--watch`.
- **Breaking:** Server-side configuration is authored in TOML only, and the CLI's config-setting flags are removed.
- **Breaking:** CLI commands are reorganized, with `sync` under `config`, `collections docs` renamed `documents`, `database-types` renamed `database-type-configs`, and the `llm` group removed.
- **Breaking:** `config push` applies `app.toml` as the complete state, so run `config pull --only app` first or omitted settings are cleared.
- **Breaking:** Availability status is no longer authored in TOML, so use `primitive <noun> enable` or `disable`.
- **Breaking:** `config diff` and `config push` compare every resource by content and agree, and a quoted number where a number is declared is now an error.
- **Fixed:** `config push` adopts existing server resources by key, lands a multi-config prompt on the first push, and round-trips cleanly with `config pull`.

### Documents and databases

- A resource can be found by a unique metadata value from both clients and the CLI.
- Documents gain a records `aggregate` endpoint matching the databases one.
- **Breaking:** Admin data routes move under role-aware `records/*`, `databases.list` requires a filter, and bulk operations take `data` instead of `fields`.
- **Breaking:** Document permission entries type `name` as optional.
- **Fixed:** Long-lived documents reopen quickly, `localOnly` edits stay local, and documents deleted on the server are evicted from the local cache.

### Webhooks

- Webhooks can verify JWTs from a remote JWKS, custom detached signatures, or Plaid.
- **Breaking:** An app is limited to 50 webhooks.
- **Fixed:** Webhook deduplication keys derive only from signed material, and JWT verification rejects a key id it cannot match.

### Swift client

- Notifications, named locks and resource metadata match the JavaScript client.
- Events are available as `AsyncStream`, `workflows.waitFor` survives disconnects, and codegen emits typed subscriptions and the full workflow factory.
- Network monitoring, `waitForLoad`, `serverTimeoutMs` and cache-first document listing work as documented.
- **Breaking:** Deprecated symbols are removed in one batch, and the libraries build in Swift 6 language mode.
- **Fixed:** Connections, the document lifecycle and write ordering match the JavaScript client, and several crashes and decoding failures are gone.

### JavaScript client

- `openDocument` errors carry typed codes.
- **Fixed:** A truncated listing no longer evicts documents, token refresh loops are capped, and switching accounts re-authenticates the connection.

### Analytics and admin console

- The admin console groups recurring errors, shows live connections, and names the step a failed run failed on.
- **Breaking:** An analytics filter set over the cap is rejected instead of truncated.
- **Fixed:** Error groups no longer split on embedded ids or merge distinct failures, and adding a user by email is reliable.

## 2026-07-24

### Documents

- Single-document `save`, `patch` and `delete` accept `upsertOn`, `precondition` and `ifNotExists` for conditional writes.
- **Fixed:** A just-created document appears in its creator's owned documents immediately.

### Sign-in

- A user can link several OAuth providers to one account.
- **Fixed:** Linking a second provider no longer locks the user out of the first.

### Analytics and workflows

- Failures are recorded as fingerprinted error events, queryable with `errors.groups`.
- A workflow fragment include accepts parameters.

### Lists and pagination

- **Breaking:** Every list endpoint returns `{ items, hasMore, nextCursor }`, and an invalid `limit` or cursor answers `400`.
- **Fixed:** The JavaScript client pages through every list result, and group memberships page fully.

### CLI and Swift client

- `primitive blob-buckets head` reads a blob's metadata, and `primitive connections list` shows active connections.
- **Fixed:** Date-time values and `[workflow.lock]` survive `config push` and `pull`, and the Swift client's `waitForInSync` waits for real confirmation.

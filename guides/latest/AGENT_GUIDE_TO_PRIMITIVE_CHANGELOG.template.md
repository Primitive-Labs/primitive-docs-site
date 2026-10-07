# Agent Guide to the Primitive Changelog

User-visible changes in each production release of the Primitive platform, newest first. Use this when upgrading an app's platform libraries or CLI: scan the entries newer than the app's current versions for new capabilities to adopt, changed behavior to accommodate, and deprecations to migrate off. Every deprecated client member is marked in the typings, so a TypeScript app lists its uses with the `@typescript-eslint/no-deprecated` lint rule (the Vue starter template's `pnpm lint` runs it) and a Swift app reads them as compiler warnings; a deprecated configuration key is reported at `config push`. Only user-visible changes are listed — new features, client API and CLI additions and changes, and behavior-changing fixes. Each release is grouped by feature area; bullets tagged **Breaking** need an app change on upgrade, and a client is named only when a change is specific to it. An `## Unreleased` section, when present, lists changes already merged since the last production release; it is date-stamped when that release ships.

<!-- changelog:insert — the nightly docs sweep maintains the `## Unreleased` section directly below this line; the production deploy date-stamps it. Keep this comment in place. -->

## Unreleased

### Agents

- An agent is a prompt of `kind = "agent"` declared in TOML and shipped with `config push`: it names the server functions and client tools the model may call, the events members record, its turn context and its limits, with model settings on a `[configs.agent]` configuration.
- A server function creates a session with `ctx.agents.createSession` from a signed-in member's call, and members then use it through `client.agents.sessions` in the JavaScript and Swift clients: `open` returns a live view of the conversation with `onChange`, and `send`, `answer`, `recordEvent`, `cancel`, `rename`, `addMember` and `removeMember` act on it.
- A tool can require a person's approval or be answered by the member's client, pausing the turn until the first valid answer, and `ctx.tool.attachArtifact` keeps an app-only value beside the call that the model never sees.
- A turn's context — a server function's return or the send's own `context`, validated against the agent's schema — renders as `turn.*` in the system prompt, with the send's timezone and language as `turn.timezone` and `turn.locale`; an agent selects what the model sees of earlier turns with a history function or `maxHistoryChars`.
- App owners and admins list, inspect and delete any member's sessions with `primitive agent-sessions list`, `get` and `delete`.

### Child apps

- `primitive apps children create <slug>` creates a child app of the current environment's app for a branch — a copy of its settings and, unless `--no-secrets`, its secrets — pushes the tree into it and selects it as a machine-local environment that `config pull`, `push`, `diff`, `pnpm dev` and the deploy scripts resolve with no other setup; `apps children list`, `get` and `delete` manage it, another parent admin attaches it with `primitive env use <slug>`, and `primitive apps list --all` includes children.
- Every admin of the parent uses a child's admin and app APIs without an invitation; only the child's owner or the parent's console owner deletes it, and `primitive apps delete <app-id> --with-children` deletes an app's children before the app.
- A child is deleted after `--idle-days` (7–30, default 30) without a config push, an interactive sign-in or a person-started function invocation; its owner is emailed at least seven days before, and any such activity renews it.
- `pnpm cf-deploy --deploy-env <env> --primitive-env <child> --preview-alias <name>` in the Vue template uploads a preview version of the worker under that alias with the child's app id, prints its URL and registers the origin on the child, so its CORS and email sign-in checks accept it while the live worker is untouched.

### Server functions

- `client.functions.listRuns` lists the task runs the signed-in user started, filtered by function, status or document, and `listRunSteps` lists one run's steps, in both clients.
- A document `batch`, `ctx.api.documents.records.bulk` and `primitive documents records bulk` accept an `upsert` operation that addresses a record by a declared unique constraint, compound or single-field: the record holding those values gets only the supplied fields, and a missing one is created. The result's new `upserted` list gives each upsert's record `id` and whether it was `created`.
- A task run that shows no progress for 30 minutes is ended `failed` with the new `errorCode` `FUNCTION_STALLED`, so polling a run never needs a timeout of its own. A step's own `timeout` and retry delay extend the wait, and sleeps, event waits and retry delays never count.
- A task run's `slice` block adds `lastProgressAt`, `openStep`, `openStepStartedAt` and `parkedAt` in both clients, and `primitive functions runs` shows a `LAST PROGRESS` column.
- An integration call made inside a server function's `step.do` names its step in `primitive integrations logs`: a STEP column, `--step <name>` to filter, and `correlation.stepId`, `correlation.stepOccurrence` and `detail.stepAttempt` under `--json`.
- **Breaking:** `ctx.api.documents.blobs.uploadWithoutId` is removed from `primitive-functions`; it only ever answered 405. Upload with `ctx.api.documents.blobs.upload` and a `blobId` you choose. The server no longer serves `PUT /documents/{documentId}/blobs` without an id.
- A server function lists what a named member holds, page by page: `ctx.api.documents.listOwnedByUser`, `ctx.api.documents.listSharedWithUser` and `ctx.api.collections.listForUser`, each taking a `userId`. Over REST they are `GET /users/{userId}/owned-documents`, `/shared-documents` and `/collections`, for an app owner or admin.
- **Fixed:** A webhook delivery signed with a newly created `{{secrets.KEY}}` signing secret verifies right away, instead of being refused with `401` for up to a minute.
- **Fixed:** `primitive functions codegen --lang swift` leaves unchanged output files untouched, so an Xcode build no longer recompiles the whole app every time.

### Large documents

- **Breaking:** In the browser, a tab left open across this release must reload before it can open a large document again: the local store now commits many saves together, so a burst such as an import finishes much faster, and an older client library's open is refused with no local data lost; do not downgrade the client library past this release for large documents.
- The JavaScript client asks the browser to keep a large document's local copy when it first opens one, and `client.getLargeDocumentStorage()` reports the browser's answer as `persistence`.
- Node clients apply a large document's writes faster: the statements a save runs are compiled once per executor and reused.
- **Fixed:** A large document's stats report the exact size of its stored records, with `sizeBasis` saying what the size measures, instead of a figure that dropped after each snapshot.
- **Fixed:** A JavaScript client that held a large document across a server archive no longer re-sends the archived writes, which made the document's export and storage grow.
- **Fixed:** In the Swift client, opening a large document the device has never held resolves only once its records are queryable, instead of reporting `isSynced` with no rows; an `openDocument` that waits on the network always resolves or throws `.networkTimeout` within `availabilityWait`, and `setLogLevel(.debug)` traces each open.
- **Fixed:** Two documents that each hold a record with the same id keep their own records in the client's local query tables, so a `count()` or `query()` scoped to one of them no longer comes back short; a local store written by an earlier release is rebuilt on its next open and keeps every row.
- **Fixed:** Writes to a large document no longer stall while it is being imported or its snapshot is built, and a Swift client whose write is refused just after the document's epoch moves catches up and keeps writing instead of stopping with `FORMAT2_RELOAD_REQUIRED`.
- **Fixed:** A record save through the REST API, a server function, a workflow or the CLI that lands just as a large document's epoch rotates is planned again on the new epoch instead of failing with `save record failed`.

### Documents

- **Breaking:** `documents.validateAccess` on a document the server does not hold — deleted, never existed, or in another app — answers `404 NOT_FOUND` in both clients instead of `DOCUMENT_DELETED`; on a document whose create has not yet reached the server it waits for the commit and answers `owner`, failing with `PENDING_CREATE` when the device is offline or the create has not landed within 30 seconds.
- An app admin, in-app or console, can delete any document of the app directly with `documents.delete`, as an app owner already could.
- **Fixed:** A compound unique constraint holds across writers: a record saved by a server function, a workflow, the REST API or the CLI now blocks a client save of the same values, and the reverse.
- **Fixed:** `me.sharedDocuments()` no longer skips documents when paging with a small `limit`. A page can now be short or empty while `hasMore` is `true`; keep following `nextCursor`.
- **Fixed:** The dev tools' Document and Blob explorers and the Swift `PrimitiveAppState.fetchDocuments()` list every owned and shared document, not only the first page of each.
- **Fixed:** A document listing fetched for a user who has since signed out is dropped instead of being kept for the next signed-in user, in both clients.

### Models and queries

- **Breaking:** A `stringset` field's values are returned sorted by Unicode code point, each value once, from every surface — model reads, queries, server reads, the CLI and the Swift client, in documents, large documents and databases — rather than in the order they were added; sort a copy to display another order.

### Collections

- **Breaking:** Collection `contextId` is removed from the API, rules, both clients and the CLI; bind a collection to an outside entity with `initialMetadata` on create and read it in rules as `md.self.<category>.<key>`.
- **Breaking:** Collection names are labels, not unique within an app, so creating or renaming a collection never answers a name `409`, and `primitive collections import` refuses a name that more than one target collection carries.
- **Fixed:** A collection's `documentCount` stays accurate when documents are added to or removed from it at the same time.
- **Fixed:** Deleting or unsharing a collection shared with a group, removing a document from it, and revoking a document's group grant no longer fail with `403` on older grants.

### Users and groups

- `groups.create` accepts `initialMetadata` in both clients, `ctx.api.groups.create` and `primitive groups create --initial-metadata`, and a group rule set can gate the create on it, as collections do.
- An app owner or admin sets a user's display name and avatar, at creation or later: `primitive users create --name --avatar-url`, `primitive users set-profile <user-id> --name|--avatar-url|--clear-avatar`, `ctx.api.users.setProfile` from a server function, and `name`/`avatarUrl` when adding a user by email. An avatar URL written this way or by `me.update` must be an `http:` or `https:` URL; any other scheme is refused with `400` and nothing is written.

### Locks

- A server function waits for a named lock with `ctx.locks.acquire`, or runs code under it with `ctx.locks.withLock`, and the JavaScript and Swift clients' blocking `acquire` waits on the server through the acquire route's new `waitMs`.

### Prompts

- A prompt run whose answer stopped at the configuration's `maxTokens` carries `truncated: true`, on success and on failure, from `ctx.prompts.run`, the member execute route and `primitive prompts execute`. A cut-off JSON answer, or a completion that used the whole limit and returned nothing, fails with the new `errorCode` `PROMPT_OUTPUT_TRUNCATED` instead of `PROMPT_OUTPUT_NOT_JSON` or `PROMPT_OUTPUT_SCHEMA_VIOLATION`; raise `maxTokens`. `primitive prompts execute` prints a warning, and a prompt test run cut off at the limit fails its `Output complete` check.
- **Breaking:** `config push` refuses a prompt `[[configs]]` entry that writes chat keys at its root or marks the live config with `isActive`, so move those keys under `[configs.chat]` and write `active = true`.
- **Fixed:** OpenRouter chat configurations now send `maxTokens` to the provider, so a configuration that sets it is capped at that many output tokens.
- **Fixed:** `config push` no longer refuses a reasoning setting because of which values a particular model supports; the provider decides. A value the model refuses fails the run with the provider's message, `upstreamStatus: 400` and the new `errorCode` `PROMPT_UPSTREAM_REJECTED`, from `ctx.prompts.run`, the member execute route and `primitive prompts execute`. A provider `400` on any prompt run carries that code.
- **Fixed:** `primitive prompts execute`, the console's prompt test and the evaluator use a system prompt or user template larger than 25 KB (stored offloaded) instead of running without it.

### Sign-in and sessions

- **Breaking:** `magicLinkRequest`, `otpRequest` and their routes are removed, and the auth and app config report only `emailSignInEnabled`; start email sign-in with `emailSignInRequest`.
- **Fixed:** Clearing an app's `googleClients` or `passkeyRpConfig` from the console no longer brings back Google client or passkey values set before those maps existed.

### API and clients

- **Breaking:** The database CEL context, deprecated in the last release, is removed: `getCelContext`/`updateCelContext` and the database `metadata` methods in both clients, `primitive databases cel-context` and `databases create --cel-context`/`--metadata`, the `databases/{databaseId}/metadata` routes, and `celContext`/`metadata` on database responses. A rule that still references `database.celContext` or `$database.metadata` is refused when saved.
- **Breaking:** List responses drop the deprecated `cursor` alias and the `grants`, `admins`, `apps`, `documents` and `databases` array keys, so read rows from `items` and page with `nextCursor`.
- **Breaking:** The deprecated `admin-data/*` database record routes and `ctx.api.databases.adminData` in `primitive-functions` are removed; use the `records/*` routes and `ctx.api.databases.records`.
- **Breaking:** The deprecated `primitive databases permissions grant` and `revoke` commands and the JavaScript client's `databases.grantPermission` and `revokePermission` are removed; use `primitive databases permissions add-manager` / `remove-manager`, or `ctx.api.databases.addManager` / `revokePermission` from a server function.
- **Breaking:** The JavaScript client removes `FunctionWaitForResult`, `documents.createOffline`, `waitForSync` and `updateLocalSnapshotFlag` (use `FunctionRunSettled`, `documents.create` and `waitForInitialSync`), and both clients drop `user_created_at_epoch_s` from analytics event input.
- **Breaking:** In the Swift client, an HTTP call made while the client is offline throws `JsBaoError` with code `.offline` without sending the request, as the JavaScript client does, so code that detects a missing network by catching `JsBaoNetworkError` must also catch `.offline`.
- **Fixed:** The JavaScript client bypasses the browser's HTTP cache on every request, so concurrent reads of the same resource no longer wait on one another.

### Configuration and CLI

- **Scoped CLI logins and admin step-ups.** `primitive login --scope <list> [--app <ids>]` asks for a session limited to those scopes and pinned to those apps, granted only after you approve it on a console page that names them. `--scope admin` asks for a 15-minute step-up held beside your login; a command refused for lacking `admin` retries once with it, and `primitive logout` ends it too. `primitive token --scope <list> [--app <ids>] [--ttl <d>]` prints a narrower token for a script or agent, with no browser unless the request is wider than your login. `primitive whoami` shows the session's kind, scope and pinned apps and any step-up, and `primitive auth sessions list` shows `step-up` sessions and `pending` approvals.
- **CI sessions.** `primitive auth sessions create --name <name> --scope <list> --app <ids> [--ttl <d>]` creates a named admin token for a CI job once you approve it in the browser, and prints it once. It is pinned to its apps, never holds `admin`, and lives 90 days unless `--ttl` says otherwise, at most 1 year. Run the CLI with it in `PRIMITIVE_TOKEN`, with no credentials file; it works on the admin and app APIs and WebSocket connections within its scope and pins. `primitive auth sessions list` shows it with its name, and once revoked its token fails on its next request.
- **Protected apps.** `protected = true` under `[app]` in `app.toml` marks an app as live with real users; `config push` applies it, and `primitive apps get`, `GET /settings`, the admin app object and the web-admin dashboard show it. Only the app's console owner can set or clear it: anyone else's change — a console admin's, or any app-user token's — is refused with `403 PROTECTED_FLAG_OWNER_ONLY` and nothing is applied, while a body carrying the current value saves as before. A value that is not a boolean is a `400`. A pulled `app.toml` always states the flag; a file without the line pushes `false`.
- **Root documents as large documents.** `rootDocumentFormat = 2` under `[app]` in `app.toml` creates each new user's root document as a large document; omit it, or set `1`, for an ordinary root. Existing root documents keep their format. `GET /settings` and the admin app object report it, and a value other than `1`, `2` or empty is refused with `400 INVALID_ROOT_DOCUMENT_FORMAT`, and a child app copies it from its parent.
- `primitive documents import` installs a large root-document export into the target user's large root document, and refuses a root export whose format differs from the target root's, naming both, before writing anything for it. The admin root-document routes report `documentFormat: 2` for a large root.
- **Breaking:** Database, collection and group type configs name their rule set with `ruleSetName` in TOML; `config push` refuses a `ruleSetId` line there.
- **Breaking:** A blob bucket's `accessPolicy` is removed from bucket TOML, the bucket API and both clients and is refused with `400 RETIRED_REQUEST_KEY`, so set `preset` (`public`, `authenticated` or `admin-only`) instead.
- **Breaking:** A prompt or integration test case names its configuration and judge prompt only by `configName`, `evaluatorPromptKey` and `evaluatorConfigName`, and `config push` refuses a case file that carries `configId`, `evaluatorPromptId` or `evaluatorConfigId`.
- **Breaking:** `config push` refuses `ruleSetId` in a blob bucket's TOML; name the rule set with `ruleSetName`, which push resolves in the app you push to, and `config pull` rewrites a file that still carries the id.
- **Fixed:** `config pull` leaves a file untouched when `config diff` reports it in sync, so its comments, key order and explicit defaults survive the pull. Files that differ from the server are still rewritten, and the pull summary reports how many files were written and how many were left unchanged.
- **Fixed:** `primitive documents export` and `export-all` no longer report success when a document's permissions, pending invitations, aliases or blob list can't be read. A 404 still means "none"; a transient failure is retried, and any other failure names the document and exits non-zero. `export-all` exports the remaining documents and lists the failed ones. A document with more than one page of blobs now exports all of them, not just the first page.
- **Fixed:** `primitive documents import` retries a transient failure, reports each document that failed as failed or incomplete and continues with the rest, exiting non-zero at the end; re-running it completes a document a previous run left without its state or some of its blobs, large documents included, and skips those already whole.
- **Fixed:** `primitive users set-role <user-id> <role>` works in its documented two-argument form instead of exiting with a missing-argument error.
- **Fixed:** `config push` of an `[invitations]` or `largeDocumentWindowDays` edit no longer leaves `config diff --only app` reporting drift, because the settings write answers with every field a read returns.

### Starter templates

- **Breaking:** The Vue template's `pnpm cf-deploy` refuses a bare environment name; name both environments with `--deploy-env` and `--primitive-env`.
- **Fixed:** The dev tools overlay keeps its own colors and fonts instead of taking on the app's theme.
- **Fixed:** In the Swift app layer, signing in after a sign-out in the same process no longer shows a false "could not connect" error or keeps the previous account's profile and documents.

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

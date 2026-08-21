# Agent Guide to the Primitive Changelog

User-visible changes in each production release of the Primitive platform, newest first. Use this when upgrading an app's platform libraries or CLI: scan the entries newer than the app's current versions for new capabilities to adopt, changed behavior to accommodate, and deprecations to migrate off. Only user-visible changes are listed — new features, client API and CLI additions and changes, and behavior-changing fixes. Platform-specific items are labeled (JavaScript client, Swift client, CLI); unlabeled items apply everywhere. An `## Unreleased` section, when present, lists changes already merged since the last production release; it is date-stamped when that release ships.

<!-- changelog:insert — the nightly docs sweep maintains the `## Unreleased` section directly below this line; the production deploy date-stamps it. Keep this comment in place. -->

## Unreleased

### New

- **Webhook verification schemes.** Inbound webhooks can verify requests with a JWT scheme (fetching keys from a remote JWKS URL), a declarative custom detached-signature configuration, or the built-in Plaid preset — all configurable in the admin console.
- **Metadata reverse lookup.** A resource can be found by a unique metadata value, in the JavaScript client, the Swift client, and the CLI.
- **Document record aggregation.** Documents gained a records `aggregate` endpoint matching the databases one, and the CLI's `documents records` and `databases records` groups expose the same verb set.
- **Workflow step execution state.** Templates can read whether an upstream step succeeded, failed, or was skipped, so `runIf` conditions no longer have to re-derive it.
- **Batch database writes in workflows.** The database write step accepts multiple records per operation call.
- **Metadata-filtered user fan-out.** The iterate-users step accepts a `metadataFilter`, limiting the fan-out to users whose metadata matches a value.
- **CLI: document management.** `primitive documents` gained read and inspection verbs, permission grant and revoke, and `create` and `delete` — with the owner assignable by email.
- **CLI: database record commands.** New `records get`, `count`, `aggregate`, `save`, and `patch` verbs, plus `databases list --owner`.
- **CLI: live log following.** Log inspection commands gained a `--watch` mode that resumes from the last entry seen.
- **Swift client: notifications, locks, and resource metadata.** A notifications inbox with push-token registration, named locks, and the full resource-metadata value API (get, set, getBatch, list, delete) — each matching the JavaScript client.
- **Swift client: events as `AsyncStream`.** Client events are available as `AsyncStream` sequences, with opt-in main-actor delivery and delivery metadata on each event.
- **Swift client: reliable workflow completion waiting.** `workflows.waitFor(runId:)` resolves a run's completion even when the run finishes while the client is disconnected, reconciling on reconnect; a typed overload decodes the run output.
- **Swift client: typed subscriptions codegen.** Database codegen emits a typed subscriptions factory, matching the JavaScript generator.
- **Admin console: error groups, connections, and run detail.** Recurring errors are grouped in a dedicated analytics view, an app's live connections are inspectable, and the workflow runs table names the step a failed run failed on.

### Changed

- **Access rules are required on prompts, workflows, and integrations.** Every create surface requires an `accessRule`, and execution fails closed — a resource with no rule refuses callers. Integration calls are checked against their own rule on every call.
- **Credentials are configured by reference, not by literal.** Webhook signing secrets and the Google client secret accept only secret references on writes, and literals are masked on reads. Existing literal signing secrets keep verifying deliveries, but editing such a webhook requires migrating it to a reference. The separate integration-secret store is retired — app secrets are the only credential store. Rotation grace periods are capped at 30 days.
- **Server-side configuration is authored in TOML only.** The CLI's config-setting flags are removed: author changes in the sync directory and push them. Integration test cases move into the config tree the same way.
- **CLI command reorganization.** `sync` now lives under `config`; `settings get` is folded into `apps get`; `collections docs` is renamed to `documents`; `database-types` is renamed to `database-type-configs`; metadata category configuration is split into its own noun; per-subject analytics is consolidated under `analytics` (the `workflows analytics` group is gone); and the deprecated `llm` group and `workflows publish` are removed. `--dir` is the sync-directory override everywhere, with `--sync-dir` kept as a hidden deprecated alias.
- **Unresolved workflow template references fail the step** instead of substituting silently. `config sync push` also validates `{{ }}` expressions statically, so unknown roots and references to undeclared steps fail at push time rather than at run time.
- **Prompt steps declare their output type.** `prompt.execute` takes `expect = "text"` (the default) or `"json"`, and `content` is typed by `expect` alone rather than by the active prompt configuration. Workflows that relied on automatic parsing need `expect = "json"`.
- **Workflow push always sends the full definition.** A field omitted from your TOML now clears the server value instead of silently preserving whatever was there.
- **`forEach` default cap raised to 500.** A 250-row provider page iterates without configuration, and an over-cap list fails with an error naming the limit and the `maxItems` override instead of being silently truncated.
- **Declarative locks apply to synchronous runs** and to `workflow.call`, not just queued runs. A step that loses a lock under `onContention: fail` is now distinguishable from an application error in run status and error analytics.
- **Role-aware records API.** Admin data routes are consolidated under `records/*` with role-aware authorization, `databases.list` requires a bounding filter, and bulk record operations carry their payload under `data` — the old `fields` key is rejected with a clear error instead of silently writing nothing.
- **Stricter analytics and webhook limits.** A filter set over the cap is rejected with an error instead of having filters silently dropped, and an app is limited to 50 webhooks.
- **Swift client: one breaking API migration.** Every deprecated symbol queued for the next major is removed in a single batch — `get*()` methods become properties, millisecond `Int` parameters become `TimeInterval`, event, awareness, and analytics payloads (and codegen row data) become `[String: JSONValue]`, generated code moves behind a `client.codegen` facade, and `databases.subscribe` returns an `EventSubscription`. Internal transport types are no longer public, and `BlobManager` is an actor.
- **Swift client: the libraries build under the Swift 6 language mode** with strict concurrency.
- **Swift client: connectivity and list options behave as described.** Network-path monitoring works, with gating and reporting separated (the ineffective probe option is removed); `includeSystem` reaches the server; `waitForLoad` and `serverTimeoutMs` are implemented; cached documents are served immediately while sync refreshes them; and `me.ownedDocuments` returns cached rows right away with a background refresh — pass `.network` for the previous always-blocking behavior.
- **JavaScript client: typed `openDocument` errors.** Fast-fail open errors carry error codes instead of being plain `Error`s, matching the Swift client.

### Fixed

- Reopening a long-lived document after a cold start is fast: stored updates are compacted durably instead of being replayed one at a time, and concurrent reconstructions of the same document are deduplicated.
- Edits to `localOnly` documents are no longer transmitted to the server, in both the JavaScript and Swift clients.
- Workflow runs are debuggable while in flight — run steps are visible before the run reaches a terminal state, child runs started by `workflow.call` appear in the runs list, and `workflows runs failures` shows the actual error for each failure and honors `--limit`.
- Workflow `forEach` corrections: sources written as `{{ }}` templates resolve, checkpoints are scoped per iteration so iterations no longer collide, classified conflict error codes survive durable error capture, and a `runIf`-gated step downstream of a skipped step is skipped instead of failing on source resolution.
- Workflow templates express boolean fallbacks correctly — `|| false`, `default:`, and the boolean filter all handle `false`.
- Workflow run status is reconciled on the server, so every client reports one consistent terminal status, and runs spawned per user by an iterate-users step finalize correctly.
- Webhook hardening: deduplication keys derive from signed material and are domain-separated, so unsigned fields cannot forge duplicate suppression; JWT verification rejects tokens whose key id cannot be matched to a resolvable key; the test endpoint refuses to send an unsignable preview and verifies caller-supplied headers; and a duplicate webhook key returns a conflict error instead of failing opaquely.
- Error-group analytics no longer mint a separate group for messages that differ only by embedded identifiers or numbers, distinct step and run failures are no longer collapsed together, timestamps are epoch seconds, JSON key casing is preserved, and each group carries an exemplar.
- Adding a user by email is reliable: addresses are normalized across every provisioning path, transient conflicts are retried, and failures are descriptive instead of an opaque server error.
- Secret references containing invisible formatting characters are rejected up front instead of failing confusingly later, and setting a configuration variable no longer misreports an unrelated write conflict as a duplicate key.
- Legacy admin app routes enforce per-app access checks, and admin sessions can refresh their tokens.
- CLI: `config sync` round-trips correctly — pull preserves `[[include]]` fragments and no longer writes workflow files a later push would reject, push lands a schema change together with an operation rewrite that depends on it, change detection agrees with `diff` and consults live server state, and `diff` detects webhook `workflowKey` and prompt `accessRule` changes.
- CLI: `init` honors the configured server as well as the `skip_install` and `dev_port` keys, query-shaped `operations execute` results print in the standard list envelope, the upgrade hint names the package manager the CLI was installed with, and a vulnerable archive-extraction dependency is patched.
- JavaScript client: device reachability no longer overwrites the app-set `networkMode`, challenge-driven token refreshes are capped so a refresh loop can no longer occur, switching accounts re-authenticates the connection as the new user, five missing event names are present on `JsBaoEvents`, and the published package no longer ships unresolvable source maps.
- Swift client: the connection lifecycle now matches the JavaScript client — re-authentication mid-connection keeps long-lived connections alive through token rotation, an authentication failure triggers a token refresh instead of failing the connection, server-pushed frames are no longer dropped, sync state and presence reset on disconnect with exactly one disconnected status per close, stalled syncs recover through a watchdog, queued updates flush on open, and forcing a reconnect no longer tears the connection down.
- Swift client: authentication is durable — sessions restore on cold start with leeway on token expiry, magic-link, one-time-code, and passkey sign-in connect automatically, concurrent unauthorized responses coalesce into a single refresh, failed refreshes retry on a backoff, a network outage during a refresh is no longer reported as an authentication failure, and signing out fully clears the session.
- Swift client: the document lifecycle matches the JavaScript client — initial sync, the close and evict cycle, metadata frames, eviction on authoritative listings, permission bootstrapping at open time, `documentLoaded` and `documentOpened` firing once per open, and typed errors on an open timeout.
- Swift client: writes are correct and ordered — outbound flushes are serialized and drained on send confirmation, typed `save(in:)` writes only the fields that changed, a zero-debounce save can no longer report synced before flushing, and documents with large externally stored updates load on a cold start.
- Swift client: `transactAndSync` on an unopened document throws instead of crashing, an un-encodable URL path segment throws, reserved characters in URL values are percent-encoded so a plus-addressed email resolves the right user, reconnect backoff no longer overflows, removing all event handlers no longer deadlocks the event emitter, and `documents.validateAccess` and `getRoot` call real endpoints.
- Swift client: opening a database no longer misreads existing data as empty, `find` and `findByUnique` return string-set members, and the in-app debug inspector's storage panel finds the local database again.
- Admin console: analytics filters on the same field merge with operator-aware deduplication instead of stacking or overwriting each other, invalid filters surface the server's error instead of failing silently, secret-reference validation matches the server's rules, and the custom detached-signature webhook editor enforces the rules it states.
- Starter templates and libraries: the Swift template's Xcode project pin stays in sync with the app's resolved packages and its development auth bypass is removed, both starter templates carry the same platform instructions, newly scaffolded Vue projects ignore `.eslintcache` and `.wrangler`, the Vue library accepts newer peer dependency versions and no longer ships unresolvable declaration maps, and the Swift client package ships consistent resolved dependencies.

## 2026-07-24

### New

- **Conditional document writes.** Single-document `save`, `patch`, and `delete` accept `upsertOn`, `precondition` (compare-and-set), and `ifNotExists`; bulk updates accept a per-operation `precondition`. A failed guard rejects the write with `CONDITION_NOT_MET` and rolls back the whole batch. Preconditions are field-equality only.
- **Multiple sign-in providers per account.** A user can link more than one OAuth provider (for example Google and Apple); signing in with either resolves the same account.
- **Error-event analytics.** Failures are recorded as fingerprinted error events in a dedicated dataset, queryable with the `errors.groups` query type — via the HTTP API, the workflow `analytics` step, or `primitive analytics errors-groups`.
- **Parameterized workflow fragments.** A workflow fragment include accepts parameters, so one fragment serves multiple call sites.
- **`primitive blob-buckets head`** reads a blob's metadata without downloading its content.
- **`primitive connections list`** inspects an app's active client connections.

### Changed

- **Unified list pagination.** Every list endpoint returns `{ items, hasMore, nextCursor }`. The previous `cursor` field and per-endpoint array keys remain as deprecated aliases for one deprecation window — migrate readers to `items` and `nextCursor`.
- **Stricter list-input validation.** Zero, negative, or non-integer `limit` values and malformed cursors now return `400` instead of being silently coerced.

### Fixed

- The JavaScript client advances through every page of a list result instead of stopping after the first.
- Users who linked a second sign-in provider are no longer locked out of the first.
- A just-created document appears in the creator's owned-documents list immediately.
- Group-membership reads paginate fully instead of silently truncating for users with many memberships.
- CLI: date-time field values are serialized correctly on `primitive config push`, and workflow lock configuration (`[workflow.lock]`) survives config push and pull.
- Swift client: `waitForWriteConfirmation` and `waitForInSync` wait for the write to actually be confirmed, and reconnect backoff no longer overflows during long outages.

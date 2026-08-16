# Agent Guide to the Primitive Changelog

User-visible changes in each production release of the Primitive platform, newest first. Use this when upgrading an app's platform libraries or CLI: scan the entries newer than the app's current versions for new capabilities to adopt, changed behavior to accommodate, and deprecations to migrate off. Only user-visible changes are listed — new features, client API and CLI additions and changes, and behavior-changing fixes. Platform-specific items are labeled (JavaScript client, Swift client, CLI); unlabeled items apply everywhere.

<!-- changelog:insert — the production release process prepends each release's entry directly below this line. Keep this comment in place. -->

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
- CLI: date-time field values are serialized correctly on `primitive sync push`, and workflow lock configuration (`[workflow.lock]`) survives sync push and pull.
- Swift client: `waitForWriteConfirmation` and `waitForInSync` wait for the write to actually be confirmed, and reconnect backoff no longer overflows during long outages.

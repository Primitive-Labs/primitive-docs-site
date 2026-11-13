# Agent Guide to Primitive Usage

Primitive meters what an app uses — requests, function runs and steps, AI tokens, realtime messages, emails, rows read and written, stored bytes — and puts an estimated cost on each figure from list prices. App admins read it by meter and day, by user, and traced to what ran it, through `primitive usage` or the admin API. Every command takes `--json` and prints what the route answers. App code reads one member's quantities from a server function or the JS client.

## Read a month

```bash
primitive usage get                     # this month to date
primitive usage get --month 2026-09     # a month that has ended
primitive usage get <app-id> --json
```

The answer has the month's `meters` (quantity, unit, `estCostMicros`, costliest first), its `days` (each day's cost and meters), `storage.current` (bytes held now per storage meter) and `total`. A parent app's read adds `children`: one line per child app, deleted ones included (`deleted: true`, `name: null`), summed into `total.children`.

Meters are named for what they count: `api.request`, `fn.invoke`, `fn.step`, `fn.cpu_ms`, `llm.tokens.in`, `llm.tokens.out`, `ws.msg.in`, `email.sent`, and others. Meters ending in `.bytes` are storage (`kind: "storage"`), counted in byte-days.

## Read by user

```bash
primitive usage top-users                                  # by api.request
primitive usage top-users --meter llm.tokens.in --limit 20
primitive usage top-users --by cost --since 2026-11-01     # by cost over every meter
primitive usage user <user-id> --since 2026-11-01          # one user's meters and owned bytes
```

`billedTo` is the user who drove the usage; `sys` is the line for usage no user drove (triggers, scheduled runs, maintenance). `usage user` accepts `<id>`, `user:<id>` or `sys`. With `--by cost` and no `--meter`, `quantity` is `null` (the ranking spans meters).

## Trace system usage

```bash
primitive usage system
primitive usage system --meter fn.invoke --limit 50
```

Each row is one `origin` (what started the work: `cron:<trigger-id>`, `webhook:<webhook-id>`, or the user id that started a chain of calls), `source` (`trigger`, `fn`, `api`, `ws`, `walk`, `agent`, `system`), `resourceId` (the function, document or other resource) and `meter`, costliest first.

## Admin API

All routes are `GET` under `/admin/api/apps/{appId}/usage`, need an admin token with `read` pinned to the app (`primitive token --scope read --app <app-id>`), and answer the app's admins (a parent app's admins for its children).

| Route | Query | Answers |
| --- | --- | --- |
| `/usage` | `month` (`YYYY-MM`, default the current month) | `meters`, `days`, `storage.current`, `children` on a parent, `total`, `truncated` |
| `/usage/users` | `meter` (default `api.request`), `by=cost`, `since`, `until`, `limit` (1–100, default 10) | `rows: [{ billedTo, quantity, estCostMicros, points }]` |
| `/usage/users/{userId}` | `since`, `until` | `meters`, `storage` (bytes the user owns now) |
| `/usage/system` | `meter`, `since`, `until`, `limit` (1–1000, default 100) | `rows: [{ origin, source, resourceId, meter, quantity, estCostMicros, points }]`, `truncated` |

`since` and `until` are ISO 8601 in UTC (a date, or a time; no zone reads as UTC). The window includes `since`, ends before `until`, defaults to the 30 days before now, and may start at most 90 days ago. Costs are `estCostMicros`, millionths of a dollar, unrounded.

| `code` (400) | Cause |
| --- | --- |
| `USAGE_MONTH_INVALID` | `month` is not `YYYY-MM`, or is in the future |
| `USAGE_WINDOW_INVALID` | `since`/`until` not ISO 8601, or `since` not before `until` |
| `USAGE_WINDOW_TOO_OLD` | `since` more than 90 days ago; read the month with `?month=` instead |
| `USAGE_METER_UNKNOWN` | `meter` is not a usage meter |
| `USAGE_RANK_INVALID` | `by` is not `cost` |
| `USAGE_LIMIT_INVALID` | `limit` out of range |
| `USAGE_USER_INVALID` | `userId` names no user (`user:` alone) |

## From app code

A server function reads one member's usage with `ctx.usage.forUser(userId, { since?, until? })`; the route is `GET /app/{appId}/api/usage/users/{userId}` (beta). The answer is `{ appId, userId, since, until, meters: [{ meter, unit, quantity, bytes, points }], storage: [{ meter, bytes }] }` with no cost field, over the same window rules as above (`since`/`until` may also be `Date`s).

```ts
const usage = await ctx.usage.forUser(member.userId, { since: new Date(Date.now() - 86_400_000) });
const tokens = usage.meters.find((line) => line.meter === "llm.tokens.in")?.quantity ?? 0;
```

Who may be named is the rule for any call about one user: a member reads themselves; an app owner or admin, a run with no caller, or a member's run of a function declaring `users:readAny` reads any member; an agent-driven run reads only its initiator. Others get `SUBJECT_USER_FORBIDDEN` (403); a non-member `SUBJECT_USER_NOT_FOUND` (404).

{{#lang ts}}
In the JS client the same read is `client.usage.forUser(userId, { since?, until? })`: a member reads their own usage, an app owner or admin any member's. A refusal rejects with a `JsBaoApiError` carrying `status` and `code`.
{{/lang}}

## Gotchas

- **Costs are estimates from list prices**, not a bill. Compare them; do not present them as charges.
- **A closed day can change until the 3rd of the next month.** Late usage is counted into it until then; a day with `frozen: true` is final. Do not cache a month's figures before then.
- **`usage.dropped` counts usage that could not be recorded.** When it is above 0, that day's other meters read low by about that many records.
- **Windows reach back 90 days.** For older usage read `usage get --month`, which has days and meters but not users.

# Agent Guide to Emptying a Primitive Database Model

> **Status: experimental.** Experimental. Production does not serve this; every other environment does.

Clear a model, not a `refresh` load with no rows or per-record deletes, to remove every record of it: one call whatever the record count, one change on the server, and one change for each device to apply.

```typescript
  try {
    const { rows } = await client.db3.records(db3Id).clear("embeddings");
    console.log(`removed ${rows} records`);
  } catch (error) {
    if (!(error instanceof Db3ClearApplyPendingError)) throw error;
    // The server emptied the model; this device has not caught up yet.
    // Don't clear again. Wait for sync before writing to the model here.
    console.log(`removed ${error.result.rows} records; this device catches up shortly`);
  }
```

CLI: `primitive db3 records clear <db3-id> <model> [--drop] [-y] [--json]` (asks first unless `-y`). HTTP: `POST db3/{db3Id}/records/{model}/clear`, body `{ "drop"?: boolean }`.

- Answers `{ model, rows, lsn, epoch, dropped, cost }`. With no live record `rows` is 0, `lsn` and `epoch` are `null`, and nothing is written.
- Needs write access. 409 `DOCUMENT_INGEST_ACTIVE` (with `sessionId`) while a bulk load is open, committed or applying. Runs no trigger, hook or `[[function.triggers.db3]]` run. Not idempotent: a retry clears what the model holds then.
- On a db3 open here and syncing, the call returns only after this device applied the clear. It returns at once when nothing was written, when the db3 is not open here, or when its sync is off.
- Racing writes end as against a deleted record. A create stamped before the writing device applied the clear (an offline queue, an in-flight save) is accepted and discarded, even for a new id. A patch or delete of a cleared record leaves it gone. An increment is refused `DB3_RECORD_NOT_FOUND`. Server writes after the clear apply.
- `drop: true` on an untyped db3 also removes the model's declaration, observed fields and registered indexes, so `schema` stops listing it. It needs the owner or an app admin (as index registration does). On a typed db3 it is refused 409 `DB3_MODEL_DROP_REFUSED_TYPED` and nothing changes. A device that still declares the model declares it again on its next open.
- To remove only the records matching a filter, use `deleteWhere` (see the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#changing-every-record-that-matches-a-filter)).
- Refill a model by clearing it and then bulk loading (see the [Databases guide](AGENT_GUIDE_TO_PRIMITIVE_DATABASES.md#bulk-loading)).

**Watch out when clearing:** if this device has not applied the clear 10 s after the server answered, the call fails `DB3_CLEAR_APPLY_PENDING` carrying the server's answer, as the example catches. The server did clear the model, so don't retry. Until the device catches up, a record it creates in the model is discarded, so hold writes to that model until it has synced.

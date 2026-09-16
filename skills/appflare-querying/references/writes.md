# Writes reference

All write methods return `Promise<Row[]>`. Every `ctx.db` write is atomic: it compiles to statements that run in one D1 batch.

## insert

```ts
await ctx.db.table.insert({ values: row | row[] });
```

- Applies runtime defaults (`v.uuid()`, `.defaultFn()`, `.defaultNow()`) and serializes JSON columns.
- Large arrays are split into several statements inside the same batch to stay under D1's 100 bound-parameter limit.
- Relation fields can be passed inside `values`:

| Relation | `id` item | `{ id, ... }` item | `{ ... }` (no id) item |
| --- | --- | --- | --- |
| `v.one` (single item) | set FK | set FK to that id | create target, set FK |
| `v.many` (item or array) | set child FK → parent | update child with object + FK | create child with FK |
| `v.manyToMany` (item or array) | link | link | create target, link |

The returned rows include the relation keys you passed (hydrated rows). Passing `ownerId` and `owner` with different values throws.

## update

```ts
await ctx.db.table.update({ where, set, limit?, allowAll?, expectRows? });
```

- A missing or empty `where` **throws** unless `allowAll: true`.
- `set` accepts plain values or SQL expressions.

```ts
import { decrement, increment, now, raw } from "appflare";

set: { balance: decrement(50), views: increment(), updatedAt: now(), score: raw(sql`max(${t.score}, 0)`) }
```

- Many-to-many fields in `set`:

```ts
set: { labels: { items: [1, { id: 2 }, { name: "new" }], mode: "merge" } } // add links (default)
set: { labels: { items: [3], mode: "overwrite" } }                          // replace links
set: { labels: [1, 2] }                                                     // shorthand = merge
```

Objects without an id are inserted into the target table first. Link changes run before the row update in the same batch, so they match the same rows as `where`.

## expectRows

```ts
await ctx.db.accounts.update({
	where: { id, balance: { gte: amount } },
	set: { balance: decrement(amount) },
	expectRows: 1,              // or { min: 1 }, { min: 1, max: 10 }
});
```

Out-of-range row counts throw `AppflareConflictError` and write nothing — optimistic concurrency without a transaction. Works on `update`, `upsert` and `delete`, including inside `batch`, where it rolls back the whole batch.

## batch

```ts
const [rowsA, rowsB] = await ctx.db.batch((tx) => [
	tx.ledgerEntries.insert({ values: entries }),
	tx.accounts.update({ where: { id }, set: { balance: decrement(cents) }, expectRows: 1 }),
]);
```

The callback must return the plans synchronously; results come back in the same order. Any failure writes nothing and publishes no mutation events.

## transaction

```ts
const result = await ctx.db.transaction(async (tx) => {
	const account = await tx.accounts.findFirst({ where: { id } }); // reads hit the committed DB
	if (!account) ctx.error(404, "Not found");
	const handle = tx.accounts.update({ where: { id }, set: { status: "closed" } });
	await ctx.scheduler.enqueue("jobs/notify", { id });             // flushed after commit
	return handle;
});
result.rows[0].status;
```

- Writes on `tx` return **handles**, not promises. `handle.rows` throws until the transaction resolves.
- Reads don't see queued writes: read first, then queue writes.
- Throwing writes nothing and drops enqueued messages.
- `batch` and `transaction` cannot be nested.

## upsert

```ts
await ctx.db.table.upsert({ values, target?, set?, expectRows? });
```

| Arg | Default |
| --- | --- |
| `target` | `["id"]` if the table has `id`, otherwise an error (`could not infer a conflict target`) |
| `set` | each row's own values (`excluded.<column>`) |

`target` columns need a primary key or unique constraint. Rows that set different columns are split into separate statements.

## delete

```ts
await ctx.db.table.delete({ where, limit?, allowAll?, expectRows? }); // returns deleted rows
```

## Ownership-scoped writes

```ts
const [row] = await ctx.db.projects.update({
	where: { id: args.id, ownerId: ctx.user.id },
	set: { name: args.name },
});
if (!row) ctx.error(404, "Project not found");
```

## Invariants for every row

A batch can't abort on a condition, so rules that must always hold belong in the schema:

```ts
accounts: table({ balance: v.int().notNull().default(0) }, {
	checks: [{ name: "accounts_balance_non_negative", sql: "balance >= 0" }],
}),
```

Violations surface as `CheckConstraintError`, and `RAISE(ABORT, 'reason')` in a trigger as `TriggerAbortError` (HTTP 400 with your message). Add triggers with `bun appflare migrate:custom --name add_guards`.

## Realtime

Each `ctx.db` write records `{ kind, table, args, rows }` in `ctx.mutationEvents`, published only **after** the write commits. Subscribed queries on that table whose `where` matches are re-run and pushed. `ctx.$db` writes are not recorded.

# Writes reference

All write methods return `Promise<Row[]>`.

## insert

```ts
await ctx.db.table.insert({ values: row | row[] });
```

- Applies runtime defaults (`v.uuid()`, `.defaultFn()`, `.defaultNow()`) and serializes JSON columns.
- Relation fields can be passed inside `values`:

| Relation | `id` item | `{ id, ... }` item | `{ ... }` (no id) item |
| --- | --- | --- | --- |
| `v.one` (single item) | set FK | set FK to that id | create target, set FK |
| `v.many` (item or array) | set child FK → parent | update child with object + FK | create child with FK |
| `v.manyToMany` (item or array) | link | link | create target, link |

The returned rows include the relation keys you passed (hydrated rows). Passing `ownerId` and `owner` with different values throws.

## update

```ts
await ctx.db.table.update({ where, set, limit? });
```

- Without `where`, **all rows** are updated.
- Many-to-many fields in `set`:

```ts
set: { labels: { items: [1, { id: 2 }, { name: "new" }], mode: "merge" } } // add links (default)
set: { labels: { items: [3], mode: "overwrite" } }                          // replace links
set: { labels: [1, 2] }                                                     // shorthand = merge
```

Objects without an id are inserted into the target table first.

## upsert

```ts
await ctx.db.table.upsert({ values, target?, set? });
```

| Arg | Default |
| --- | --- |
| `target` | `["id"]` if the table has `id`, otherwise an error (`Unable to infer conflict target`) |
| `set` | first element of `values` (undefined keys dropped) |

`target` columns need a primary key or unique constraint. For arrays, pass an explicit `set` or upsert rows individually.

## delete

```ts
await ctx.db.table.delete({ where?, limit? }); // returns deleted rows; no where = all rows
```

## Ownership-scoped writes

```ts
const [row] = await ctx.db.projects.update({
	where: { id: args.id, ownerId: ctx.user.id },
	set: { name: args.name },
});
if (!row) ctx.error(404, "Project not found");
```

## Atomicity

D1 has no interactive transactions, so nested relation writes and many-to-many updates run sequentially without rollback. For atomic multi-statement writes:

```ts
await ctx.$db.batch([
	ctx.$db.insert(schema.orders).values({ id, userId }),
	ctx.$db.update(schema.inventory).set({ stock: sql`${schema.inventory.stock} - 1` }).where(eq(schema.inventory.sku, sku)),
]);
```

`ctx.$db` writes skip runtime defaults (supply ids yourself) and don't emit realtime events.

## Realtime

Each `ctx.db` write records `{ kind, table, args, rows }` in `ctx.mutationEvents`. After the mutation, scheduler task or cron job succeeds, subscribed queries on that table whose `where` matches are re-run and pushed.

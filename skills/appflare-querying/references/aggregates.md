# Aggregates reference

## count

```ts
await ctx.db.posts.count();                                      // all rows
await ctx.db.posts.count({ where: { ownerId } });                // filtered
await ctx.db.posts.count({ field: "ownerId", distinct: true });  // distinct non-null values
await ctx.db.posts.count({ with: { comments: { where: { id: { gte: 10_000 } } } } });
```

Args: `where?`, `field?` (column or `relation.column` path), `distinct?`, `with?`. Returns `Promise<number>`.

## sum, avg, min, max

```ts
await ctx.db.orders.sum({ field: "totalMinor", where: { status: "paid" } });
await ctx.db.reviews.avg({ field: "rating", where: { productId } });
await ctx.db.products.min({ field: "priceMinor" });
await ctx.db.posts.max({ field: "createdAt" });   // Date column → Date
```

- `sum` and `avg` need a numeric field and return `Promise<number | null>`.
- `min` and `max` accept any comparable column and keep its type.
- All accept `where` and a `relation.column` path with `with`; `sum`/`avg` also accept `distinct`.
- `null` means no rows matched.

## groupBy

```ts
const rows = await ctx.db.ledgerEntries.groupBy({
	by: ["category"],
	where: { accountId },
	_count: true,
	_sum: { amount: true },
	orderBy: { aggregate: "_sum", field: "amount", direction: "desc" },
	limit: 20,
});

rows[0].category;     // string
rows[0]._count;       // number
rows[0]._sum.amount;  // number | null
```

| Arg | Meaning |
| --- | --- |
| `by` | Columns to group by; they appear on each row |
| `where` | Filter before grouping |
| `_count` | `true` for row counts, or `{ field: true }` for non-null counts |
| `_sum` / `_avg` | `{ numericField: true }` |
| `_min` / `_max` | `{ field: true }`, keeps the column type |
| `orderBy` | `{ column }` (must be in `by`) or `{ aggregate, field }` (must be selected) |
| `limit` / `offset` | Same default (100) and maximum (1000) as `findMany` |

## Relation aggregates per row

```ts
const posts = await ctx.db.posts.findMany({
	with: { comments: { _count: true, _avg: { id: true } } },
});

posts[0].commentsAggregate.count;  // number
posts[0].commentsAggregate.avg.id; // number
```

- The key is `<relationName>Aggregate`.
- `_count: true` adds `count`; `_avg: { numericField: true }` adds `avg.numericField`.
- These are computed in JavaScript from the loaded relation rows, so use `groupBy` or `count` for large relations.

## Patterns

Dashboard stats in one handler:

```ts
const [total, open, revenue] = await Promise.all([
	ctx.db.tasks.count({ where: { projectId } }),
	ctx.db.tasks.count({ where: { projectId, done: false } }),
	ctx.db.orders.sum({ field: "totalMinor", where: { projectId } }),
]);
return { total, open, revenue: revenue ?? 0 };
```

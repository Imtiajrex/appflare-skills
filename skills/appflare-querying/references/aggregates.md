# Aggregates reference

## count

```ts
await ctx.db.posts.count();                                      // all rows
await ctx.db.posts.count({ where: { ownerId } });                // filtered
await ctx.db.posts.count({ field: "ownerId", distinct: true });  // distinct non-null values
await ctx.db.posts.count({ with: { comments: { where: { id: { gte: 10_000 } } } } });
```

Args: `where?`, `field?` (column or `relation.column` path), `distinct?`, `with?`. Returns `Promise<number>`.

## avg

```ts
await ctx.db.reviews.avg({ field: "rating", where: { productId } });
await ctx.db.posts.avg({
	field: "comments.id",
	with: { comments: { where: { id: { gte: 10_000 } } } },
});
```

Args: `field` (required, numeric column or relation path), `where?`, `distinct?`, `with?`. Returns `Promise<number | null>`, where `null` means no rows.

## Relation aggregates per row

```ts
const posts = await ctx.db.posts.findMany({
	with: {
		comments: { _count: true, _avg: { id: true } },
	},
});

posts[0].commentsAggregate.count;  // number
posts[0].commentsAggregate.avg.id; // number
```

- The key is `<relationName>Aggregate`.
- `_count: true` adds `count`.
- `_avg: { numericField: true }` adds `avg.numericField`.
- Combine with a relation `where` to aggregate a filtered subset.

## Patterns

Dashboard stats in one handler:

```ts
const [total, open, avgRating] = await Promise.all([
	ctx.db.tasks.count({ where: { projectId } }),
	ctx.db.tasks.count({ where: { projectId, done: false } }),
	ctx.db.reviews.avg({ field: "rating", where: { projectId } }),
]);
return { total, open, avgRating: avgRating ?? 0 };
```

Use `ctx.$db` with Drizzle's `groupBy` for `sum`, `min`, `max` or grouped results.

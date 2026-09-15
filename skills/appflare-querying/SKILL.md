---
name: appflare-querying
description: Read and write data in Appflare handlers with the typed ctx.db API, including findMany and findFirst with where operators (eq, in, gt, regex/$options, exists, includes, geoWithin), relation loading and filtering with "with", orderBy, limit/offset and cursor pagination, insert with nested relations, update with many-to-many items/mode, upsert with target/set, delete, count and avg aggregates, and the ctx.$db raw Drizzle escape hatch. Use when writing database logic, filters, search, joins, pagination, counts, upserts or bulk updates inside Appflare query, mutation, scheduler or cron handlers.
metadata:
  author: appflare
  version: "0.2.55"
---

# Appflare data access (`ctx.db`)

## Default patterns

```ts
// list with filters, relations, sort and limit
const tasks = await ctx.db.tasks.findMany({
	where: {
		projectId: args.projectId,
		...(args.search ? { title: { regex: args.search, $options: "i" } } : {}),
		done: false,
	},
	with: { assignee: true, labels: true },
	orderBy: { column: "id", direction: "desc" },
	limit: args.limit,
});

// single row (or null)
const task = await ctx.db.tasks.findFirst({ where: { id: args.id } });

// create
const [created] = await ctx.db.tasks.insert({ values: { title, projectId } });

// update (always scope with where, ideally including ownership)
const [updated] = await ctx.db.tasks.update({ where: { id, assigneeId: ctx.user.id }, set: { done: true } });

// delete
const removed = await ctx.db.tasks.delete({ where: { id } });

// counts
const open = await ctx.db.tasks.count({ where: { projectId, done: false } });
```

## Procedure

1. Check the schema for the table name (the key in `schema({...})`) and the field names. Relation FKs are `<relation>Id`.
2. Put every filter in `where`. For optional filters, spread them in conditionally and never pass `undefined` values.
3. Load related rows with `with`. To filter parents by related rows, use the relation name inside `where`.
4. For lists, add `orderBy` and `limit`. For pages, return `{ rows, nextCursor, hasMore }`.
5. For writes, scope `update` and `delete` with `where`, and check the returned array length to detect "not found".

## Cursor pagination template

```ts
args: {
	cursor: z.coerce.number().int().optional(),
	pageSize: z.coerce.number().int().min(1).max(50).default(20),
},
handler: async (ctx, args) => {
	const rows = await ctx.db.posts.findMany({
		where: args.cursor ? { id: { lt: args.cursor } } : {},
		orderBy: { column: "id", direction: "desc" },
		limit: args.pageSize,
	});
	return { rows, nextCursor: rows.at(-1)?.id, hasMore: rows.length === args.pageSize };
},
```

## Gotchas

- `regex` is **SQL `LIKE '%value%'`**, a substring match and not a real regex. `$options: "i"` makes it case-insensitive.
- There is **no OR** operator. Use `in` for one field, or `ctx.$db` for cross-field OR.
- Filter on FK **fields** (`ownerId`), not relation names, when you mean "equals this id". A relation name in `where` means "has a related row matching…".
- `update` or `delete` without `where` affects **every row**.
- Every write method returns an **array**. Destructure (`const [row] = …`) and check for `undefined`.
- `upsert` defaults `target` to `"id"`, and `set` to the **first** values item, which is wrong for batch upserts. Pass `set` explicitly.
- Nested and many-to-many writes aren't atomic on D1. Use `ctx.$db.batch([...])` for all-or-nothing writes.
- `ctx.$db` bypasses runtime defaults, JSON handling and realtime events. Prefer `ctx.db`.
- `v.date()` filters accept `Date` or epoch milliseconds.
- `_count`/`_avg` inside `with` add `<relation>Aggregate` (e.g. `row.commentsAggregate.count`).

## References

- Read [references/where-operators.md](references/where-operators.md) when building non-trivial filters: all operators, relation filters, JSON array and object filters, dates, geo distance, ordering.
- Read [references/writes.md](references/writes.md) when inserting nested relations, updating many-to-many links (`items`/`mode`), upserting, or needing atomic batches.
- Read [references/aggregates.md](references/aggregates.md) when computing counts, averages, distinct counts or per-row relation aggregates.

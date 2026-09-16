# Where operators and read options

## findMany / findFirst args

| Arg | Shape |
| --- | --- |
| `where` | `{ field: value \| operators, relationName: where, and, or, not, geoWithin }` |
| `with` | `{ relation: true \| { where?, limit?, columns?, with?, _count?, _avg? } }` |
| `orderBy` | `{ column, direction?: "asc" \| "desc" }` or an array |
| `limit`, `offset` | numbers. `limit` defaults to 100 and cannot exceed 1000 |
| `columns` | `{ field: true }` |

`findFirst` returns a row or `undefined`, and `findMany` returns an array.

## Field operators

| Operator | SQL meaning | Example |
| --- | --- | --- |
| (plain value) | `=` | `{ status: "open" }` |
| `eq` / `ne` | `=` / `<>`; `null` → `IS NULL` / `IS NOT NULL` | `{ status: { ne: "archived" } }` |
| `in` / `nin` | `IN` / `NOT IN`; `in: []` matches nothing, `nin: []` is a no-op | `{ id: { in: [1, 2, 3] } }` |
| `gt` `gte` `lt` `lte` | comparisons | `{ price: { gte: 10, lt: 100 } }` |
| `exists` | `IS NOT NULL` (`true`) / `IS NULL` (`false`) | `{ deletedAt: { exists: false } }` |
| `contains` | substring, `%`/`_` escaped | `{ title: { contains: "100%" } }` |
| `startsWith` / `endsWith` | prefix / suffix | `{ slug: { startsWith: "2026-" } }` |
| `regex` | alias of `contains` | `{ title: { regex: "hello" } }` |
| `options` | `"i"` → case-insensitive | `{ title: { contains: "Hello", options: "i" } }` |

`$`-prefixed aliases (`$eq`, `$gte`, `$options`, …) are accepted. All conditions are ANDed. **Unknown operators and unknown columns throw `AppflareQueryError`.**

## Combinators

```ts
where: {
	or: [{ ownerId: uid }, { assigneeId: uid }],   // at least one branch
	and: [{ views: { gte: 10 } }, { views: { lt: 100 } }],
	not: { status: "archived" },
	done: false,                                   // ANDed with the combinators
}
```

Branches are full `where` objects and nest. `or: []` matches nothing; a branch that ends up empty makes the whole `or` match everything.

## Relation filters (EXISTS)

```ts
where: { comments: { text: { contains: "bug" } } }         // v.many: has a matching comment
where: { owner: { email: "a@b.dev" } }                     // v.one: owner matches
where: { labels: { name: { in: ["urgent", "blocked"] } } } // v.manyToMany
```

Self-relations are aliased automatically. A relation filter whose values are all `undefined` is skipped.

## JSON array columns

| Operator | Meaning |
| --- | --- |
| `includes: [a, b]` | array contains all |
| `includesAny: [a, b]` | array contains at least one |
| `length: n` | `json_array_length = n` |
| `eq: [...]` / `ne: [...]` | whole-array equality |

Arrays of objects accept element property conditions. A row matches when some element matches:

```ts
where: { items: { sku: "A1", qty: { gte: 2 } } } // supports eq, ne, gt, gte, lt, lte, in per property
```

## Dates

`v.date()` fields accept `Date | number` (epoch ms) in comparisons: `{ createdAt: { gte: Date.now() - 86_400_000 } }`.

## Geo distance

```ts
where: {
	geoWithin: {
		$geometry: { latitude: 51.5, longitude: -0.12 },   // or { coordinates: [lng, lat] }
		latitudeField: "lat",
		longitudeField: "lng",
		minDistance: 0,        // or gte/gt (inclusive)
		maxDistance: 10_000,   // meters; or lte/lt (inclusive)
	},
}
```

It uses the Haversine formula with an Earth radius of 6,371,000 m. Both field names are required.

## orderBy

```ts
orderBy: [{ column: "priority", direction: "desc" }, { column: "id" }]
```

An unknown column throws.

## Optional filters

```ts
where: {
	...(args.status ? { status: args.status } : {}),
	...(args.from ? { createdAt: { gte: args.from } } : {}),
}
```

## iterate (full scans)

```ts
for await (const row of ctx.db.accounts.iterate({ where: { status: "active" }, pageSize: 200 })) {
	// keyset pagination on the primary key; no limit ceiling
}
```

Takes the same `where`, `with` and `columns` as `findMany`, plus `pageSize` and `direction`. Needs a single-column primary key, which must stay in `columns`.

# Where operators and read options

## findMany / findFirst args

| Arg | Shape |
| --- | --- |
| `where` | `{ field: value \| operators, relationName: where, geoWithin }` |
| `with` | `{ relation: true \| { where?, limit?, columns?, with?, _count?, _avg? } }` |
| `orderBy` | `{ column, direction?: "asc" \| "desc" }` or an array |
| `limit`, `offset` | numbers |
| `columns` | `{ field: true }` |

`findFirst` returns a row or `null`, and `findMany` returns an array.

## Field operators

| Operator | SQL meaning | Example |
| --- | --- | --- |
| (plain value) | `=` | `{ status: "open" }` |
| `eq` / `ne` | `=` / `<>` | `{ status: { ne: "archived" } }` |
| `in` / `nin` | `IN` / `NOT IN` | `{ id: { in: [1, 2, 3] } }` |
| `gt` `gte` `lt` `lte` | comparisons | `{ price: { gte: 10, lt: 100 } }` |
| `exists` | `IS NOT NULL` (`true`) / `IS NULL` (`false`) | `{ deletedAt: { exists: false } }` |
| `regex` | `LIKE '%v%'` | `{ title: { regex: "hello" } }` |
| `$options` | `"i"` → `lower(col) LIKE lower(pattern)` | `{ title: { regex: "Hello", $options: "i" } }` |

`$`-prefixed aliases (`$eq`, `$gte`, `$lt`, …) are accepted. All conditions are ANDed.

## Relation filters (EXISTS)

```ts
where: { comments: { text: { regex: "bug" } } }            // v.many: has a matching comment
where: { owner: { email: "a@b.dev" } }                     // v.one: owner matches
where: { labels: { name: { in: ["urgent", "blocked"] } } } // v.manyToMany
```

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

It uses the Haversine formula with an Earth radius of 6,371,000 m. Both field names are required. Unknown fields log a warning and the filter is skipped.

## orderBy

```ts
orderBy: [{ column: "priority", direction: "desc" }, { column: "id" }]
```

## Optional filters

```ts
where: {
	...(args.status ? { status: args.status } : {}),
	...(args.from ? { createdAt: { gte: args.from } } : {}),
}
```

## OR logic

There is no OR operator. For a single field, use `in`. Otherwise use raw Drizzle:

```ts
import { or, eq } from "drizzle-orm";
import * as schema from "../_generated/schema.compiled";

await ctx.$db.select().from(schema.tasks)
	.where(or(eq(schema.tasks.ownerId, uid), eq(schema.tasks.assigneeId, uid)));
```

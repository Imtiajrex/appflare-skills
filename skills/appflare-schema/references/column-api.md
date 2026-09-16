# Column API reference

Import with `import { schema, table, v } from "appflare";`

## `schema(tables, options?)`

- `tables`: `Record<string, table(...)>`. Keys are the table names used in `ctx.db.<name>` and relations.
- `options.enums`: optional `Record<string, EnumDefinition>`.
- If the entry file exports several schemas, set `schemaDsl.exportName`.

## `table(shape, options?)`

`shape` values must be column builders or relation helpers. Anything else throws `Invalid table field`.

| Option | Shape | Notes |
| --- | --- | --- |
| `sqlName` | `string` | SQL table name |
| `indexes` | `{ columns: string[]; unique?: boolean; name?: string }[]` | Composite indexes; `columns` are field names, including inferred FKs |
| `checks` | `{ name: string; sql: string }[]` | CHECK constraints; `sql` uses SQL column names |

```ts
accounts: table(
	{ status: v.string().notNull().default("active"), balanceMinor: v.int().notNull().default(0) },
	{
		indexes: [{ columns: ["status", "balanceMinor"] }],
		checks: [{ name: "accounts_balance_non_negative", sql: "balance_minor >= 0" }],
	},
),
```

## Builders

| Builder | Options | SQL | TS type |
| --- | --- | --- | --- |
| `v.int(opts?)` | `sqlName` | integer | number |
| `v.number(opts?)` | same as int | integer | number |
| `v.string(opts?)` | `sqlName`, `length` | text / text(n) | string |
| `v.boolean(opts?)` | `sqlName` | integer (boolean mode) | boolean |
| `v.date(opts?)` | `sqlName` | integer (timestamp_ms) | Date |
| `v.uuid(opts?)` | `sqlName`, `length` (default 36) | text PK NOT NULL | string (runtime `crypto.randomUUID()`) |
| `v.enum(values \| enumBuilder, opts?)` | `sqlName` | text | union |
| `v.enumArray(values \| enumBuilder, opts?)` | `sqlName` | text array | union[] |
| `v.defineEnum(name, values)` | | reusable `EnumBuilder` | |
| `v.array(elementBuilder)` | | text (JSON) | element[] |
| `v.object({ key: builder })` | | text (JSON) | object |

JSON element and shape builders map as follows: `v.string()` to string, `v.int()`/`v.number()` to number, `v.boolean()` to boolean, `v.date()` to date, and nested `v.object()`/`v.array()` to nested shapes.

## Modifiers (immutable, chainable)

| Modifier | Meaning |
| --- | --- |
| `.notNull()` | NOT NULL; required on insert unless a default exists |
| `.nullable()` | nullable (default) |
| `.primaryKey({ autoIncrement? })` | primary key |
| `.unique(name?)` | unique constraint (optional name) |
| `.index(name?)` | index (optional name) |
| `.default(value)` | SQL DEFAULT |
| `.defaultFn(() => value)` | runtime default in `ctx.db.insert` |
| `.defaultNow()` | runtime `new Date()` default |
| `.references(table, column = "id", { onDelete?, onUpdate? })` | manual FK |
| `.sql(name)` | SQL column name |
| `.array()` | enum array |

## Insert optionality and nullability

A field is optional on insert when it is a primary key, isn't `notNull`, is auto-increment, or has `.default` or `.defaultFn` (including `defaultNow` and `uuid`).

A column with any default is **typed non-null** on select and rejects an explicit `null` on insert, even though the SQL column stays nullable. Add `.nullable()` for columns that really store nulls. Rows written before the default existed can still be `NULL`.

## Naming strategy

`camelToSnake` (the only strategy): `createdAt` becomes `created_at`, the `petDocuments` table becomes `pet_documents`, and `owner: v.one(...)` becomes `owner_id`.

## Generated artifacts

`<outDir>/schema.compiled.js` (Drizzle tables and relations), `schema.types.js` (row types) and `schema.zod.js` (Zod schemas). Override the paths with `schemaDsl.outFile`, `typesOutFile` and `zodOutFile`.

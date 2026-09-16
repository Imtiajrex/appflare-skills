# Relations reference

## `v.one(targetTable, fieldOrOptions?, options?)`

Belongs-to. It adds an FK column on **this** table.

| Option | Default | Notes |
| --- | --- | --- |
| `field` (or 2nd arg string) | `<relationName>Id` | FK field name |
| `referenceField` | `"id"` | column on target |
| `fkType` | inferred from target column, else `"string"` | |
| `sqlName` | snake_case of field | |
| `nullable` / `notNull` | NOT NULL | pass `nullable: true` for optional |
| `onDelete` / `onUpdate` | none | `cascade`, `set null`, `set default`, `restrict`, `no action` |
| `unique` | false | unique index on the FK (one-to-one) |
| `index` | false | index on the FK |
| `relationName` | none | pairs this relation with the `v.many` using the same name |

```ts
author: v.one("users"),                       // authorId
reviewer: v.one("users", "reviewedBy"),       // reviewedBy
parent: v.one("categories", { nullable: true, onDelete: "set null" }),
```

## `v.many(targetTable, fieldOrOptions?, options?)`

Has-many. It adds an FK column on the **target** table. It takes the same options as `v.one`, and the default field is `<singularized source table>Id`.

```ts
posts: table({ comments: v.many("comments") }),  // comments.postId
comments: table({ post: v.one("posts") }),       // same postId column
```

If the inverse `v.one` uses another name, pass that name: `v.many("comments", "articleId")`.

### Several relations to the same table

```ts
messages: table({
	sender: v.one("authors", { relationName: "sender" }),
	recipient: v.one("authors", { relationName: "recipient", nullable: true }),
}),
authors: table({
	sentMessages: v.many("messages", { relationName: "sender" }),        // reuses senderId
	receivedMessages: v.many("messages", { relationName: "recipient" }), // reuses recipientId
}),
```

Without `relationName`, a `v.many` whose target has several matching `v.one` relations fails generation with a message listing the candidates. A `relationName` with no matching `v.one` fails too.

## `v.manyToMany(targetTable, options?)`

It synthesizes a junction table.

| Option | Default |
| --- | --- |
| `junctionTable` | `<left><Right>Links`, with tables sorted by `table:referenceField` (e.g. `petsTripsLinks`) |
| `sourceField` / `targetField` | `<singular table>Id`. Self-relations use `source<Singular>Id` / `target<Singular>Id` |
| `sourceSqlName` / `targetSqlName` | snake_case |
| `referenceField` / `targetReferenceField` | `"id"` |
| `onDelete` / `onUpdate` | none |

Rules:

- Declare it on both tables to query from both sides, and keep the options identical. Conflicts throw `manyToMany pair '…' has conflicting … values`.
- The junction name must not match an existing table.
- The junction has no payload columns.
- Identical resolved field names throw `resolves to duplicate junction fields`. Set `sourceField` and `targetField`.

## Using relations in code

```ts
// read
await ctx.db.tasks.findMany({ where: { projectId }, with: { project: true, labels: true } });

// write FK directly
await ctx.db.tasks.insert({ values: { title, projectId } });

// many-to-many links
await ctx.db.tasks.update({
	where: { id },
	set: { labels: { items: [1, 2], mode: "overwrite" } },
});
```

## Common compile errors

| Error | Fix |
| --- | --- |
| `Invalid table field 'x'` | The value isn't a `v.*` builder or relation helper |
| `No schema() export found` | Export `schema(...)`, or set `schemaDsl.exportName` |
| `manyToMany auto junction table '…' conflicts` | Set `junctionTable` |
| `conflicting junctionTable values` | Make both declarations match |

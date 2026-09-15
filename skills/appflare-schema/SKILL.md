---
name: appflare-schema
description: Define and change the Appflare schema DSL (schema, table, v) and apply D1 migrations. Covers column builders (v.int, v.string, v.boolean, v.date, v.uuid, v.enum, v.array, v.object), modifiers (notNull, unique, default, defaultNow), relations with v.one, v.many and v.manyToMany including foreign key naming and junction tables, and the appflare migrate workflow. Use when adding a table, column, enum, JSON field or relation, fixing FK or nullability issues, or running migrations in an Appflare project.
metadata:
  author: appflare
  version: "0.2.55"
---

# Appflare schema

## Workflow

- [ ] Open the schema entry (`schemaDsl.entry` in `appflare.config.ts`, usually `schema.ts`)
- [ ] Edit tables inside the exported `schema({...})`
- [ ] Run `bun appflare dev` from the backend package and fix any compiler error it prints
- [ ] Run `bun appflare migrate --local` and review the new SQL file in `drizzle/`
- [ ] Update the handlers and client code that use the changed fields

## Template

```ts
import { schema, table, v } from "appflare";

export const schemas = schema({
	projects: table({
		id: v.uuid(),                                   // text PK, crypto.randomUUID() on insert
		name: v.string().notNull(),
		slug: v.string().notNull().unique(),
		status: v.enum(["active", "archived"]).notNull().default("active"),
		tags: v.array(v.string()),                      // JSON text
		settings: v.object({ color: v.string(), public: v.boolean() }),
		createdAt: v.date().defaultNow(),               // runtime default on insert
		owner: v.one("users"),                          // ownerId (NOT NULL) → users.id
		tasks: v.many("tasks"),                         // adds projectId to tasks
	}),
	tasks: table({
		id: v.int().primaryKey({ autoIncrement: true }),
		title: v.string().notNull(),
		done: v.boolean().notNull().default(false),
		dueAt: v.date(),
		project: v.one("projects"),                     // projectId: matches v.many above
		assignee: v.one("users", { nullable: true }),   // assigneeId (nullable)
		labels: v.manyToMany("labels"),
	}),
	labels: table({
		id: v.int().primaryKey({ autoIncrement: true }),
		name: v.string().notNull().unique(),
		tasks: v.manyToMany("tasks"),                   // junction: labelsTasksLinks (labelId, taskId)
	}),
});
```

## Rules to apply

- **Primary keys:** use `v.uuid()` for string ids or `v.int().primaryKey({ autoIncrement: true })` for numeric ids.
- **Nullability:** plain columns are nullable and optional. Add `.notNull()` to required fields. FKs from `v.one` and `v.many` are **NOT NULL** unless `{ nullable: true }`.
- **FK names:** `field: v.one("t")` creates `fieldId`. `source.rel: v.many("target")` adds `<singular source>Id` to the target. Give the inverse `v.one` the same name (`project` ↔ `projects.tasks`), or pass the name explicitly (`v.many("tasks", "parentProjectId")`).
- **Many-to-many:** declare `v.manyToMany` on both tables with identical options. The junction holds links only. If the link needs data (a role, a timestamp), create an explicit join table with two `v.one` relations.
- **Defaults:** `.default(x)` is a SQL default. `.defaultFn(fn)`, `.defaultNow()` and `v.uuid()` are runtime defaults applied only by `ctx.db.insert` (not by raw `ctx.$db`).
- **Auth tables** (`users`, `sessions`, `accounts`, `verifications`) come from Better Auth. Reference them (`v.one("users")`) but never define them.
- **SQL naming** is camelCase to snake_case automatically. Use `.sql("name")` or `table(shape, { sqlName })` only to match an existing database.

## Gotchas

- `v.number()` is an alias of `v.int()` and creates an INTEGER column. There is no float column type.
- `v.date()` is stored as epoch milliseconds, and filters accept `Date` or numbers.
- `.defaultNow()` doesn't add a SQL default, so rows inserted with raw SQL get `NULL`.
- Reciprocal `manyToMany` declarations with different `junctionTable` or field options fail generation.
- `onDelete: "set null"` requires `nullable: true` on the relation.
- `migrate` reads the compiled schema, so always run `bun appflare dev` first.
- Pass only one of `--local`, `--remote` or `--preview` to `migrate`.
- Adding a Better Auth plugin changes auth tables. Regenerate and migrate afterwards.

## References

- Read [references/column-api.md](references/column-api.md) when you need every `v.*` builder, option or modifier, or the SQL and TypeScript mapping.
- Read [references/relations.md](references/relations.md) when configuring relation options (custom FK names, reference fields, junction names, FK actions, self-relations) or debugging a relation compile error.

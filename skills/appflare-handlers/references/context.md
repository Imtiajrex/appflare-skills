# AppflareContext reference

`import type { AppflareContext } from "<relative>/_generated/handlers";`

| Field | Description |
| --- | --- |
| `ctx.db.<table>` | Typed API: `findMany`, `findFirst`, `iterate`, `insert`, `update`, `upsert`, `delete`, `count`, `sum`, `avg`, `min`, `max`, `groupBy` (see appflare-querying) |
| `ctx.db.batch(fn)` / `ctx.db.transaction(fn)` | Atomic multi-table writes |
| `ctx.$db` | Raw Drizzle D1 instance with the compiled and auth schemas. `ctx.$db.select().from(schema.posts)` or `ctx.$db.query.posts.findMany({...})` |
| `ctx.user` | Better Auth user plus `role`, `banned`, `banReason`, `banExpires` (admin plugin) and `user.additionalFields`. `null` when anonymous or in scheduler/cron |
| `ctx.session` | Better Auth session, or `null` |
| `ctx.auth` | Better Auth internal adapter, e.g. `await ctx.auth.listUsers()` |
| `ctx.storage` | `put({ path, body, contentType?, customMetadata? })`, `get({ path })`, `delete({ path })`, `list({ prefix?, cursor?, limit?, delimiter? })` |
| `ctx.scheduler.enqueue(task, payload?, { delaySeconds? })` | Send a queue message. Inside `ctx.db.transaction`, it is sent only after the commit |
| `ctx.env` | Worker bindings and vars (same object as `ctx.context.env`) |
| `ctx.context` | Hono context: `ctx.context.req.header("x")`, `ctx.context.req.raw.cf`. In scheduler/cron only `env` exists |
| `ctx.mutationEvents` | `{ kind, table, args, rows }[]` recorded by `ctx.db` writes after they commit |
| `ctx.error(status, message, details?)` | Throws. The response is `status` with `{ message, details }` |

## Response mapping

| Outcome | Status | Body |
| --- | --- | --- |
| return value | 200 | JSON of value |
| Zod validation failure | 400 | issues |
| `authRequired` and no user | 401 | `{ message: "Unauthorized" }` |
| `ctx.error(s, m, d)` | s | `{ message: m, details: d }` |
| UNIQUE violation | 409 | `{ code: "unique_violation", table, fields }` |
| `expectRows` guard failed | 409 | `{ code: "expect_rows_failed" }` |
| FOREIGN KEY violation | 409 | `{ code: "foreign_key_violation" }` |
| `RAISE(ABORT, 'reason')` trigger | 400 | `{ code: "trigger_abort", message: "reason" }` |
| CHECK violation | 400 | `{ code: "check_violation", constraint }` |
| NOT NULL violation | 400 | `{ code: "not_null_violation", table, fields }` |
| other throw | 500 | `{ message: "Internal error", requestId }` — details only in the logs |

Use `ctx.error` for expected failures; anything else is sanitized, so don't rely on a thrown `Error`'s message reaching the client. Catch database errors with `isDbError(error, "unique")` when you want to handle them yourself.

## Secrets and env

- `ctx.env.NAME` (or `ctx.context.env.NAME`) always works in request handlers.
- `process.env.NAME` works when `compatibility_flags` includes `nodejs_compat_populate_process_env`.
- Set secrets with `bunx wrangler secret put NAME` and non-secret vars in `wranglerOverrides.vars`.

## Typing helpers

```ts
import type { AppflareContext } from "../_generated/handlers";

export async function requireOwner(ctx: AppflareContext, projectId: string) {
	const project = await ctx.db.projects.findFirst({ where: { id: projectId } });
	if (!project || project.ownerId !== ctx.user?.id) ctx.error(404, "Project not found");
	return project;
}
```

Shared helpers like this can live under `scanDir`, because only `export const x = query(...)`-style exports are treated as handlers.

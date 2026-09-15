# AppflareContext reference

`import type { AppflareContext } from "<relative>/_generated/handlers";`

| Field | Description |
| --- | --- |
| `ctx.db.<table>` | Typed API: `findMany`, `findFirst`, `insert`, `update`, `upsert`, `delete`, `count`, `avg` (see appflare-querying) |
| `ctx.$db` | Raw Drizzle D1 instance with the compiled and auth schemas. `ctx.$db.select().from(schema.posts)` or `ctx.$db.query.posts.findMany({...})` |
| `ctx.user` | Better Auth user plus `role`, `banned`, `banReason`, `banExpires` (admin plugin) and `user.additionalFields`. `null` when anonymous or in scheduler/cron |
| `ctx.session` | Better Auth session, or `null` |
| `ctx.auth` | Better Auth internal adapter, e.g. `await ctx.auth.listUsers()` |
| `ctx.storage` | `put({ path, body, contentType?, customMetadata? })`, `get({ path })`, `delete({ path })`, `list({ prefix?, cursor?, limit?, delimiter? })` |
| `ctx.scheduler.enqueue(task, payload?, { delaySeconds? })` | Send a queue message for a scheduler task |
| `ctx.context` | Hono context: `ctx.context.env.SECRET`, `ctx.context.req.header("x")`, `ctx.context.req.raw.cf`. In scheduler/cron only `env` exists |
| `ctx.mutationEvents` | `{ kind, table, args, rows }[]` recorded by `ctx.db` writes |
| `ctx.error(status, message, details?)` | Throws. The response is `status` with `{ message, details }` |

## Response mapping

| Outcome | Status | Body |
| --- | --- | --- |
| return value | 200 | JSON of value |
| Zod validation failure | 400 | issues |
| `authRequired` and no user | 401 | `{ message: "Unauthorized" }` |
| `ctx.error(s, m, d)` | s | `{ message: m, details: d }` |
| other throw | 500 | `{ message }` |

## Secrets and env

- `ctx.context.env.NAME` always works in request handlers.
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

# appflare.config.ts reference

## Top level

| Key | Required | Default | Notes |
| --- | --- | --- | --- |
| `scanDir` | yes | | handler root |
| `exclude` | no | `[]` | globs (relative to `scanDir`) skipped by discovery; `*.test.*`, `*.spec.*` and `__tests__/` always are |
| `outDir` | yes | | generated output |
| `schema` | yes (≥1) | | drizzle-kit schema modules |
| `schemaDsl` | no | | `{ entry, exportName?, outFile?, typesOutFile?, zodOutFile?, namingStrategy?: "camelToSnake" }` |
| `database` | yes | | object or array: `{ binding, databaseName, databaseId, previewDatabaseId?, migrationsDir?, query?: { defaultLimit, maxLimit } }` |
| `kv` | no | `[]` | `{ binding, id, previewId? }` |
| `r2` | no | `[]` | `{ binding, bucketName, previewBucketName?, jurisdiction? }` |
| `auth` | yes | | `{ enabled, basePath, options: BetterAuthOptions, clientOptions }` |
| `scheduler` | no | `{ enabled: true, binding: "APPFLARE_SCHEDULER_QUEUE" }` | `queue` defaults to `<worker name>-scheduler` |
| `realtime` | no | enabled | `{ enabled, binding: "APPFLARE_REALTIME", className: "AppflareRealtimeDurableObject", objectName: "global", subscribePath: "/realtime/subscribe", websocketPath: "/realtime/ws", protocol: "appflare.realtime.v1" }` |
| `wranglerOutDir` / `wranglerOutPath` | no | `outDir` | where `wrangler.json` goes |
| `wranglerOverrides` | no | | deep merge (arrays replace) |
| `build` | no | `true` | run `tsc --build` if `tsconfig.json` exists |

## Values read at build time from `auth.options`

- `plugins` containing `admin({ roles: {...} })` → `UserRole` union, typed `ctx.user.role`, patched `role` column type
- `user.additionalFields` (`string`, `number`, `boolean`, `date`) → extra `ctx.user` fields

## Generated `wrangler.json`

| Key | Source |
| --- | --- |
| `name` | `wranglerOverrides.name` or `appflare-worker` |
| `main` | `./src/index.ts`, so override it |
| `d1_databases[]` | `database` (`preview_database_id` defaults to `databaseId`) |
| `kv_namespaces[]` | `kv` |
| `r2_buckets[]` | `r2` |
| `queues.producers` / `queues.consumers` | `scheduler`, when scheduler or cron handlers exist |
| `triggers.crons` | all `cron` handler expressions |
| `durable_objects.bindings` + `migrations` (`appflare-realtime-v1`) | `realtime.enabled` |

## Generated files in `outDir`

`server.js` (exports `fetch`, `queue`, `scheduled`, `AppflareRealtimeDurableObject`), `handlers.js`, `handlers.context.js`, `handlers.execution.js`, `handlers.routes.js`, `client.js`, `client/**`, `schema.compiled.js`, `schema.types.js`, `schema.zod.js`, `auth.config.js`, `auth.schema.js`, `drizzle.config.js` and `admin.routes.js`, each with a `.d.ts`.

## Runtime HTTP surface

| Path | Purpose |
| --- | --- |
| `GET /queries/**` | queries |
| `POST /mutations/**` | mutations |
| `GET, POST <auth.basePath>/*` | Better Auth |
| `/storage/upload`, `/storage/download`, `/storage/object`, `/storage/list` | R2 |
| `POST /realtime/subscribe`, `POST /realtime/unsubscribe`, `GET /realtime/ws` | realtime |
| `/admin` | dashboard (role `admin`) |

## Env vars

| Var | Effect |
| --- | --- |
| `ALLOWED_DOMAINS` | comma-separated CORS allow-list. `*` or unset allows all |
| `BETTER_AUTH_SECRET` | Better Auth secret (use `wrangler secret put`) |

## CLI errors

| Message | Cause |
| --- | --- |
| `Appflare CLI must be run with Bun.` | ran under Node |
| `Only one of --local, --remote, or --preview can be set.` | multiple target flags |
| `better-auth generation failed` | invalid `auth.options`, or `@better-auth/cli` isn't installed |
| `drizzle-kit generate failed` | schema modules not generated, or a drizzle-kit prompt was interrupted |
| `TypeScript build failed` | type errors in your package (use `--no-build` to isolate) |
| `Duplicate handler operation discovered` | two handlers resolve to the same route or task name |
| `Relation '…' is ambiguous` | several `v.one` relations to the same table; give both sides a `relationName` |
| `Migration name may only contain letters, numbers, - and _` | bad `--name` for `migrate:custom` |

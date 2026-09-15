# Scheduler tasks and cron jobs

## Scheduler (Cloudflare Queues)

```ts
// src/billing/invoices.ts → task name "billing/invoices/generate"
import * as z from "zod";
import { scheduler } from "../../_generated/handlers";

export const generate = scheduler({
	args: { orderId: z.string() },   // optional; payload validated when the task runs
	handler: async (ctx, args) => {
		const order = await ctx.db.orders.findFirst({ where: { id: args.orderId } });
		if (!order) return;
		// ...
	},
});
```

Enqueue it from any handler:

```ts
await ctx.scheduler.enqueue("billing/invoices/generate", { orderId });
await ctx.scheduler.enqueue("billing/invoices/generate", { orderId }, { delaySeconds: 300 });
```

### Task name rule

`<last directory>/<file name without extension>/<export>`. Files directly in `scanDir` use `root`, and a leading `schedulers/` or `crons/` segment is stripped.

| File | Export | Name |
| --- | --- | --- |
| `src/ball.ts` | `send` | `root/ball/send` |
| `src/bun/test.ts` | `sendEmail` | `bun/test/sendEmail` |
| `src/doctor/certificates/create.ts` | `generateCertificate` | `certificates/create/generateCertificate` |

### Semantics

- One context per queue batch, and messages run sequentially.
- `ctx.user` and `ctx.session` are `null`, and `ctx.context` has only `env`.
- Invalid payloads (Zod) and thrown errors are logged and the message is **not retried**. Make tasks idempotent.
- `ctx.db` writes publish realtime updates after each task.
- `ctx.storage` still runs `storageManager` checks with `ctx.user === null`, so allow that case in your rules if tasks use storage.

### Wiring

- Config: `scheduler: { enabled: true, binding: "MY_QUEUE_BINDING", queue: "my-queue" }` (defaults: enabled, `APPFLARE_SCHEDULER_QUEUE`, `<worker>-scheduler`).
- Queue producer and consumer entries appear in `wrangler.json` only when at least one scheduler or cron handler exists.
- Create the queue first: `bunx wrangler queues create my-queue`.
- `Scheduler queue binding is not configured` means the binding is missing from `wrangler.json`, or it wasn't regenerated or deployed.

## Cron (Cron Triggers)

```ts
import { cron } from "../../_generated/handlers";

export const cleanup = cron({
	cronTrigger: ["0 3 * * *", "0 15 * * *"],
	handler: async (ctx) => {
		await ctx.db.sessions.delete({ where: { expiresAt: { lt: Date.now() } } });
	},
});
```

- `cronTrigger` must be a **string literal** or an **array of string literals**, because it is parsed from source.
- Expressions are deduplicated into `triggers.crons`. Every handler listing the fired expression runs.
- The handler has no args and the same context rules as scheduler tasks.

### Local testing

```bash
bunx wrangler dev --test-scheduled
curl "http://localhost:8787/__scheduled?cron=0+3+*+*+*"
```

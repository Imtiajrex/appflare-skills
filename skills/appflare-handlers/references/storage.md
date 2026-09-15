# Storage (R2) reference

Requires `r2: { binding, bucketName }` in `appflare.config.ts`. The first bucket is used.

## Authorization: `storageManager`

```ts
// src/storage.ts
import { storageManager } from "../_generated/handlers";

export const storageRules = storageManager({
	handler: async (ctx, { path, method }) => {
		if (path.startsWith("/public/") && (method === "get" || method === "list")) return true;
		if (ctx.user && path.startsWith(`/users/${ctx.user.id}/`)) return true;
		if (ctx.user?.role === "admin") return true;
		return false;
	},
});
```

- `path` always starts with `/`. For `list`, it is `/` + prefix.
- `method` is one of `put`, `get`, `delete`, `list`, `download` or `preview`. HTTP downloads check `get`.
- Access is allowed if **any** manager returns `true`. With **no** managers everything is denied (403 `Storage access denied`).
- The checks also run for `ctx.storage` calls inside handlers, using that request's `ctx.user`.

## Server API

```ts
await ctx.storage.put({ path: `users/${ctx.user.id}/a.png`, body, contentType: "image/png", customMetadata: {} });
const obj = await ctx.storage.get({ path: "public/a.png" });   // R2 object with body, or null
await ctx.storage.delete({ path: `users/${ctx.user.id}/a.png` });
const list = await ctx.storage.list({ prefix: "public/", limit: 100, cursor, delimiter: "/" });
```

`body` can be a `ReadableStream`, `ArrayBuffer`, `ArrayBufferView`, `string` or `Blob`. Leading slashes are stripped before the R2 call.

## HTTP routes (generated)

| Route | Input | Output |
| --- | --- | --- |
| `POST /storage/upload` | JSON `{ path, contentType?, base64Body }` | `{ path, method: "PUT", uploaded: true }` |
| `GET /storage/download` | `?path=&fileName=` | streamed file (attachment when `fileName` is set) |
| `DELETE /storage/object` | `?path=` | `{ ok: true, path }` |
| `GET /storage/list` | `?prefix=&cursor=&limit=&delimiter=` | R2 list JSON |
| `GET /storage/preview` | | **501**, not implemented |

## Recommended patterns

- **Large or binary uploads:** accept the data in a mutation and stream it into `ctx.storage.put`, or keep uploads small. The client upload base64-encodes files into JSON.
- **Public images:** allow anonymous `get` for a `/public/` prefix and use `appflare.storage.preview({ path })` as the `<img src>`.
- **Private files:** `fetch(appflare.storage.preview({ path }), { headers: { authorization: "Bearer …" } })` and then create a blob URL.
- Keep object keys tied to ids (`users/<userId>/…`) so manager rules stay simple.

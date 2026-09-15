# Auth and storage from the client

## Server prerequisites (`appflare.config.ts` → `auth.options`)

- `emailAndPassword: { enabled: true }` for email/password sign-in
- `plugins: [bearer()]`, which is required for token auth through `onSetAuthToken`
- `trustedOrigins: [...]`, containing your web origins and app schemes
- optional: `admin()` for roles, `emailOTP({...})`, `expo()` for Expo
- the `BETTER_AUTH_SECRET` Worker secret

## Client flows

```ts
// sign up / sign in
const { error } = await appflare.auth.signUp.email({ email, password, name });
await appflare.auth.signIn.email({ email, password });

// current session
const { data: session } = await appflare.auth.getSession();
session?.user.id;

// sign out
await appflare.auth.signOut();
localStorage.removeItem("appflare-token");
```

The token is captured from the `set-auth-token` response header and passed to `onSetAuthToken`. Every later auth, query, mutation, storage and realtime request sends `Authorization: Bearer <onGetAuthToken()>`.

## Client plugins

`auth.clientOptions` in the backend config supplies **types** only. Register the same plugins at runtime:

```ts
import { adminClient, emailOTPClient } from "better-auth/client/plugins";

new Appflare({
	endpoint,
	authOptions: { plugins: [emailOTPClient(), adminClient()] },
	onGetAuthToken,
	onSetAuthToken,
});

await appflare.auth.emailOtp.sendVerificationOtp({ email, type: "sign-in" });
```

## Role-aware UI

`session.user.role` exists when the admin plugin is enabled. Hide UI based on it, but always enforce roles on the server with `authRequired` and `middleware`.

## Cookies vs bearer

If a request carries cookies, the server ignores the `Authorization` header on auth routes. Use bearer tokens for mobile and cross-origin apps.

## Storage from the client

```ts
await appflare.storage.upload({ path: `users/${userId}/avatar.png`, body: file, contentType: file.type });
const url = appflare.storage.preview({ path: `public/logo.png` }); // GET /storage/download?path=…
await appflare.storage.delete({ path: `users/${userId}/avatar.png` });
const listing = await appflare.storage.list({ prefix: `users/${userId}/` });
```

- `upload` base64-encodes a `File` into JSON, so keep files small. A `string` body must be Latin-1 (it uses `btoa`); otherwise pass `base64Body`.
- `preview()` is a plain URL, so `<img src>` works only for paths the storage rules allow anonymously. For private files, `fetch` it with the bearer header.
- Don't rely on `download()`, which parses JSON while the route streams the file.
- 403 `Storage access denied` means the backend's `storageManager` rules rejected the path or method.

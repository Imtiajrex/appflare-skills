---
name: appflare-client
description: Use the Appflare generated client in frontends, including creating new Appflare() with endpoint, wsEndpoint and bearer-token storage, calling appflare.queries/appflare.mutations with .run() and { data, error }, the React and React Native hooks useQuery, useInfiniteQuery and useMutation from appflare/react with TanStack Query, realtime subscriptions, Better Auth sign-up and sign-in through appflare.auth, and uploads through appflare.storage. Use when wiring a web or mobile app to an Appflare backend, fetching or mutating data from UI code, adding live updates, or implementing login.
metadata:
  author: appflare
  version: "0.3.0"
---

# Appflare client

## Workflow

- [ ] Find the generated client: `<backend package>/_generated/client`, or `dist/_generated/client` when the backend builds with `tsc`
- [ ] Create **one** shared client module (e.g. `lib/appflare.ts`)
- [ ] Wrap the React tree in `QueryClientProvider`
- [ ] Call routes through `appflare.queries.*` and `appflare.mutations.*`. Names mirror handler file paths
- [ ] After backend changes, run `bun appflare dev` in the backend so the client types update

## Shared client

```ts
// lib/appflare.ts
import { Appflare } from "my-backend/_generated/client";

export const appflare = new Appflare({
	endpoint: import.meta.env.VITE_API_URL,          // e.g. http://localhost:8787
	wsEndpoint: import.meta.env.VITE_API_WS_URL,     // e.g. ws://localhost:8787 (needed for realtime)
	onGetAuthToken: () => localStorage.getItem("appflare-token") ?? "",
	onSetAuthToken: (token) => localStorage.setItem("appflare-token", token),
});
```

For React Native, store the token with AsyncStorage or SecureStore. On the Android emulator, use `http://10.0.2.2:8787` instead of localhost.

## Plain calls

```ts
const { data, error } = await appflare.queries.tasks.listTasks.run({ projectId, limit: 20 });
if (error) throw new Error(`${error.status} ${error.message}`);

await appflare.mutations.tasks.completeTask.run({ id: 42 });
```

## React

```tsx
import { useQueryClient } from "@tanstack/react-query";
import { useMutation, useQuery } from "appflare/react";
import { appflare } from "../lib/appflare";

export function Tasks({ projectId }: { projectId: string }) {
	const queryClient = useQueryClient();
	const tasks = useQuery(appflare.queries.tasks.listTasks, { projectId }, {
		realtime: { enabled: true },
	});
	const complete = useMutation(appflare.mutations.tasks.completeTask, {
		onSuccess: () =>
			queryClient.invalidateQueries({ queryKey: appflare.queries.tasks.listTasks.queryKey() }),
	});

	if (tasks.isLoading) return <p>Loading…</p>;
	if (tasks.error) return <p>{tasks.error.message}</p>;
	return tasks.data?.map((t) => (
		<button key={t.id} onClick={() => complete.mutate({ id: t.id })}>{t.title}</button>
	));
}
```

## Auth

```ts
await appflare.auth.signUp.email({ email, password, name });
await appflare.auth.signIn.email({ email, password }); // token stored via onSetAuthToken
await appflare.auth.signOut();                         // then clear your stored token
```

The server needs Better Auth's `bearer()` plugin so responses include `set-auth-token`.

## Gotchas

- `run()` returns `{ data, error }` for HTTP failures and doesn't throw. It **does** throw a `ZodError` if the args fail the route schema on the client.
- Hooks throw `Error` objects that carry a `status` property, so use TanStack's `error` state.
- A `Date` returned by a handler arrives as an ISO **string**, even though the type says `Date`.
- Query args go in the URL, and the server converts them to the handler's arg types, so numbers, booleans, arrays and objects work without client-side tricks.
- A route whose handler takes no args (`args: {}`) is callable with no arguments: `route.run()` and `useQuery(route)`.
- Unhandled server errors return `{ message: "Internal error", requestId }`. Show `error.message` for expected failures (`ctx.error`) and log the `requestId` otherwise. Constraint failures carry a `code` such as `unique_violation` in `error.body`.
- Realtime needs `wsEndpoint`. `subscribe()` only pushes **changes**; hooks load initial data for you.
- `useInfiniteQuery` realtime pushes replace only the **first** page.
- `route.queryKey()` with no args matches all cached variants, which makes it a good key for invalidation.
- `clientOptions` in `appflare.config.ts` only types `appflare.auth`. Pass runtime Better Auth client plugins through `new Appflare({ authOptions: { plugins: [...] } })`.
- `appflare.storage.download()` expects JSON but the route streams bytes. Use `storage.preview({ path })` as a URL instead.

## References

- Read [references/react-hooks.md](references/react-hooks.md) when you need hook options (queryOptions, requestOptions, realtime callbacks), infinite pagination, or client types such as `InferRouteInput` and `InferRouteOutput`.
- Read [references/auth.md](references/auth.md) when implementing sign-in or sign-up, OTP or admin client plugins, token storage, role-aware UI, or storage uploads from the client.

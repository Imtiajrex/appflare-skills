# Client and hooks reference

## `new Appflare(options)`

| Option | Description |
| --- | --- |
| `endpoint` (required) | Worker URL |
| `wsEndpoint` | WebSocket base for realtime (defaults to `endpoint`) |
| `authPath` | default `/api/auth` |
| `authOptions` | Better Auth client options (runtime plugins) |
| `fetch` | custom fetch for auth |
| `requestOptions` | default `{ headers, signal, onError }` for routes |
| `onGetAuthToken()` | returns the bearer token (string or Promise) |
| `onSetAuthToken(token)` | called when a `set-auth-token` header arrives |

Instance members: `queries`, `mutations`, `auth`, `storage`, `endpoint`, `wsEndpoint`.

## Route objects

```ts
const route = appflare.queries.tasks.listTasks;
await route.run(args?, { headers?, signal?, onError? }); // { data, error }
route.schema;                    // z.object of handler args
route.queryKey(args?);           // ["appflare", "query", "/queries/tasks/listTasks", args]
route.subscribe({ args, onChange, onError?, authToken?, signal? }); // { remove() }
```

Mutations have only `run` and `schema`. The `error` object has `status`, `message`, `body`, `route`, `method` and `responseText`.

## Types

```ts
import type { InferRouteInput, InferRouteOutput } from "my-backend/_generated/client";
type Args = InferRouteInput<typeof appflare.queries.tasks.listTasks>;
type Result = InferRouteOutput<typeof appflare.queries.tasks.listTasks>;
```

## `useQuery(route, args?, options?)`

| Option | Description |
| --- | --- |
| `queryOptions` | TanStack `useQuery` options except `queryFn` (you may set `queryKey`, `enabled`, `staleTime`, `select`…) |
| `requestOptions` | per-call headers, signal, onError |
| `realtime.enabled` | subscribe and write pushes into the cache |
| `realtime.authToken` / `realtime.requestOptions` | subscription auth |
| `realtime.onChange(data, update)` / `realtime.onError(err)` | callbacks |

To skip a query conditionally, use `queryOptions: { enabled: Boolean(projectId) }`. Pass args that satisfy the schema even when the query is disabled.

## `useMutation(route, mutationOptions?)`

This is TanStack `useMutation` without `mutationFn`. Call `mutate(args)`, or `mutate()` when the mutation has no required args. `mutateAsync` resolves to the data and throws an `Error` with `status`.

Extra options:

| Option | Description |
| --- | --- |
| `optimistic(args, cache)` | instant cache changes, rolled back on error, kept across refetches while pending |
| `updateCache(result, args, cache)` | write the server response into the cache on success |
| `invalidates: [route, ...]` | refetch these routes on success |
| `reconcile` | default `true`: refetch queries `optimistic` touched once settled |

```tsx
const toggle = useMutation(appflare.mutations.tasks.toggleTask, {
	optimistic: ({ id }, cache) =>
		cache.update(appflare.queries.tasks.listTasks, (tasks) =>
			tasks.map((t) => (t.id === id ? { ...t, done: !t.done } : t)),
		),
});
```

Updaters must be pure (they re-run on fresh server data while the mutation is pending) and return the query's own result type.

## `useAppflareCache()` / `createAppflareCache(queryClient)`

Typed cache access by route. Omitting `args` targets every cached variant; updaters skip queries without data.

```ts
cache.get(route, args?)                         // useQuery data | undefined
cache.set(route, args, data)
cache.update(route, args?, (data) => data)      // useQuery results
cache.getInfinite(route, args?)                 // InfiniteData | undefined
cache.updateInfinite(route, args?, (data) => data)
cache.updatePages(route, args?, (page, index) => page)
cache.invalidate(route, args?)                  // both kinds
const tx = cache.optimistic((c) => c.update(...)); tx.commit({ reconcile: true }) / tx.rollback()
```

Direct `cache` writes are permanent. Results cached under a custom `queryOptions.queryKey` aren't reachable.

## `useInfiniteQuery(route, args?, options?)`

```tsx
const feed = useInfiniteQuery(appflare.queries.tasks.listTasksPage, { pageSize: 20 }, {
	pageParamToArgs: (args, cursor: number | undefined) => ({ ...args, cursor }),
	queryOptions: {
		initialPageParam: undefined,
		getNextPageParam: (last) => (last.hasMore ? last.nextCursor : undefined),
	},
});
const rows = feed.data?.pages.flatMap((p) => p.rows) ?? [];
```

The handler should return `{ rows, nextCursor, hasMore }` (see appflare-querying pagination). Realtime pushes replace page 1 only. The cache key is `[...route.queryKey(args), "infinite"]`.

## Provider and local-first caching

```tsx
import { AppflareQueryProvider, createAppflareQueryClient } from "appflare/react";

const queryClient = createAppflareQueryClient(); // offlineFirst queries, pausing mutations, 24h gcTime
<AppflareQueryProvider client={queryClient} persist={{ storage: localStorage, buster: "v1" }}>
	<App />
</AppflareQueryProvider>
```

`persist` saves appflare route results (server-confirmed data only) and restores them at startup. Options: `storage` (sync or async, e.g. `AsyncStorage`), `key`, `maxAge` (24h), `buster`, `throttleMs`, `shouldPersist`, `serialize`/`deserialize`. A plain `QueryClientProvider` still works. On React Native, wire `onlineManager` to NetInfo so paused mutations resume.

Install with `bun add @tanstack/react-query react`.

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

The handler should return `{ rows, nextCursor, hasMore }` (see appflare-querying pagination). Realtime pushes replace page 1 only.

## Provider

```tsx
const queryClient = new QueryClient();
<QueryClientProvider client={queryClient}><App /></QueryClientProvider>
```

Install with `bun add @tanstack/react-query react`.

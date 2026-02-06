# Convex + TanStack Query Integration Fix

## Problem Description

The application was experiencing the following error when using `convexQuery` in routes:

```
Missing queryFn: '["convexQuery","todos:list",{}]'
```

This error occurred because `ConvexQueryClient` was created in `src/integrations/convex/provider.tsx`, but it was **never connected to the TanStack Query `QueryClient`** that routes/loaders use.

## Root Cause Analysis

### What Was Missing

1. **Missing Package**: The `@tanstack/react-router-with-query` package was not installed, which is required for integrating TanStack Query with TanStack Router.

2. **Improper Wiring**: The `ConvexQueryClient` was created but never connected to a `QueryClient` instance:
   - `src/integrations/convex/provider.tsx` created a `ConvexQueryClient`
   - No `QueryClient` was created with the proper `queryFn` and `queryKeyHashFn` from `convexQueryClient`
   - The `convexQueryClient.connect(queryClient)` method was never called

3. **Incorrect Router Context**: The router context only included `queryClient`, not `convexQueryClient`.

4. **Duplicate Provider Setup**: `ConvexProvider` was being used in `RootDocument` instead of the router's `Wrap` component.

### Why This Caused the Error

`convexQuery(...)` only works if:
1. A `ConvexQueryClient` is created
2. A `QueryClient` is created with `queryFn` and `queryKeyHashFn` from that `ConvexQueryClient`
3. The `convexQueryClient` is connected to the `queryClient`
4. Both clients are available in the router context

Without this wiring, TanStack Query doesn't know how to handle Convex query keys like `["convexQuery","todos:list",{}]`, resulting in the "Missing queryFn" error.

## Solution Implemented

### Step 1: Install Missing Dependencies

```bash
npm install @tanstack/react-router-with-query @tanstack/react-query
```

### Step 2: Update Router Configuration

**File**: `src/router.tsx`

Replaced the router setup with proper Convex + TanStack Query wiring:

```typescript
import { createRouter } from "@tanstack/react-router";
import { QueryClient } from "@tanstack/react-query";
import { routerWithQueryClient } from "@tanstack/react-router-with-query";
import { ConvexQueryClient } from "@convex-dev/react-query";
import { ConvexProvider } from "convex/react";
import { routeTree } from "./routeTree.gen";

export function getRouter() {
  const CONVEX_URL = (import.meta as any).env.VITE_CONVEX_URL!;
  if (!CONVEX_URL) {
    console.error("missing envar VITE_CONVEX_URL");
  }

  const convexQueryClient = new ConvexQueryClient(CONVEX_URL);

  const queryClient: QueryClient = new QueryClient({
    defaultOptions: {
      queries: {
        queryKeyHashFn: convexQueryClient.hashFn(),
        queryFn: convexQueryClient.queryFn(),
      },
    },
  });
  convexQueryClient.connect(queryClient);

  const router = routerWithQueryClient(
    createRouter({
      routeTree,
      defaultPreload: "intent",
      context: { queryClient, convexQueryClient },
      scrollRestoration: true,
      Wrap: ({ children }) => (
        <ConvexProvider client={convexQueryClient.convexClient}>
          {children}
        </ConvexProvider>
      ),
    }),
    queryClient,
  );

  return router;
}
```

**Key Changes**:
- Created `ConvexQueryClient` instance with `VITE_CONVEX_URL`
- Configured `QueryClient` with `queryKeyHashFn` and `queryFn` from `convexQueryClient`
- Connected `convexQueryClient` to `queryClient` using `.connect()`
- Used `routerWithQueryClient` to wire the router with the query client
- Added `Wrap` component to provide `ConvexProvider` with `convexClient`
- Included both `queryClient` and `convexQueryClient` in the router context

### Step 3: Update Root Route Context

**File**: `src/routes/__root.tsx`

Updated the route context to match the router:

```typescript
import type { QueryClient } from "@tanstack/react-query";
import type { ConvexQueryClient } from "@convex-dev/react-query";

interface MyRouterContext {
  queryClient: QueryClient;
  convexQueryClient: ConvexQueryClient;
}

export const Route = createRootRouteWithContext<MyRouterContext>()({
  // ... rest of route configuration
});
```

Removed the `ConvexProvider` wrapper from `RootDocument` since the router `Wrap` already provides it.

### Step 4: Remove Obsolete File

**File**: `src/integrations/convex/provider.tsx`

Deleted this file entirely since `ConvexQueryClient` is now created and wired directly in the router.

### Step 5: Verify Route Usage

Routes can now safely use `convexQuery`:

```typescript
import { createFileRoute } from "@tanstack/react-router";
import { useSuspenseQuery } from "@tanstack/react-query";
import { convexQuery } from "@convex-dev/react-query";
import { api } from "@/convex/_generated/api";

export const Route = createFileRoute("/test2")({
  loader: async ({ context }) => {
    await context.queryClient.ensureQueryData(
      convexQuery(api.todos.list, {})
    );
    return null;
  },
  component: RouteComponent,
});

function RouteComponent() {
  const { data } = useSuspenseQuery(
    convexQuery(api.todos.list, {})
  );

  return <pre>{JSON.stringify(data, null, 2)}</pre>;
}
```

## Verification

After implementing these changes:
- The "Missing queryFn" error was resolved
- Convex queries now work correctly in routes
- Data is successfully fetched from the Convex backend
- The application can display todo items from the database

## Key Takeaways

1. **Always connect ConvexQueryClient to QueryClient**: The `.connect()` method is essential for the integration to work.

2. **Use routerWithQueryClient**: This is the proper way to integrate TanStack Query with TanStack Router.

3. **Provide both clients in context**: Routes need access to both `queryClient` and `convexQueryClient`.

4. **Use the router's Wrap for providers**: The `Wrap` component in the router configuration is the correct place to provide `ConvexProvider`.

5. **Follow official quickstarts**: The Convex + TanStack Start quickstart provides the correct pattern for this integration.

## References

- [Convex + TanStack Query Setup](https://docs.convex.dev/client/tanstack/tanstack-query/#setup)
- [TanStack Start Quickstart](https://docs.convex.dev/quickstart/tanstack-start)
- [Convex + TanStack Query Queries](https://docs.convex.dev/client/tanstack/tanstack-query/#queries)
- [TanStack Start SSR with convexQuery](https://docs.convex.dev/client/tanstack/tanstack-start/#server-side-rendering)

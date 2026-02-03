# Convex Next.js App Router Reference

## Setup

### ConvexClientProvider

Create a client provider for the App Router:

```typescript
// app/ConvexClientProvider.tsx
"use client";

import { ConvexProvider, ConvexReactClient } from "convex/react";
import { ReactNode } from "react";

const convex = new ConvexReactClient(process.env.NEXT_PUBLIC_CONVEX_URL!);

export function ConvexClientProvider({ children }: { children: ReactNode }) {
  return <ConvexProvider client={convex}>{children}</ConvexProvider>;
}
```

Wrap your app in the root layout:

```typescript
// app/layout.tsx
import { ConvexClientProvider } from "./ConvexClientProvider";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <ConvexClientProvider>{children}</ConvexClientProvider>
      </body>
    </html>
  );
}
```

---

## Server Rendering

### Preloading Data for Client Components

Use `preloadQuery` in Server Components + `usePreloadedQuery` in Client Components for SSR with reactivity:

```typescript
// app/TasksWrapper.tsx (Server Component)
import { preloadQuery } from "convex/nextjs";
import { api } from "@/convex/_generated/api";
import { Tasks } from "./Tasks";

export async function TasksWrapper() {
  const preloadedTasks = await preloadQuery(api.tasks.list, {
    list: "default",
  });
  return <Tasks preloadedTasks={preloadedTasks} />;
}
```

```typescript
// app/Tasks.tsx (Client Component)
"use client";

import { Preloaded, usePreloadedQuery } from "convex/react";
import { api } from "@/convex/_generated/api";

export function Tasks(props: {
  preloadedTasks: Preloaded<typeof api.tasks.list>;
}) {
  const tasks = usePreloadedQuery(props.preloadedTasks);
  return (
    <ul>
      {tasks.map((task) => (
        <li key={task._id}>{task.text}</li>
      ))}
    </ul>
  );
}
```

### Accessing Preloaded Query Result

Use `preloadedQueryResult` to access the data in Server Components:

```typescript
import { preloadQuery, preloadedQueryResult } from "convex/nextjs";
import { api } from "@/convex/_generated/api";

export async function ConditionalRender() {
  const preloaded = await preloadQuery(api.tasks.list, { list: "default" });
  const tasks = preloadedQueryResult(preloaded);

  if (tasks.length === 0) {
    return <EmptyState />;
  }

  return <Tasks preloadedTasks={preloaded} />;
}
```

### Non-Reactive Server Components

Use `fetchQuery` for static server-rendered content (no reactivity):

```typescript
// app/StaticTasks.tsx (Server Component)
import { fetchQuery } from "convex/nextjs";
import { api } from "@/convex/_generated/api";

export async function StaticTasks() {
  const tasks = await fetchQuery(api.tasks.list, { list: "default" });
  return (
    <ul>
      {tasks.map((task) => (
        <li key={task._id}>{task.text}</li>
      ))}
    </ul>
  );
}
```

---

## Server Actions and Route Handlers

### Server Actions

Use `fetchMutation` in Server Actions:

```typescript
// app/example/page.tsx
import { api } from "@/convex/_generated/api";
import { fetchMutation, fetchQuery } from "convex/nextjs";
import { revalidatePath } from "next/cache";

export default async function TasksPage() {
  const tasks = await fetchQuery(api.tasks.list, { list: "default" });

  async function createTask(formData: FormData) {
    "use server";
    await fetchMutation(api.tasks.create, {
      text: formData.get("text") as string,
    });
    revalidatePath("/example");
  }

  return (
    <form action={createTask}>
      <input name="text" placeholder="New task" />
      <button type="submit">Add</button>
    </form>
  );
}
```

### Route Handlers

Use `fetchQuery`, `fetchMutation`, `fetchAction` in Route Handlers:

```typescript
// app/api/tasks/route.ts
import { NextResponse } from 'next/server';
import { api } from '@/convex/_generated/api';
import { fetchQuery, fetchMutation } from 'convex/nextjs';

export async function GET() {
  const tasks = await fetchQuery(api.tasks.list, { list: 'default' });
  return NextResponse.json(tasks);
}

export async function POST(request: Request) {
  const { text } = await request.json();
  const id = await fetchMutation(api.tasks.create, { text });
  return NextResponse.json({ id });
}
```

---

## Server-Side Authentication

### Auth Token Helper

Create a reusable auth token helper:

```typescript
// app/auth.ts

// For Clerk
import { auth } from '@clerk/nextjs/server';

export async function getAuthToken() {
  const authResult = await auth();
  return (await authResult.getToken({ template: 'convex' })) ?? undefined;
}

// For Auth0
// import { getSession } from '@auth0/nextjs-auth0';
// export async function getAuthToken() {
//   const session = await getSession();
//   return session?.tokenSet.idToken;
// }
```

### Authenticated Preloading

Pass the token to `preloadQuery`:

```typescript
// app/AuthenticatedTasks.tsx
import { preloadQuery } from "convex/nextjs";
import { api } from "@/convex/_generated/api";
import { getAuthToken } from "./auth";
import { Tasks } from "./Tasks";

export async function AuthenticatedTasks() {
  const token = await getAuthToken();
  const preloadedTasks = await preloadQuery(
    api.tasks.list,
    { list: "default" },
    { token }
  );
  return <Tasks preloadedTasks={preloadedTasks} />;
}
```

### Authenticated Server Actions

```typescript
import { fetchMutation } from 'convex/nextjs';
import { getAuthToken } from './auth';

async function createTask(formData: FormData) {
  'use server';
  const token = await getAuthToken();
  await fetchMutation(
    api.tasks.create,
    { text: formData.get('text') as string },
    { token }
  );
}
```

---

## Authentication Providers

### Clerk Setup (Client + Server)

```typescript
// app/ConvexClientProvider.tsx
"use client";

import { ClerkProvider, useAuth } from "@clerk/nextjs";
import { ConvexProviderWithClerk } from "convex/react-clerk";
import { ConvexReactClient } from "convex/react";
import { ReactNode } from "react";

const convex = new ConvexReactClient(process.env.NEXT_PUBLIC_CONVEX_URL!);

export function ConvexClientProvider({ children }: { children: ReactNode }) {
  return (
    <ClerkProvider>
      <ConvexProviderWithClerk client={convex} useAuth={useAuth}>
        {children}
      </ConvexProviderWithClerk>
    </ClerkProvider>
  );
}
```

### Auth0 Setup (Client Only)

```typescript
// app/ConvexClientProvider.tsx
"use client";

import { Auth0Provider } from "@auth0/auth0-react";
import { ConvexReactClient } from "convex/react";
import { ConvexProviderWithAuth0 } from "convex/react-auth0";
import { ReactNode } from "react";

const convex = new ConvexReactClient(process.env.NEXT_PUBLIC_CONVEX_URL!);

export function ConvexClientProvider({ children }: { children: ReactNode }) {
  return (
    <Auth0Provider
      domain={process.env.NEXT_PUBLIC_AUTH0_DOMAIN!}
      clientId={process.env.NEXT_PUBLIC_AUTH0_CLIENT_ID!}
      authorizationParams={{
        redirect_uri:
          typeof window === "undefined" ? undefined : window.location.origin,
      }}
      useRefreshTokens={true}
      cacheLocation="localstorage"
    >
      <ConvexProviderWithAuth0 client={convex}>
        {children}
      </ConvexProviderWithAuth0>
    </Auth0Provider>
  );
}
```

### Auth State Components

```typescript
import { Authenticated, Unauthenticated, AuthLoading } from "convex/react";

function App() {
  return (
    <>
      <AuthLoading>Loading...</AuthLoading>
      <Unauthenticated>Please log in</Unauthenticated>
      <Authenticated>
        <AuthenticatedContent />
      </Authenticated>
    </>
  );
}
```

---

## Configuration

### Environment Variables

```bash
# .env.local
NEXT_PUBLIC_CONVEX_URL=https://your-deployment.convex.cloud

# For Clerk
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...
CLERK_SECRET_KEY=sk_...

# For Auth0
NEXT_PUBLIC_AUTH0_DOMAIN=your-domain.auth0.com
NEXT_PUBLIC_AUTH0_CLIENT_ID=your-client-id
```

### Custom Deployment URL

Pass URL directly to server functions:

```typescript
const tasks = await fetchQuery(
  api.tasks.list,
  { list: 'default' },
  { url: 'https://your-deployment.convex.cloud' }
);
```

---

## Important Considerations

### Consistency

`preloadQuery` and `fetchQuery` use stateless HTTP requests. Multiple calls are **not guaranteed to return consistent data** from the same database snapshot.

```typescript
// ⚠️ May see inconsistent data between these two calls
const users = await preloadQuery(api.users.list, {});
const posts = await preloadQuery(api.posts.list, {});

// ✅ Better: Single query that returns all needed data
const data = await preloadQuery(api.dashboard.getData, {});
```

### Caching

`preloadQuery` uses `cache: 'no-store'` by default — Server Components using it will not be statically rendered.

### When to Use Each Pattern

| Pattern                              | Reactive | Server-Rendered | Use Case                       |
| ------------------------------------ | -------- | --------------- | ------------------------------ |
| `useQuery`                           | ✅       | ❌              | Client-only, real-time updates |
| `preloadQuery` + `usePreloadedQuery` | ✅       | ✅              | SSR with reactivity            |
| `fetchQuery`                         | ❌       | ✅              | Static content, Route Handlers |
| `fetchMutation`                      | N/A      | ✅              | Server Actions, Route Handlers |

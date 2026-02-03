# Convex Functions Reference

## Function Types Overview

| Type         | DB Access      | Network | Deterministic | Cached | Use For                      |
| ------------ | -------------- | ------- | ------------- | ------ | ---------------------------- |
| `query`      | Read           | No      | Yes           | Yes    | Fetching data, subscriptions |
| `mutation`   | Read/Write     | No      | Yes           | No     | Writing data (transactional) |
| `action`     | Via `ctx.run*` | Yes     | No            | No     | External APIs, AI, email     |
| `httpAction` | Via `ctx.run*` | Yes     | No            | No     | Webhooks, custom endpoints   |

### Public vs Internal

|                 | Public                        | Internal                                              |
| --------------- | ----------------------------- | ----------------------------------------------------- |
| Definition      | `query`, `mutation`, `action` | `internalQuery`, `internalMutation`, `internalAction` |
| Client callable | Yes                           | No                                                    |
| API reference   | `api.file.function`           | `internal.file.function`                              |
| Use for         | Client-facing APIs            | Scheduling, crons, actions calling DB                 |

---

## Query Functions

Queries are **deterministic**, **cached**, and **reactive**. They re-run automatically when underlying data changes.

### QueryCtx Capabilities

```typescript
import { query, QueryCtx } from './_generated/server';

export const example = query({
  args: {},
  handler: async (ctx: QueryCtx, args) => {
    // ✅ Database reads
    const doc = await ctx.db.get(userId);
    const docs = await ctx.db.query('messages').take(50);

    // ✅ Auth
    const identity = await ctx.auth.getUserIdentity();

    // ✅ Storage URLs
    const url = await ctx.storage.getUrl(storageId);

    // ❌ NO network requests (fetch)
    // ❌ NO database writes
    // ❌ NO scheduling
  }
});
```

### Query Characteristics

- **No `fetch()`** — use actions for external APIs
- **No writes** — `ctx.db.insert/patch/replace/delete` not available
- **Deterministic** — `Math.random()` and `Date.now()` are seeded/frozen by Convex
- **Return `undefined`** → converted to `null` on client
- **Caching** — same args return cached data
- **Reactivity** — subscribers get new data when underlying data changes
- **Consistency** — all reads see same database snapshot

---

## Mutation Functions

Mutations write data and run as **transactions** — all writes commit together or none do.

### MutationCtx Capabilities

```typescript
import { mutation, MutationCtx } from './_generated/server';
import { internal } from './_generated/api';

export const example = mutation({
  args: {},
  handler: async (ctx: MutationCtx, args) => {
    // ✅ Everything QueryCtx can do, plus:

    // ✅ Database writes
    const id = await ctx.db.insert('messages', { body: 'Hello' });
    await ctx.db.patch(id, { edited: true });
    await ctx.db.replace(id, { body: 'New' });
    await ctx.db.delete(id);

    // ✅ Scheduling (use internal.* not api.*)
    await ctx.scheduler.runAfter(0, internal.jobs.process, { id });
    await ctx.scheduler.runAt(timestamp, internal.jobs.remind, {});

    // ✅ Storage writes
    await ctx.storage.delete(storageId);

    // ❌ NO network requests (fetch)
  }
});
```

### Mutation Characteristics

- **Transactional** — all writes succeed or all fail
- **Auto-retry on OCC** — Convex retries on optimistic concurrency conflicts
- **Deterministic** — same restrictions as queries
- **Ordered execution** — from React/Rust clients, mutations execute in order
- Always `await` all promises

---

## Action Functions

Actions can call external APIs but have **no transactional guarantees**.

### ActionCtx Capabilities

```typescript
import { action, ActionCtx } from './_generated/server';
import { internal } from './_generated/api';

export const example = action({
  args: {},
  handler: async (ctx: ActionCtx, args) => {
    // ✅ Network requests
    const response = await fetch('https://api.example.com/data');

    // ✅ Call queries/mutations (use internal.*)
    const user = await ctx.runQuery(internal.users.get, { id });
    await ctx.runMutation(internal.users.update, { id, data });

    // ✅ Scheduling
    await ctx.scheduler.runAfter(0, internal.jobs.process, {});

    // ✅ Auth (propagated to runQuery/runMutation)
    const identity = await ctx.auth.getUserIdentity();

    // ✅ Vector search
    const results = await ctx.vectorSearch('docs', 'by_embedding', {
      vector: embedding,
      limit: 10
    });

    // ✅ Storage
    const storageId = await ctx.storage.store(blob);

    // ❌ NO direct database access (use runQuery/runMutation)
  }
});
```

### Action Best Practices

```typescript
// ❌ BAD: Multiple sequential DB calls — each is separate transaction
export const bad = action({
  handler: async (ctx) => {
    const user = await ctx.runQuery(internal.users.get, { id });
    const posts = await ctx.runQuery(internal.posts.byUser, { userId });
    // These could see inconsistent data!
  }
});

// ✅ GOOD: Single query returns all needed data
export const good = action({
  handler: async (ctx) => {
    const data = await ctx.runQuery(internal.users.getWithPosts, { id });
    // Consistent snapshot
  }
});
```

### ctx.runAction Best Practice

**Only use `ctx.runAction` for crossing JS runtimes** (default → Node.js). Otherwise, use helper functions:

```typescript
// ❌ BAD: Unnecessary overhead
await ctx.runAction(internal.helpers.doSomething, { data });

// ✅ GOOD: Direct helper call (same runtime)
import { doSomething } from './helpers';
await doSomething(data);
```

### Dangling Promises

Always `await` all promises. Async tasks running when the function returns might not complete:

```typescript
// ❌ BAD: Promise may not complete
export const bad = action({
  handler: async (ctx) => {
    fetch('https://api.example.com/notify'); // No await!
    return 'done';
  }
});

// ✅ GOOD: Await all promises
export const good = action({
  handler: async (ctx) => {
    await fetch('https://api.example.com/notify');
    return 'done';
  }
});
```

### Action Limits

- **Timeout**: 10 minutes
- **Memory**: 512MB (Node.js), 64MB (Convex runtime)
- **Concurrent operations**: 1000 (queries, mutations, fetch requests)
- **No automatic retry** — actions may have side effects, caller must handle errors

### Node.js Runtime

```typescript
// convex/nodeActions.ts
'use node'; // Enable Node.js runtime

import { action } from './_generated/server';

export const processWithNode = action({
  handler: async (ctx) => {
    // Can use Node.js APIs and npm packages requiring Node
    const crypto = await import('crypto');
  }
});
```

**Note**: Files with `"use node"` can only contain actions, not queries or mutations.

---

## Internal Functions

Internal functions cannot be called from clients — only from other Convex functions.

```typescript
import {
  internalQuery,
  internalMutation,
  internalAction
} from './_generated/server';
import { internal } from './_generated/api';

// Only callable from other Convex functions
export const getUser = internalQuery({
  args: { userId: v.id('users') },
  handler: async (ctx, { userId }) => {
    return await ctx.db.get(userId);
  }
});

// Can skip argument validation for internal functions
export const processJob = internalMutation({
  handler: async (ctx, args: { jobId: Id<'jobs'>; data: JobData }) => {
    // Internal functions can accept complex types without validators
  }
});
```

### When to Use Internal Functions

| Use Case             | Why Internal                                       |
| -------------------- | -------------------------------------------------- |
| Scheduled functions  | `ctx.scheduler.runAfter(0, internal.jobs.run, {})` |
| Cron jobs            | Crons call internal functions                      |
| Action DB access     | `ctx.runQuery(internal.data.get, {})`              |
| Sensitive operations | No direct client access                            |
| Complex arg types    | Can skip validators for internal-only              |

### Always Use `internal.*` for Scheduling

```typescript
// ✅ CORRECT
await ctx.scheduler.runAfter(0, internal.jobs.cleanup, {});

// ❌ WRONG — api.* exposes to clients
await ctx.scheduler.runAfter(0, api.jobs.cleanup, {});
```

---

## Helper Functions

Put business logic in helper functions, keep query/mutation handlers thin.

### Pattern: Model Layer

```typescript
// convex/model/users.ts
import { QueryCtx, MutationCtx } from '../_generated/server';
import { Doc, Id } from '../_generated/dataModel';

export async function getCurrentUser(
  ctx: QueryCtx
): Promise<Doc<'users'> | null> {
  const identity = await ctx.auth.getUserIdentity();
  if (!identity) return null;

  return await ctx.db
    .query('users')
    .withIndex('by_tokenIdentifier', (q) =>
      q.eq('tokenIdentifier', identity.tokenIdentifier)
    )
    .unique();
}

export async function requireUser(ctx: QueryCtx): Promise<Doc<'users'>> {
  const user = await getCurrentUser(ctx);
  if (!user) throw new Error('Unauthorized');
  return user;
}
```

### Using Helpers

```typescript
// convex/messages.ts
import { query } from './_generated/server';
import { v } from 'convex/values';
import * as Users from './model/users';

export const list = query({
  args: { channelId: v.id('channels') },
  handler: async (ctx, { channelId }) => {
    await Users.requireUser(ctx);

    return await ctx.db
      .query('messages')
      .withIndex('by_channel', (q) => q.eq('channelId', channelId))
      .order('desc')
      .take(50);
  }
});
```

**Note**: Mutations can call helpers that take `QueryCtx` since `MutationCtx` extends it.

---

## Function Naming

Functions are named by file path + export name:

```typescript
// convex/messages.ts
export const list = query({...});        // api.messages.list
export const send = mutation({...});     // api.messages.send

// convex/channels/members.ts
export const add = mutation({...});      // api.channels.members.add

// Default export
export default query({...});             // api.messages.default
```

Non-JS clients use string format:

- `api.messages.list` → `"messages:list"`
- `api.channels.members.add` → `"channels/members:add"`

---

## Calling from Clients

### React

```typescript
import { useQuery, useMutation, useAction } from 'convex/react';
import { api } from '../convex/_generated/api';

function MyComponent() {
  // Queries: reactive, cached
  const data = useQuery(api.messages.list, { channelId });

  // Mutations: ordered execution
  const send = useMutation(api.messages.send);
  await send({ body: 'Hello' });

  // Actions: parallel execution (not ordered!)
  const process = useAction(api.ai.process);
  await process({ documentId });
}
```

**Important**: Unlike mutations, actions from a single client run in parallel. If order matters, wait for each action to complete before starting the next.

---

## Limits

| Limit                    | Value                    |
| ------------------------ | ------------------------ |
| Query/Mutation timeout   | 2 minutes                |
| Action timeout           | 10 minutes               |
| Action memory (Node.js)  | 512MB                    |
| Action memory (Convex)   | 64MB                     |
| Scheduled functions/call | 1000                     |
| Argument size            | 8MB (16MB for mutations) |

See [ERROR_HANDLING.md](ERROR_HANDLING.md) for read/write limits and error patterns.
See [HTTP_ACTIONS.md](HTTP_ACTIONS.md) for HTTP action details.
See [VALIDATION.md](VALIDATION.md) for argument validation.
See [RUNTIMES.md](RUNTIMES.md) for runtime details and bundling.

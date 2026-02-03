# Convex Authentication Reference

> **For comprehensive auth patterns** (providers, debugging, storing users, webhooks), see the `convex-auth` skill and provider-specific skills (`convex-clerk`, `convex-workos`).

This reference covers Row-Level Security and Convex Auth (first-party).

## Quick Auth Check

```typescript
const identity = await ctx.auth.getUserIdentity();
if (!identity) throw new Error('Unauthorized');

// Key fields
identity.tokenIdentifier; // Unique ID (use for DB lookups)
identity.subject; // Auth provider's user ID
identity.email; // Email (if configured)
```

## Row-Level Security Pattern

Centralize access rules for multi-tenant or resource-based authorization:

```typescript
// convex/model/rls.ts
import { QueryCtx } from '../_generated/server';
import { Doc, Id } from '../_generated/dataModel';

type TableName = 'messages' | 'documents' | 'projects';

type AccessRules = {
  [K in TableName]: (
    ctx: QueryCtx,
    doc: Doc<K>,
    userId: Id<'users'>
  ) => Promise<boolean>;
};

const rules: AccessRules = {
  messages: async (ctx, doc, userId) => {
    // User can access messages in channels they're a member of
    const membership = await ctx.db
      .query('channelMembers')
      .withIndex('by_channel_and_user', (q) =>
        q.eq('channelId', doc.channelId).eq('userId', userId)
      )
      .unique();
    return !!membership;
  },

  documents: async (ctx, doc, userId) => {
    // Owner access OR public documents
    return doc.ownerId === userId || doc.isPublic === true;
  },

  projects: async (ctx, doc, userId) => {
    // Team member check
    const member = await ctx.db
      .query('projectMembers')
      .withIndex('by_project_user', (q) =>
        q.eq('projectId', doc._id).eq('userId', userId)
      )
      .unique();
    return !!member;
  }
};

export async function canAccess<T extends TableName>(
  ctx: QueryCtx,
  table: T,
  doc: Doc<T>
): Promise<boolean> {
  const user = await getCurrentUser(ctx);
  if (!user) return false;
  return rules[table](ctx, doc, user._id);
}

export async function requireAccess<T extends TableName>(
  ctx: QueryCtx,
  table: T,
  doc: Doc<T>
): Promise<void> {
  if (!(await canAccess(ctx, table, doc))) {
    throw new Error('Access denied');
  }
}
```

### Usage

```typescript
import { canAccess, requireAccess } from './model/rls';

export const getDocument = query({
  args: { documentId: v.id('documents') },
  handler: async (ctx, { documentId }) => {
    const doc = await ctx.db.get(documentId);
    if (!doc) throw new Error('Not found');

    await requireAccess(ctx, 'documents', doc);
    return doc;
  }
});
```

## Convex Auth (First-Party)

Convex's built-in auth solution - no external provider needed.

### Setup

```bash
npm install @convex-dev/auth @auth/core
```

```typescript
// convex/auth.ts
import { convexAuth } from '@convex-dev/auth/server';
import GitHub from '@auth/core/providers/github';
import Google from '@auth/core/providers/google';
import { Password } from '@convex-dev/auth/providers/Password';

export const { auth, signIn, signOut, store } = convexAuth({
  providers: [
    GitHub,
    Google,
    Password // Email/password auth
  ]
});
```

### Schema

```typescript
// convex/schema.ts
import { defineSchema } from 'convex/server';
import { authTables } from '@convex-dev/auth/server';

export default defineSchema({
  ...authTables, // Adds users, sessions, etc.

  // Your tables
  posts: defineTable({
    authorId: v.id('users'),
    content: v.string()
  })
});
```

### HTTP Routes

```typescript
// convex/http.ts
import { httpRouter } from 'convex/server';
import { auth } from './auth';

const http = httpRouter();
auth.addHttpRoutes(http);

export default http;
```

### Client Setup

```typescript
// src/main.tsx
import { ConvexAuthProvider } from "@convex-dev/auth/react";

<ConvexAuthProvider client={convex}>
  <App />
</ConvexAuthProvider>
```

### Using Auth in Functions

```typescript
import { auth } from './auth';
import { query } from './_generated/server';

export const currentUser = query({
  args: {},
  handler: async (ctx) => {
    const userId = await auth.getUserId(ctx);
    if (!userId) return null;
    return await ctx.db.get(userId);
  }
});
```

## Resource Authorization Helpers

```typescript
// convex/model/auth.ts
import { QueryCtx } from '../_generated/server';
import { Doc, Id } from '../_generated/dataModel';
import { ConvexError } from 'convex/values';

export async function requireChannelAccess(
  ctx: QueryCtx,
  channelId: Id<'channels'>
): Promise<Doc<'channels'>> {
  const user = await requireUser(ctx);
  const channel = await ctx.db.get(channelId);

  if (!channel) throw new ConvexError('Channel not found');

  const membership = await ctx.db
    .query('channelMembers')
    .withIndex('by_channel_and_user', (q) =>
      q.eq('channelId', channelId).eq('userId', user._id)
    )
    .unique();

  if (!membership) throw new ConvexError('Access denied');

  return channel;
}

export async function requireAdmin(ctx: QueryCtx): Promise<Doc<'users'>> {
  const user = await requireUser(ctx);
  if (user.role !== 'admin') {
    throw new ConvexError('Admin access required');
  }
  return user;
}
```

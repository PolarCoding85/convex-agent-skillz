# Storing Users in Convex Database

Store user information in your database when:

- Functions need info about OTHER users (not just current)
- You need data beyond JWT fields (roles, preferences, etc.)
- You want to link users to other documents

## Schema Design

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from 'convex/server';
import { v } from 'convex/values';

export default defineSchema({
  users: defineTable({
    // Choose ONE identifier strategy:

    // Option A: tokenIdentifier (works across multiple providers)
    tokenIdentifier: v.string(),

    // Option B: externalId (provider's user ID, from identity.subject)
    // externalId: v.string(),

    // User data
    name: v.string(),
    email: v.optional(v.string()),
    imageUrl: v.optional(v.string()),

    // Custom fields
    role: v.optional(v.union(v.literal('admin'), v.literal('member')))
  }).index('by_token', ['tokenIdentifier'])
  // Or: .index("by_external_id", ["externalId"])
});
```

## Method 1: Store via Client Mutation

Call a mutation when user logs in. Simpler but requires client coordination.

### Store User Mutation

```typescript
// convex/users.ts
import { mutation, query, QueryCtx } from './_generated/server';

export const store = mutation({
  args: {},
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) {
      throw new Error('Called store without authentication');
    }

    // Check if user exists
    const existing = await ctx.db
      .query('users')
      .withIndex('by_token', (q) =>
        q.eq('tokenIdentifier', identity.tokenIdentifier)
      )
      .unique();

    if (existing) {
      // Update if name changed
      if (existing.name !== identity.name) {
        await ctx.db.patch(existing._id, {
          name: identity.name ?? 'Anonymous'
        });
      }
      return existing._id;
    }

    // Create new user
    return await ctx.db.insert('users', {
      tokenIdentifier: identity.tokenIdentifier,
      name: identity.name ?? 'Anonymous',
      email: identity.email,
      imageUrl: identity.pictureUrl
    });
  }
});

export const current = query({
  args: {},
  handler: async (ctx) => {
    return await getCurrentUser(ctx);
  }
});
```

### Client Hook (React)

```typescript
// src/hooks/useStoreUser.ts
import { useConvexAuth, useMutation } from 'convex/react';
import { useEffect, useState } from 'react';
import { api } from '../convex/_generated/api';
import { Id } from '../convex/_generated/dataModel';

export function useStoreUser() {
  const { isAuthenticated } = useConvexAuth();
  const [userId, setUserId] = useState<Id<'users'> | null>(null);
  const storeUser = useMutation(api.users.store);

  useEffect(() => {
    if (!isAuthenticated) {
      setUserId(null);
      return;
    }

    storeUser().then(setUserId);
  }, [isAuthenticated, storeUser]);

  return {
    isLoading: isAuthenticated && userId === null,
    isAuthenticated: isAuthenticated && userId !== null,
    userId
  };
}
```

## Method 2: Store via Webhooks (Recommended)

Use provider webhooks for production. More reliable - users created even without visiting your app.

### Webhook Handler

```typescript
// convex/http.ts
import { httpRouter } from 'convex/server';
import { httpAction } from './_generated/server';
import { internal } from './_generated/api';
import { Webhook } from 'svix';

const http = httpRouter();

http.route({
  path: '/clerk-users-webhook', // or /workos-users-webhook
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    // Validate webhook signature
    const event = await validateWebhook(request);
    if (!event) {
      return new Response('Invalid signature', { status: 400 });
    }

    switch (event.type) {
      case 'user.created':
      case 'user.updated':
        await ctx.runMutation(internal.users.upsert, { data: event.data });
        break;
      case 'user.deleted':
        await ctx.runMutation(internal.users.remove, {
          externalId: event.data.id
        });
        break;
    }

    return new Response(null, { status: 200 });
  })
});

async function validateWebhook(req: Request) {
  const payload = await req.text();
  const headers = {
    'svix-id': req.headers.get('svix-id')!,
    'svix-timestamp': req.headers.get('svix-timestamp')!,
    'svix-signature': req.headers.get('svix-signature')!
  };

  const wh = new Webhook(process.env.WEBHOOK_SECRET!);
  try {
    return wh.verify(payload, headers);
  } catch {
    return null;
  }
}

export default http;
```

### Internal Mutations

```typescript
// convex/users.ts
import { internalMutation } from './_generated/server';
import { v } from 'convex/values';

export const upsert = internalMutation({
  args: { data: v.any() },
  handler: async (ctx, { data }) => {
    const attributes = {
      externalId: data.id,
      name:
        `${data.first_name ?? ''} ${data.last_name ?? ''}`.trim() ||
        'Anonymous',
      email: data.email_addresses?.[0]?.email_address,
      imageUrl: data.image_url
    };

    const existing = await ctx.db
      .query('users')
      .withIndex('by_external_id', (q) => q.eq('externalId', data.id))
      .unique();

    if (existing) {
      await ctx.db.patch(existing._id, attributes);
    } else {
      await ctx.db.insert('users', attributes);
    }
  }
});

export const remove = internalMutation({
  args: { externalId: v.string() },
  handler: async (ctx, { externalId }) => {
    const user = await ctx.db
      .query('users')
      .withIndex('by_external_id', (q) => q.eq('externalId', externalId))
      .unique();

    if (user) {
      await ctx.db.delete(user._id);
    }
  }
});
```

## User Helper Functions

```typescript
// convex/model/users.ts
import { QueryCtx, MutationCtx } from '../_generated/server';
import { ConvexError } from 'convex/values';
import { Doc } from '../_generated/dataModel';

export async function getCurrentUser(
  ctx: QueryCtx
): Promise<Doc<'users'> | null> {
  const identity = await ctx.auth.getUserIdentity();
  if (!identity) return null;

  return await ctx.db
    .query('users')
    .withIndex('by_token', (q) =>
      q.eq('tokenIdentifier', identity.tokenIdentifier)
    )
    .unique();
}

export async function getCurrentUserOrThrow(
  ctx: QueryCtx
): Promise<Doc<'users'>> {
  const user = await getCurrentUser(ctx);
  if (!user) throw new ConvexError('User not found');
  return user;
}

export async function requireAuth(ctx: QueryCtx) {
  const identity = await ctx.auth.getUserIdentity();
  if (!identity) throw new ConvexError('Not authenticated');
  return identity;
}
```

## Using User IDs as Foreign Keys

```typescript
// convex/messages.ts
import { mutation, query } from './_generated/server';
import { v } from 'convex/values';
import { getCurrentUserOrThrow } from './model/users';

export const send = mutation({
  args: { body: v.string() },
  handler: async (ctx, { body }) => {
    const user = await getCurrentUserOrThrow(ctx);

    await ctx.db.insert('messages', {
      body,
      userId: user._id, // Foreign key to users table
      createdAt: Date.now()
    });
  }
});

export const list = query({
  args: {},
  handler: async (ctx) => {
    const messages = await ctx.db.query('messages').order('desc').take(50);

    return Promise.all(
      messages.map(async (msg) => {
        const user = await ctx.db.get(msg.userId);
        return { ...msg, author: user?.name ?? 'Unknown' };
      })
    );
  }
});
```

## Waiting for User to be Stored (Client)

When using webhooks, user might not exist immediately after login:

```typescript
// src/hooks/useCurrentUser.ts
import { useConvexAuth, useQuery } from 'convex/react';
import { api } from '../convex/_generated/api';

export function useCurrentUser() {
  const { isLoading: authLoading, isAuthenticated } = useConvexAuth();
  const user = useQuery(api.users.current);

  return {
    isLoading: authLoading || (isAuthenticated && user === undefined),
    isAuthenticated: isAuthenticated && user !== null,
    user
  };
}
```

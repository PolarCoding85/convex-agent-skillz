# Clerk Webhooks for Convex

Sync Clerk users to your Convex database automatically via webhooks.

## Setup in Clerk Dashboard

1. Go to **Webhooks** > **Add Endpoint**
2. Set **Endpoint URL**: `https://<your-deployment>.convex.site/clerk-users-webhook`
   - Find deployment name in Convex Dashboard > Settings > URL
   - Note: Domain ends in `.site`, NOT `.cloud`
3. Under **Message Filtering**, select all `user` events:
   - `user.created`
   - `user.updated`
   - `user.deleted`
4. Click **Create**
5. Copy the **Signing Secret** (starts with `whsec_`)
6. In Convex Dashboard > Settings > Environment Variables, add:
   - Name: `CLERK_WEBHOOK_SECRET`
   - Value: `whsec_...`

## Schema

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from 'convex/server';
import { v } from 'convex/values';

export default defineSchema({
  users: defineTable({
    clerkId: v.string(), // Clerk user ID
    name: v.string(),
    email: v.optional(v.string()),
    imageUrl: v.optional(v.string())
  }).index('by_clerk_id', ['clerkId'])
});
```

## HTTP Endpoint

```typescript
// convex/http.ts
import { httpRouter } from 'convex/server';
import { httpAction } from './_generated/server';
import { internal } from './_generated/api';
import type { WebhookEvent } from '@clerk/backend';
import { Webhook } from 'svix';

const http = httpRouter();

http.route({
  path: '/clerk-users-webhook',
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    const event = await validateClerkWebhook(request);
    if (!event) {
      return new Response('Invalid webhook signature', { status: 400 });
    }

    switch (event.type) {
      case 'user.created':
      case 'user.updated':
        await ctx.runMutation(internal.users.upsertFromClerk, {
          data: event.data
        });
        break;

      case 'user.deleted':
        if (event.data.id) {
          await ctx.runMutation(internal.users.deleteFromClerk, {
            clerkId: event.data.id
          });
        }
        break;

      default:
        console.log('Unhandled Clerk webhook event:', event.type);
    }

    return new Response(null, { status: 200 });
  })
});

async function validateClerkWebhook(
  request: Request
): Promise<WebhookEvent | null> {
  const payload = await request.text();
  const svixHeaders = {
    'svix-id': request.headers.get('svix-id')!,
    'svix-timestamp': request.headers.get('svix-timestamp')!,
    'svix-signature': request.headers.get('svix-signature')!
  };

  const wh = new Webhook(process.env.CLERK_WEBHOOK_SECRET!);

  try {
    return wh.verify(payload, svixHeaders) as unknown as WebhookEvent;
  } catch (error) {
    console.error('Webhook verification failed:', error);
    return null;
  }
}

export default http;
```

## Internal Mutations

```typescript
// convex/users.ts
import { internalMutation, query, QueryCtx } from './_generated/server';
import type { UserJSON } from '@clerk/backend';
import { v, Validator } from 'convex/values';

export const upsertFromClerk = internalMutation({
  args: { data: v.any() as Validator<UserJSON> },
  handler: async (ctx, { data }) => {
    const userAttributes = {
      clerkId: data.id,
      name:
        [data.first_name, data.last_name].filter(Boolean).join(' ') ||
        'Anonymous',
      email: data.email_addresses?.[0]?.email_address,
      imageUrl: data.image_url
    };

    const existingUser = await ctx.db
      .query('users')
      .withIndex('by_clerk_id', (q) => q.eq('clerkId', data.id))
      .unique();

    if (existingUser) {
      await ctx.db.patch(existingUser._id, userAttributes);
      return existingUser._id;
    }

    return await ctx.db.insert('users', userAttributes);
  }
});

export const deleteFromClerk = internalMutation({
  args: { clerkId: v.string() },
  handler: async (ctx, { clerkId }) => {
    const user = await ctx.db
      .query('users')
      .withIndex('by_clerk_id', (q) => q.eq('clerkId', clerkId))
      .unique();

    if (user) {
      await ctx.db.delete(user._id);
    } else {
      console.warn(`User not found for Clerk ID: ${clerkId}`);
    }
  }
});

// Query helpers
export const current = query({
  args: {},
  handler: async (ctx) => {
    return await getCurrentUser(ctx);
  }
});

export async function getCurrentUser(ctx: QueryCtx) {
  const identity = await ctx.auth.getUserIdentity();
  if (!identity) return null;

  // identity.subject is the Clerk user ID
  return await ctx.db
    .query('users')
    .withIndex('by_clerk_id', (q) => q.eq('clerkId', identity.subject))
    .unique();
}

export async function getCurrentUserOrThrow(ctx: QueryCtx) {
  const user = await getCurrentUser(ctx);
  if (!user) throw new Error('User not found');
  return user;
}
```

## Install Dependencies

```bash
npm install svix @clerk/backend
```

## Testing Webhooks

1. Deploy your code: `npx convex dev`
2. Create/update a user in Clerk Dashboard
3. Check Convex Dashboard > Logs for webhook events
4. Verify user appears in Convex Dashboard > Data

## Troubleshooting

| Issue                | Cause             | Fix                                     |
| -------------------- | ----------------- | --------------------------------------- |
| 400 Bad Request      | Invalid signature | Check CLERK_WEBHOOK_SECRET matches      |
| Webhook not received | Wrong URL         | Ensure URL ends in `.site` not `.cloud` |
| User not created     | Mutation error    | Check Convex logs for errors            |
| Missing env var      | Not deployed      | Set env var, run `npx convex dev`       |

## Waiting for User on Client

Since webhooks are async, user might not exist immediately after login:

```typescript
// src/hooks/useCurrentUser.ts
import { useConvexAuth, useQuery } from 'convex/react';
import { api } from '../convex/_generated/api';

export function useCurrentUser() {
  const { isLoading: authLoading, isAuthenticated } = useConvexAuth();
  const user = useQuery(api.users.current, isAuthenticated ? {} : 'skip');

  return {
    isLoading: authLoading || (isAuthenticated && user === undefined),
    isAuthenticated: isAuthenticated && user !== null && user !== undefined,
    user
  };
}
```

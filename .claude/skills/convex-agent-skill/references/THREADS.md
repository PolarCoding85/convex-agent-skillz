# Threads

Threads group messages in a linear history. All messages are associated with a thread.

## Creating Threads

### In Mutation or Action

```typescript
import { createThread } from '@convex-dev/agent';

const threadId = await createThread(ctx, components.agent);

// With metadata
const threadId = await createThread(ctx, components.agent, {
  userId,
  title: 'Support conversation',
  summary: 'User needs help with billing'
});
```

### Via Agent (Action Only)

Returns thread object with convenience methods:

```typescript
const { threadId, thread } = await agent.createThread(ctx, {
  userId,
  title: 'Chat session'
});

// Use thread object
const result = await thread.generateText({ prompt });
```

## Continue Existing Thread

```typescript
const { thread } = await agent.continueThread(ctx, { threadId });

// Thread methods
await thread.getMetadata();
await thread.updateMetadata({ patch: { title: 'New title' } });
await thread.generateText({ prompt });
await thread.streamText({ prompt });
await thread.generateObject({ prompt, schema });
await thread.streamObject({ prompt, schema });
```

## Generate in Thread

Any agent can generate in any thread:

```typescript
const result = await agent.generateText(ctx, { threadId }, { prompt });
```

Multiple agents can contribute to the same thread:

```typescript
// Agent 1 responds
await supportAgent.generateText(
  ctx,
  { threadId },
  { prompt: 'Initial question' }
);

// Agent 2 continues
await billingAgent.generateText(
  ctx,
  { threadId },
  { prompt: 'Follow up on billing' }
);
```

## Thread Metadata

```typescript
// Get metadata
const metadata = await thread.getMetadata();
// { userId, title, summary, ... }

// Update metadata
await thread.updateMetadata({
  patch: {
    title: 'Updated title',
    summary: 'New summary'
  }
});
```

## Delete Threads

### Single Thread

```typescript
// Asynchronous (mutation or action)
await agent.deleteThreadAsync(ctx, { threadId });

// Synchronous batches (action only)
await agent.deleteThreadSync(ctx, { threadId });
```

### By User

```typescript
await agent.deleteThreadsByUserId(ctx, { userId });
```

### All User Data

```typescript
// Asynchronous
await ctx.runMutation(components.agent.users.deleteAllForUserIdAsync, {
  userId
});

// Synchronous (action only)
await ctx.runAction(components.agent.users.deleteAllForUserId, { userId });
```

## List Threads by User

```typescript
const threads = await ctx.runQuery(
  components.agent.threads.listThreadsByUserId,
  { userId, paginationOpts: { cursor: null, numItems: 10 } }
);
```

## Get Messages in Thread

```typescript
import { listMessages, listUIMessages } from '@convex-dev/agent';

// Raw messages
const messages = await listMessages(ctx, components.agent, {
  threadId,
  excludeToolMessages: true,
  paginationOpts: { cursor: null, numItems: 10 }
});

// UI-friendly messages
const uiMessages = await listUIMessages(ctx, components.agent, {
  threadId,
  paginationOpts: { cursor: null, numItems: 10 }
});
```

## Thread Authorization

Always authorize thread access in your functions:

```typescript
async function authorizeThreadAccess(ctx: QueryCtx, threadId: string) {
  const userId = await getAuthUserId(ctx);
  if (!userId) throw new Error('Not authenticated');

  const thread = await ctx.runQuery(components.agent.threads.get, { threadId });
  if (!thread || thread.userId !== userId) {
    throw new Error('Not authorized');
  }
  return thread;
}

export const chat = action({
  args: { threadId: v.string(), prompt: v.string() },
  handler: async (ctx, { threadId, prompt }) => {
    await authorizeThreadAccess(ctx, threadId);
    return await agent.generateText(ctx, { threadId }, { prompt });
  }
});
```

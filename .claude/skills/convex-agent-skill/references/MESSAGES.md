# Messages

Messages store conversation history in threads.

## Retrieving Messages

### Server Query

```typescript
import { listUIMessages, paginationOptsValidator } from '@convex-dev/agent';

export const listThreadMessages = query({
  args: {
    threadId: v.string(),
    paginationOpts: paginationOptsValidator
  },
  handler: async (ctx, args) => {
    await authorizeThreadAccess(ctx, args.threadId);
    return await listUIMessages(ctx, components.agent, args);
  }
});
```

### For Streaming

```typescript
import { vStreamArgs, listUIMessages, syncStreams } from '@convex-dev/agent';

export const listMessages = query({
  args: {
    threadId: v.string(),
    paginationOpts: paginationOptsValidator,
    streamArgs: vStreamArgs
  },
  handler: async (ctx, args) => {
    const paginated = await listUIMessages(ctx, components.agent, args);
    const streams = await syncStreams(ctx, components.agent, args);
    return { ...paginated, streams };
  }
});
```

## React Hooks

### useUIMessages

```typescript
import { useUIMessages } from '@convex-dev/agent/react';

const { results, status, loadMore } = useUIMessages(
  api.chat.listMessages,
  { threadId },
  { initialNumItems: 10, stream: true } // stream: true for delta streaming
);
```

### useThreadMessages (Legacy)

```typescript
import { useThreadMessages, toUIMessages } from '@convex-dev/agent';

const { results } = useThreadMessages(api.chat.listMessages, { threadId });
const uiMessages = toUIMessages(results);
```

## UIMessage Type

Extended from AI SDK with additional fields:

```typescript
interface UIMessage {
  // From AI SDK
  parts: Array<TextPart | FilePart | ImagePart | ToolCallPart | ToolResultPart>;
  content: string;
  role: 'user' | 'assistant' | 'system';

  // Agent extensions
  key: string; // Unique identifier
  order: number; // Order in thread
  stepOrder: number; // Step order within same order
  status: 'streaming' | 'pending' | 'complete' | 'error';
  agentName?: string; // Agent that generated message
  text: string; // Extracted text content
  _creationTime: number; // Timestamp
}
```

## Saving Messages

### Automatic Saving

Messages saved automatically when using `generateText`, `streamText`, etc.

### Manual Saving

```typescript
import { saveMessage } from '@convex-dev/agent';

// Basic save
const { messageId } = await saveMessage(ctx, components.agent, {
  threadId,
  prompt: 'User message',
});

// Full options
const { messageId } = await saveMessage(ctx, components.agent, {
  threadId,
  userId,
  message: { role: 'assistant', content: 'Response' },
  agentName: 'support-agent',
  metadata: { reasoning: '...', usage: {...}, sources: [...] },
  embedding: { vector: [...], model: 'text-embedding-3-small' },
});
```

### Via Agent Class

```typescript
// Single message
const { messageId } = await agent.saveMessage(ctx, {
  threadId,
  userId,
  prompt,
  metadata,
});

// Multiple messages
const { messages } = await agent.saveMessages(ctx, {
  threadId,
  userId,
  messages: [{ role: 'user', content: '...' }],
  metadata: [{ ... }],
});
```

## Storage Options

```typescript
const result = await agent.generateText(
  ctx,
  { threadId },
  { prompt },
  {
    storageOptions: {
      saveMessages: 'promptAndOutput' // default
      // 'all' - save all input messages
      // 'none' - don't save any
      // 'promptAndOutput' - save prompt and output only
    }
  }
);
```

## Message Ordering

Each message has `order` and `stepOrder`:

- `order`: increments for each new user message/prompt
- `stepOrder`: increments for responses to same prompt

```typescript
// User message: order=1, stepOrder=0
// Assistant response: order=1, stepOrder=1
// Tool call: order=1, stepOrder=2
// Tool result: order=1, stepOrder=3
// Final response: order=1, stepOrder=4

// Next user message: order=2, stepOrder=0
```

Using `promptMessageId` associates responses with the prompt's order.

## Deleting Messages

### By ID

```typescript
await agent.deleteMessage(ctx, { messageId });
await agent.deleteMessages(ctx, { messageIds: [id1, id2] });
```

### By Order Range

```typescript
// Delete all at order 1
await agent.deleteMessageRange(ctx, {
  threadId,
  startOrder: 1,
  endOrder: 2 // exclusive
});

// Delete specific step range
await agent.deleteMessageRange(ctx, {
  threadId,
  startOrder: 1,
  startStepOrder: 2,
  endOrder: 2,
  endStepOrder: 5
});
```

## Optimistic Updates

```typescript
import { optimisticallySendMessage } from '@convex-dev/agent/react';

const sendMessage = useMutation(api.chat.sendMessage).withOptimisticUpdate(
  optimisticallySendMessage(api.chat.listMessages)
);

// Custom args mapping
const sendMessage = useMutation(api.chat.sendMessage).withOptimisticUpdate(
  (store, args) => {
    optimisticallySendMessage(api.chat.listMessages)(store, {
      threadId: args.threadId,
      prompt: args.userMessage // Map your args to prompt
    });
  }
);
```

## Utilities

```typescript
import {
  serializeDataOrUrl, // Serialize DataContent/URL for Convex
  filterOutOrphanedToolMessages, // Remove tool calls without results
  extractText, // Extract text from ModelMessage
  toUIMessages // Convert MessageDocs to UIMessages
} from '@convex-dev/agent';
```

## Validators and Types

```typescript
import {
  vMessage, // Validator for ModelMessage
  MessageDoc, // Type for stored message
  vMessageDoc, // Validator for MessageDoc
  Thread, // Thread object from continueThread
  ThreadDoc, // Type for thread metadata
  vThreadDoc, // Validator for ThreadDoc
  AgentComponent, // Type for components.agent
  ToolCtx, // Context type for createTool
  UIMessage // Extended UI message type
} from '@convex-dev/agent';
```

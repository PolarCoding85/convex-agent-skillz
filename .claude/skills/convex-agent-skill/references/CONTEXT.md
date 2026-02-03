# LLM Context Configuration

By default, the Agent provides message history from the thread as context. Customize with `contextOptions`.

## Context Options

```typescript
const result = await agent.generateText(
  ctx,
  { threadId },
  { prompt },
  {
    contextOptions: {
      // Exclude tool call/result messages
      excludeToolMessages: true, // default

      // Recent messages to include (after search messages)
      recentMessages: 100, // default

      // Search options for text/vector search
      searchOptions: {
        limit: 10, // Max messages from search
        textSearch: false, // Use text search
        vectorSearch: false, // Use vector search (requires textEmbeddingModel)
        // Expand search results with surrounding messages
        messageRange: { before: 2, after: 1 }
      },

      // Search across user's other threads
      searchOtherThreads: false // default
    }
  }
);
```

## Context Handler

Full control over messages sent to LLM:

```typescript
const agent = new Agent(components.agent, {
  languageModel,
  contextHandler: async (ctx, args) => {
    // args contains:
    // - search: messages from text/vector search
    // - recent: recent thread messages
    // - inputMessages: messages passed to generateText
    // - inputPrompt: prompt as { role: 'user', content }
    // - existingResponses: previous responses for same order
    // - allMessages: all of the above combined

    // Default behavior:
    return args.allMessages;

    // Or customize:
    const memories = await getUserMemories(ctx, args.userId);
    const examples = [
      { role: 'user', content: 'Example input' },
      { role: 'assistant', content: 'Example output' }
    ];

    return [
      ...memories,
      ...examples,
      ...args.search,
      ...args.recent,
      ...args.inputMessages,
      ...args.inputPrompt,
      ...args.existingResponses
    ];
  }
});
```

## Fetch Context Manually

Get context messages without calling LLM:

```typescript
import { fetchContextWithPrompt } from '@convex-dev/agent';

const { messages } = await fetchContextWithPrompt(ctx, components.agent, {
  prompt, // or messages
  promptMessageId, // optional: use existing message as prompt
  userId,
  threadId,
  contextOptions
});
```

## Search Messages Manually

```typescript
import type { MessageDoc } from '@convex-dev/agent';

const messages: MessageDoc[] = await agent.fetchContextMessages(ctx, {
  threadId,
  searchText: prompt, // Optional for text/vector search
  targetMessageId: messageId, // Optional: target search to specific message
  userId, // Required if searchOtherThreads is true
  contextOptions
});
```

### Without Agent Class

```typescript
import { fetchRecentAndSearchMessages } from '@convex-dev/agent';

const { recentMessages, searchMessages } = await fetchRecentAndSearchMessages(
  ctx,
  components.agent,
  {
    threadId,
    searchText: prompt,
    targetMessageId,
    contextOptions,
    getEmbedding: async (text) => {
      const embedding = await textEmbeddingModel.embed(text);
      return { embedding, textEmbeddingModel };
    }
  }
);
```

## Searching Other Threads

Enable cross-thread search for user:

```typescript
const result = await agent.generateText(
  ctx,
  { threadId },
  { prompt },
  {
    contextOptions: {
      searchOtherThreads: true // Uses hybrid text + vector search
    }
  }
);
```

Requires `userId` to be set on thread or passed in call.

## Passing Extra Context

Add messages without saving them:

```typescript
const ragResults = await rag.search(ctx, { query: prompt });

const result = await agent.generateText(
  ctx,
  { threadId },
  {
    messages: [
      { role: 'system', content: `Context:\n${ragResults.text}` },
      ...otherContextMessages
    ],
    prompt // Final user message
  }
);
```

Message order sent to LLM:

1. System prompt (from `instructions` or passed)
2. Messages from `contextOptions`
3. `messages` argument
4. `prompt` argument (as user message)

## Embeddings Management

### Generate Embeddings

```typescript
import { embedMessages } from '@convex-dev/agent';

const embeddings = await embedMessages(
  ctx,
  { userId, threadId, textEmbeddingModel, ...config },
  [{ role: 'user', content: 'What is love?' }]
);
```

### Save Embeddings for Existing Messages

```typescript
await agent.generateAndSaveEmbeddings(ctx, { messageIds });
```

### Migrate Embeddings

```typescript
// Get messages needing new embeddings
const messages = await ctx.runQuery(components.agent.vector.index.paginate, {
  vectorDimension: 1536,
  targetModel: 'gpt-4o-mini',
  cursor: null,
  limit: 10
});

// Update embeddings
await ctx.runMutation(components.agent.vector.index.updateBatch, {
  vectors: [{ model: 'gpt-4o-mini', vector: embedding, id: msg.embeddingId }]
});

// Or delete old and insert new (if dimension changed)
await ctx.runMutation(components.agent.vector.index.deleteBatch, {
  ids: [embeddingId1, embeddingId2]
});

await ctx.runMutation(components.agent.vector.index.insertBatch, {
  vectorDimension: 1536,
  vectors: [
    {
      model: 'gpt-4o-mini',
      table: 'messages',
      userId,
      threadId,
      vector: embedding,
      messageId
    }
  ]
});
```

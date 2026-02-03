# Persistent Text Streaming Component

Stream text to clients via HTTP while persisting to database for reconnection and multi-client sync.

> **Note:** For AI Agent use cases, prefer the Agent component's built-in delta streaming. Use this component for standalone text streaming without agent infrastructure.

## Installation

```bash
npm install @convex-dev/persistent-text-streaming
```

```typescript
// convex/convex.config.ts
import { defineApp } from 'convex/server';
import persistentTextStreaming from '@convex-dev/persistent-text-streaming/convex.config';

const app = defineApp();
app.use(persistentTextStreaming);
export default app;
```

## Setup

```typescript
// convex/streaming.ts
import {
  PersistentTextStreaming,
  StreamId
} from '@convex-dev/persistent-text-streaming';
import { components } from './_generated/api';

const streaming = new PersistentTextStreaming(
  components.persistentTextStreaming
);

// Create a stream and store the ID
export const createChat = mutation({
  args: { prompt: v.string() },
  handler: async (ctx, args) => {
    const streamId = await streaming.createStream(ctx);
    const chatId = await ctx.db.insert('chats', {
      prompt: args.prompt,
      streamId,
      body: ''
    });

    // Schedule the streaming action
    await ctx.scheduler.runAfter(0, internal.streaming.generateResponse, {
      chatId,
      streamId,
      prompt: args.prompt
    });

    return chatId;
  }
});

// Query for the stream body (persisted)
export const getChatBody = query({
  args: { streamId: v.string() },
  handler: async (ctx, args) => {
    return await streaming.getStreamBody(ctx, args.streamId as StreamId);
  }
});
```

## HTTP Streaming Action

```typescript
// convex/http.ts
import { httpAction } from './_generated/server';
import { internal } from './_generated/api';

export const chatStream = httpAction(async (ctx, request) => {
  const url = new URL(request.url);
  const streamId = url.searchParams.get('streamId') as StreamId;

  // Create stream writer
  const { body, write, close, error } = await streaming.stream(ctx, streamId);

  // Generate and stream
  (async () => {
    try {
      const response = await openai.chat.completions.create({
        model: 'gpt-4o-mini',
        messages: [{ role: 'user', content: prompt }],
        stream: true
      });

      for await (const chunk of response) {
        const text = chunk.choices[0]?.delta?.content || '';
        write(text); // Streams to HTTP AND persists to DB
      }
      close();
    } catch (e) {
      error(e);
    }
  })();

  return new Response(body, {
    headers: { 'Content-Type': 'text/event-stream' }
  });
});
```

## React Client

```typescript
import { useStream, StreamId } from "@convex-dev/persistent-text-streaming/react";

function ChatMessage({ streamId, isDriver }: { streamId: StreamId; isDriver: boolean }) {
  const convexSiteUrl = import.meta.env.VITE_CONVEX_URL
    .replace(".convex.cloud", ".convex.site");

  const { text, status } = useStream(
    api.streaming.getChatBody,                    // Query for persisted body
    new URL(`${convexSiteUrl}/chat-stream`),      // HTTP streaming endpoint
    isDriver,                                     // true = drive the stream, false = subscribe only
    streamId
  );

  return (
    <div>
      {text}
      {status === "streaming" && <span className="cursor" />}
    </div>
  );
}
```

### Parameters

- **Query**: Fetches persisted stream body for non-drivers or after reconnection
- **URL**: HTTP endpoint for live streaming
- **isDriver**: Only the original requester should be `true`; other clients use `false` to subscribe via database
- **streamId**: Identifier linking stream to your data

## How It Works

```
┌─────────────────┐     HTTP Stream    ┌─────────────────┐
│  Driver Client  │◄──────────────────►│   HTTP Action   │
│  (isDriver=true)│                    │                 │
└─────────────────┘                    │  ┌───────────┐  │
                                       │  │ LLM API   │  │
┌─────────────────┐     DB Subscribe   │  └───────────┘  │
│  Other Clients  │◄──────────────────►│        │        │
│ (isDriver=false)│                    │        ▼        │
└─────────────────┘                    │  ┌───────────┐  │
                                       │  │ Database  │  │
                                       │  │(persisted)│  │
                                       │  └───────────┘  │
                                       └─────────────────┘
```

- **Driver** gets low-latency HTTP stream directly
- **Non-drivers** subscribe to database updates (batched by sentence/paragraph)
- **Reconnection** reads from database, then resumes streaming
- **Persistence** keeps full text even after stream ends

## Chunking Options

Control how text is batched for database writes:

```typescript
const { body, write, close } = await streaming.stream(ctx, streamId, {
  chunkBy: 'sentence' // "word" | "sentence" | "paragraph" | number (bytes)
});
```

Smaller chunks = more responsive for DB subscribers but more writes.
Larger chunks = fewer writes but more lag for non-driver clients.

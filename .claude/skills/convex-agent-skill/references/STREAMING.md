# Agent Streaming Patterns

## Delta Streaming (Recommended)

Saves streaming chunks to database for multi-client sync and reconnection support.

### Server Setup

```typescript
// convex/chat.ts
import { vStreamArgs, listUIMessages, syncStreams } from '@convex-dev/agent';
import { paginationOptsValidator } from 'convex/server';

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

### Generation with Deltas

```typescript
export const chat = action({
  args: { threadId: v.string(), prompt: v.string() },
  handler: async (ctx, { threadId, prompt }) => {
    await agent.streamText(
      ctx,
      { threadId },
      { prompt },
      { saveStreamDeltas: true }
    );
  }
});
```

### Delta Options

```typescript
await agent.streamText(
  ctx,
  { threadId },
  { prompt },
  {
    saveStreamDeltas: {
      chunking: 'line', // 'word' | 'line' | regex | function
      throttleMs: 500, // Debounce saves
      returnImmediately: true // Don't wait for stream to finish
    }
  }
);
```

### React Client

```typescript
import { useUIMessages, useSmoothText, type UIMessage } from '@convex-dev/agent/react';

function Chat({ threadId }: { threadId: string }) {
  const { results, status, loadMore } = useUIMessages(
    api.chat.listMessages,
    { threadId },
    { initialNumItems: 20, stream: true } // Enable streaming
  );

  return (
    <div>
      {results.map((msg) => (
        <Message key={msg.key} message={msg} />
      ))}
    </div>
  );
}

function Message({ message }: { message: UIMessage }) {
  const [visibleText] = useSmoothText(message.text, {
    startStreaming: message.status === 'streaming',
  });

  return <div>{visibleText}</div>;
}
```

### SmoothText Component

```typescript
import { SmoothText } from '@convex-dev/agent/react';

<SmoothText text={message.text} />
```

## HTTP Streaming (Traditional)

Direct HTTP streaming for single-client scenarios.

### HTTP Action Setup

```typescript
// convex/http.ts
import { httpAction } from './_generated/server';
import { httpRouter } from 'convex/server';

export const chatStream = httpAction(async (ctx, request) => {
  const { threadId, prompt } = await request.json();
  const result = await agent.streamText(ctx, { threadId }, { prompt });
  return result.toDataStreamResponse();
  // Or: return result.toUIMessageStreamResponse();
});

const http = httpRouter();
http.route({ path: '/chat-stream', method: 'POST', handler: chatStream });
export default http;
```

### Client Consumption

```typescript
const response = await fetch(`${CONVEX_SITE_URL}/chat-stream`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ threadId, prompt })
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  const chunk = decoder.decode(value);
  // Process chunk
}
```

## Combining Both Approaches

Stream via HTTP while also saving deltas:

```typescript
const result = await agent.streamText(
  ctx,
  { threadId },
  { prompt },
  { saveStreamDeltas: { returnImmediately: true } }
);

return result.toUIMessageStreamResponse();
```

## Consuming Stream in Action

```typescript
export const processStream = action({
  handler: async (ctx) => {
    const result = await agent.streamText(
      ctx,
      { threadId },
      { prompt },
      { saveStreamDeltas: { returnImmediately: true } }
    );

    for await (const textPart of result.textStream) {
      console.log(textPart);
    }
  }
});
```

## Advanced: DeltaStreamer (Without Agent)

Stream without using Agent's wrapper:

```typescript
import { DeltaStreamer, compressUIMessageChunks } from '@convex-dev/agent';
import { streamText } from 'ai';

async function stream(ctx: ActionCtx, threadId: string, order: number) {
  const streamer = new DeltaStreamer(
    components.agent,
    ctx,
    {
      throttleMs: 100,
      compress: compressUIMessageChunks,
      onAsyncAbort: async () => console.error('Aborted')
    },
    {
      threadId,
      format: 'UIMessageChunk',
      order,
      stepOrder: 0
    }
  );

  const response = streamText({
    model: openai.chat('gpt-4o-mini'),
    prompt: 'Tell me a joke',
    abortSignal: streamer.abortController.signal,
    onError: (error) => streamer.fail(error.message)
  });

  void streamer.consumeStream(response.toUIMessageStream());

  return {
    response,
    streamId: await streamer.getStreamId()
  };
}
```

### Fetch Deltas Without Agent

```typescript
import { syncStreams, vStreamArgs } from '@convex-dev/agent';

export const listStreams = query({
  args: { threadId: v.string(), streamArgs: vStreamArgs },
  handler: async (ctx, args) => {
    return await syncStreams(ctx, components.agent, {
      ...args,
      includeStatuses: ['streaming', 'aborted', 'finished']
    });
  }
});
```

```typescript
// React
import { useStreamingUIMessages } from '@convex-dev/agent/react';

const messages = useStreamingUIMessages(api.example.listStreams, { threadId });
```

# Debugging

Tools and techniques for debugging Agent issues.

## Playground

The Agent Playground provides an interactive UI for testing and debugging:

- View thread history and message metadata
- Inspect tool calls and results
- Test prompts with different context settings
- See token usage and timing

Access via Convex Dashboard → Agent component → Playground tab.

## Log Raw Requests/Responses

Log LLM API calls:

```typescript
const agent = new Agent(components.agent, {
  languageModel: openai.chat('gpt-4o-mini'),
  rawRequestResponseHandler: async (ctx, { request, response }) => {
    console.log('LLM Request:', JSON.stringify(request, null, 2));
    console.log('LLM Response:', JSON.stringify(response, null, 2));

    // Or save to database for later analysis
    await ctx.runMutation(internal.debug.logRequest, {
      request,
      response,
      timestamp: Date.now()
    });
  }
});
```

## Log Context Messages

See exactly what the LLM receives:

```typescript
const agent = new Agent(components.agent, {
  languageModel: openai.chat('gpt-4o-mini'),
  contextHandler: async (ctx, { allMessages }) => {
    console.log('Context messages:', allMessages.length);
    console.log('Messages:', JSON.stringify(allMessages, null, 2));
    return allMessages;
  }
});
```

## Inspect Database

In Convex Dashboard → Data tab:

1. Select "agent" component above table list
2. Key tables:
   - `threads` - Thread metadata
   - `messages` - Message history with status, order, stepOrder
   - `streamingMessages` - Active streams
   - `streamDeltas` - Stream chunks
   - `files` - Tracked file references

### Messages Table Fields

```typescript
{
  threadId: string,
  agentName: string,
  status: 'pending' | 'complete' | 'error',
  order: number,      // User message sequence
  stepOrder: number,  // Response sequence within order
  message: {
    role: 'user' | 'assistant' | 'tool',
    content: string | ContentPart[],
  },
  usage?: { promptTokens, completionTokens, totalTokens },
}
```

## Common Issues

### Type Errors on `components.agent`

Run `npx convex dev` to generate component types:

```bash
npx convex dev
```

### Circular Dependencies

Add explicit return types to handlers:

```typescript
// Bad - can cause circular dependency
export const myWorkflow = workflow.define({
  handler: async (step, args) => {
    const result = await step.runAction(internal.other.action, {});
    return result;
  }
});

// Good - explicit return type
export const myWorkflow = workflow.define({
  handler: async (step, args): Promise<string> => {
    const result = await step.runAction(internal.other.action, {});
    return result;
  }
});
```

Also add return types to regular functions:

```typescript
export const myAction = action({
  args: { prompt: v.string() },
  handler: async (ctx, { prompt }): Promise<string> => {
    return await agent.generateText(ctx, { threadId }, { prompt });
  }
});
```

### Tool Type Errors

Annotate tool handler return types:

```typescript
const myTool = createTool({
  description: '...',
  args: z.object({ ... }),
  handler: async (ctx, args): Promise<string> => {
    // Without return type, can cause inference issues
    return 'result';
  },
});
```

### Message Not Appearing

1. Check `status` field in messages table
2. Verify `threadId` matches
3. Check for errors in Convex logs
4. Ensure stream completed (for streaming)

### Streaming Not Working

1. Check `streamArgs` in query
2. Verify `saveStreamDeltas: true` in generation
3. Check `stream: true` in `useUIMessages`
4. Look at `streamingMessages` and `streamDeltas` tables

### Context Issues

1. Log context via `contextHandler`
2. Check `contextOptions` settings
3. Verify embeddings exist (for vector search)
4. Check `searchOtherThreads` and `userId`

## Log Streaming Integration

Use Convex Log Streaming with Axiom or similar:

```typescript
rawRequestResponseHandler: async (ctx, { request, response, threadId, agentName }) => {
  // These logs go to your log streaming destination
  console.log(JSON.stringify({
    type: 'llm_request',
    threadId,
    agentName,
    model: request.model,
    promptTokens: response.usage?.promptTokens,
    completionTokens: response.usage?.completionTokens,
    timestamp: Date.now(),
  }));
},
```

Configure in Dashboard → Settings → Log Streams.

## Testing

```typescript
// convex/tests/agent.test.ts
import { convexTest } from 'convex-test';
import { expect, test } from 'vitest';
import schema from '../schema';

test('agent creates thread', async () => {
  const t = convexTest(schema);

  const threadId = await t.action(api.chat.createThread, {});
  expect(threadId).toBeDefined();

  const thread = await t.run(async (ctx) => {
    return await ctx.db.query('threads').first();
  });
  expect(thread).toBeDefined();
});
```

## Debug Checklist

1. [ ] Check Convex Dashboard logs for errors
2. [ ] Verify `npx convex dev` ran successfully
3. [ ] Inspect messages table for status
4. [ ] Log context messages to see LLM input
5. [ ] Check return types on all handlers
6. [ ] Verify tools are properly registered
7. [ ] Check streaming setup if using deltas
8. [ ] Verify authentication/authorization
9. [ ] Look for OCC retry errors (index issues)
10. [ ] Check component installation in `convex.config.ts`

# Human Agents

Mix human and AI responses in the same thread for customer support and hybrid workflows.

## Save User Message Without AI Response

```typescript
import { saveMessage } from '@convex-dev/agent';

await saveMessage(ctx, components.agent, {
  threadId,
  prompt: 'User question here'
});
// No AI response generated
```

## Save Human as Agent

Human responds as an agent:

```typescript
await saveMessage(ctx, components.agent, {
  threadId,
  agentName: 'Alex', // Human agent name
  message: { role: 'assistant', content: 'Human support response' }
});
```

## Human Agent Metadata

Track human agent details:

```typescript
await saveMessage(ctx, components.agent, {
  threadId,
  agentName: 'Alex',
  message: { role: 'assistant', content: 'Response from support' },
  metadata: {
    provider: 'human',
    providerMetadata: {
      human: {
        agentId: 'agent-123',
        department: 'billing',
        responseTime: 45000
      }
    }
  }
});
```

## Routing: AI vs Human

### Database Flag

```typescript
// Store assignment in your database
const thread = await ctx.db.get(threadId);
if (thread.assignedTo === 'human') {
  // Route to human agent queue
  await ctx.scheduler.runAfter(0, internal.support.notifyAgent, { threadId });
} else {
  // Generate AI response
  await agent.generateText(ctx, { threadId }, { prompt });
}
```

### LLM Classification

```typescript
const result = await agent.generateObject(
  ctx,
  { threadId },
  {
    prompt: `Should this be handled by AI or human? Question: ${userMessage}`,
    schema: z.object({
      handler: z.enum(['ai', 'human']),
      reason: z.string()
    })
  }
);

if (result.object.handler === 'human') {
  await routeToHuman(ctx, threadId);
} else {
  await agent.generateText(ctx, { threadId }, { prompt: userMessage });
}
```

### Tool-based Escalation

LLM can request human help:

```typescript
const askHuman = tool({
  description: 'Request human assistance for complex issues',
  parameters: z.object({
    question: z.string().describe('Question for human agent'),
    context: z.string().describe('Relevant context')
  })
  // No execute - handled externally
});

export const handleChat = action({
  args: { threadId: v.string(), prompt: v.string() },
  handler: async (ctx, { threadId, prompt }) => {
    const result = await agent.generateText(
      ctx,
      { threadId },
      {
        prompt,
        tools: { askHuman }
      }
    );

    // Check for human escalation requests
    const humanRequests = result.toolCalls
      .filter((tc) => tc.toolName === 'askHuman')
      .map(({ toolCallId, args }) => ({
        toolCallId,
        question: args.question,
        context: args.context
      }));

    if (humanRequests.length > 0) {
      await ctx.runMutation(internal.support.createTickets, {
        threadId,
        requests: humanRequests
      });
    }
  }
});
```

## Human Response as Tool Result

When human responds to tool call:

```typescript
export const submitHumanResponse = internalAction({
  args: {
    threadId: v.string(),
    messageId: v.string(),
    toolCallId: v.string(),
    response: v.string(),
    humanName: v.string()
  },
  handler: async (ctx, args) => {
    // Save human response as tool result
    await agent.saveMessage(ctx, {
      threadId: args.threadId,
      message: {
        role: 'tool',
        content: [
          {
            type: 'tool-result',
            result: args.response,
            toolCallId: args.toolCallId,
            toolName: 'askHuman'
          }
        ]
      },
      metadata: {
        provider: 'human',
        providerMetadata: {
          human: { name: args.humanName }
        }
      }
    });

    // Continue AI generation with human input
    await agent.generateText(
      ctx,
      { threadId: args.threadId },
      {
        promptMessageId: args.messageId
      }
    );
  }
});
```

## Support Queue Pattern

```typescript
// Schema
defineTable('supportQueue', {
  threadId: v.string(),
  userId: v.string(),
  status: v.union(
    v.literal('pending'),
    v.literal('assigned'),
    v.literal('resolved')
  ),
  assignedAgent: v.optional(v.string()),
  priority: v.number()
}).index('by_status', ['status', 'priority']);

// Escalate to human
export const escalateToHuman = mutation({
  args: { threadId: v.string(), priority: v.number() },
  handler: async (ctx, { threadId, priority }) => {
    await ctx.db.insert('supportQueue', {
      threadId,
      userId: await getAuthUserId(ctx),
      status: 'pending',
      priority
    });
  }
});

// Agent claims ticket
export const claimTicket = mutation({
  args: { ticketId: v.id('supportQueue') },
  handler: async (ctx, { ticketId }) => {
    const agentId = await getAuthUserId(ctx);
    await ctx.db.patch(ticketId, {
      status: 'assigned',
      assignedAgent: agentId
    });
  }
});

// Agent responds
export const submitResponse = action({
  args: { threadId: v.string(), response: v.string() },
  handler: async (ctx, { threadId, response }) => {
    const agentId = await getAuthUserId(ctx);
    const agent = await ctx.runQuery(api.agents.get, { agentId });

    await saveMessage(ctx, components.agent, {
      threadId,
      agentName: agent.name,
      message: { role: 'assistant', content: response },
      metadata: {
        provider: 'human',
        providerMetadata: { human: { agentId } }
      }
    });
  }
});
```

## Handoff Patterns

### AI to Human

```typescript
// AI generates response and flags for review
const result = await agent.generateText(
  ctx,
  { threadId },
  {
    prompt,
    tools: { flagForReview }
  }
);

// Or based on confidence
if (result.usage?.totalTokens > 1000) {
  // Complex response, get human review
  await escalateToHuman(ctx, threadId, { reason: 'complex_response' });
}
```

### Human to AI

```typescript
// Human marks resolved, AI takes over
export const resolveAndHandoff = mutation({
  args: { threadId: v.string() },
  handler: async (ctx, { threadId }) => {
    // Mark human ticket resolved
    const ticket = await ctx.db
      .query('supportQueue')
      .withIndex('by_threadId', (q) => q.eq('threadId', threadId))
      .first();

    if (ticket) {
      await ctx.db.patch(ticket._id, { status: 'resolved' });
    }

    // AI takes over future messages
    await ctx.db.patch(threadId, { assignedTo: 'ai' });
  }
});
```

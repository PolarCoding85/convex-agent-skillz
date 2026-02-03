# Agent Tools

Tools allow LLMs to call external services, query databases, and perform actions.

## Defining Tools

Provide tools at different levels (each overrides the previous):

1. Agent constructor: `new Agent(components.agent, { tools: {...} })`
2. Creating thread: `agent.createThread(ctx, { tools: {...} })`
3. Continuing thread: `agent.continueThread(ctx, { threadId, tools: {...} })`
4. Generation call: `agent.generateText(ctx, { threadId }, { tools: {...} })`

## Convex Tool (createTool)

Access Convex context in tool handlers:

```typescript
import { createTool } from '@convex-dev/agent';
import { z } from 'zod/v3';

const searchIdeas = createTool({
  description: 'Search for ideas in the database',
  args: z.object({
    query: z.string().describe('The query to search for')
  }),
  handler: async (ctx, { query }): Promise<Array<Idea>> => {
    // ctx has: agent, userId, threadId, messageId
    // Plus ActionCtx: auth, storage, runMutation, runAction, runQuery
    return await ctx.runQuery(api.ideas.search, { query });
  }
});
```

## Standard AI SDK Tool

```typescript
import { tool } from 'ai';
import { z } from 'zod/v3';

const getWeather = tool({
  description: 'Get current weather for a location',
  parameters: z.object({
    city: z.string().describe('City name')
  }),
  execute: async ({ city }) => {
    const response = await fetch(`https://api.weather.com/${city}`);
    return response.json();
  }
});
```

## Runtime Tool Definition

Define tools at runtime with access to scope variables:

```typescript
function createBookTools(ctx: ActionCtx, bookId: Id<'books'>) {
  return {
    getChapter: tool({
      description: 'Get a chapter from the book',
      parameters: z.object({ chapterNum: z.number() }),
      execute: async ({ chapterNum }) => {
        return await ctx.runQuery(api.books.getChapter, { bookId, chapterNum });
      }
    })
  };
}

// Use in agent
const agent = new Agent(components.agent, {
  languageModel,
  tools: createBookTools(ctx, bookId)
});
```

## Tool Context (ToolCtx)

Default context passed to `createTool` handlers:

```typescript
type ToolCtx = ActionCtx & {
  agent: Agent;
  userId?: string;
  threadId?: string;
  messageId?: string;
};
```

## Custom Context in Tools

Add custom fields to tool context:

```typescript
// Define agent with custom context type
const myAgent = new Agent<{ orgId: string }>(components.agent, { ... });

// Pass custom context when generating
await myAgent.generateText({ ...ctx, orgId: '123' }, { threadId }, { prompt });

// Use in tool
type MyCtx = ToolCtx & { orgId: string };

const myTool = createTool({
  args: z.object({ ... }),
  description: '...',
  handler: async (ctx: MyCtx, args) => {
    const orgData = await ctx.runQuery(api.orgs.get, { orgId: ctx.orgId });
    // ...
  },
});
```

## Using Tools for Multi-Step Generation

Enable automatic tool handling with `stopWhen` or `maxSteps`:

```typescript
const agent = new Agent(components.agent, {
  languageModel: openai.chat('gpt-4o-mini'),
  tools: { searchIdeas, createTicket },
  stopWhen: stepCountIs(10) // Or maxSteps: 10
});

// Tools are called automatically until stop condition
const result = await agent.generateText(ctx, { threadId }, { prompt });
```

## Agent as a Tool

Use one agent within another:

```typescript
const agentTool = createTool({
  description: `Ask a question to agent ${expertAgent.name}`,
  args: z.object({
    message: z.string().describe('The message to ask')
  }),
  handler: async (ctx, { message }, options): Promise<string> => {
    const { thread } = await expertAgent.createThread(ctx, {
      userId: ctx.userId
    });

    const result = await thread.generateText(
      {
        // Pass through messages from current generation
        messages: [...options.messages, { role: 'user', content: message }]
      },
      {
        storageOptions: { saveMessages: 'all' }
      }
    );

    return result.text;
  }
});
```

## Direct LLM in Tool (No Thread)

```typescript
import { generateText } from 'ai';

const llmTool = createTool({
  description: 'Ask a question to an LLM',
  args: z.object({
    message: z.string()
  }),
  handler: async (ctx, { message }, options): Promise<string> => {
    const result = await generateText({
      system: 'You are a helpful assistant.',
      messages: [...options.messages, { role: 'user', content: message }],
      model: openai.chat('gpt-4o-mini')
    });
    return result.text;
  }
});
```

## Best Practices

1. **Use `.describe()` on zod schemas** - Provides parameter descriptions to LLM
2. **Annotate return types** - Prevents type cycles: `handler: async (...): Promise<string>`
3. **Set appropriate `maxSteps`** - Prevents infinite tool loops
4. **Keep tools focused** - Single responsibility per tool
5. **Handle errors gracefully** - Return error messages, don't throw
6. **Use `createTool` for DB access** - Gets Convex context automatically

# Workflows

Build durable, multi-step agent workflows that survive server restarts.

## Basic Chaining

Simple sequential agent calls:

```typescript
export const getAdvice = action({
  args: { location: v.string(), threadId: v.string() },
  handler: async (ctx, { location, threadId }) => {
    // Step 1: Get weather
    await weatherAgent.generateText(
      ctx,
      { threadId },
      {
        prompt: `What is the weather in ${location}?`
      }
    );

    // Step 2: Get fashion advice (includes previous messages)
    await fashionAgent.generateText(
      ctx,
      { threadId },
      {
        prompt: 'What should I wear based on the weather?'
      }
    );
  }
});
```

## Workflow Component

For reliable, durable workflows use the Workflow component:

```bash
npm install @convex-dev/workflow
```

```typescript
// convex/convex.config.ts
import workflow from '@convex-dev/workflow/convex.config';
app.use(workflow);
```

### Define Workflow

```typescript
import { WorkflowManager } from '@convex-dev/workflow';

const workflow = new WorkflowManager(components.workflow);

export const supportWorkflow = workflow.define({
  args: { prompt: v.string(), userId: v.string() },
  handler: async (step, { prompt, userId }): Promise<string> => {
    // Step 1: Create thread (mutation step)
    const { threadId } = await createThread(step, components.agent, { userId });

    // Step 2: Save message (mutation step)
    const { messageId } = await saveMessage(step, components.agent, {
      threadId,
      prompt
    });

    // Step 3: Generate response (action step)
    const { text } = await step.runAction(
      internal.agents.generateSupport,
      { threadId, promptMessageId: messageId },
      { retry: true }
    );

    // Step 4: Send notification (mutation step)
    await step.runMutation(internal.notifications.send, {
      userId,
      message: text
    });

    return text;
  }
});
```

### Agent Functions in Workflows

Some Agent functions work directly with workflow `step`:

```typescript
import { createThread, saveMessage } from '@convex-dev/agent';

// These work in workflows (call step.runMutation internally)
const { threadId } = await createThread(step, components.agent, { userId });
const { messageId } = await saveMessage(step, components.agent, {
  threadId,
  prompt
});
```

### Agent as Action Steps

For LLM calls, use action steps:

```typescript
// Export agent as action
export const generateSupport = supportAgent.asTextAction({
  stopWhen: stepCountIs(10)
});

// Use in workflow
const { text } = await step.runAction(
  internal.agents.generateSupport,
  { threadId, promptMessageId: messageId },
  { retry: true }
);
```

### Structured Output Action

```typescript
export const getStructuredSupport = supportAgent.asObjectAction({
  schema: z.object({
    analysis: z.string().describe('Analysis of the request'),
    instruction: z.string().describe('Suggested action')
  })
});
```

## Reliability Components

### Action Retrier

Simple retry for single actions:

```typescript
import { ActionRetrier } from '@convex-dev/action-retrier';

const retrier = new ActionRetrier(components.actionRetrier);

await retrier.run(ctx, internal.agents.generate, { threadId, prompt });
```

### Workpool

Manage concurrent LLM calls:

```typescript
import { Workpool } from '@convex-dev/workpool';

const pool = new Workpool(components.workpool, {
  maxParallelism: 5
});

// Queue work
await pool.enqueueAction(ctx, internal.agents.generate, { threadId, prompt });
```

### Workflow

Full durable execution with steps:

```typescript
// Each step is recorded and can be resumed
await step.runAction(internal.agents.generate, args, { retry: true });
await step.runMutation(internal.db.save, args);
await step.sleep(5000); // Durable sleep
```

## Complex Patterns

### Parallel Agents

```typescript
export const parallelWorkflow = workflow.define({
  handler: async (step, { prompts }) => {
    // Fan out to multiple agents
    const results = await Promise.all(
      prompts.map((prompt) =>
        step.runAction(internal.agents.analyze, { prompt })
      )
    );

    // Combine results
    return await step.runAction(internal.agents.summarize, {
      inputs: results
    });
  }
});
```

### Dynamic Routing

```typescript
export const routingWorkflow = workflow.define({
  handler: async (step, { prompt }) => {
    // Triage step
    const { department } = await step.runAction(internal.agents.triage, {
      prompt
    });

    // Route to appropriate agent
    const agentAction =
      department === 'billing'
        ? internal.agents.billing
        : internal.agents.support;

    return await step.runAction(agentAction, { prompt });
  }
});
```

### Human-in-the-Loop

```typescript
export const approvalWorkflow = workflow.define({
  handler: async (step, { draft }) => {
    // Generate draft
    const { text } = await step.runAction(internal.agents.draft, { draft });

    // Wait for human approval (external signal)
    const approved = await step.waitForSignal('approval', {
      timeout: 24 * 60 * 60 * 1000 // 24 hours
    });

    if (approved) {
      await step.runMutation(internal.publish, { text });
    }
  }
});
```

## Best Practices

1. **Use `promptMessageId`** - Enables safe retries without duplicating prompts
2. **Annotate return types** - Prevents circular dependency issues
3. **Keep steps small** - Each step's args/return count toward bandwidth
4. **Pass IDs not data** - Save large data to DB, pass IDs in steps
5. **Handle failures** - Use workflow's `onComplete` for error handling
6. **Set step retries** - `{ retry: true }` for transient failures

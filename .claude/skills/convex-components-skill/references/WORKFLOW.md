# Workflow Component

Durable, long-running workflows with retries, delays, and guaranteed completion.

## Installation

```bash
npm install @convex-dev/workflow
```

```typescript
// convex/convex.config.ts
import workflow from '@convex-dev/workflow/convex.config';

app.use(workflow);
```

## Setup

```typescript
// convex/workflows.ts
import { WorkflowManager } from '@convex-dev/workflow';
import { components } from './_generated/api';

export const workflow = new WorkflowManager(components.workflow, {
  // Optional global settings
  workpoolOptions: {
    maxParallelism: 10, // Max concurrent workflow steps
    retryActionsByDefault: true,
    defaultRetryBehavior: {
      maxAttempts: 3,
      initialBackoffMs: 100,
      base: 2
    }
  }
});
```

## Defining Workflows

```typescript
import { v } from 'convex/values';
import { internal } from './_generated/api';

export const onboardingWorkflow = workflow.define({
  args: {
    userId: v.id('users'),
    email: v.string()
  },
  handler: async (step, args): Promise<void> => {
    // Step 1: Send welcome email
    await step.runAction(internal.emails.sendWelcome, {
      email: args.email
    });

    // Step 2: Wait for email verification (external event)
    await step.awaitEvent({ name: 'emailVerified' });

    // Step 3: Generate personalized content
    const content = await step.runAction(
      internal.ai.generateWelcomeContent,
      { userId: args.userId },
      { retry: true } // Enable retry for this step
    );

    // Step 4: Save to database
    await step.runMutation(internal.users.completeOnboarding, {
      userId: args.userId,
      content
    });
  }
});
```

## Starting Workflows

```typescript
export const signUp = mutation({
  args: { email: v.string(), name: v.string() },
  handler: async (ctx, args) => {
    const userId = await ctx.db.insert('users', args);

    // Start workflow
    const workflowId = await workflow.start(
      ctx,
      internal.workflows.onboardingWorkflow,
      { userId, email: args.email }
    );

    return { userId, workflowId };
  }
});
```

## Workflow Steps

### Running Functions

```typescript
// Query (read-only)
const data = await step.runQuery(internal.users.get, { userId });

// Mutation (database writes)
await step.runMutation(internal.users.update, { userId, status: 'active' });

// Action (external calls, side effects)
const result = await step.runAction(internal.api.callExternal, { endpoint });
```

### Nested Workflows

```typescript
// Run entire workflow as a single step
await step.runWorkflow(internal.workflows.subWorkflow, { data });
```

### Delays

```typescript
// Wait 5 minutes
await step.sleepMs(5 * 60 * 1000);

// Wait until specific time
await step.sleepUntil(futureTimestamp);
```

### Waiting for Events

```typescript
// Wait for external event (webhook, user action, etc.)
const eventData = await step.awaitEvent({
  name: 'paymentReceived',
  timeoutMs: 24 * 60 * 60 * 1000 // 24 hour timeout
});

// In another function, send the event:
await workflow.sendEvent(ctx, workflowId, {
  name: 'paymentReceived',
  data: { amount: 100 }
});
```

## Parallel Execution

```typescript
// Run steps in parallel
const [result1, result2] = await Promise.all([
  step.runAction(internal.api.fetch1, {}),
  step.runAction(internal.api.fetch2, {})
]);
```

## Retry Configuration

```typescript
// Per-step retry
await step.runAction(internal.api.unreliable, args, {
  retry: {
    maxAttempts: 5,
    initialBackoffMs: 500,
    base: 2
  }
});

// Disable retry for specific step
await step.runMutation(internal.db.write, args, { retry: false });
```

## Completion Callbacks

```typescript
import { vWorkflowId } from '@convex-dev/workflow';
import { vResultValidator } from '@convex-dev/workpool';

export const startWithCallback = mutation({
  handler: async (ctx, args) => {
    await workflow.start(ctx, internal.workflows.myWorkflow, args, {
      onComplete: internal.workflows.handleComplete,
      context: { originalArgs: args }
    });
  }
});

export const handleComplete = mutation({
  args: {
    workflowId: vWorkflowId,
    result: vResultValidator,
    context: v.any()
  },
  handler: async (ctx, { result, context }) => {
    if (result.kind === 'success') {
      console.log('Workflow completed:', result.returnValue);
    } else if (result.kind === 'error') {
      console.error('Workflow failed:', result.error);
    } else if (result.kind === 'canceled') {
      console.log('Workflow was canceled');
    }
  }
});
```

## Status Tracking

```typescript
import { vWorkflowId } from '@convex-dev/workflow';

export const getWorkflowStatus = query({
  args: { workflowId: vWorkflowId },
  handler: async (ctx, { workflowId }) => {
    return await workflow.status(ctx, workflowId);
  }
});
```

## Cancellation

```typescript
export const cancelWorkflow = mutation({
  args: { workflowId: vWorkflowId },
  handler: async (ctx, { workflowId }) => {
    await workflow.cancel(ctx, workflowId);
  }
});
```

## Workflow Rules

### Must Be Deterministic

Workflow handlers replay from the beginning on each step. Non-deterministic code breaks replay.

```typescript
// ❌ BAD - Non-deterministic
handler: async (step, args) => {
  const random = Math.random(); // Different each replay!
  if (random > 0.5) {
    /* ... */
  }
};

// ✅ GOOD - Deterministic
handler: async (step, args) => {
  const data = await step.runQuery(internal.data.get, {}); // Same each replay
  if (data.shouldProcess) {
    /* ... */
  }
};
```

### No Direct Side Effects

Side effects must be in steps:

```typescript
// ❌ BAD
handler: async (step, args) => {
  await fetch('https://api.example.com'); // Not durable!
};

// ✅ GOOD
handler: async (step, args) => {
  await step.runAction(internal.api.fetchExternal, {}); // Durable
};
```

### Don't Change Step Order

Once a workflow is running, don't reorder/remove steps. This causes determinism violations.

```typescript
// If workflows are in-flight, don't change this:
handler: async (step, args) => {
  await step.runAction(internal.step1, {}); // Step 1
  await step.runAction(internal.step2, {}); // Step 2 - don't remove/move!
};
```

## Integration with Agent Component

```typescript
// Export agent as workflow step
export const agentStep = supportAgent.asAction({ maxSteps: 10 });

export const supportWorkflow = workflow.define({
  args: { threadId: v.string(), question: v.string() },
  handler: async (step, args): Promise<void> => {
    // Run agent as workflow step with retries
    const response = await step.runAction(
      internal.agents.agentStep,
      {
        threadId: args.threadId,
        generateText: { prompt: args.question }
      },
      { retry: true }
    );

    // Continue with other steps
    await step.runMutation(internal.tickets.resolve, {
      answer: response
    });
  }
});
```

## Parallelism Limits

- **Free tier**: Max 20 concurrent steps
- **Pro tier**: Max 100 concurrent steps
- Use batching for high-volume work

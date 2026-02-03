# Action Retrier Component

Automatically retry failed actions with exponential backoff.

## Installation

```bash
npm install @convex-dev/action-retrier
```

```typescript
// convex/convex.config.ts
import actionRetrier from '@convex-dev/action-retrier/convex.config';
app.use(actionRetrier);
```

## Basic Setup

```typescript
// convex/retrier.ts
import { ActionRetrier } from '@convex-dev/action-retrier';
import { components } from './_generated/api';

export const retrier = new ActionRetrier(components.actionRetrier);
```

## Running Actions with Retry

### From a Mutation

```typescript
import { mutation, internalAction } from './_generated/server';
import { internal } from './_generated/api';
import { v } from 'convex/values';
import { retrier } from './retrier';

export const sendEmail = mutation({
  args: { to: v.string(), subject: v.string() },
  handler: async (ctx, args) => {
    // Retries up to 4 times with exponential backoff
    const runId = await retrier.run(ctx, internal.email.send, args);
    return runId;
  }
});

export const send = internalAction({
  args: { to: v.string(), subject: v.string() },
  handler: async (ctx, { to, subject }) => {
    // This might fail due to network issues
    await sendEmailViaProvider(to, subject);
    return { sent: true };
  }
});
```

### From an Action

```typescript
export const processOrder = internalAction({
  handler: async (ctx) => {
    const runId = await retrier.run(ctx, internal.payments.chargeCard, {
      orderId: '123'
    });
    return runId;
  }
});
```

## Configuration

### Global Defaults

```typescript
const retrier = new ActionRetrier(components.actionRetrier, {
  initialBackoffMs: 250, // Initial delay after failure (default: 250)
  base: 2, // Exponential backoff base (default: 2)
  maxFailures: 4 // Max retry attempts (default: 4)
});
```

### Per-Run Override

```typescript
const runId = await retrier.run(
  ctx,
  internal.example.myAction,
  { arg: 'value' },
  {
    initialBackoffMs: 1000, // Start with 1s delay
    base: 3, // Triple backoff each time
    maxFailures: 6 // Try up to 6 times
  }
);
```

### Backoff Schedule Example

With defaults (`initialBackoffMs: 250`, `base: 2`, `maxFailures: 4`):

| Attempt | Delay Before     |
| ------- | ---------------- |
| 1       | Immediate        |
| 2       | 250ms            |
| 3       | 500ms            |
| 4       | 1000ms           |
| Give up | After 4 failures |

## Handling Completion

### onComplete Callback

```typescript
import { runResultValidator, runIdValidator } from '@convex-dev/action-retrier';

export const startTask = mutation({
  handler: async (ctx) => {
    await retrier.run(
      ctx,
      internal.tasks.process,
      { taskId: '123' },
      { onComplete: internal.tasks.handleResult }
    );
  }
});

export const handleResult = internalMutation({
  args: {
    id: runIdValidator,
    result: runResultValidator
  },
  handler: async (ctx, { id, result }) => {
    if (result.type === 'success') {
      console.log('Succeeded:', result.returnValue);
    } else if (result.type === 'failed') {
      console.log('Failed after all retries:', result.error);
    } else if (result.type === 'canceled') {
      console.log('Was canceled');
    }
  }
});
```

**Note:** `onComplete` is guaranteed to run exactly once.

## Checking Status

```typescript
export const checkTask = query({
  args: { runId: v.string() },
  handler: async (ctx, { runId }) => {
    const status = await retrier.status(ctx, runId);

    if (status.type === 'inProgress') {
      return { status: 'running' };
    }

    // status.type === "completed"
    if (status.result.type === 'success') {
      return { status: 'done', value: status.result.returnValue };
    } else if (status.result.type === 'failed') {
      return { status: 'failed', error: status.result.error };
    } else {
      return { status: 'canceled' };
    }
  }
});
```

## Canceling Runs

```typescript
export const cancelTask = mutation({
  args: { runId: v.string() },
  handler: async (ctx, { runId }) => {
    await retrier.cancel(ctx, runId);
  }
});
```

**Note:** Currently executing actions are canceled best-effort (may still complete). Status will reflect cancellation.

## Cleanup

Runs store return values in the database. Clean up after completion:

```typescript
// Manual cleanup
await retrier.cleanup(ctx, runId);

// Automatic cleanup happens after 7 days
```

### Polling Pattern with Cleanup

```typescript
export const waitForResult = internalAction({
  args: { runId: v.string() },
  handler: async (ctx, { runId }) => {
    try {
      while (true) {
        const status = await retrier.status(ctx, runId);

        if (status.type === 'inProgress') {
          await new Promise((r) => setTimeout(r, 1000));
          continue;
        }

        console.log('Completed:', status.result);
        return status.result;
      }
    } finally {
      await retrier.cleanup(ctx, runId);
    }
  }
});
```

## Debugging

Set log level for more detail:

```bash
npx convex env set ACTION_RETRIER_LOG_LEVEL DEBUG
```

Log levels: `DEBUG`, `INFO` (default), `ERROR`

## When to Use Action Retrier

**Good candidates for retry:**

- Sending emails/notifications
- Calling external APIs
- File uploads to cloud storage
- Payment processing (with idempotency keys)
- Webhook deliveries

**Not suitable for retry:**

- Non-idempotent operations (posting tweets, creating orders)
- Actions with side effects that can't be repeated
- Time-sensitive operations where stale data is worse than failure

## Idempotency

**Important:** Only retry idempotent actions—operations safe to run multiple times.

```typescript
// ✅ Safe to retry - uses idempotency key
export const chargeCard = internalAction({
  args: { orderId: v.string(), amount: v.number() },
  handler: async (ctx, { orderId, amount }) => {
    await stripe.charges.create({
      amount,
      idempotency_key: orderId // Prevents duplicate charges
    });
  }
});

// ❌ Not safe to retry - could post multiple times
export const postTweet = internalAction({
  args: { message: v.string() },
  handler: async (ctx, { message }) => {
    await twitter.post(message); // Could duplicate!
  }
});
```

## Action Retrier vs Workpool vs Workflow

| Feature             | Action Retrier | Workpool        | Workflow    |
| ------------------- | -------------- | --------------- | ----------- |
| Retries             | ✅ Built-in    | ✅ Configurable | ✅ Per-step |
| Parallelism control | ❌             | ✅              | ✅          |
| Multi-step          | ❌             | ❌              | ✅          |
| Delays/scheduling   | ❌             | ❌              | ✅          |
| Complexity          | Low            | Medium          | High        |

**Use Action Retrier when:**

- Single action needs retry logic
- Simple fire-and-forget with reliability

**Use Workpool when:**

- Need to limit concurrent executions
- Priority queues for different work types

**Use Workflow when:**

- Multi-step processes
- Need delays between steps
- Complex orchestration

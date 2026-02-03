# Workpool Component

Queue actions and mutations with parallelism limits, retries, and completion callbacks.

## Installation

```bash
npm install @convex-dev/workpool
```

```typescript
// convex/convex.config.ts
import workpool from '@convex-dev/workpool/convex.config';

app.use(workpool);
// Separate pools for different priorities
app.use(workpool, { name: 'emailWorkpool' });
app.use(workpool, { name: 'scrapeWorkpool' });
```

## Configuration

```typescript
// convex/workpools.ts
import { Workpool } from '@convex-dev/workpool';
import { components } from './_generated/api';

// High-priority email pool
export const emailPool = new Workpool(components.emailWorkpool, {
  maxParallelism: 10,

  // Retry configuration
  retryActionsByDefault: true,
  defaultRetryBehavior: {
    maxAttempts: 3,
    initialBackoffMs: 1000,
    base: 2 // Exponential backoff
  },

  // Status retention
  statusTtl: 24 * 60 * 60 * 1000 // 1 day (default)
});

// Lower-priority scraping pool
export const scrapePool = new Workpool(components.scrapeWorkpool, {
  maxParallelism: 5
});

// Serial execution (prevents OCC conflicts)
export const counterPool = new Workpool(components.workpool, {
  maxParallelism: 1
});
```

## Enqueueing Work

### Actions

```typescript
export const processUser = action({
  args: { userId: v.id('users') },
  handler: async (ctx, { userId }) => {
    // Do work...
  }
});

export const signUp = mutation({
  handler: async (ctx, args) => {
    const userId = await ctx.db.insert('users', args);

    // Enqueue async work
    const workId = await emailPool.enqueueAction(
      ctx,
      internal.emails.sendWelcome,
      { userId }
    );

    return { userId, workId };
  }
});
```

### Mutations

```typescript
const workId = await counterPool.enqueueMutation(
  ctx,
  internal.counters.increment,
  { key: 'signups' }
);
```

### Queries (less common)

```typescript
const workId = await pool.enqueueQuery(ctx, internal.reports.generate, {
  month: '2024-01'
});
```

### Batch Enqueueing

```typescript
// More efficient than multiple single calls
await scrapePool.enqueueActionBatch(ctx, internal.scraper.fetchPage, [
  { url: 'https://example.com/page1' },
  { url: 'https://example.com/page2' },
  { url: 'https://example.com/page3' }
]);
```

## Completion Callbacks

```typescript
import { vResultValidator, vWorkIdValidator } from '@convex-dev/workpool';

export const startWork = mutation({
  handler: async (ctx, args) => {
    await emailPool.enqueueAction(
      ctx,
      internal.emails.send,
      { to: args.email },
      {
        onComplete: internal.work.handleComplete,
        context: { email: args.email } // Pass-through data
      }
    );
  }
});

export const handleComplete = mutation({
  args: {
    workId: vWorkIdValidator,
    result: vResultValidator,
    context: v.any()
  },
  handler: async (ctx, { result, context }) => {
    if (result.kind === 'success') {
      console.log('Email sent:', context.email);
    } else if (result.kind === 'error') {
      console.error('Failed:', result.error);
      // Maybe retry or alert
    } else if (result.kind === 'canceled') {
      console.log('Work was canceled');
    }
  }
});
```

## Status Tracking

```typescript
import { vWorkIdValidator } from '@convex-dev/workpool';

export const getStatus = query({
  args: { workId: vWorkIdValidator },
  handler: async (ctx, { workId }) => {
    return await emailPool.status(ctx, workId);
    // Returns: { state: "pending" | "running" | "completed" | "failed" | "canceled", ... }
  }
});
```

## Cancellation

```typescript
export const cancelWork = mutation({
  args: { workId: vWorkIdValidator },
  handler: async (ctx, { workId }) => {
    await pool.cancel(ctx, workId);
  }
});

// Cancel all pending work
export const cancelAll = mutation({
  handler: async (ctx) => {
    await pool.cancelAll(ctx);
  }
});
```

## Scheduling for Later

```typescript
// Delay execution
await pool.enqueueAction(
  ctx,
  internal.tasks.process,
  { id },
  {
    delayMs: 5000 // 5 second delay
  }
);

// Schedule for specific time
await pool.enqueueAction(
  ctx,
  internal.reports.generate,
  {},
  {
    runAtTime: Date.now() + 60 * 60 * 1000 // 1 hour from now
  }
);
```

## Retry Configuration

```typescript
// Per-enqueue retry settings
await pool.enqueueAction(
  ctx,
  internal.api.callExternal,
  { endpoint },
  {
    retry: {
      maxAttempts: 5,
      initialBackoffMs: 500,
      base: 2
    }
  }
);

// Disable retry for specific call
await pool.enqueueAction(ctx, internal.tasks.process, {}, { retry: false });
```

## Use Cases

### Priority Queues

```typescript
// Critical emails get their own high-parallelism pool
await emailPool.enqueueAction(ctx, internal.emails.sendVerification, {
  userId
});

// Background scraping uses lower-priority pool
await scrapePool.enqueueAction(ctx, internal.scraper.fetch, { url });
```

### Preventing OCC Conflicts

```typescript
// Serial execution prevents conflicts
const counterPool = new Workpool(components.counterPool, { maxParallelism: 1 });

// Multiple calls won't conflict
await counterPool.enqueueMutation(ctx, internal.counters.increment, {});
await counterPool.enqueueMutation(ctx, internal.counters.increment, {});
```

### Building Workflows

Use Workpool's `onComplete` to chain operations:

```typescript
export const step1 = mutation({
  handler: async (ctx, args) => {
    await pool.enqueueAction(ctx, internal.pipeline.step1, args, {
      onComplete: internal.pipeline.afterStep1,
      context: args
    });
  }
});

export const afterStep1 = mutation({
  args: { result: vResultValidator, context: v.any() },
  handler: async (ctx, { result, context }) => {
    if (result.kind !== 'success') return;

    await pool.enqueueAction(ctx, internal.pipeline.step2, {
      ...context,
      step1Result: result.returnValue
    });
  }
});
```

> For complex multi-step workflows, consider the [Workflow component](WORKFLOW.md) instead.

## Parallelism Limits

- **Free tier**: Max 20 across all workpools
- **Pro tier**: Max 100 across all workpools
- **High volume**: Use batching within each work item

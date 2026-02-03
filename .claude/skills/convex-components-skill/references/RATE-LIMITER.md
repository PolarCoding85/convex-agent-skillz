# Rate Limiter Component

Application-layer rate limiting with token bucket and fixed window algorithms.

## Installation

```bash
npm install @convex-dev/rate-limiter
```

```typescript
// convex/convex.config.ts
import rateLimiter from '@convex-dev/rate-limiter/convex.config';
app.use(rateLimiter);
```

## Configuration

```typescript
// convex/rateLimiter.ts
import { RateLimiter, MINUTE, HOUR, SECOND } from '@convex-dev/rate-limiter';
import { components } from './_generated/api';

export const rateLimiter = new RateLimiter(components.rateLimiter, {
  // Fixed window: resets at period boundaries
  freeTrialSignUp: { kind: 'fixed window', rate: 100, period: HOUR },

  // Token bucket: smooth rate with burst capacity
  sendMessage: {
    kind: 'token bucket',
    rate: 10, // 10 per minute sustained
    period: MINUTE,
    capacity: 3 // Allow burst of 3
  },

  // Per-user with sharding for high throughput
  llmTokens: {
    kind: 'token bucket',
    rate: 40000,
    period: MINUTE,
    shards: 100 // Reduces OCC conflicts
  }
});
```

## Usage

### Basic Limiting

```typescript
export const sendMessage = mutation({
  args: { content: v.string() },
  handler: async (ctx, args) => {
    const userId = await getAuthUserId(ctx);

    // Check and consume
    const { ok, retryAfter } = await rateLimiter.limit(ctx, 'sendMessage', {
      key: userId // Per-user limit
    });

    if (!ok) {
      throw new Error(`Rate limited. Retry after ${retryAfter}ms`);
    }

    // Proceed with action
    await ctx.db.insert('messages', { content: args.content });
  }
});
```

### Auto-Throw

```typescript
// Throws ConvexError if rate limited
await rateLimiter.limit(ctx, 'sendMessage', {
  key: userId,
  throws: true
});
```

### Check Without Consuming

```typescript
const { ok, retryAfter } = await rateLimiter.check(ctx, 'sendMessage', {
  key: userId
});

if (!ok) {
  // Show warning but don't block yet
}
```

### Reserve Future Capacity

```typescript
// For scheduled work - reserve now, consume later
const { ok, retryAfter } = await rateLimiter.limit(ctx, 'llmRequests', {
  key: userId,
  reserve: true
});

if (!ok && retryAfter) {
  // Schedule for when capacity is available
  await ctx.scheduler.runAfter(retryAfter, internal.process.doWork, args);
  return;
}
```

### Consume Multiple Units

```typescript
// LLM token tracking
await rateLimiter.limit(ctx, 'llmTokens', {
  key: userId,
  count: estimatedTokens
});
```

### Reset on Success

```typescript
// Reset failed login counter on successful login
await rateLimiter.reset(ctx, 'failedLogins', { key: userId });
```

## React Hook

```typescript
// convex/rateLimiter.ts - expose hook API
export const { getRateLimit, getServerTime } = rateLimiter.hookAPI<DataModel>(
  'sendMessage',
  { key: (ctx) => getAuthUserId(ctx) }
);
```

```typescript
// React component
import { useRateLimit } from "@convex-dev/rate-limiter/react";

function SendButton() {
  const { status, check } = useRateLimit(api.rateLimiter.getRateLimit);

  if (status && !status.ok) {
    return <p>Rate limited. Retry in {status.retryAfter}ms</p>;
  }

  return <button onClick={handleSend}>Send</button>;
}
```

## Error Handling

```typescript
import { isRateLimitError } from '@convex-dev/rate-limiter';

try {
  await sendMessage({ content });
} catch (e) {
  if (isRateLimitError(e)) {
    toast.error(`Try again in ${e.data.retryAfter}ms`);
  }
}
```

## Algorithm Comparison

| Algorithm    | Best For      | Behavior                                            |
| ------------ | ------------- | --------------------------------------------------- |
| Fixed Window | Simple quotas | Resets at period boundary; can burst 2x at boundary |
| Token Bucket | Smooth rates  | Refills continuously; `capacity` allows burst       |

## Sharding for Scale

High-throughput limits benefit from sharding:

```typescript
globalRequests: {
  kind: "token bucket",
  rate: 10000,
  period: MINUTE,
  shards: 50,  // Splits capacity across 50 buckets
},
```

Tradeoff: May under-utilize capacity (randomness) but prevents OCC conflicts.

## Integration with Agent Component

```typescript
const sharedConfig = {
  usageHandler: async (ctx, { usage, userId }) => {
    if (!userId) return;

    // Track actual token usage after generation
    await rateLimiter.limit(ctx, 'tokenUsage', {
      key: userId,
      count: usage.totalTokens,
      reserve: true // Go negative if needed, block future requests
    });
  }
};

const agent = new Agent(components.agent, {
  name: 'assistant',
  chat: openai.chat('gpt-4o-mini'),
  ...sharedConfig
});
```

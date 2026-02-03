# Rate Limiting

Control request rates to prevent abuse and manage API budgets.

## Setup

```bash
npm install @convex-dev/rate-limiter
```

```typescript
// convex/convex.config.ts
import rateLimiter from '@convex-dev/rate-limiter/convex.config';
app.use(rateLimiter);
```

## Define Rate Limits

```typescript
// convex/rateLimiting.ts
import { MINUTE, SECOND, RateLimiter } from '@convex-dev/rate-limiter';
import { components } from './_generated/api';

export const rateLimiter = new RateLimiter(components.rateLimiter, {
  // Per-user message rate
  sendMessage: {
    kind: 'fixed window',
    period: 5 * SECOND,
    rate: 1,
    capacity: 2 // Burst allowance
  },

  // Global message rate
  globalSendMessage: {
    kind: 'token bucket',
    period: MINUTE,
    rate: 1000
  },

  // Per-user token usage
  tokenUsagePerUser: {
    kind: 'token bucket',
    period: MINUTE,
    rate: 2000,
    capacity: 10000
  },

  // Global token usage
  globalTokenUsage: {
    kind: 'token bucket',
    period: MINUTE,
    rate: 100000
  }
});
```

## Pre-flight Checks

Check limits before processing:

```typescript
export const sendMessage = mutation({
  args: { threadId: v.string(), prompt: v.string() },
  handler: async (ctx, { threadId, prompt }) => {
    const userId = await getAuthUserId(ctx);

    // Check message rate
    await rateLimiter.limit(ctx, 'sendMessage', { key: userId, throws: true });
    await rateLimiter.limit(ctx, 'globalSendMessage', { throws: true });

    // Estimate and check tokens
    const estimatedTokens = await estimateTokens(ctx, threadId, prompt);
    await rateLimiter.check(ctx, 'tokenUsagePerUser', {
      key: userId,
      count: estimatedTokens,
      throws: true
    });
    await rateLimiter.check(ctx, 'globalTokenUsage', {
      count: estimatedTokens,
      reserve: true,
      throws: true
    });

    // Save and schedule
    const { messageId } = await saveMessage(ctx, components.agent, {
      threadId,
      prompt
    });

    await ctx.scheduler.runAfter(0, internal.chat.generateResponse, {
      threadId,
      promptMessageId: messageId,
      userId
    });
  }
});
```

## Track Actual Usage

Use `usageHandler` to track real token consumption:

```typescript
import { Agent, type Config } from '@convex-dev/agent';

const sharedConfig = {
  usageHandler: async (ctx, { usage, userId }) => {
    if (!userId) return;

    // Record actual usage with reservation
    await rateLimiter.limit(ctx, 'tokenUsagePerUser', {
      key: userId,
      count: usage.totalTokens,
      reserve: true // Allow temporary negative balance
    });

    await rateLimiter.limit(ctx, 'globalTokenUsage', {
      count: usage.totalTokens,
      reserve: true
    });
  }
} satisfies Config;

const agent = new Agent(components.agent, {
  languageModel: openai.chat('gpt-4o-mini'),
  ...sharedConfig
});
```

## Token Estimation

```typescript
import { fetchContextMessages } from '@convex-dev/agent';

async function estimateTokens(
  ctx: QueryCtx,
  threadId: string,
  prompt: string
): Promise<number> {
  // Rough estimate: 4 chars per token
  const promptTokens = prompt.length / 4;
  const estimatedOutput = promptTokens * 3 + 1;

  // Get previous usage from history
  const messages = await fetchContextMessages(ctx, components.agent, {
    threadId,
    searchText: prompt,
    contextOptions: { recentMessages: 2 }
  });

  const lastUsage = messages.reverse().find((m) => m.usage);
  const historyTokens = lastUsage?.usage?.totalTokens ?? 1;

  return historyTokens + promptTokens + estimatedOutput;
}
```

## Client-side Handling

### Show Rate Limit Status

```typescript
// convex/rateLimiting.ts
export const { getRateLimit, getServerTime } = rateLimiter.hookAPI<DataModel>(
  'sendMessage',
  { key: (ctx) => getAuthUserId(ctx) }
);
```

```typescript
// React component
import { useRateLimit } from '@convex-dev/rate-limiter/react';

function ChatInput() {
  const { status } = useRateLimit(api.rateLimiting.getRateLimit);

  if (status && !status.ok) {
    return (
      <div>
        Rate limit exceeded. Try again in {formatTime(status.retryAt)}
      </div>
    );
  }

  return <input ... />;
}
```

### Handle Rate Limit Errors

```typescript
import { isRateLimitError } from '@convex-dev/rate-limiter';

const handleSubmit = async () => {
  try {
    await sendMessage({ threadId, prompt });
  } catch (e) {
    if (isRateLimitError(e)) {
      toast({
        title: 'Rate limit exceeded',
        description: `Try again in ${formatTime(e.data.retryAfter)}`
      });
    }
  }
};
```

## Rate Limit Types

### Fixed Window

Resets at fixed intervals:

```typescript
{
  kind: 'fixed window',
  period: 5 * SECOND,
  rate: 1,        // 1 per period
  capacity: 2,    // Burst up to 2
}
```

### Token Bucket

Accumulates tokens over time:

```typescript
{
  kind: 'token bucket',
  period: MINUTE,
  rate: 1000,     // Tokens added per period
  capacity: 1000, // Max tokens stored
}
```

## Methods

```typescript
// Consume tokens (fails if insufficient)
await rateLimiter.limit(ctx, 'name', { key, count, throws });

// Check without consuming (for pre-flight)
await rateLimiter.check(ctx, 'name', { key, count, throws, reserve });

// Get current status
const status = await rateLimiter.status(ctx, 'name', { key });
// { ok: boolean, retryAt?: number }
```

## Best Practices

1. **Layer rate limits** - Per-user and global limits
2. **Estimate before, track after** - Check estimates, record actual
3. **Use reservations** - Allow temporary overdraft, prevent future requests
4. **Show client feedback** - Real-time rate limit status
5. **Separate message and token limits** - Different concerns

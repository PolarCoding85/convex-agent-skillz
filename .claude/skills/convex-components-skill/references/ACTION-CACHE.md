# Action Cache Component

Cache expensive action results with optional TTL expiration.

## Installation

```bash
npm install @convex-dev/action-cache
```

```typescript
// convex/convex.config.ts
import cache from '@convex-dev/action-cache/convex.config';
app.use(cache);
```

## Basic Setup

```typescript
// convex/example.ts
import { action, internalAction } from './_generated/server';
import { internal, components } from './_generated/api';
import { ActionCache } from '@convex-dev/action-cache';
import { v } from 'convex/values';

// Define cache for an expensive action
const llmCache = new ActionCache(components.actionCache, {
  action: internal.example.generateResponse,
  name: 'llm-v1', // Version identifier
  ttl: 1000 * 60 * 60 * 24 // 24 hours (optional)
});

// Public action that uses cache
export const askQuestion = action({
  args: { question: v.string() },
  // NOTE: Must explicitly type return value for cached actions
  handler: async (ctx, args): Promise<{ answer: string }> => {
    // Fetches from cache or runs action on miss
    return await llmCache.fetch(ctx, { question: args.question });
  }
});

// The expensive action being cached
export const generateResponse = internalAction({
  args: { question: v.string() },
  handler: async (ctx, { question }): Promise<{ answer: string }> => {
    const response = await callLLM(question);
    return { answer: response };
  }
});
```

## Configuration Options

```typescript
const cache = new ActionCache(components.actionCache, {
  action: internal.example.myAction, // Required: action to cache
  name: 'my-action-v1', // Optional: cache key prefix (defaults to function name)
  ttl: 1000 * 60 * 60 // Optional: TTL in ms (undefined = forever)
});
```

### Cache Key

The cache key is composed of:

1. **name** - Identifies function version (defaults to action name)
2. **args** - The arguments passed to the action

**Important:** Change the `name` when your action's return value format changes to avoid returning stale/incompatible data.

## Cache Operations

### Fetch (Default Behavior)

```typescript
// Returns cached value if exists, otherwise runs action
const result = await cache.fetch(ctx, { input: 'hello' });
```

### Force Refresh

```typescript
// Always runs action and updates cache
// Other concurrent requests still get cache hits
const result = await cache.fetch(ctx, { input: 'hello' }, { force: true });
```

This is useful for:

- Warming cache via cron jobs
- Ensuring fresh data for specific requests
- Background refresh while serving stale data

### Check Cache Without Running Action

```typescript
// Returns cached value or undefined (never runs action)
const cached = await cache.get(ctx, { input: 'hello' });
if (cached === undefined) {
  // Cache miss - handle accordingly
}
```

## Cache Invalidation

### Remove Single Entry

```typescript
await cache.remove(ctx, { input: 'hello' });
```

### Remove All Entries for Name

```typescript
// Removes all entries matching current cache name
await cache.removeAll(ctx);
```

### Remove All Entries (Global)

```typescript
import { removeAll } from '@convex-dev/action-cache';

// Removes ALL cached values across all names
await removeAll(ctx, components.actionCache);
```

## Multiple Caches

You can use the same component for multiple actions:

```typescript
const geocodingCache = new ActionCache(components.actionCache, {
  action: internal.geo.geocode,
  name: 'geocode',
  ttl: 1000 * 60 * 60 * 24 * 7 // 1 week (locations don't change)
});

const weatherCache = new ActionCache(components.actionCache, {
  action: internal.weather.fetch,
  name: 'weather',
  ttl: 1000 * 60 * 5 // 5 minutes (weather changes frequently)
});

const embedCache = new ActionCache(components.actionCache, {
  action: internal.ai.embed,
  name: 'embed-v2' // Bump version when embedding model changes
  // No TTL - embeddings are deterministic
});
```

## Versioning Pattern

When your action implementation changes:

```typescript
// Before: using text-embedding-ada-002
const embedCache = new ActionCache(components.actionCache, {
  action: internal.ai.embed,
  name: 'embed-v1'
});

// After: upgraded to text-embedding-3-small
const embedCache = new ActionCache(components.actionCache, {
  action: internal.ai.embed,
  name: 'embed-v2' // New name = fresh cache
});

// Optionally clean up old entries
await embedCache.removeAll(ctx); // Only removes "embed-v2"
```

## Pre-warming with Cron

Prevent cold cache latency by pre-warming:

```typescript
// convex/crons.ts
import { cronJobs } from 'convex/server';

const crons = cronJobs();

crons.interval(
  'warm-popular-queries',
  { minutes: 30 },
  internal.cache.warmPopularQueries
);

export default crons;

// convex/cache.ts
export const warmPopularQueries = internalAction({
  handler: async (ctx) => {
    const popularQueries = ['how to start', 'pricing', 'contact'];

    for (const query of popularQueries) {
      // Force refresh ensures cache is updated
      await llmCache.fetch(ctx, { question: query }, { force: true });
    }
  }
});
```

## TTL Behavior

- **With TTL**: Expired entries are deleted on access and via daily cleanup cron
- **Without TTL**: Entries persist indefinitely until manually removed
- **Race condition handling**: Multiple concurrent misses may all run the action; last write wins

## Use Cases

| Scenario                | TTL Suggestion          |
| ----------------------- | ----------------------- |
| LLM responses           | 1-24 hours              |
| Geocoding               | 1 week+                 |
| Weather data            | 5-15 minutes            |
| Embeddings              | None (deterministic)    |
| API rate-limited data   | Match API cache headers |
| User profile enrichment | 1-6 hours               |

## Type Safety Note

When returning cached results, you must explicitly type the handler return value due to TypeScript circular inference:

```typescript
// ✅ Correct - explicit return type
export const myAction = action({
  args: { input: v.string() },
  handler: async (ctx, args): Promise<{ result: string }> => {
    return await cache.fetch(ctx, args);
  }
});

// ❌ Incorrect - will cause type errors
export const myAction = action({
  args: { input: v.string() },
  handler: async (ctx, args) => {
    // Missing return type
    return await cache.fetch(ctx, args);
  }
});
```

See [Convex docs on circular type inference](https://docs.convex.dev/functions/actions#dealing-with-circular-type-inference) for more details.

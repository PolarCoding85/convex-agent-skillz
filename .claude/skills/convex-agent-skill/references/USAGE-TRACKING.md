# Usage Tracking

Track token usage for billing, analytics, and cost management.

## Usage Handler

Track usage on every LLM call:

```typescript
import { Agent, type UsageHandler } from '@convex-dev/agent';

const usageHandler: UsageHandler = async (ctx, args) => {
  const {
    userId, // User who triggered the call
    threadId, // Thread ID
    agentName, // Agent name
    model, // Model used (e.g., 'gpt-4o-mini')
    provider, // Provider (e.g., 'openai')
    usage, // Token counts
    providerMetadata // Provider-specific data
  } = args;

  await ctx.runMutation(internal.usage.record, {
    userId,
    agentName,
    model,
    provider,
    usage
  });
};

const agent = new Agent(components.agent, {
  languageModel: openai.chat('gpt-4o-mini'),
  usageHandler
});
```

## Usage Object

```typescript
interface Usage {
  promptTokens: number;
  completionTokens: number;
  totalTokens: number;
}
```

## Store Usage in Database

### Schema

```typescript
// convex/schema.ts
import { vUsage, vProviderMetadata } from '@convex-dev/agent';

export default defineSchema({
  rawUsage: defineTable({
    userId: v.string(),
    agentName: v.optional(v.string()),
    model: v.string(),
    provider: v.string(),
    usage: vUsage,
    providerMetadata: v.optional(vProviderMetadata),
    billingPeriod: v.string() // e.g., '2024-01'
  }).index('billingPeriod_userId', ['billingPeriod', 'userId']),

  invoices: defineTable({
    userId: v.string(),
    billingPeriod: v.string(),
    amount: v.number(),
    status: v.union(
      v.literal('pending'),
      v.literal('paid'),
      v.literal('failed')
    )
  }).index('billingPeriod_userId', ['billingPeriod', 'userId'])
});
```

### Mutation

```typescript
// convex/usage.ts
export const record = internalMutation({
  args: {
    userId: v.string(),
    agentName: v.optional(v.string()),
    model: v.string(),
    provider: v.string(),
    usage: vUsage,
    providerMetadata: v.optional(vProviderMetadata)
  },
  handler: async (ctx, args) => {
    const billingPeriod = getBillingPeriod(Date.now());
    await ctx.db.insert('rawUsage', { ...args, billingPeriod });
  }
});

function getBillingPeriod(timestamp: number): string {
  const date = new Date(timestamp);
  return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}`;
}
```

## Generate Invoices

```typescript
// convex/invoicing.ts
export const generateInvoices = internalMutation({
  handler: async (ctx) => {
    const previousPeriod = getPreviousBillingPeriod();

    // Get all users with usage
    const usageRecords = await ctx.db
      .query('rawUsage')
      .withIndex('billingPeriod_userId', (q) =>
        q.eq('billingPeriod', previousPeriod)
      )
      .collect();

    // Group by user
    const byUser = groupBy(usageRecords, (r) => r.userId);

    for (const [userId, records] of Object.entries(byUser)) {
      const totalTokens = records.reduce(
        (sum, r) => sum + r.usage.totalTokens,
        0
      );
      const amount = calculateCost(totalTokens, records);

      // Check for existing invoice
      const existing = await ctx.db
        .query('invoices')
        .withIndex('billingPeriod_userId', (q) =>
          q.eq('billingPeriod', previousPeriod).eq('userId', userId)
        )
        .first();

      if (!existing) {
        await ctx.db.insert('invoices', {
          userId,
          billingPeriod: previousPeriod,
          amount,
          status: 'pending'
        });
      }
    }
  }
});

function calculateCost(totalTokens: number, records: UsageRecord[]): number {
  // Example pricing per 1M tokens
  const pricing: Record<string, number> = {
    'gpt-4o': 5.0,
    'gpt-4o-mini': 0.15,
    'text-embedding-3-small': 0.02
  };

  let cost = 0;
  for (const record of records) {
    const rate = pricing[record.model] ?? 1.0;
    cost += (record.usage.totalTokens / 1_000_000) * rate;
  }
  return Math.round(cost * 100) / 100; // Round to cents
}
```

## Schedule Invoice Generation

```typescript
// convex/crons.ts
import { cronJobs } from 'convex/server';
import { internal } from './_generated/api';

const crons = cronJobs();

crons.monthly(
  'generateInvoices',
  { day: 2, hourUTC: 0, minuteUTC: 0 }, // 2nd day of month
  internal.invoicing.generateInvoices,
  {}
);

export default crons;
```

## Query Usage

```typescript
// Get user's usage for current period
export const getCurrentUsage = query({
  args: {},
  handler: async (ctx) => {
    const userId = await getAuthUserId(ctx);
    const billingPeriod = getBillingPeriod(Date.now());

    const records = await ctx.db
      .query('rawUsage')
      .withIndex('billingPeriod_userId', (q) =>
        q.eq('billingPeriod', billingPeriod).eq('userId', userId)
      )
      .collect();

    const totalTokens = records.reduce(
      (sum, r) => sum + r.usage.totalTokens,
      0
    );

    return {
      billingPeriod,
      totalTokens,
      recordCount: records.length,
      byModel: groupAndSumByModel(records)
    };
  }
});
```

## Usage Dashboard

```typescript
// React component
function UsageDashboard() {
  const usage = useQuery(api.usage.getCurrentUsage);

  if (!usage) return <Loading />;

  return (
    <div>
      <h2>Usage for {usage.billingPeriod}</h2>
      <p>Total tokens: {usage.totalTokens.toLocaleString()}</p>

      <h3>By Model</h3>
      <ul>
        {Object.entries(usage.byModel).map(([model, tokens]) => (
          <li key={model}>
            {model}: {tokens.toLocaleString()} tokens
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## Custom Context for Tracking

Pass additional context for detailed tracking:

```typescript
// Define agent with custom context
const agent = new Agent<{ teamId: string; projectId: string }>(
  components.agent,
  {
    languageModel: openai.chat('gpt-4o-mini'),
    usageHandler: async (ctx, { usage, userId }) => {
      await ctx.runMutation(internal.usage.record, {
        userId,
        teamId: ctx.teamId,
        projectId: ctx.projectId,
        usage
      });
    }
  }
);

// Pass context when generating
await agent.generateText(
  { ...ctx, teamId: 'team-123', projectId: 'proj-456' },
  { threadId },
  { prompt }
);
```

## Best Practices

1. **Store raw usage** - Keep detailed records for auditing
2. **Aggregate for display** - Compute summaries on demand
3. **Use billing periods** - Index by period for efficient queries
4. **Handle multiple models** - Different models have different costs
5. **Schedule invoice generation** - Automate with crons
6. **Track by dimensions** - User, team, project, agent, model

# Convex Advanced Database Reference

## System Tables

System tables provide read-only access to metadata for built-in Convex features.

### Available System Tables

| Table                  | Purpose                          |
| ---------------------- | -------------------------------- |
| `_scheduled_functions` | Metadata for scheduled functions |
| `_storage`             | Metadata for stored files        |

### Querying System Tables

Use `ctx.db.system.get` and `ctx.db.system.query` (same API as regular tables):

```typescript
import { query } from './_generated/server';
import { v } from 'convex/values';

// Get a specific scheduled function
export const getScheduledJob = query({
  args: { id: v.id('_scheduled_functions') },
  handler: async (ctx, { id }) => {
    return await ctx.db.system.get(id);
  }
});

// List all scheduled functions
export const listScheduledJobs = query({
  args: {},
  handler: async (ctx) => {
    return await ctx.db.system.query('_scheduled_functions').collect();
  }
});

// Get file metadata
export const getFileMetadata = query({
  args: { storageId: v.id('_storage') },
  handler: async (ctx, { storageId }) => {
    return await ctx.db.system.get(storageId);
  }
});

// List stored files with pagination
export const listFiles = query({
  args: { paginationOpts: paginationOptsValidator },
  handler: async (ctx, { paginationOpts }) => {
    return await ctx.db.system.query('_storage').paginate(paginationOpts);
  }
});
```

### Key Points

- System table queries are **reactive** — they update in real-time like regular queries
- Use pagination for large result sets (same `paginate()` API)
- Read-only — cannot write to system tables directly
- Useful for building admin dashboards, job monitors, file browsers

---

## Schema Philosophy

Convex takes a **flexible-first** approach to schemas, letting you move fast early while maintaining type safety.

### Implicit Typing

When you insert documents, Convex automatically tracks field types:

```typescript
// No schema defined yet — Convex tracks types implicitly
await ctx.db.insert('users', {
  name: 'Alice',
  age: 30,
  email: 'alice@example.com'
});
```

View inferred schemas in the Dashboard to see what types Convex detected.

### Handling Schema Evolution

When field types change, Convex tracks them as unions:

```typescript
// Initially stored age as string
await ctx.db.insert('users', { name: 'Bob', age: '25' });

// Later stored age as number
await ctx.db.insert('users', { name: 'Carol', age: 30 });

// Convex infers: age: v.union(v.string(), v.number())
```

### When to Formalize

Start without a schema for rapid prototyping, then formalize when:

- Your data model stabilizes
- You want compile-time type checking
- You need to enforce constraints
- You're preparing for production

```typescript
// convex/schema.ts — Formalize when ready
import { defineSchema, defineTable } from 'convex/server';
import { v } from 'convex/values';

export default defineSchema({
  users: defineTable({
    name: v.string(),
    age: v.number(), // Now enforced
    email: v.string()
  }).index('by_email', ['email'])
});
```

### Migration Pattern

When formalizing a schema with mixed types, migrate existing data:

```typescript
// Migration mutation
export const migrateAgeField = internalMutation({
  handler: async (ctx) => {
    const users = await ctx.db.query('users').collect();

    for (const user of users) {
      if (typeof user.age === 'string') {
        await ctx.db.patch(user._id, {
          age: parseInt(user.age, 10)
        });
      }
    }
  }
});
```

---

## OCC and Atomicity

Convex uses **Optimistic Concurrency Control (OCC)** to ensure data consistency without locks.

### How OCC Works

1. **Read Phase**: Transaction reads documents, recording their versions
2. **Proposal Phase**: Transaction proposes writes based on those reads
3. **Commit Phase**: Writes commit **only if** all read versions are still current

```typescript
// This transfer is automatically atomic
export const transfer = mutation({
  args: {
    from: v.id('accounts'),
    to: v.id('accounts'),
    amount: v.number()
  },
  handler: async (ctx, { from, to, amount }) => {
    const sender = await ctx.db.get(from); // Read v1
    const receiver = await ctx.db.get(to); // Read v3

    if (!sender || !receiver) throw new Error('Account not found');
    if (sender.balance < amount) throw new Error('Insufficient funds');

    // These writes only commit if sender is still v1 and receiver is still v3
    await ctx.db.patch(from, { balance: sender.balance - amount });
    await ctx.db.patch(to, { balance: receiver.balance + amount });
  }
});
```

### Automatic Retries

When a conflict occurs (another transaction modified a document you read), Convex automatically retries your mutation. This works because:

1. **Mutations are deterministic** — no side effects outside Convex
2. **No external calls** — mutations can't call APIs or write files
3. **Idempotent reads** — same inputs produce same outputs

```typescript
// ✅ Safe — Convex can retry this
export const incrementCounter = mutation({
  handler: async (ctx, { counterId }) => {
    const counter = await ctx.db.get(counterId);
    await ctx.db.patch(counterId, {
      value: counter.value + 1
    });
  }
});

// ❌ This is why actions can't write directly — they're not deterministic
export const badPattern = action({
  handler: async (ctx) => {
    const random = Math.random(); // Non-deterministic!
    // If this could write to DB and needed retry, we'd get different results
  }
});
```

### Serializability

Convex provides **true serializability**, not just snapshot isolation:

| Guarantee       | Snapshot Isolation | Convex (Serializable) |
| --------------- | ------------------ | --------------------- |
| Atomic commits  | ✅                 | ✅                    |
| No dirty reads  | ✅                 | ✅                    |
| No lost updates | ✅                 | ✅                    |
| No write skew   | ❌                 | ✅                    |

This means concurrent transactions always yield correct results — no anomalies.

### Practical Implications

**You don't need to think about this day-to-day.** Write mutations as if they always succeed:

```typescript
// Just write the obvious code — Convex handles conflicts
export const claimSeat = mutation({
  args: { eventId: v.id('events'), seatNumber: v.number() },
  handler: async (ctx, { eventId, seatNumber }) => {
    const event = await ctx.db.get(eventId);

    // Check if seat is available
    const existingClaim = await ctx.db
      .query('seatClaims')
      .withIndex('by_event_seat', (q) =>
        q.eq('eventId', eventId).eq('seatNumber', seatNumber)
      )
      .unique();

    if (existingClaim) {
      throw new Error('Seat already taken');
    }

    // Claim the seat — OCC ensures no double-booking
    await ctx.db.insert('seatClaims', {
      eventId,
      seatNumber,
      userId: user._id
    });
  }
});
```

### When You Might See OCC Errors

High contention on the same documents can cause repeated retries:

- **Hot counters**: Many users incrementing the same counter
- **Queue-like patterns**: Multiple workers claiming from the same queue
- **Global state**: Frequently updated singleton documents

**Solutions**:

- Use the [Sharded Counter component](https://www.convex.dev/components) for counters
- Design for less contention (partition by user, time, etc.)
- Use the [Workpool component](https://www.convex.dev/components) for job queues

### OCC vs Pessimistic Locking

| Aspect          | OCC (Convex)      | Pessimistic Locking    |
| --------------- | ----------------- | ---------------------- |
| Contention      | Retry on conflict | Wait for lock          |
| Deadlocks       | Impossible        | Possible               |
| Failed clients  | No impact         | Can block others       |
| Low contention  | Faster            | Slower (lock overhead) |
| High contention | More retries      | Waiting                |

OCC is ideal for web applications where conflicts are rare and you want high throughput without lock management complexity.

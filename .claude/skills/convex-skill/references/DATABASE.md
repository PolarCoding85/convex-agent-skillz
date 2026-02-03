# Convex Database Reference

## Tables and Documents

Tables hold your app's data. They spring into existence when you add the first document:

```typescript
// `friends` table doesn't exist yet
await ctx.db.insert('friends', { name: 'Jamie' });
// Now it does, and it has one document
```

Documents are JavaScript objects with fields and values. They can contain nested objects and arrays.

### System Fields

Every document automatically has:

```typescript
{
  _id: Id<"tableName">,       // Unique, globally unique identifier
  _creationTime: number,      // Unix timestamp (ms) when created
  // ... your fields
}
```

---

## Schema Definition

Schemas ensure documents match expected types and provide end-to-end TypeScript safety.

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from 'convex/server';
import { v } from 'convex/values';

export default defineSchema({
  users: defineTable({
    name: v.string(),
    email: v.string(),
    tokenIdentifier: v.string(),
    imageUrl: v.optional(v.string())
  })
    .index('by_email', ['email'])
    .index('by_tokenIdentifier', ['tokenIdentifier']),

  channels: defineTable({
    name: v.string(),
    ownerId: v.id('users'),
    isPrivate: v.boolean()
  }).index('by_owner', ['ownerId']),

  messages: defineTable({
    channelId: v.id('channels'),
    authorId: v.id('users'),
    body: v.string(),
    edited: v.optional(v.boolean())
  })
    .index('by_channel', ['channelId'])
    .index('by_author', ['authorId'])
    .index('by_channel_author', ['channelId', 'authorId'])
});
```

### Schema Options

```typescript
defineSchema(
  {
    // tables...
  },
  {
    // Validate documents at runtime (default: true)
    schemaValidation: true,

    // Only allow tables defined in schema (default: true)
    // Set to false to allow accessing undefined tables (typed as `any`)
    strictTableNameTypes: true
  }
);
```

### Validators Reference

```typescript
// Primitives
v.string(); // string
v.number(); // number (float64)
v.int64(); // bigint (-2^63 to 2^63-1)
v.boolean(); // boolean
v.null(); // null
v.bytes(); // ArrayBuffer

// References
v.id('tableName'); // Id<"tableName">

// Collections
v.array(v.string()); // string[]
v.object({ key: v.string() }); // { key: string }
v.record(v.string(), v.number()); // Record<string, number>

// Optional and unions
v.optional(v.string()); // string | undefined
v.union(v.string(), v.null()); // string | null
v.literal('active'); // "active"

// Escape hatch
v.any(); // any
```

### Union Types for Polymorphic Tables

```typescript
defineTable(
  v.union(
    v.object({
      kind: v.literal('text'),
      body: v.string()
    }),
    v.object({
      kind: v.literal('image'),
      url: v.string(),
      width: v.number()
    })
  )
);
```

### Circular References

Schema validation runs on every write, so circular references need nullable fields:

```typescript
export default defineSchema({
  users: defineTable({
    preferencesId: v.id('preferences')
  }),
  preferences: defineTable({
    userId: v.union(v.id('users'), v.null()) // Nullable to break cycle
  })
});

// Create in order:
const preferencesId = await ctx.db.insert('preferences', { userId: null });
const userId = await ctx.db.insert('users', { preferencesId });
await ctx.db.patch(preferencesId, { userId });
```

---

## Data Types

| Convex Type | JS/TS Type  | Validator         | Notes                                      |
| ----------- | ----------- | ----------------- | ------------------------------------------ |
| Id          | string      | `v.id(tableName)` | Globally unique document reference         |
| Null        | null        | `v.null()`        | Use instead of `undefined`                 |
| Int64       | bigint      | `v.int64()`       | -2^63 to 2^63-1                            |
| Float64     | number      | `v.number()`      | IEEE-754 double (includes NaN, Infinity)   |
| Boolean     | boolean     | `v.boolean()`     |                                            |
| String      | string      | `v.string()`      | UTF-8, max 1MB                             |
| Bytes       | ArrayBuffer | `v.bytes()`       | Max 1MB                                    |
| Array       | Array       | `v.array(v)`      | Max 8192 elements                          |
| Object      | Object      | `v.object({})`    | Max 1024 fields, plain objects only        |
| Record      | Record      | `v.record(k, v)`  | Dynamic keys (ASCII, no `$` or `_` prefix) |

### Type Limits

- Documents: max 1MB total size
- Nesting: max 16 levels deep
- Arrays: max 8192 elements
- Objects: max 1024 fields
- Field names: non-empty, no `$` or `_` prefix

### Working with `undefined`

`undefined` is **not** a valid Convex value. Key behaviors:

```typescript
// Objects with undefined fields → field is omitted
await ctx.db.insert('users', { name: 'Al', nickname: undefined });
// Stored as: { name: "Al" }

// In patch: undefined REMOVES the field
await ctx.db.patch(id, { nickname: undefined });  // Removes nickname

// Querying: undefined matches missing fields
.withIndex('by_nickname', (q) => q.eq('nickname', undefined))
// Matches docs without nickname field

// Match docs that HAVE a field (exclude undefined)
.withIndex('by_a', (q) => q.gte('a', null))

// Functions returning undefined → converted to null on client
```

### Working with Dates and Times

Convex has no special date type. Options:

```typescript
// Option 1: UTC timestamp (recommended for points in time)
{
  createdAt: Date.now(),  // number in milliseconds
}
// Convert back: new Date(doc.createdAt)

// Option 2: ISO string (for calendar dates/times with timezone)
{
  scheduledAt: '2024-03-21T14:37:15Z',
  timezone: 'America/New_York'  // IANA timezone
}
```

---

## Document IDs and Relationships

Every document has a globally unique `_id` generated on insert:

```typescript
const userId = await ctx.db.insert('users', { name: 'Michael Jordan' });
const user = await ctx.db.get(userId);
console.log(user._id === userId); // true
```

### References Between Tables

Embed IDs to create relationships:

```typescript
// One-to-many: Book belongs to User
await ctx.db.insert('books', {
  title: 'Foundation',
  ownerId: user._id // Reference to users table
});

// Follow the reference
const book = await ctx.db.get(bookId);
const owner = await ctx.db.get(book.ownerId);

// Query by reference
const userBooks = await ctx.db
  .query('books')
  .withIndex('by_owner', (q) => q.eq('ownerId', user._id))
  .collect();
```

### Serializing IDs

IDs are strings, safe for URLs and external storage:

```typescript
// Client: cast string to Id type
import { Id } from '../convex/_generated/dataModel';
const taskId = localStorage.getItem('taskId') as Id<'tasks'>;

// Backend: validate with v.id()
export const getTask = query({
  args: { taskId: v.id('tasks') }, // Validates it's a tasks ID
  handler: async (ctx, { taskId }) => {
    return await ctx.db.get(taskId);
  }
});
```

### Data Modeling Best Practices

Keep documents relatively small. Avoid deeply nested objects and large arrays (>10 elements). Use separate tables with references instead.

---

## Indexes

Indexes speed up queries by organizing documents in a specific order. Without indexes, queries perform **full table scans**.

**Rule**: Query performance depends on how many documents are in the index range, not total table size.

### Index Limits

- Max **16 fields** per index
- Max **32 indexes** per table
- No duplicate fields within an index
- `_creationTime` is automatically appended (don't add it)
- Reserved: `by_id`, `by_creation_time`

### Defining Indexes

```typescript
defineTable({
  channelId: v.id('channels'),
  authorId: v.id('users'),
  status: v.string()
})
  .index('by_channel', ['channelId'])
  .index('by_channel_author', ['channelId', 'authorId'])
  .index('by_status_channel', ['status', 'channelId']);
```

### Index Range Expressions

`withIndex` accepts a **range expression** in strict order:

1. **Zero or more `.eq()` conditions** — must match index field order
2. **Optionally one lower bound** — `.gt()` or `.gte()`
3. **Optionally one upper bound** — `.lt()` or `.lte()`

```typescript
// Index: ["channelId", "authorId"] + auto _creationTime

// ✅ Just equality on first field
.withIndex('by_channel_author', (q) => q.eq('channelId', id))

// ✅ Equality on both fields
.withIndex('by_channel_author', (q) =>
  q.eq('channelId', channelId).eq('authorId', authorId)
)

// ✅ Equality + range on next field
.withIndex('by_channel_author', (q) =>
  q.eq('channelId', channelId)
   .eq('authorId', authorId)
   .gt('_creationTime', timestamp)
)

// ❌ INVALID: Can't skip fields
.withIndex('by_channel_author', (q) =>
  q.eq('channelId', channelId).gt('_creationTime', timestamp)
)

// ❌ INVALID: Must query fields in order
.withIndex('by_channel_author', (q) => q.eq('authorId', authorId))
```

### Compound Index Design

A compound index `["a", "b", "c"]` efficiently supports queries on:

- `a` only
- `a` and `b`
- `a`, `b`, and `c`

Does **not** support: `b` only, `c` only, `b` and `c`

```typescript
// ❌ Redundant — compound index handles single-field queries
.index('by_channel', ['channelId'])
.index('by_channel_author', ['channelId', 'authorId'])

// ✅ Only need compound index (unless sort order differs)
.index('by_channel_author', ['channelId', 'authorId'])
```

**Exception**: Keep both if you need different sort orders:

- `by_channel` sorts by `[channelId, _creationTime]`
- `by_channel_author` sorts by `[channelId, authorId, _creationTime]`

### Sorting with Indexes

Indexes enable efficient sorted queries for leaderboards, recent items, etc:

```typescript
// Schema: index for sorting by score
players: defineTable({
  username: v.string(),
  highestScore: v.number()
}).index('by_highest_score', ['highestScore']);

// Top 10 highest scoring players
const topPlayers = await ctx.db
  .query('players')
  .withIndex('by_highest_score')
  .order('desc')
  .take(10);
```

**Important**: When using `withIndex` without a range expression, always use `.first()`, `.unique()`, `.take(n)`, or `.paginate()` to avoid full table scans.

### Type Ordering in Indexes

When a field contains mixed types, ascending order is:

```
undefined < null < bigint < number < boolean < string < ArrayBuffer < Array < Object
```

### Staged Indexes (Large Tables)

For large tables, use staged indexes to avoid blocking deploys:

```typescript
// Step 1: Create staged index (doesn't block deploy)
.index('by_channel', { fields: ['channelId'], staged: true })

// Step 2: Monitor backfill progress in Dashboard → Indexes

// Step 3: After backfill completes, enable the index
.index('by_channel', ['channelId'])  // Remove staged option
```

---

## Reading Data

### Get by ID

```typescript
const user = await ctx.db.get(userId); // Doc | null
```

### Query Builder

```typescript
const results = await ctx.db
  .query('messages')
  .withIndex('by_channel', (q) => q.eq('channelId', channelId))
  .order('desc') // "asc" (default) or "desc"
  .filter((q) => q.eq(q.field('edited'), true))
  .collect();
```

### Terminal Methods

| Method            | Returns            | Scans         | Use When                           |
| ----------------- | ------------------ | ------------- | ---------------------------------- |
| `.first()`        | `Doc \| null`      | Until 1 found | Need first match                   |
| `.unique()`       | `Doc \| null`      | Until 2 found | Expect 0-1 results (throws if >1)  |
| `.take(n)`        | `Doc[]`            | Until n found | Need limited results               |
| `.collect()`      | `Doc[]`            | Entire range  | Need all (use with bounded range!) |
| `.paginate(opts)` | `PaginationResult` | One page      | Building paginated UI              |

### Query Performance Guidelines

```typescript
// ✅ FAST: Index narrows + take limits scan
await ctx.db
  .query('messages')
  .withIndex('by_channel', (q) => q.eq('channelId', channelId))
  .order('desc')
  .take(50);

// ✅ FAST: Unique/first stop at first match
await ctx.db
  .query('users')
  .withIndex('by_email', (q) => q.eq('email', email))
  .unique();

// ⚠️ POTENTIALLY SLOW: collect() on large range
await ctx.db
  .query('messages')
  .withIndex('by_channel', (q) => q.eq('channelId', channelId))
  .collect(); // OK if <1000 messages

// ❌ SLOW: Full table scan with filter
await ctx.db
  .query('messages')
  .filter((q) => q.eq(q.field('channelId'), channelId))
  .collect();

// ❌ SLOW: No index range expression
await ctx.db
  .query('messages')
  .withIndex('by_channel') // No range!
  .collect();
```

### Streaming with `for await...of`

For complex filtering that can't use indexes:

```typescript
export const firstPostWithTag = query({
  args: { tag: v.string() },
  handler: async (ctx, { tag }) => {
    for await (const post of ctx.db.query('posts')) {
      if (post.tags.includes(tag)) {
        return post; // Stop at first match
      }
    }
    return null;
  }
});
```

---

## Filters

`.filter()` runs **after** the index range, checking each document. Use indexes when possible for better performance.

### Comparison Operators

```typescript
// Equality
.filter((q) => q.eq(q.field('name'), 'Alex'))      // name === "Alex"
.filter((q) => q.neq(q.field('status'), 'done'))   // status !== "done"

// Comparisons
.filter((q) => q.lt(q.field('age'), 18))           // age < 18
.filter((q) => q.lte(q.field('age'), 18))          // age <= 18
.filter((q) => q.gt(q.field('age'), 18))           // age > 18
.filter((q) => q.gte(q.field('age'), 18))          // age >= 18
```

### Arithmetic Operators

```typescript
// Carpets with area > 100
.filter((q) => q.gt(q.mul(q.field('height'), q.field('width')), 100))

// Available operators:
q.add(l, r)   // l + r
q.sub(l, r)   // l - r
q.mul(l, r)   // l * r
q.div(l, r)   // l / r
q.mod(l, r)   // l % r
q.neg(x)      // -x
```

### Combining Operators

```typescript
// AND: name === "Alex" && age >= 18
.filter((q) =>
  q.and(q.eq(q.field('name'), 'Alex'), q.gte(q.field('age'), 18))
)

// OR: name === "Alex" || name === "Emma"
.filter((q) =>
  q.or(q.eq(q.field('name'), 'Alex'), q.eq(q.field('name'), 'Emma'))
)

// NOT: !(status === "archived")
.filter((q) => q.not(q.eq(q.field('status'), 'archived')))
```

### TypeScript Filtering

For complex logic not supported by `.filter()`:

```typescript
const messages = await ctx.db
  .query('messages')
  .withIndex('by_channel', (q) => q.eq('channelId', channelId))
  .take(200);

// Complex filtering in TypeScript
const filtered = messages.filter(
  (m) => m.body.length > 10 && !m.metadata?.hidden
);
```

---

## Complex Queries

Convex uses JavaScript for complex logic instead of SQL. Results are always consistent.

### Joins

```typescript
export const eventAttendees = query({
  args: { eventId: v.id('events') },
  handler: async (ctx, { eventId }) => {
    const event = await ctx.db.get(eventId);
    return Promise.all(
      (event?.attendeeIds ?? []).map((userId) => ctx.db.get(userId))
    );
  }
});
```

### Aggregations

```typescript
export const averagePurchasePrice = query({
  args: { email: v.string() },
  handler: async (ctx, { email }) => {
    const purchases = await ctx.db
      .query('purchases')
      .withIndex('by_buyer', (q) => q.eq('buyer', email))
      .collect();
    const sum = purchases.reduce((a, { value: b }) => a + b, 0);
    return sum / purchases.length;
  }
});
```

For large-scale aggregations, use the [Aggregate](https://www.convex.dev/components/aggregate) or [Sharded Counter](https://www.convex.dev/components/sharded-counter) components.

### Group By

```typescript
export const purchasesByBuyer = query({
  args: {},
  handler: async (ctx) => {
    const purchases = await ctx.db.query('purchases').collect();
    return purchases.reduce(
      (counts, { buyer }) => ({
        ...counts,
        [buyer]: (counts[buyer] ?? 0) + 1
      }),
      {} as Record<string, number>
    );
  }
});
```

---

## Pagination

### Backend

```typescript
import { query } from './_generated/server';
import { paginationOptsValidator } from 'convex/server';
import { v } from 'convex/values';

export const listMessages = query({
  args: {
    channelId: v.id('channels'),
    paginationOpts: paginationOptsValidator
  },
  handler: async (ctx, { channelId, paginationOpts }) => {
    return await ctx.db
      .query('messages')
      .withIndex('by_channel', (q) => q.eq('channelId', channelId))
      .order('desc')
      .paginate(paginationOpts);
  }
});
```

### Transforming Paginated Results

```typescript
export const listMessages = query({
  args: { paginationOpts: paginationOptsValidator },
  handler: async (ctx, { paginationOpts }) => {
    const results = await ctx.db
      .query('messages')
      .order('desc')
      .paginate(paginationOpts);

    return {
      ...results,
      page: results.page.map((msg) => ({
        ...msg,
        bodyPreview: msg.body.slice(0, 100)
      }))
    };
  }
});
```

### React Client

```typescript
import { usePaginatedQuery } from 'convex/react';
import { api } from '../convex/_generated/api';

function MessageList({ channelId }) {
  const { results, status, loadMore } = usePaginatedQuery(
    api.messages.listMessages,
    { channelId },
    { initialNumItems: 25 }
  );

  return (
    <>
      {results.map((msg) => <Message key={msg._id} message={msg} />)}
      {status === 'CanLoadMore' && (
        <button onClick={() => loadMore(25)}>Load More</button>
      )}
    </>
  );
}
```

### Pagination Status Values

| Status               | Meaning                  |
| -------------------- | ------------------------ |
| `"LoadingFirstPage"` | Initial load in progress |
| `"CanLoadMore"`      | More results available   |
| `"LoadingMore"`      | Loading next page        |
| `"Exhausted"`        | No more results          |

### Manual Pagination (Non-React)

```typescript
async function getAllMessages(client, channelId) {
  let cursor = null;
  let isDone = false;
  const allResults = [];

  while (!isDone) {
    const {
      page,
      continueCursor,
      isDone: done
    } = await client.query(api.messages.listMessages, {
      channelId,
      paginationOpts: { numItems: 100, cursor }
    });
    allResults.push(...page);
    cursor = continueCursor;
    isDone = done;
  }

  return allResults;
}
```

---

## Writing Data

### Basic Operations

```typescript
// Insert — returns new document ID
const id = await ctx.db.insert('messages', {
  channelId,
  authorId: user._id,
  body: 'Hello world'
});

// Patch — shallow merge (undefined removes field)
await ctx.db.patch(id, {
  edited: true,
  editedAt: Date.now()
});

// Replace — replace all non-system fields
await ctx.db.replace(id, {
  channelId,
  authorId: user._id,
  body: 'Completely new content'
});

// Delete
await ctx.db.delete(id);
```

### Bulk Operations

The entire mutation is a single transaction. Loop inserts are efficient:

```typescript
export const bulkInsertProducts = mutation({
  args: {
    products: v.array(
      v.object({
        name: v.string(),
        price: v.number()
      })
    )
  },
  handler: async (ctx, { products }) => {
    // All inserts are queued and executed in one transaction
    for (const product of products) {
      await ctx.db.insert('products', product);
    }
  }
});
```

### Patch vs Replace

```typescript
// Original: { text: "foo", status: { done: true } }

// Patch: shallow merge
await ctx.db.patch(id, { tag: 'bar', status: { archived: true } });
// Result: { text: "foo", tag: "bar", status: { archived: true } }

// Replace: complete replacement
await ctx.db.replace(id, { text: 'new', status: { fresh: true } });
// Result: { text: "new", status: { fresh: true } }
```

---

## Data Modeling Patterns

### One-to-Many

```typescript
// Schema
messages: defineTable({
  channelId: v.id('channels'),
  body: v.string()
}).index('by_channel', ['channelId']);

// Query children
const messages = await ctx.db
  .query('messages')
  .withIndex('by_channel', (q) => q.eq('channelId', channelId))
  .collect();
```

### Many-to-Many (Junction Table)

```typescript
// Schema
channelMembers: defineTable({
  channelId: v.id('channels'),
  userId: v.id('users'),
  role: v.union(v.literal('admin'), v.literal('member'))
})
  .index('by_channel', ['channelId'])
  .index('by_user', ['userId'])
  .index('by_channel_user', ['channelId', 'userId']);

// Get user's channels
const memberships = await ctx.db
  .query('channelMembers')
  .withIndex('by_user', (q) => q.eq('userId', userId))
  .collect();

const channels = await Promise.all(
  memberships.map((m) => ctx.db.get(m.channelId))
);
```

### Denormalization

Store computed data for efficient reads:

```typescript
// Schema with denormalized counts
channels: defineTable({
  name: v.string(),
  memberCount: v.number(),
  lastMessageAt: v.optional(v.number())
});

// Update denormalized data in mutations
export const sendMessage = mutation({
  handler: async (ctx, { channelId, body }) => {
    await ctx.db.insert('messages', { channelId, body, authorId });
    await ctx.db.patch(channelId, {
      lastMessageAt: Date.now()
    });
  }
});
```

---

## Search

For comprehensive search documentation, see [SEARCH.md](SEARCH.md).

### Full-Text Search (Quick Reference)

```typescript
// Schema
documents: defineTable({
  title: v.string(),
  content: v.string()
}).searchIndex('search_content', {
  searchField: 'content',
  filterFields: ['title']
});

// Query (in queries or mutations — reactive)
const results = await ctx.db
  .query('documents')
  .withSearchIndex('search_content', (q) =>
    q.search('content', searchTerm).eq('title', 'Guide')
  )
  .take(10);
```

### Vector Search (Quick Reference)

```typescript
// Schema
documents: defineTable({
  content: v.string(),
  embedding: v.array(v.float64()),
  category: v.string()
}).vectorIndex('by_embedding', {
  vectorField: 'embedding',
  dimensions: 1536,
  filterFields: ['category']
});

// Query (ACTIONS ONLY — not reactive)
const results = await ctx.vectorSearch('documents', 'by_embedding', {
  vector: queryEmbedding,
  limit: 10,
  filter: (q) => q.eq('category', 'tech')
});
```

**Key difference**: Full-text search works in queries/mutations (reactive). Vector search requires actions (not reactive).

---

## TypeScript Types

### `Doc<TableName>`

```typescript
import { Doc } from '../convex/_generated/dataModel';

function MessageView(props: { message: Doc<'messages'> }) {
  // message has full type information
}
```

### `Id<TableName>`

```typescript
import { Id } from '../convex/_generated/dataModel';

const userId: Id<'users'> = user._id;
```

---

## Limits

| Resource                          | Limit     |
| --------------------------------- | --------- |
| Document size                     | 1MB       |
| Nesting depth                     | 16 levels |
| Array elements                    | 8,192     |
| Object fields                     | 1,024     |
| Fields per index                  | 16        |
| Indexes per table                 | 32        |
| Documents read per query/mutation | 16,384    |
| Documents written per mutation    | 8,192     |

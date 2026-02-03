# Convex Search Reference

## Search Types Overview

| Type             | Available In        | Reactive | Use Case                                  |
| ---------------- | ------------------- | -------- | ----------------------------------------- |
| Full-Text Search | `query`, `mutation` | Yes      | Keyword search, typeahead, search boxes   |
| Vector Search    | `action` only       | No       | Semantic similarity, RAG, recommendations |

---

## Full-Text Search

Full-text search finds documents matching keywords within a string field. It's reactive, consistent, and transactional.

### Defining Search Indexes

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from 'convex/server';
import { v } from 'convex/values';

export default defineSchema({
  messages: defineTable({
    body: v.string(),
    channel: v.string(),
    authorId: v.id('users')
  }).searchIndex('search_body', {
    searchField: 'body',
    filterFields: ['channel', 'authorId']
  }),

  articles: defineTable({
    title: v.string(),
    content: v.string(),
    category: v.string(),
    isPublished: v.boolean()
  }).searchIndex('search_content', {
    searchField: 'content',
    filterFields: ['category', 'isPublished']
  })
});
```

### Search Index Requirements

- **Exactly 1 search field** — must be `v.string()`
- **Up to 16 filter fields** — for equality filtering
- Nested fields supported via dot notation: `"metadata.title"`
- Counts toward 32 indexes per table limit

### Running Search Queries

```typescript
import { query } from './_generated/server';
import { v } from 'convex/values';

export const searchMessages = query({
  args: {
    searchTerm: v.string(),
    channel: v.optional(v.string())
  },
  handler: async (ctx, { searchTerm, channel }) => {
    let searchQuery = ctx.db
      .query('messages')
      .withSearchIndex('search_body', (q) => {
        let search = q.search('body', searchTerm);
        if (channel) {
          search = search.eq('channel', channel);
        }
        return search;
      });

    return await searchQuery.take(20);
  }
});
```

### Search Filter Expression Structure

A search filter expression chains:

1. **Exactly 1 `.search()`** — the full-text search on searchField
2. **0+ `.eq()` conditions** — equality filters on filterFields

```typescript
.withSearchIndex('search_body', (q) =>
  q.search('body', 'hello world')        // Required: search expression
   .eq('channel', '#general')            // Optional: filter by channel
   .eq('authorId', userId)               // Optional: filter by author
)
```

### Typeahead / Prefix Search

The **last term** in your search query automatically enables prefix matching:

```typescript
// User types "hel" → matches "hello", "help", "helicopter"
.withSearchIndex('search_body', (q) => q.search('body', 'hel'))

// User types "hello wor" → "hello" exact match, "wor" prefix matches "world"
.withSearchIndex('search_body', (q) => q.search('body', 'hello wor'))
```

### Additional Filtering with `.filter()`

For filters not in `filterFields`, use `.filter()` after the search:

```typescript
const recentMessages = await ctx.db
  .query('messages')
  .withSearchIndex('search_body', (q) =>
    q.search('body', searchTerm).eq('channel', channel)
  )
  .filter((q) => q.gt(q.field('_creationTime'), Date.now() - 10 * 60000))
  .take(20);
```

**⚠️ Performance**: Put as many filters as possible in `.withSearchIndex()`. The `.filter()` method checks documents one-by-one after the search index returns results.

### Pagination

Search results support standard pagination:

```typescript
import { paginationOptsValidator } from 'convex/server';

export const searchMessagesPaginated = query({
  args: {
    searchTerm: v.string(),
    paginationOpts: paginationOptsValidator
  },
  handler: async (ctx, { searchTerm, paginationOpts }) => {
    return await ctx.db
      .query('messages')
      .withSearchIndex('search_body', (q) => q.search('body', searchTerm))
      .paginate(paginationOpts);
  }
});
```

### Full-Text Search Limits

| Limit                        | Value         |
| ---------------------------- | ------------- |
| Search fields per index      | 1             |
| Filter fields per index      | 16            |
| Terms (words) per query      | 16            |
| Filter expressions per query | 8             |
| Max results scanned          | 1024          |
| Max term length              | 32 characters |

---

## Vector Search

Vector search finds documents similar to a provided vector (embedding). Essential for RAG, semantic search, and recommendations.

**⚠️ Actions Only**: Vector search is only available in actions, not queries or mutations.

### Defining Vector Indexes

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from 'convex/server';
import { v } from 'convex/values';

export default defineSchema({
  documents: defineTable({
    content: v.string(),
    embedding: v.array(v.float64()),
    category: v.string(),
    authorId: v.id('users')
  }).vectorIndex('by_embedding', {
    vectorField: 'embedding',
    dimensions: 1536, // Must match your embedding model (e.g., OpenAI)
    filterFields: ['category', 'authorId']
  })
});
```

### Vector Index Requirements

- **Exactly 1 vector field** — `v.array(v.float64())`
- **Dimensions**: 2–4096 (must match embedding model)
- **Up to 16 filter fields**
- **Max 4 vector indexes per table**
- Counts toward 32 indexes per table limit

### Running Vector Searches

```typescript
import { action, internalQuery } from './_generated/server';
import { internal } from './_generated/api';
import { v } from 'convex/values';

// Action performs vector search
export const searchSimilar = action({
  args: {
    query: v.string(),
    category: v.optional(v.string())
  },
  handler: async (ctx, { query, category }) => {
    // 1. Generate embedding from your provider
    const embedding = await generateEmbedding(query);

    // 2. Perform vector search
    const results = await ctx.vectorSearch('documents', 'by_embedding', {
      vector: embedding,
      limit: 10,
      filter: category ? (q) => q.eq('category', category) : undefined
    });

    // 3. Load full documents
    const documents = await ctx.runQuery(internal.documents.getByIds, {
      ids: results.map((r) => r._id)
    });

    return documents;
  }
});

// Internal query to load documents
export const getByIds = internalQuery({
  args: { ids: v.array(v.id('documents')) },
  handler: async (ctx, { ids }) => {
    const docs = [];
    for (const id of ids) {
      const doc = await ctx.db.get(id);
      if (doc) docs.push(doc);
    }
    return docs;
  }
});
```

### Vector Search API

```typescript
const results = await ctx.vectorSearch('tableName', 'indexName', {
  vector: number[],           // Required: query vector (same dimensions as index)
  limit?: number,             // Optional: 1-256, default 10
  filter?: (q) => Expression  // Optional: equality filters
});

// Returns: Array<{ _id: Id<"tableName">, _score: number }>
// _score ranges from -1 (least similar) to 1 (most similar)
```

### Vector Search Filter Expressions

Filters support `.eq()` and `.or()`:

```typescript
// Single equality
filter: (q) => q.eq('category', 'tech');

// OR on same field
filter: (q) => q.or(q.eq('category', 'tech'), q.eq('category', 'science'));

// OR across different fields (both must be in filterFields)
filter: (q) => q.or(q.eq('category', 'tech'), q.eq('authorId', userId));
```

### Score-Based Filtering

Filter by similarity score in your action code:

```typescript
const results = await ctx.vectorSearch('documents', 'by_embedding', {
  vector: embedding,
  limit: 50
});

// Keep only highly similar results
const highQuality = results.filter((r) => r._score >= 0.8);
```

### Advanced Pattern: Separate Embedding Table

For large documents, store embeddings separately to avoid loading them on every read:

```typescript
// convex/schema.ts
export default defineSchema({
  // Embeddings stored separately
  documentEmbeddings: defineTable({
    embedding: v.array(v.float64()),
    category: v.string()
  }).vectorIndex('by_embedding', {
    vectorField: 'embedding',
    dimensions: 1536,
    filterFields: ['category']
  }),

  // Documents reference embeddings
  documents: defineTable({
    title: v.string(),
    content: v.string(),
    category: v.string(),
    embeddingId: v.optional(v.id('documentEmbeddings'))
  }).index('by_embedding', ['embeddingId'])
});
```

```typescript
// Load documents from embedding IDs
export const getDocumentsByEmbeddingIds = internalQuery({
  args: { ids: v.array(v.id('documentEmbeddings')) },
  handler: async (ctx, { ids }) => {
    const docs = [];
    for (const embeddingId of ids) {
      const doc = await ctx.db
        .query('documents')
        .withIndex('by_embedding', (q) => q.eq('embeddingId', embeddingId))
        .unique();
      if (doc) docs.push(doc);
    }
    return docs;
  }
});
```

### Vector Search Limits

| Limit                        | Value  |
| ---------------------------- | ------ |
| Vector indexes per table     | 4      |
| Dimensions                   | 2–4096 |
| Filter fields per index      | 16     |
| Filter expressions per query | 64     |
| Max results (limit)          | 256    |

---

## Comparison: Full-Text vs Vector Search

| Feature               | Full-Text Search        | Vector Search          |
| --------------------- | ----------------------- | ---------------------- |
| Available in          | Queries, Mutations      | Actions only           |
| Reactive              | ✅ Yes                  | ❌ No                  |
| Search type           | Keyword/term matching   | Semantic similarity    |
| Index field type      | `v.string()`            | `v.array(v.float64())` |
| Max indexes per table | 32 (shared)             | 4                      |
| Pagination            | ✅ Supported            | ❌ Use limit           |
| Ordering              | Relevance (BM25)        | Similarity (cosine)    |
| Use case              | Search boxes, typeahead | RAG, recommendations   |

---

## Common Patterns

### Hybrid Search (Text + Vector)

Combine both for best results:

```typescript
export const hybridSearch = action({
  args: { query: v.string() },
  handler: async (ctx, { query }) => {
    // Vector search for semantic matches
    const embedding = await generateEmbedding(query);
    const vectorResults = await ctx.vectorSearch('articles', 'by_embedding', {
      vector: embedding,
      limit: 20
    });

    // Text search for exact keyword matches
    const textResults = await ctx.runQuery(internal.articles.textSearch, {
      query
    });

    // Merge and deduplicate results
    const allIds = new Set([
      ...vectorResults.map((r) => r._id),
      ...textResults.map((r) => r._id)
    ]);

    return await ctx.runQuery(internal.articles.getByIds, {
      ids: Array.from(allIds)
    });
  }
});
```

### RAG Pattern

```typescript
export const askQuestion = action({
  args: { question: v.string() },
  handler: async (ctx, { question }) => {
    // 1. Embed the question
    const embedding = await generateEmbedding(question);

    // 2. Find relevant context
    const results = await ctx.vectorSearch('knowledge', 'by_embedding', {
      vector: embedding,
      limit: 5
    });

    // 3. Load context documents
    const context = await ctx.runQuery(internal.knowledge.getByIds, {
      ids: results.map((r) => r._id)
    });

    // 4. Generate answer with context
    const answer = await generateAnswer(question, context);

    // 5. Optionally save the interaction
    await ctx.runMutation(internal.conversations.save, {
      question,
      answer,
      contextIds: results.map((r) => r._id)
    });

    return answer;
  }
});
```

---

## Best Practices

### Full-Text Search

- ✅ Use `filterFields` for common equality filters (much faster than `.filter()`)
- ✅ Use `.take(n)` or pagination instead of `.collect()` (1024 doc limit)
- ✅ Design for typeahead — users expect instant results as they type
- ❌ Don't put all your filtering in `.filter()` — use `filterFields`

### Vector Search

- ✅ Store embeddings in a separate table for large documents
- ✅ Use `filterFields` to narrow search space before vector comparison
- ✅ Filter by `_score` for quality control
- ✅ Cache embeddings — don't regenerate for the same content
- ❌ Don't call vector search from queries/mutations (actions only)
- ❌ Don't return raw embeddings to clients (large, not useful)

### Index Design

```typescript
// ✅ Good: filterFields match your common query patterns
.searchIndex('search_messages', {
  searchField: 'body',
  filterFields: ['channelId', 'authorId', 'isDeleted']
})

// ❌ Bad: Missing common filter, will need slow .filter()
.searchIndex('search_messages', {
  searchField: 'body',
  filterFields: []  // Now every filter is slow
})
```

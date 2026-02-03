# RAG (Retrieval-Augmented Generation)

Enhance LLM responses with custom knowledge bases.

## Built-in Message Search

The Agent has built-in hybrid text & vector search for message history:

```typescript
const agent = new Agent(components.agent, {
  languageModel: openai.chat('gpt-4o-mini'),
  textEmbeddingModel: openai.embedding('text-embedding-3-small'),
  contextOptions: {
    searchOptions: {
      textSearch: true,
      vectorSearch: true,
      limit: 10,
    },
    searchOtherThreads: true, // Search user's other threads
  },
});
```

## RAG Component

For custom knowledge bases, use the RAG component:

```bash
npm install @convex-dev/rag
```

```typescript
// convex/convex.config.ts
import rag from '@convex-dev/rag/convex.config';
app.use(rag);
```

### Setup

```typescript
import { RAG } from '@convex-dev/rag';

const rag = new RAG(components.rag, {
  textEmbeddingModel: openai.embedding('text-embedding-3-small'),
});
```

### Add Content

```typescript
await rag.add(ctx, {
  namespace: 'global',     // or userId for user-specific
  key: 'doc-123',          // unique key for updates
  text: 'Document content...',
  importance: 0.8,         // 0-1, affects search ranking
  metadata: { source: 'wiki' },
});
```

### Search

```typescript
const results = await rag.search(ctx, {
  namespace: 'global',
  query: 'user question',
  limit: 10,
});
// results.text contains combined relevant chunks
```

## RAG Approaches

### 1. Prompt-based RAG

Always inject context into prompts:

```typescript
export const askQuestion = action({
  args: { threadId: v.string(), prompt: v.string() },
  handler: async (ctx, { threadId, prompt }) => {
    // Search for context
    const context = await rag.search(ctx, {
      namespace: 'global',
      query: prompt,
      limit: 10,
    });

    // Inject context into prompt
    const result = await agent.generateText(ctx, { threadId }, {
      prompt: `# Context:\n${context.text}\n\n---\n\n# Question:\n${prompt}`,
    });

    return result.text;
  },
});
```

**Pros:** Simple, predictable
**Cons:** Always searches even when not needed

### 2. Tool-based RAG

Let LLM decide when to search:

```typescript
const searchContext = createTool({
  description: 'Search knowledge base for relevant information',
  args: z.object({
    query: z.string().describe('Search query'),
  }),
  handler: async (ctx, { query }): Promise<string> => {
    const results = await rag.search(ctx, {
      namespace: ctx.userId ?? 'global',
      query,
      limit: 10,
    });
    return results.text;
  },
});

const addKnowledge = createTool({
  description: 'Add new information to knowledge base',
  args: z.object({
    content: z.string().describe('Information to store'),
    key: z.string().describe('Unique identifier'),
  }),
  handler: async (ctx, { content, key }): Promise<string> => {
    await rag.add(ctx, {
      namespace: ctx.userId ?? 'global',
      key,
      text: content,
    });
    return 'Knowledge saved successfully';
  },
});

const agent = new Agent(components.agent, {
  languageModel: openai.chat('gpt-4o-mini'),
  tools: { searchContext, addKnowledge },
  stopWhen: stepCountIs(5),
});
```

**Pros:** LLM decides when to search, can add knowledge
**Cons:** Less predictable, uses more tokens

## Ingesting Content

### Text Files

```typescript
await rag.add(ctx, {
  namespace: 'docs',
  key: `file-${fileId}`,
  text: fileContent,
});
```

### Images

Use LLM to describe images:

```typescript
const description = await generateText({
  model: openai.chat('gpt-4o'),
  messages: [{
    role: 'user',
    content: [
      { type: 'image', image: imageUrl },
      { type: 'text', text: 'Describe this image in detail' },
    ],
  }],
});

await rag.add(ctx, {
  namespace: 'images',
  key: `image-${imageId}`,
  text: description.text,
  metadata: { imageUrl },
});
```

### PDFs

Parse client-side with PDF.js (recommended) or server-side with LLM:

```typescript
// Server-side (slower, uses tokens)
const result = await generateText({
  model: openai.chat('gpt-4o'),
  messages: [{
    role: 'user',
    content: [
      { type: 'file', data: pdfUrl, mimeType: 'application/pdf' },
      { type: 'text', text: 'Extract all text from this PDF' },
    ],
  }],
});

await rag.add(ctx, {
  namespace: 'pdfs',
  key: `pdf-${pdfId}`,
  text: result.text,
});
```

## Advanced RAG Features

### Namespaces

Isolate search domains:

```typescript
// Global knowledge
await rag.add(ctx, { namespace: 'global', ... });

// Per-user knowledge
await rag.add(ctx, { namespace: userId, ... });

// Per-team knowledge
await rag.add(ctx, { namespace: `team-${teamId}`, ... });
```

### Custom Filters

Filter during vector search:

```typescript
await rag.add(ctx, {
  namespace: 'docs',
  key: 'doc-123',
  text: content,
  metadata: { category: 'billing', date: '2024-01' },
});

const results = await rag.search(ctx, {
  namespace: 'docs',
  query: prompt,
  filter: { category: 'billing' },
});
```

### Chunk Context

Get surrounding chunks:

```typescript
const results = await rag.search(ctx, {
  namespace: 'docs',
  query: prompt,
  includeChunkContext: true, // Include surrounding chunks
});
```

### Importance Weighting

Prioritize certain documents:

```typescript
await rag.add(ctx, {
  namespace: 'docs',
  key: 'important-doc',
  text: content,
  importance: 1.0, // High priority
});

await rag.add(ctx, {
  namespace: 'docs',
  key: 'background-doc',
  text: content,
  importance: 0.3, // Lower priority
});
```

## Combining Agent History with RAG

```typescript
const agent = new Agent(components.agent, {
  languageModel: openai.chat('gpt-4o-mini'),
  textEmbeddingModel: openai.embedding('text-embedding-3-small'),
  tools: { searchKnowledge },
  contextOptions: {
    searchOptions: {
      vectorSearch: true, // Search message history
      limit: 5,
    },
  },
});

// Agent can search both message history (automatic) 
// and knowledge base (via tool)
```
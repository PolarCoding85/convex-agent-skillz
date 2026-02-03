# ProseMirror Sync Component

Real-time collaborative document editing with Tiptap or BlockNote.

## Installation

```bash
npm install @convex-dev/prosemirror-sync
```

```typescript
// convex/convex.config.ts
import prosemirrorSync from '@convex-dev/prosemirror-sync/convex.config';
app.use(prosemirrorSync);
```

## Overview

Syncs ProseMirror documents between multiple clients using operational transformations (OT). Works with:

- **Tiptap** - Flexible, extensible editor based on ProseMirror
- **BlockNote** - Notion-like block editor with great UX out of the box

Data lives in your Convex database alongside your app data.

## Backend Setup

```typescript
// convex/prosemirror.ts
import { components } from './_generated/api';
import { ProsemirrorSync } from '@convex-dev/prosemirror-sync';

const prosemirrorSync = new ProsemirrorSync(components.prosemirrorSync);

export const {
  getSnapshot,
  submitSnapshot,
  latestVersion,
  getSteps,
  submitSteps
} = prosemirrorSync.syncApi({
  // Authorization hooks (optional but recommended)
  checkRead(ctx, id) {
    // Validate user can read this document
    // const user = await ctx.auth.getUserIdentity();
  },
  checkWrite(ctx, id) {
    // Validate user can write to this document
  },
  async onSnapshot(ctx, id, snapshot, version) {
    // React to snapshots (e.g., extract text for search)
  }
});
```

## React Setup

### With Tiptap

```tsx
import { useTiptapSync } from '@convex-dev/prosemirror-sync/tiptap';
import { EditorContent, EditorProvider } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import { api } from '../convex/_generated/api';

function TiptapEditor({ docId }: { docId: string }) {
  const sync = useTiptapSync(api.prosemirror, docId);

  if (sync.isLoading) {
    return <p>Loading...</p>;
  }

  if (sync.initialContent === null) {
    return (
      <button onClick={() => sync.create({ type: 'doc', content: [] })}>
        Create document
      </button>
    );
  }

  return (
    <EditorProvider
      content={sync.initialContent}
      extensions={[StarterKit, sync.extension]}
    >
      <EditorContent editor={null} />
    </EditorProvider>
  );
}
```

### With BlockNote

```tsx
import { useBlockNoteSync } from '@convex-dev/prosemirror-sync/blocknote';
import { BlockNoteView } from '@blocknote/mantine';
import '@blocknote/mantine/style.css';
import { api } from '../convex/_generated/api';

const EMPTY_DOC = { type: 'doc', content: [] };

function BlockNoteEditor({ docId }: { docId: string }) {
  const sync = useBlockNoteSync(api.prosemirror, docId);

  if (sync.isLoading) {
    return <p>Loading...</p>;
  }

  if (!sync.editor) {
    return (
      <button onClick={() => sync.create(EMPTY_DOC)}>Create document</button>
    );
  }

  return <BlockNoteView editor={sync.editor} />;
}
```

**Note:** BlockNote doesn't support React 19 `<StrictMode>`. Remove `<React.StrictMode>` blocks and set `reactStrictMode: false` in Next.js config.

## Authorization

```typescript
export const { ... } = prosemirrorSync.syncApi({
  async checkRead(ctx, id) {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");

    // Example: Check document access
    const doc = await ctx.db.get(id);
    if (!doc.members.includes(identity.subject)) {
      throw new Error("Access denied");
    }
  },

  async checkWrite(ctx, id) {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");

    // Example: Check write permission
    const doc = await ctx.db.get(id);
    if (doc.status === "locked") {
      throw new Error("Document is locked");
    }
  },
});
```

## Snapshot Hooks

React to document changes for indexing, search, or webhooks:

```typescript
export const { ... } = prosemirrorSync.syncApi({
  async onSnapshot(ctx, id, snapshot, version) {
    // Extract text content for search
    const schema = getSchema(extensions);
    const node = schema.nodeFromJSON(JSON.parse(snapshot));
    const textContent = node.textContent;

    // Store for full-text search
    await ctx.db.patch(id, { searchText: textContent });
  },
});
```

### Extract Text Content

**Tiptap:**

```typescript
import { getSchema } from '@tiptap/core';

const schema = getSchema(extensions);
const node = schema.nodeFromJSON(JSON.parse(snapshot));
const content = node.textContent;
```

**BlockNote:**

```typescript
import { BlockNoteEditor } from '@blocknote/core';

const editor = BlockNoteEditor.create({ _headless: true });
const node = editor.pmSchema.nodeFromJSON(JSON.parse(snapshot));
const content = node.textContent;
```

## Server-Side Transforms

Modify documents from the server (e.g., AI-generated content):

```typescript
import { getSchema } from '@tiptap/core';
import { EditorState } from '@tiptap/pm/state';

export const insertAIContent = action({
  args: { id: v.string(), content: v.string() },
  handler: async (ctx, { id, content }) => {
    const schema = getSchema(extensions);

    await prosemirrorSync.transform(ctx, id, schema, (doc) => {
      const tr = EditorState.create({ doc }).tr;
      tr.insertText(content, 0);
      return tr;
    });
  }
});
```

### With Version Checking

```typescript
import { Transform } from '@tiptap/pm/transform';

export const insertWithCheck = action({
  args: { id: v.string() },
  handler: async (ctx, { id }) => {
    const schema = getSchema(extensions);

    // Get current state
    const { doc, version } = await prosemirrorSync.getDoc(ctx, id, schema);

    // Do slow AI work
    const aiContent = await generateAIContent(doc);

    // Apply transform
    await prosemirrorSync.transform(ctx, id, schema, (doc, v) => {
      if (v !== version) {
        // Document changed during AI processing
        // Decide: abort, merge, or overwrite
      }

      const tr = new Transform(doc);
      tr.insert(0, schema.text(aiContent));
      return tr;
    });
  }
});
```

## Creating Documents

### Client-Side

```typescript
// Wait for load to check if document exists
if (!sync.isLoading && sync.initialContent === null) {
  sync.create({ type: 'doc', content: [] });
}
```

### Server-Side

```typescript
await prosemirrorSync.create(ctx, 'doc-id', { type: 'doc', content: [] });
```

## Tiptap vs BlockNote

| Feature             | Tiptap           | BlockNote       |
| ------------------- | ---------------- | --------------- |
| Customization       | ⭐⭐⭐ High      | ⭐⭐ Medium     |
| Out-of-box UX       | ⭐⭐ Medium      | ⭐⭐⭐ Polished |
| Extension ecosystem | ⭐⭐⭐ Large     | ⭐⭐ Growing    |
| Learning curve      | ⭐⭐ Medium      | ⭐⭐⭐ Easy     |
| Data model          | ProseMirror JSON | Custom blocks   |

**Important:** Data models differ between Tiptap and BlockNote. Switching editors later requires data migration. Experiment before launching.

## Styling

### Tiptap with Tailwind

```tsx
<EditorProvider
  content={sync.initialContent}
  extensions={[StarterKit, sync.extension]}
  editorProps={{
    attributes: {
      class: "prose prose-sm sm:prose lg:prose-lg focus:outline-none",
    },
  }}
>
```

Install `@tailwindcss/typography` plugin.

### BlockNote with shadcn

```bash
npm install @blocknote/shadcn
```

```tsx
import { BlockNoteView } from '@blocknote/shadcn';
import '@blocknote/shadcn/style.css';
```

## How It Works

1. **Initial Load**: Client fetches latest snapshot
2. **Local Changes**: Detected and submitted as "steps"
3. **Remote Changes**: Fetched and applied via OT
4. **Snapshots**: Debounced saves reduce history size
5. **Conflict Resolution**: OT merges concurrent edits

## Data Storage

Component stores:

- **Snapshots**: Full document state at intervals
- **Steps**: Incremental changes between snapshots

New clients load latest snapshot + subsequent steps.

## Cleanup

Delete old snapshots and steps to save storage:

```typescript
// Delete steps older than snapshot
await prosemirrorSync.deleteSteps(ctx, id, upToVersion);
```

## Best Practices

1. **Use consistent extensions** - Server schema must match client
2. **Authorize access** - Implement `checkRead`/`checkWrite`
3. **Extract text** - Use `onSnapshot` for search indexing
4. **Handle offline** - Documents can be created offline
5. **Test collaboration** - Open multiple tabs with same URL
6. **Choose editor early** - Switching requires data migration

## Limitations

- Documents limited to ~1MB
- No offline sync between browser tabs
- No Yjs support (use ProseMirror OT instead)
- BlockNote incompatible with React 19 StrictMode

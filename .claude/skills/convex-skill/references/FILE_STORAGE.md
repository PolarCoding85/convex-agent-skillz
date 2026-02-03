# Convex File Storage Reference

## Overview

File Storage enables uploading, storing, serving, and deleting files in Convex. All file types are supported. Files are stored in the `_storage` system table and referenced by `Id<"_storage">`.

## Quick Reference

| Operation           | Context               | Method                            | Notes                         |
| ------------------- | --------------------- | --------------------------------- | ----------------------------- |
| Generate upload URL | Mutation              | `ctx.storage.generateUploadUrl()` | Returns URL for client upload |
| Store file          | Action/HTTP Action    | `ctx.storage.store(blob)`         | Returns `Id<"_storage">`      |
| Get file URL        | Query/Mutation/Action | `ctx.storage.getUrl(storageId)`   | Returns serving URL           |
| Get file blob       | Action/HTTP Action    | `ctx.storage.get(storageId)`      | Returns `Blob \| null`        |
| Delete file         | Mutation/Action       | `ctx.storage.delete(storageId)`   | Permanently removes file      |
| Get metadata        | Query/Mutation        | `ctx.db.system.get(storageId)`    | Returns file metadata         |
| List all files      | Query/Mutation        | `ctx.db.system.query("_storage")` | Query system table            |

---

## Uploading Files

### Method 1: Upload URLs (Recommended for Large Files)

Best for arbitrarily large files. Uses 3-step process:

```typescript
// convex/files.ts
import { mutation } from './_generated/server';
import { v } from 'convex/values';

// Step 1: Generate upload URL
export const generateUploadUrl = mutation({
  args: {},
  handler: async (ctx) => {
    // Can add auth check here to control who can upload
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Unauthorized');

    return await ctx.storage.generateUploadUrl();
  }
});

// Step 3: Save storage ID to database
export const saveFile = mutation({
  args: {
    storageId: v.id('_storage'),
    fileName: v.string()
  },
  handler: async (ctx, { storageId, fileName }) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Unauthorized');

    return await ctx.db.insert('files', {
      storageId,
      fileName,
      uploadedBy: identity.subject,
      uploadedAt: Date.now()
    });
  }
});
```

**Client-side upload (React):**

```typescript
// src/UploadForm.tsx
import { useMutation } from 'convex/react';
import { api } from '../convex/_generated/api';
import { FormEvent, useRef, useState } from 'react';

export function UploadForm() {
  const generateUploadUrl = useMutation(api.files.generateUploadUrl);
  const saveFile = useMutation(api.files.saveFile);
  const fileInput = useRef<HTMLInputElement>(null);
  const [uploading, setUploading] = useState(false);

  async function handleUpload(event: FormEvent) {
    event.preventDefault();
    const file = fileInput.current?.files?.[0];
    if (!file) return;

    setUploading(true);
    try {
      // Step 1: Get upload URL
      const uploadUrl = await generateUploadUrl();

      // Step 2: POST file to upload URL
      const result = await fetch(uploadUrl, {
        method: 'POST',
        headers: { 'Content-Type': file.type },
        body: file
      });
      const { storageId } = await result.json();

      // Step 3: Save to database
      await saveFile({ storageId, fileName: file.name });
    } finally {
      setUploading(false);
      if (fileInput.current) fileInput.current.value = '';
    }
  }

  return (
    <form onSubmit={handleUpload}>
      <input type="file" ref={fileInput} disabled={uploading} />
      <button type="submit" disabled={uploading}>
        {uploading ? 'Uploading...' : 'Upload'}
      </button>
    </form>
  );
}
```

### Method 2: HTTP Action Upload (≤20MB)

Single request, but limited to 20MB:

```typescript
// convex/http.ts
import { httpRouter } from 'convex/server';
import { httpAction } from './_generated/server';
import { internal } from './_generated/api';

const http = httpRouter();

http.route({
  path: '/upload',
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    // Store file directly
    const blob = await request.blob();
    const storageId = await ctx.storage.store(blob);

    // Get metadata from headers/params
    const fileName = request.headers.get('X-File-Name') || 'unnamed';

    // Save to database
    await ctx.runMutation(internal.files.saveFile, { storageId, fileName });

    return new Response(JSON.stringify({ storageId }), {
      status: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': process.env.CLIENT_ORIGIN!
      }
    });
  })
});

// Handle CORS preflight
http.route({
  path: '/upload',
  method: 'OPTIONS',
  handler: httpAction(async (_, request) => {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': process.env.CLIENT_ORIGIN!,
        'Access-Control-Allow-Methods': 'POST',
        'Access-Control-Allow-Headers': 'Content-Type, X-File-Name',
        'Access-Control-Max-Age': '86400'
      }
    });
  })
});

export default http;
```

---

## Storing Files from Actions

Store files fetched from external APIs or generated server-side:

```typescript
// convex/images.ts
import { action, internalMutation } from './_generated/server';
import { internal } from './_generated/api';
import { v } from 'convex/values';

export const generateAndStore = action({
  args: { prompt: v.string() },
  handler: async (ctx, { prompt }) => {
    // Generate or fetch image from external API
    const response = await fetch(
      'https://api.openai.com/v1/images/generations',
      {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ prompt, size: '1024x1024' })
      }
    );

    const { data } = await response.json();
    const imageUrl = data[0].url;

    // Download the image
    const imageResponse = await fetch(imageUrl);
    const imageBlob = await imageResponse.blob();

    // Store in Convex
    const storageId = await ctx.storage.store(imageBlob);

    // Save to database
    await ctx.runMutation(internal.images.saveGenerated, {
      storageId,
      prompt
    });

    return storageId;
  }
});

export const saveGenerated = internalMutation({
  args: {
    storageId: v.id('_storage'),
    prompt: v.string()
  },
  handler: async (ctx, { storageId, prompt }) => {
    await ctx.db.insert('generatedImages', {
      storageId,
      prompt,
      createdAt: Date.now()
    });
  }
});
```

---

## Serving Files

### Method 1: URL from Queries (Simplest)

Generate URLs in queries for reactive file serving:

```typescript
// convex/files.ts
import { query } from './_generated/server';
import { v } from 'convex/values';

export const getFileUrl = query({
  args: { storageId: v.id('_storage') },
  handler: async (ctx, { storageId }) => {
    return await ctx.storage.getUrl(storageId);
  }
});

export const listFilesWithUrls = query({
  args: {},
  handler: async (ctx) => {
    const files = await ctx.db.query('files').collect();

    return Promise.all(
      files.map(async (file) => ({
        ...file,
        url: await ctx.storage.getUrl(file.storageId)
      }))
    );
  }
});
```

**React usage:**

```typescript
function FileList() {
  const files = useQuery(api.files.listFilesWithUrls);

  return (
    <ul>
      {files?.map((file) => (
        <li key={file._id}>
          {file.fileName}
          {file.url && <img src={file.url} alt={file.fileName} />}
        </li>
      ))}
    </ul>
  );
}
```

### Method 2: HTTP Action Serving (Access Control)

For fine-grained access control at serve time (≤20MB):

```typescript
// convex/http.ts
import { httpRouter } from 'convex/server';
import { httpAction } from './_generated/server';
import { Id } from './_generated/dataModel';

const http = httpRouter();

http.route({
  path: '/file',
  method: 'GET',
  handler: httpAction(async (ctx, request) => {
    const { searchParams } = new URL(request.url);
    const storageId = searchParams.get('id') as Id<'_storage'>;

    if (!storageId) {
      return new Response('Missing storage ID', { status: 400 });
    }

    // Optional: Add auth check
    // const token = request.headers.get('Authorization');
    // if (!await verifyToken(token)) {
    //   return new Response('Unauthorized', { status: 401 });
    // }

    const blob = await ctx.storage.get(storageId);
    if (!blob) {
      return new Response('File not found', { status: 404 });
    }

    return new Response(blob, {
      headers: {
        'Content-Type': blob.type || 'application/octet-stream',
        'Cache-Control': 'public, max-age=31536000'
      }
    });
  })
});

export default http;
```

---

## Deleting Files

```typescript
// convex/files.ts
import { mutation, internalMutation } from './_generated/server';
import { internal } from './_generated/api';
import { v } from 'convex/values';

export const deleteFile = mutation({
  args: { fileId: v.id('files') },
  handler: async (ctx, { fileId }) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Unauthorized');

    const file = await ctx.db.get(fileId);
    if (!file) throw new Error('File not found');

    // Delete from storage
    await ctx.storage.delete(file.storageId);

    // Delete database record
    await ctx.db.delete(fileId);
  }
});

// For cleanup jobs
export const deleteOrphanedFiles = internalMutation({
  args: {},
  handler: async (ctx) => {
    // Get all storage IDs referenced in files table
    const files = await ctx.db.query('files').collect();
    const referencedIds = new Set(files.map((f) => f.storageId));

    // Get all stored files
    const storedFiles = await ctx.db.system.query('_storage').collect();

    // Delete unreferenced files
    for (const stored of storedFiles) {
      if (!referencedIds.has(stored._id)) {
        await ctx.storage.delete(stored._id);
      }
    }
  }
});
```

---

## File Metadata

Access file metadata via the `_storage` system table:

```typescript
// convex/files.ts
import { query } from './_generated/server';
import { v } from 'convex/values';

export const getMetadata = query({
  args: { storageId: v.id('_storage') },
  handler: async (ctx, { storageId }) => {
    return await ctx.db.system.get(storageId);
  }
});

export const listAllStoredFiles = query({
  args: {},
  handler: async (ctx) => {
    return await ctx.db.system.query('_storage').collect();
  }
});

export const getFilesOverSize = query({
  args: { minBytes: v.number() },
  handler: async (ctx, { minBytes }) => {
    const files = await ctx.db.system.query('_storage').collect();
    return files.filter((f) => f.size >= minBytes);
  }
});
```

**Metadata document structure:**

```typescript
{
  _id: Id<"_storage">,       // Storage ID
  _creationTime: number,     // Upload timestamp (ms)
  contentType?: string,      // MIME type (if provided)
  sha256: string,            // Base16 encoded checksum
  size: number               // File size in bytes
}
```

---

## Schema Patterns

### Basic File Reference

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from 'convex/server';
import { v } from 'convex/values';

export default defineSchema({
  files: defineTable({
    storageId: v.id('_storage'),
    fileName: v.string(),
    uploadedBy: v.string(),
    uploadedAt: v.number()
  }).index('by_uploader', ['uploadedBy']),

  // User profile with avatar
  users: defineTable({
    name: v.string(),
    avatarId: v.optional(v.id('_storage'))
  }),

  // Message with optional attachment
  messages: defineTable({
    content: v.string(),
    authorId: v.id('users'),
    attachmentId: v.optional(v.id('_storage')),
    attachmentType: v.optional(
      v.union(v.literal('image'), v.literal('document'), v.literal('video'))
    )
  })
});
```

### Multiple Attachments

```typescript
// convex/schema.ts
export default defineSchema({
  posts: defineTable({
    content: v.string(),
    authorId: v.id('users')
  }),

  // Separate table for attachments (one-to-many)
  attachments: defineTable({
    postId: v.id('posts'),
    storageId: v.id('_storage'),
    fileName: v.string(),
    fileType: v.string(),
    order: v.number()
  }).index('by_post', ['postId'])
});
```

---

## Complete Example: Image Gallery

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from 'convex/server';
import { v } from 'convex/values';

export default defineSchema({
  images: defineTable({
    storageId: v.id('_storage'),
    title: v.string(),
    description: v.optional(v.string()),
    uploadedBy: v.id('users'),
    uploadedAt: v.number(),
    tags: v.array(v.string())
  })
    .index('by_uploader', ['uploadedBy'])
    .index('by_uploadedAt', ['uploadedAt'])
});
```

```typescript
// convex/images.ts
import { query, mutation } from './_generated/server';
import { v } from 'convex/values';
import * as Users from './model/users';

export const generateUploadUrl = mutation({
  args: {},
  handler: async (ctx) => {
    await Users.requireUser(ctx);
    return await ctx.storage.generateUploadUrl();
  }
});

export const create = mutation({
  args: {
    storageId: v.id('_storage'),
    title: v.string(),
    description: v.optional(v.string()),
    tags: v.array(v.string())
  },
  handler: async (ctx, args) => {
    const user = await Users.requireUser(ctx);

    return await ctx.db.insert('images', {
      storageId: args.storageId,
      title: args.title,
      description: args.description,
      uploadedBy: user._id,
      uploadedAt: Date.now(),
      tags: args.tags
    });
  }
});

export const list = query({
  args: {
    limit: v.optional(v.number())
  },
  handler: async (ctx, { limit = 50 }) => {
    const images = await ctx.db
      .query('images')
      .withIndex('by_uploadedAt')
      .order('desc')
      .take(limit);

    return Promise.all(
      images.map(async (image) => ({
        ...image,
        url: await ctx.storage.getUrl(image.storageId)
      }))
    );
  }
});

export const remove = mutation({
  args: { imageId: v.id('images') },
  handler: async (ctx, { imageId }) => {
    const user = await Users.requireUser(ctx);
    const image = await ctx.db.get(imageId);

    if (!image) throw new Error('Image not found');
    if (image.uploadedBy !== user._id) throw new Error('Unauthorized');

    // Delete file and record
    await ctx.storage.delete(image.storageId);
    await ctx.db.delete(imageId);
  }
});
```

---

## Limits

| Limit                        | Value               |
| ---------------------------- | ------------------- |
| Upload URL expiry            | 1 hour              |
| Upload POST timeout          | 2 minutes           |
| HTTP Action request/response | 20MB                |
| Upload via URL               | Unlimited file size |

## Best Practices

### DO ✅

- Validate storage IDs with `v.id("_storage")`
- Check auth before generating upload URLs
- Delete storage files when deleting database records
- Use upload URLs for files >20MB
- Store file metadata (name, type) in your own tables

### DON'T ❌

- Don't expose `generateUploadUrl` without auth
- Don't forget to handle CORS for HTTP action uploads
- Don't store sensitive files without access control
- Don't assume files exist — check for null from `getUrl`/`get`
- Don't use HTTP actions for large files (>20MB)

# Files and Images

Send images and files to LLMs via Agent messages.

## Recommended Pattern

Upload first, then generate asynchronously:

### 1. Save File

```typescript
import { storeFile } from '@convex-dev/agent';

export const uploadFile = action({
  args: {
    /* file data */
  },
  handler: async (ctx, args) => {
    const { file } = await storeFile(
      ctx,
      components.agent,
      new Blob([args.bytes], { type: args.mimeType }),
      {
        filename: args.filename,
        sha256: args.sha256 // Optional for deduplication
      }
    );
    return file; // { fileId, url, storageId }
  }
});
```

### 2. Send Message with File

```typescript
import { getFile, saveMessage } from '@convex-dev/agent';

export const submitQuestion = mutation({
  args: { threadId: v.string(), fileId: v.string(), question: v.string() },
  handler: async (ctx, { threadId, fileId, question }) => {
    const { filePart, imagePart } = await getFile(
      ctx,
      components.agent,
      fileId
    );

    const { messageId } = await saveMessage(ctx, components.agent, {
      threadId,
      message: {
        role: 'user',
        content: [
          imagePart ?? filePart, // Prefer image if available
          { type: 'text', text: question }
        ]
      },
      metadata: { fileIds: [fileId] } // Track file usage
    });

    await ctx.scheduler.runAfter(0, internal.chat.generateResponse, {
      threadId,
      promptMessageId: messageId
    });

    return messageId;
  }
});
```

### 3. Generate Response

```typescript
export const generateResponse = internalAction({
  args: { threadId: v.string(), promptMessageId: v.string() },
  handler: async (ctx, { threadId, promptMessageId }) => {
    await agent.generateText(ctx, { threadId }, { promptMessageId });
  }
});
```

## Inline File Saving

Files passed to `generateText` in actions are auto-saved if >64KB:

```typescript
export const askAboutImage = action({
  args: { threadId: v.string(), imageBytes: v.bytes() },
  handler: async (ctx, { threadId, imageBytes }) => {
    const result = await agent.generateText(
      ctx,
      { threadId },
      {
        message: {
          role: 'user',
          content: [
            { type: 'image', image: imageBytes, mimeType: 'image/png' },
            { type: 'text', text: 'What is in this image?' }
          ]
        }
      }
    );
    return result.text;
  }
});
```

## Using URLs Directly

Store file yourself and pass URL:

```typescript
export const askAboutImage = action({
  args: { threadId: v.string(), imageBlob: v.bytes() },
  handler: async (ctx, { threadId, imageBlob }) => {
    const blob = new Blob([imageBlob], { type: 'image/png' });
    const storageId = await ctx.storage.store(blob);
    const url = await ctx.storage.getUrl(storageId);

    const result = await agent.generateText(
      ctx,
      { threadId },
      {
        message: {
          role: 'user',
          content: [
            { type: 'image', image: url, mimeType: blob.type },
            { type: 'text', text: 'What is this?' }
          ]
        }
      }
    );

    return result.text;
  }
});
```

## Generating Images

Create images with external APIs and save to thread:

```typescript
import { storeFile, saveMessage } from '@convex-dev/agent';
import OpenAI from 'openai';

export const generateImage = action({
  args: { threadId: v.string(), prompt: v.string() },
  handler: async (ctx, { threadId, prompt }) => {
    const openai = new OpenAI();

    // Generate image
    const response = await openai.images.generate({
      model: 'dall-e-3',
      prompt,
      response_format: 'b64_json'
    });

    const imageData = response.data[0].b64_json!;
    const imageBlob = new Blob([Buffer.from(imageData, 'base64')], {
      type: 'image/png'
    });

    // Store file
    const { file } = await storeFile(ctx, components.agent, imageBlob, {
      filename: 'generated.png'
    });

    // Save as assistant message
    const { imagePart } = await getFile(ctx, components.agent, file.fileId);

    await saveMessage(ctx, components.agent, {
      threadId,
      agentName: 'image-generator',
      message: {
        role: 'assistant',
        content: [imagePart]
      },
      metadata: { fileIds: [file.fileId] }
    });

    return file.url;
  }
});
```

## File Tracking

Files are tracked for usage counting:

```typescript
// Track in metadata
await saveMessage(ctx, components.agent, {
  threadId,
  message: { ... },
  metadata: { fileIds: [fileId1, fileId2] },
});
```

## Vacuuming Unused Files

Clean up files no longer referenced:

```typescript
// convex/files/vacuum.ts
import { getOrphanedFiles, deleteFile } from '@convex-dev/agent';

export const vacuumFiles = internalMutation({
  handler: async (ctx) => {
    const orphaned = await getOrphanedFiles(ctx, components.agent, {
      limit: 100,
      olderThan: Date.now() - 7 * 24 * 60 * 60 * 1000 // 7 days
    });

    for (const file of orphaned) {
      await deleteFile(ctx, components.agent, file.fileId);
      await ctx.storage.delete(file.storageId);
    }
  }
});

// Schedule cleanup
crons.daily('vacuumFiles', { hourUTC: 3 }, internal.files.vacuumFiles);
```

## Cloud vs Local Development

Cloud backend has public storage URLs. For local development with external LLM APIs:

```bash
# Use ngrok to proxy local storage
ngrok http 3000
```

Or use `CONVEX_SITE_URL` to configure accessible URLs.

## File Utilities

```typescript
import {
  storeFile, // Store blob to file storage + agent tracking
  getFile, // Get file/image parts for messages
  deleteFile, // Delete file from storage + tracking
  getOrphanedFiles // Find files not referenced by messages
} from '@convex-dev/agent';
```

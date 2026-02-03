# Convex Runtimes Reference

Convex functions run in two runtimes:

- **Default Convex runtime** — fast, no cold starts, browser-like APIs
- **Node.js runtime** — full Node.js access, for actions only

## Default Convex Runtime

All queries, mutations, and actions run here by default. Similar to Cloudflare Workers.

### Advantages

- **No cold starts** — always ready to handle requests
- **Web standard APIs** — familiar browser-like environment
- **Low overhead** — direct, efficient database access
- **Same file** — queries, mutations, and actions can coexist

### Supported APIs

#### Network

```typescript
fetch(url, options); // Actions only
(Request, Response);
Headers;
(Blob, File);
FormData;
```

#### Encoding

```typescript
(TextEncoder, TextDecoder);
(atob(), btoa());
```

#### Streams

```typescript
ReadableStream;
WritableStream;
TransformStream;
```

#### Crypto

```typescript
crypto.getRandomValues()
crypto.subtle.*
CryptoKey
```

#### Other

```typescript
URL, URLSearchParams
Event, EventTarget
console.*
```

### Determinism in Queries and Mutations

Queries and mutations must be deterministic for caching and reactivity.

#### Math.random()

Convex provides a seeded pseudo-random number generator:

```typescript
export const example = mutation({
  handler: async (ctx) => {
    const rand1 = Math.random(); // Different each function call
    const rand2 = Math.random(); // Different from rand1
    // But same args always produce same sequence
  }
});

// Global Math.random() is fixed at deploy time
const globalRand = Math.random(); // Never changes between calls
```

#### Date.now()

Time is frozen when the function starts:

```typescript
export const example = mutation({
  handler: async (ctx) => {
    const t1 = Date.now(); // Time when function started
    await ctx.db.insert('items', { data: 'hello' });
    const t2 = Date.now(); // Same as t1!
  }
});

// Global Date.now() is frozen at deploy time
const deployTime = Date.now(); // Time of deployment
```

---

## Node.js Runtime

For actions that need Node.js APIs or npm packages that don't work in the default runtime.

### Enabling Node.js

Add `"use node"` directive at the top of the file:

```typescript
// convex/nodeActions.ts
'use node';

import { action } from './_generated/server';
import sharp from 'sharp';

export const processImage = action({
  handler: async (ctx, { imageData }) => {
    // Use Node.js npm packages
    const processed = await sharp(imageData).resize(200, 200).toBuffer();
    return processed;
  }
});
```

### Restrictions

- **Actions only** — cannot use `query` or `mutation` in `"use node"` files
- **No imports from non-node files** — files without `"use node"` cannot import from files with it
- **Cold starts** — Node.js actions may have startup latency
- **Separate file** — keep Node.js actions in dedicated files

### When to Use Node.js

- npm packages requiring Node.js APIs (fs, crypto, etc.)
- Libraries with native bindings
- Code using Node.js-specific features

```typescript
// ❌ These won't work in default runtime
import fs from 'fs';
import { createHash } from 'crypto';

// ✅ Use "use node" for these
('use node');
import { createHash } from 'crypto';
```

### Node.js Version

Configure in `convex.json`:

```json
{
  "node": {
    "version": "22"
  }
}
```

Supported versions: 20 (default), 22

---

## Bundling

Convex uses esbuild to bundle your code and dependencies.

### How It Works

1. `npx convex dev` or `npx convex deploy` runs
2. esbuild traverses `convex/` folder
3. Functions and dependencies are bundled
4. Bundle is sent to Convex servers

### Supported Module Formats

- **ESM** (recommended): `import`/`export`
- **CommonJS**: `require`/`module.exports`

Both work — esbuild handles conversion.

### Bundle Size Limits

| Limit                        | Value |
| ---------------------------- | ----- |
| Total bundle size            | 32MB  |
| External packages (zipped)   | 45MB  |
| External packages (unzipped) | 240MB |

### Debugging Bundle Size

```bash
# Generate bundle for inspection
npx convex dev --once --debug-bundle-path /tmp/myBundle

# Visualize with source-map-explorer
npx source-map-explorer /tmp/myBundle/**/*.js
```

- `isolate/` — Default runtime bundle
- `node/` — Node.js runtime bundle

---

## External Packages

For Node.js actions, mark packages as external to avoid bundling issues.

### Why Use External Packages

- **Dynamic imports** — Some packages use runtime `require()`
- **Native bindings** — Packages with compiled code
- **Large packages** — Reduce bundle size

### Configuration

In `convex.json`:

```json
{
  "node": {
    "externalPackages": ["*"]
  }
}
```

Or specify individual packages:

```json
{
  "node": {
    "externalPackages": ["sharp", "aws-sdk", "puppeteer"]
  }
}
```

### How External Packages Work

1. Package is NOT bundled into your code
2. On first push, Convex installs it from npm
3. Version matches your local `node_modules`
4. Subsequent pushes use cached packages

### Common External Package Candidates

- `sharp` — Image processing
- `aws-sdk` — AWS services
- `snowflake-sdk` — Database driver
- `pdf-parse` — PDF extraction
- `tiktoken` — Token counting

### Troubleshooting

#### Version Mismatch

External package version comes from your local `node_modules`. After changing `package.json`:

```bash
npm install  # Update local node_modules
npx convex deploy  # Now pushes correct version
```

#### Import Errors

If you see CommonJS import errors:

```typescript
// ❌ May fail with external packages
import { Foo } from 'some-module';

// ✅ Use default import
import SomeModule from 'some-module';
const { Foo } = SomeModule;
```

---

## Debugging

### Console Methods

```typescript
// Logging
console.log('Info message', data);
console.info('Info');
console.warn('Warning');
console.error('Error');
console.debug('Debug');

// Stack trace
console.trace('Where am I?');

// Timing
console.time('operation');
// ... do work ...
console.timeEnd('operation'); // Prints: operation: 123ms

console.time('multi-step');
console.timeLog('multi-step', 'Step 1 done');
console.timeLog('multi-step', 'Step 2 done');
console.timeEnd('multi-step');
```

### Viewing Logs

**Development:**

- Browser console (logs streamed from dev deployment)
- Terminal with `npx convex dev`
- Dashboard → Logs

**Production:**

- Dashboard → Logs
- Log streaming integrations (Axiom, Datadog, etc.)
- `npx convex logs` CLI command

### Request ID

Every error includes a Request ID:

```
Error: Something went wrong [Request ID: abc123xyz...]
```

To find related logs:

1. Copy the Request ID
2. Dashboard → Logs → Filter by Request ID
3. View all logs for that specific request

### Debugging Node.js Import Errors

If you see errors about `fs` or `node:fs` not being available:

```bash
npx convex dev --once --debug-node-apis
```

This uses slower bundling to trace which import causes the error.

### Common Issues

#### "Cannot use 'import' for CommonJS"

The package needs Node.js runtime:

```typescript
// Add to top of file
'use node';
```

#### "Module not found" after deploy

Package needs to be external:

```json
{
  "node": {
    "externalPackages": ["problematic-package"]
  }
}
```

#### Function timeout

- Check for infinite loops
- Add logging to identify slow operations
- Consider breaking into smaller functions
- For long-running work, use scheduled functions

---

## Choosing the Right Runtime

| Scenario                          | Runtime | Why                      |
| --------------------------------- | ------- | ------------------------ |
| Database queries                  | Default | Must be deterministic    |
| Database writes                   | Default | Must be deterministic    |
| Simple fetch calls                | Default | Fast, no cold starts     |
| npm packages (browser-compatible) | Default | Fast, no cold starts     |
| npm packages (Node.js-only)       | Node.js | Required for Node APIs   |
| Image processing (sharp)          | Node.js | Native bindings          |
| PDF processing                    | Node.js | Often needs Node APIs    |
| Crypto operations                 | Default | Web Crypto API available |

### Decision Flow

```
Is this a query or mutation?
├── Yes → Default runtime (required)
└── No (action) →
    Does it need Node.js-specific packages?
    ├── Yes → Node.js runtime ("use node")
    └── No → Default runtime (faster)
```

---

## Best Practices

### 1. Prefer Default Runtime

```typescript
// ✅ Most actions can use default runtime
export const callAPI = action({
  handler: async (ctx) => {
    const response = await fetch('https://api.example.com/data');
    return await response.json();
  }
});
```

### 2. Isolate Node.js Code

```typescript
// convex/imageProcessing.ts
'use node';

import sharp from 'sharp';
import { action } from './_generated/server';

// Only image processing actions here
export const resize = action({...});
export const convert = action({...});
```

```typescript
// convex/api.ts
// No "use node" — uses fast default runtime

export const fetchData = action({...});
export const processData = mutation({...});
```

### 3. Use External Packages for Large Dependencies

```json
{
  "node": {
    "externalPackages": ["aws-sdk", "sharp", "puppeteer"]
  }
}
```

### 4. Monitor Bundle Size

Regularly check bundle size during development:

```bash
npx convex dev --once --debug-bundle-path /tmp/bundle
du -sh /tmp/bundle/*
```

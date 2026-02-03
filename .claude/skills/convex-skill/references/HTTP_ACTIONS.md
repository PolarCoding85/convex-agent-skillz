# Convex HTTP Actions Reference

HTTP actions let you build custom HTTP endpoints in Convex for webhooks, public APIs, and file handling.

## Basic HTTP Action

```typescript
// convex/http.ts
import { httpRouter } from 'convex/server';
import { httpAction } from './_generated/server';

const http = httpRouter();

http.route({
  path: '/hello',
  method: 'GET',
  handler: httpAction(async (ctx, request) => {
    return new Response('Hello from Convex!', { status: 200 });
  })
});

export default http;
```

**Endpoint URL**: `https://<deployment-name>.convex.site/hello`

---

## Routing

### Exact Path Match

```typescript
http.route({
  path: '/api/users',
  method: 'GET',
  handler: getUsers
});
```

### Path Prefix Match

```typescript
http.route({
  pathPrefix: '/api/users/',
  method: 'GET',
  handler: httpAction(async (ctx, request) => {
    // Extract user ID from URL
    const url = new URL(request.url);
    const userId = url.pathname.replace('/api/users/', '');
    // ...
  })
});
// Matches: /api/users/123, /api/users/abc, etc.
```

### Multiple Methods

```typescript
// Handle GET
http.route({
  path: '/api/items',
  method: 'GET',
  handler: listItems
});

// Handle POST
http.route({
  path: '/api/items',
  method: 'POST',
  handler: createItem
});
```

---

## Request Handling

### Accessing Request Data

```typescript
http.route({
  path: '/webhook',
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    // JSON body
    const body = await request.json();

    // Form data
    const formData = await request.formData();

    // Raw text
    const text = await request.text();

    // Binary data
    const blob = await request.blob();
    const buffer = await request.arrayBuffer();

    // Headers
    const contentType = request.headers.get('Content-Type');
    const auth = request.headers.get('Authorization');

    // URL and query params
    const url = new URL(request.url);
    const queryParam = url.searchParams.get('filter');

    return new Response('OK');
  })
});
```

### Response Types

```typescript
// JSON response
return new Response(JSON.stringify({ success: true }), {
  status: 200,
  headers: { 'Content-Type': 'application/json' }
});

// Text response
return new Response('Hello', { status: 200 });

// Binary response
return new Response(blobData, {
  headers: { 'Content-Type': 'image/png' }
});

// Empty response
return new Response(null, { status: 204 });

// Error response
return new Response('Not found', { status: 404 });
```

---

## HTTP Action Context

HTTP actions have access to `ActionCtx`:

```typescript
http.route({
  path: '/api/data',
  method: 'GET',
  handler: httpAction(async (ctx, request) => {
    // Read from database via query
    const data = await ctx.runQuery(internal.data.list, {});

    // Write to database via mutation
    await ctx.runMutation(internal.data.create, { name: 'New Item' });

    // Call another action
    await ctx.runAction(internal.external.notify, {});

    // Schedule for later
    await ctx.scheduler.runAfter(0, internal.jobs.process, {});

    // File storage
    const url = await ctx.storage.getUrl(storageId);
    const blob = await ctx.storage.get(storageId);
    const newId = await ctx.storage.store(someBlob);

    // Auth (if using Authorization header)
    const identity = await ctx.auth.getUserIdentity();

    return new Response(JSON.stringify(data));
  })
});
```

---

## CORS (Cross-Origin Resource Sharing)

### Basic CORS Headers

```typescript
http.route({
  path: '/api/public',
  method: 'GET',
  handler: httpAction(async (ctx, request) => {
    const data = await ctx.runQuery(internal.data.getPublic, {});

    return new Response(JSON.stringify(data), {
      status: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': process.env.CLIENT_ORIGIN!,
        Vary: 'origin'
      }
    });
  })
});
```

### Handling Preflight (OPTIONS)

```typescript
// Preflight handler
http.route({
  path: '/api/data',
  method: 'OPTIONS',
  handler: httpAction(async (ctx, request) => {
    const headers = request.headers;

    // Validate preflight request
    if (
      headers.get('Origin') !== null &&
      headers.get('Access-Control-Request-Method') !== null &&
      headers.get('Access-Control-Request-Headers') !== null
    ) {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': process.env.CLIENT_ORIGIN!,
          'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
          'Access-Control-Allow-Headers': 'Content-Type, Authorization',
          'Access-Control-Max-Age': '86400' // 24 hours
        }
      });
    }

    return new Response(null, { status: 400 });
  })
});

// Actual handler
http.route({
  path: '/api/data',
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    const data = await request.json();
    await ctx.runMutation(internal.data.create, data);

    return new Response(JSON.stringify({ success: true }), {
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': process.env.CLIENT_ORIGIN!,
        Vary: 'origin'
      }
    });
  })
});
```

### CORS Helper Function

```typescript
function corsHeaders(origin: string) {
  return {
    'Access-Control-Allow-Origin': origin,
    Vary: 'origin'
  };
}

function corsResponse(body: string | null, status: number) {
  return new Response(body, {
    status,
    headers: {
      'Content-Type': 'application/json',
      ...corsHeaders(process.env.CLIENT_ORIGIN!)
    }
  });
}
```

---

## Authentication

### JWT Token Authentication

```typescript
http.route({
  path: '/api/protected',
  method: 'GET',
  handler: httpAction(async (ctx, request) => {
    // Convex automatically validates JWT from Authorization header
    const identity = await ctx.auth.getUserIdentity();

    if (!identity) {
      return new Response('Unauthorized', { status: 401 });
    }

    const data = await ctx.runQuery(internal.data.getForUser, {
      userId: identity.subject
    });

    return new Response(JSON.stringify(data), {
      headers: { 'Content-Type': 'application/json' }
    });
  })
});
```

**Client call:**

```typescript
const token = await getJWTToken(); // From your auth provider

fetch('https://your-deployment.convex.site/api/protected', {
  headers: {
    Authorization: `Bearer ${token}`
  }
});
```

### API Key Authentication

```typescript
http.route({
  path: '/api/webhook',
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    const apiKey = request.headers.get('X-API-Key');

    if (apiKey !== process.env.WEBHOOK_API_KEY) {
      return new Response('Invalid API key', { status: 401 });
    }

    const body = await request.json();
    await ctx.runMutation(internal.webhooks.process, { data: body });

    return new Response('OK', { status: 200 });
  })
});
```

---

## Webhook Patterns

### Stripe Webhook

```typescript
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

http.route({
  path: '/webhook/stripe',
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    const signature = request.headers.get('stripe-signature');
    const body = await request.text();

    let event: Stripe.Event;
    try {
      event = stripe.webhooks.constructEvent(
        body,
        signature!,
        process.env.STRIPE_WEBHOOK_SECRET!
      );
    } catch (err) {
      console.error('Webhook signature verification failed');
      return new Response('Invalid signature', { status: 400 });
    }

    // Process the event
    await ctx.runMutation(internal.payments.handleStripeEvent, {
      type: event.type,
      data: event.data.object
    });

    return new Response('OK', { status: 200 });
  })
});
```

### Generic Webhook with HMAC Validation

```typescript
import { createHmac } from 'crypto';

function verifyHmacSignature(
  payload: string,
  signature: string,
  secret: string
): boolean {
  const expected = createHmac('sha256', secret).update(payload).digest('hex');
  return signature === `sha256=${expected}`;
}

http.route({
  path: '/webhook/external',
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    const signature = request.headers.get('X-Signature');
    const body = await request.text();

    if (!verifyHmacSignature(body, signature!, process.env.WEBHOOK_SECRET!)) {
      return new Response('Invalid signature', { status: 401 });
    }

    const data = JSON.parse(body);
    await ctx.runMutation(internal.webhooks.process, { data });

    return new Response('OK', { status: 200 });
  })
});
```

---

## File Handling

### File Upload

```typescript
http.route({
  path: '/upload',
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    const blob = await request.blob();
    const storageId = await ctx.storage.store(blob);

    // Optionally save metadata
    const fileName = request.headers.get('X-File-Name') || 'unnamed';
    await ctx.runMutation(internal.files.saveMetadata, {
      storageId,
      fileName
    });

    return new Response(JSON.stringify({ storageId }), {
      status: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': process.env.CLIENT_ORIGIN!
      }
    });
  })
});

// OPTIONS for CORS preflight
http.route({
  path: '/upload',
  method: 'OPTIONS',
  handler: httpAction(async () => {
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
```

### File Download

```typescript
http.route({
  path: '/download',
  method: 'GET',
  handler: httpAction(async (ctx, request) => {
    const url = new URL(request.url);
    const storageId = url.searchParams.get('id') as Id<'_storage'>;

    if (!storageId) {
      return new Response('Missing file ID', { status: 400 });
    }

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
```

---

## Debugging HTTP Actions

### Step 1: Verify Deployment

Check Dashboard → Functions for an `http` entry. If missing:

- File must be named exactly `http.ts` or `http.js`
- Must export `httpRouter()` as default
- Run `npx convex dev` without errors

### Step 2: Test with curl

```bash
# Get your URL from Dashboard → Settings → URL and Deploy Key
# Use the .convex.site URL (not .convex.cloud)

curl -X GET https://your-deployment.convex.site/your-path

curl -X POST https://your-deployment.convex.site/webhook \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}'
```

### Step 3: Check Browser Network Tab

If curl works but browser doesn't:

- Open DevTools → Network
- Look for CORS errors
- Verify URL ends in `.convex.site`
- Check that OPTIONS preflight is handled

### Step 4: View Logs

- Dashboard → Logs shows all HTTP action executions
- Add `console.log` statements for debugging
- Check for errors and stack traces

---

## Limits

| Limit              | Value                    |
| ------------------ | ------------------------ |
| Request body size  | 20MB                     |
| Response body size | 20MB                     |
| Timeout            | Same as actions (10 min) |

**Supported body methods**: `.text()`, `.json()`, `.blob()`, `.arrayBuffer()`

---

## Best Practices

### 1. Always Validate Input

```typescript
http.route({
  path: '/api/create',
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    let body;
    try {
      body = await request.json();
    } catch {
      return new Response('Invalid JSON', { status: 400 });
    }

    if (!body.name || typeof body.name !== 'string') {
      return new Response('Missing or invalid "name"', { status: 400 });
    }

    // Process valid input...
  })
});
```

### 2. Use Internal Functions

```typescript
// ✅ Use internal functions for database operations
await ctx.runMutation(internal.data.create, { name });

// ❌ Don't use public API
await ctx.runMutation(api.data.create, { name });
```

### 3. Return Appropriate Status Codes

```typescript
// Success
return new Response(JSON.stringify(data), { status: 200 }); // OK
return new Response(null, { status: 201 }); // Created
return new Response(null, { status: 204 }); // No Content

// Client errors
return new Response('Bad request', { status: 400 });
return new Response('Unauthorized', { status: 401 });
return new Response('Forbidden', { status: 403 });
return new Response('Not found', { status: 404 });
return new Response('Conflict', { status: 409 });

// Server errors
return new Response('Internal error', { status: 500 });
```

### 4. Handle Errors Gracefully

```typescript
http.route({
  path: '/api/process',
  method: 'POST',
  handler: httpAction(async (ctx, request) => {
    try {
      const body = await request.json();
      const result = await ctx.runMutation(internal.process.run, body);
      return new Response(JSON.stringify(result), { status: 200 });
    } catch (error) {
      console.error('Processing failed:', error);

      if (error instanceof ConvexError) {
        return new Response(JSON.stringify({ error: error.data }), {
          status: 400
        });
      }

      return new Response('Internal server error', { status: 500 });
    }
  })
});
```

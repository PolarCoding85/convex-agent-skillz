# Convex Error Handling Reference

## Error Types

| Error Type         | Cause                            | Convex Behavior                        |
| ------------------ | -------------------------------- | -------------------------------------- |
| Application Errors | Your code throws `ConvexError`   | Sent to client with `data` payload     |
| Developer Errors   | Bugs in function code            | Logged, generic error to client (prod) |
| Read/Write Limit   | Too much data in one transaction | Transaction fails                      |
| Internal Convex    | Network/infrastructure issues    | Auto-retry by Convex                   |

---

## Application Errors with ConvexError

Use `ConvexError` for expected failures that should be communicated to clients:

```typescript
import { ConvexError } from 'convex/values';
import { mutation } from './_generated/server';
import { v } from 'convex/values';

export const assignRole = mutation({
  args: { roleId: v.id('roles'), userId: v.id('users') },
  handler: async (ctx, { roleId, userId }) => {
    const role = await ctx.db.get(roleId);
    if (!role) {
      throw new ConvexError('Role not found');
    }

    const existing = await ctx.db
      .query('assignments')
      .withIndex('by_role', (q) => q.eq('roleId', roleId))
      .first();

    if (existing) {
      throw new ConvexError('Role is already assigned');
    }

    await ctx.db.insert('assignments', { roleId, userId });
  }
});
```

### Structured Error Data

Pass any Convex-serializable data to `ConvexError`:

```typescript
// Simple string
throw new ConvexError('Role is already taken');
// error.data === "Role is already taken"

// Structured object
throw new ConvexError({
  code: 'ROLE_TAKEN',
  message: 'Role is already assigned',
  roleId: roleId
});
// error.data === { code: "ROLE_TAKEN", message: "...", roleId: "..." }

// With numeric code
throw new ConvexError({
  code: 409,
  reason: 'conflict'
});
```

### Benefits of ConvexError

- **Bubbles through call stack** — throw from helpers, catch in handlers
- **Works across function calls** — `runQuery`, `runMutation`, `runAction` propagate errors
- **Mutations roll back** — throwing prevents transaction from committing
- **Preserved in production** — `data` payload is not redacted

---

## Handling Errors on the Client

### In Mutations (Promises)

```typescript
import { ConvexError } from 'convex/values';
import { useMutation } from 'convex/react';
import { api } from '../convex/_generated/api';

function AssignRoleButton({ roleId, userId }) {
  const assignRole = useMutation(api.roles.assign);

  const handleClick = async () => {
    try {
      await assignRole({ roleId, userId });
    } catch (error) {
      if (error instanceof ConvexError) {
        // Application error — access structured data
        const data = error.data as { code: string; message: string };
        showToast(data.message);
      } else {
        // Developer/system error
        showToast('Something went wrong');
      }
    }
  };

  return <button onClick={handleClick}>Assign Role</button>;
}
```

### In Queries (Error Boundaries)

For queries, use React Error Boundaries:

```tsx
import { ErrorBoundary } from 'react-error-boundary';

function App() {
  return (
    <ErrorBoundary
      fallback={<div>Something went wrong</div>}
      onError={(error) => {
        // Report to error tracking service
        reportError(error);
      }}
    >
      <ConvexProvider client={convex}>
        <MyApp />
      </ConvexProvider>
    </ErrorBoundary>
  );
}
```

**Note**: Queries are deterministic — there's no concept of "retry". Same args always produce same result (or same error).

---

## Alternative: Return Error Values

For some use cases, returning different values is cleaner than throwing:

```typescript
export const createUser = mutation({
  args: { email: v.string(), name: v.string() },
  returns: v.union(
    v.object({ success: v.literal(true), userId: v.id('users') }),
    v.object({ success: v.literal(false), error: v.string() })
  ),
  handler: async (ctx, { email, name }) => {
    const existing = await ctx.db
      .query('users')
      .withIndex('by_email', (q) => q.eq('email', email))
      .first();

    if (existing) {
      return { success: false, error: 'EMAIL_IN_USE' };
    }

    const userId = await ctx.db.insert('users', { email, name });
    return { success: true, userId };
  }
});
```

**Client usage:**

```typescript
const result = await createUser({ email, name });
if (result.success) {
  redirect(`/users/${result.userId}`);
} else {
  showError(result.error);
}
```

---

## Error Handling in Actions

Actions **cannot be auto-retried** because they may have side effects:

```typescript
export const sendEmail = action({
  args: { to: v.string(), subject: v.string(), body: v.string() },
  handler: async (ctx, { to, subject, body }) => {
    try {
      await fetch('https://api.sendgrid.com/v3/mail/send', {
        method: 'POST',
        headers: { Authorization: `Bearer ${process.env.SENDGRID_KEY}` },
        body: JSON.stringify({ to, subject, body })
      });
    } catch (error) {
      // Log for debugging
      console.error('Email send failed:', error);

      // Throw ConvexError for client
      throw new ConvexError({
        code: 'EMAIL_FAILED',
        message: 'Failed to send email'
      });
    }
  }
});
```

### Retry Pattern for Actions

Implement retry logic explicitly:

```typescript
export const sendWithRetry = action({
  args: { to: v.string(), subject: v.string(), body: v.string() },
  handler: async (ctx, args) => {
    const maxRetries = 3;
    let lastError;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        await sendEmail(args);
        return { success: true };
      } catch (error) {
        lastError = error;
        if (attempt < maxRetries) {
          // Exponential backoff
          await new Promise((r) => setTimeout(r, Math.pow(2, attempt) * 1000));
        }
      }
    }

    throw new ConvexError({
      code: 'MAX_RETRIES_EXCEEDED',
      message: `Failed after ${maxRetries} attempts`,
      lastError: String(lastError)
    });
  }
});
```

---

## Read/Write Limit Errors

Convex enforces limits to guarantee performance:

| Limit              | Value                     |
| ------------------ | ------------------------- |
| Documents read     | 16,384 per query/mutation |
| Documents written  | 8,192 per mutation        |
| Data read          | 64MB per query/mutation   |
| Data written       | 64MB per mutation         |
| Unique tables read | 8,192                     |
| Database calls     | 8,192                     |

### Avoiding Limit Errors

```typescript
// ❌ BAD: Unbounded query
const allUsers = await ctx.db.query('users').collect();

// ✅ GOOD: Use pagination or limits
const users = await ctx.db.query('users').take(100);

// ❌ BAD: Full table scan with filter
const activeUsers = await ctx.db
  .query('users')
  .filter((q) => q.eq(q.field('status'), 'active'))
  .collect();

// ✅ GOOD: Use index
const activeUsers = await ctx.db
  .query('users')
  .withIndex('by_status', (q) => q.eq('status', 'active'))
  .take(100);
```

**Warning**: Functions close to limits will log warnings. Add indexes to reduce scanned documents.

---

## Dev vs Production Error Messages

| Environment | Error Details          | Stack Trace |
| ----------- | ---------------------- | ----------- |
| Development | Full message           | Yes         |
| Production  | Generic "Server Error" | No          |

**Exception**: `ConvexError.data` is always preserved, even in production.

```typescript
// In production, client sees:
// - Regular Error: "Server Error"
// - ConvexError: Full data payload

// Always use ConvexError for user-facing errors
throw new ConvexError({
  message: 'User-friendly message',
  code: 'SPECIFIC_CODE'
});
```

---

## Debugging Errors

### Console Logging

```typescript
export const debugMutation = mutation({
  handler: async (ctx, args) => {
    console.log('Args received:', args);
    console.time('database-read');

    const data = await ctx.db.query('items').take(10);
    console.timeEnd('database-read');

    console.warn('Potential issue:', someCondition);
    console.error('Error details:', error);
  }
});
```

Available methods: `log`, `info`, `warn`, `error`, `debug`, `trace`, `time`, `timeLog`, `timeEnd`

### Finding Logs by Request ID

Every error includes a Request ID: `[Request ID: abc123...]`

1. Copy the Request ID from the error
2. Paste into Dashboard → Logs → Filter
3. View all logs for that request

### Dashboard Logs

View logs at:

- Development: Browser console (logs streamed from dev deployment)
- Production: Dashboard → Logs page

---

## Best Practices

### 1. Use ConvexError for Expected Failures

```typescript
// ✅ User-facing validation errors
throw new ConvexError({ code: 'INVALID_INPUT', field: 'email' });

// ✅ Business logic failures
throw new ConvexError({
  code: 'INSUFFICIENT_FUNDS',
  required: 100,
  available: 50
});
```

### 2. Let Developer Errors Bubble

```typescript
// ❌ Don't catch everything
try {
  const user = await ctx.db.get(userId);
  return user.name; // If user is null, let it throw
} catch (e) {
  return null; // Hides bugs!
}

// ✅ Only catch expected errors
const user = await ctx.db.get(userId);
if (!user) {
  throw new ConvexError('User not found');
}
return user.name;
```

### 3. Wrap External Calls in Try/Catch

```typescript
export const callExternalAPI = action({
  handler: async (ctx, args) => {
    try {
      const response = await fetch('https://api.example.com/data');
      if (!response.ok) {
        throw new ConvexError({
          code: 'API_ERROR',
          status: response.status
        });
      }
      return await response.json();
    } catch (error) {
      if (error instanceof ConvexError) throw error;

      // Network or parsing error
      throw new ConvexError({
        code: 'EXTERNAL_SERVICE_FAILED',
        message: 'Could not reach external service'
      });
    }
  }
});
```

### 4. Set Up Error Reporting

Configure log streaming and exception reporting for production:

- Log streams: Axiom, Datadog, Sentry, etc.
- Exception reporting: Catches and reports unhandled errors

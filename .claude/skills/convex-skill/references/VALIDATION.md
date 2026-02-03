# Convex Validation Reference

Argument and return value validators ensure that functions are called with correct types. **This is critical for security** — without validation, malicious users can call public functions with unexpected arguments.

## Adding Validators

```typescript
import { mutation, query } from './_generated/server';
import { v } from 'convex/values';

export const send = mutation({
  args: {
    body: v.string(),
    author: v.string()
  },
  returns: v.null(),
  handler: async (ctx, args) => {
    // args is fully typed based on validators
    await ctx.db.insert('messages', { body: args.body, author: args.author });
  }
});
```

**Key points:**

- TypeScript types are inferred from validators — no need for manual type annotations
- Validators throw if objects contain undeclared properties (prevents typos, data leaks)
- Even `args: {}` helps — TypeScript errors if client passes unexpected arguments

---

## Validator Reference

### Primitives

```typescript
v.string(); // string
v.number(); // number (float64)
v.int64(); // bigint (-2^63 to 2^63-1)
v.boolean(); // boolean
v.null(); // null (use instead of undefined)
v.bytes(); // ArrayBuffer
```

### IDs

```typescript
v.id('users'); // Id<"users"> — type-safe table reference
v.id('messages');
```

### Collections

```typescript
v.array(v.string()); // string[]
v.object({ name: v.string() }); // { name: string }
v.record(v.string(), v.number()); // Record<string, number>
```

### Optional & Nullable

```typescript
v.optional(v.string()); // string | undefined
v.union(v.string(), v.null()); // string | null
v.nullable(v.string()); // string | null (shorthand)
```

### Unions & Literals

```typescript
// Enum-like unions
v.union(v.literal('pending'), v.literal('active'), v.literal('archived'));

// Mixed type unions
v.union(v.string(), v.number());
```

### Any (Escape Hatch)

```typescript
v.any(); // any — use sparingly
```

---

## Complete Example

```typescript
import { v } from 'convex/values';

export const createUser = mutation({
  args: {
    // Required fields
    name: v.string(),
    email: v.string(),

    // Optional field
    nickname: v.optional(v.string()),

    // Nullable field
    avatarUrl: v.union(v.string(), v.null()),

    // Enum
    role: v.union(v.literal('user'), v.literal('admin')),

    // Nested object
    settings: v.object({
      theme: v.union(v.literal('light'), v.literal('dark')),
      notifications: v.boolean()
    }),

    // Array of IDs
    teamIds: v.array(v.id('teams')),

    // Dynamic key-value
    metadata: v.record(v.string(), v.string())
  },
  handler: async (ctx, args) => {
    // args is fully typed
  }
});
```

---

## Return Value Validation

```typescript
export const getUser = query({
  args: { id: v.id('users') },
  returns: v.union(
    v.object({
      _id: v.id('users'),
      name: v.string(),
      email: v.string()
    }),
    v.null()
  ),
  handler: async (ctx, { id }) => {
    return await ctx.db.get(id);
  }
});
```

**Benefits:**

- TypeScript checks handler return type matches validator
- Prevents accidentally returning sensitive data to clients
- Documents API contract

---

## Reusing Validators

Define validators once, share across functions and schemas:

```typescript
// convex/validators.ts
import { v } from 'convex/values';

export const statusValidator = v.union(
  v.literal('active'),
  v.literal('inactive')
);

export const userValidator = v.object({
  name: v.string(),
  email: v.string(),
  status: statusValidator,
  profileUrl: v.optional(v.string())
});
```

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from 'convex/server';
import { userValidator } from './validators';

export default defineSchema({
  users: defineTable(userValidator).index('by_email', ['email'])
});
```

```typescript
// convex/users.ts
import { userValidator } from './validators';

export const create = mutation({
  args: userValidator.fields, // Reuse validator fields
  handler: async (ctx, args) => {
    return await ctx.db.insert('users', args);
  }
});
```

---

## Extending Object Validators

Object validators support `.pick`, `.omit`, `.extend`, and `.partial`:

### .pick() — Select Specific Fields

```typescript
const userValidator = v.object({
  name: v.string(),
  email: v.string(),
  status: v.string(),
  profileUrl: v.optional(v.string())
});

// Only name and profileUrl
const publicUser = userValidator.pick('name', 'profileUrl');
// Result: { name: string, profileUrl?: string }
```

### .omit() — Exclude Fields

```typescript
// Everything except status and profileUrl
const userWithoutStatus = userValidator.omit('status', 'profileUrl');
// Result: { name: string, email: string }
```

### .extend() — Add Fields

```typescript
// Add system fields
const userDocument = userValidator.extend({
  _id: v.id('users'),
  _creationTime: v.number()
});
// Result: { _id, _creationTime, name, email, status, profileUrl? }
```

### .partial() — Make All Fields Optional

```typescript
// Useful for patch/update operations
const userPatch = userValidator.partial();
// Result: { name?: string, email?: string, status?: string, profileUrl?: string }
```

### Combining Methods

```typescript
// Create a patch validator without certain fields
const userUpdateArgs = userValidator
  .omit('email') // Can't change email
  .partial(); // All remaining fields optional
```

---

## Extracting TypeScript Types

Use `Infer` to get TypeScript types from validators:

```typescript
import { Infer, v } from 'convex/values';

const messageValidator = v.object({
  body: v.string(),
  authorId: v.id('users'),
  channelId: v.id('channels')
});

// Extract the type
type Message = Infer<typeof messageValidator>;
// Result: { body: string, authorId: Id<"users">, channelId: Id<"channels"> }

// Use in helper functions
function formatMessage(message: Message): string {
  return `${message.body}`;
}
```

---

## Records vs Objects

**Objects** — fixed, known keys:

```typescript
v.object({
  name: v.string(),
  age: v.number()
});
// { name: string, age: number }
```

**Records** — dynamic keys:

```typescript
v.record(v.string(), v.number());
// Record<string, number> — any string key, number value

// Can use IDs as keys
v.record(v.id('users'), v.boolean());
// Record<Id<"users">, boolean>
```

**Limitations:**

- Record keys must be ASCII, non-empty, not start with `$` or `_`
- Cannot use string literals as record keys

---

## Validation Gotchas

### Extra Properties Are Rejected

```typescript
const args = { name: 'Alice', extra: 'field' };
// ❌ Throws: "extra" is not a valid field
```

This prevents:

- Typos in field names going unnoticed
- Accidentally accepting/returning unintended data

### undefined vs null

```typescript
// ❌ undefined is NOT a valid Convex value
v.optional(v.string()); // string | undefined — for args only

// ✅ Use null for database storage
v.union(v.string(), v.null()); // string | null
```

Functions returning `undefined` are converted to `null` on the client.

### Objects Must Be Plain

Convex only supports plain JavaScript objects (no custom prototypes):

```typescript
// ✅ Plain object
{ name: 'Alice', age: 30 }

// ❌ Class instance
new User('Alice', 30)
```

### Field Name Restrictions

- Cannot start with `$` or `_` (reserved for system)
- Must be non-empty

---

## Security Best Practices

### Always Validate Public Functions

```typescript
// ❌ BAD: No validation — vulnerable to malicious input
export const bad = mutation({
  handler: async (ctx, args: any) => {
    await ctx.db.insert('users', args);
  }
});

// ✅ GOOD: Explicit validation
export const good = mutation({
  args: {
    name: v.string(),
    email: v.string()
  },
  handler: async (ctx, args) => {
    await ctx.db.insert('users', args);
  }
});
```

### Use Internal Functions for Sensitive Operations

```typescript
// Public function — validated, limited access
export const updateProfile = mutation({
  args: { name: v.string() },
  handler: async (ctx, { name }) => {
    const user = await requireUser(ctx);
    await ctx.db.patch(user._id, { name });
  }
});

// Internal function — called from actions, can skip validation
export const adminUpdate = internalMutation({
  handler: async (ctx, args: { userId: Id<'users'>; data: Partial<User> }) => {
    await ctx.db.patch(args.userId, args.data);
  }
});
```

### Validate Return Values for Public Functions

```typescript
export const getUser = query({
  args: { id: v.id('users') },
  returns: v.object({
    _id: v.id('users'),
    name: v.string()
    // Note: NOT including sensitive fields like passwordHash
  }),
  handler: async (ctx, { id }) => {
    const user = await ctx.db.get(id);
    if (!user) return null;
    return { _id: user._id, name: user.name };
  }
});
```

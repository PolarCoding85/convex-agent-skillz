# Migrations Component

Run stateful online migrations on live data with tracking, resumability, and status monitoring.

## Installation

```bash
npm install @convex-dev/migrations
```

```typescript
// convex/convex.config.ts
import migrations from '@convex-dev/migrations/convex.config';
app.use(migrations);
```

## Basic Setup

```typescript
// convex/migrations.ts
import { Migrations } from '@convex-dev/migrations';
import { components, internal } from './_generated/api';
import { DataModel } from './_generated/dataModel';

// Create migrations instance (DataModel provides type safety)
export const migrations = new Migrations<DataModel>(components.migrations);

// Export runner for CLI usage
export const run = migrations.runner();
```

## Defining Migrations

### Simple Field Update

```typescript
export const setDefaultPlan = migrations.define({
  table: 'teams',
  migrateOne: async (ctx, team) => {
    if (team.plan === undefined) {
      await ctx.db.patch(team._id, { plan: 'basic' });
    }
  }
});
```

### Shorthand for Patching

```typescript
// Return object to patch the document
export const clearField = migrations.define({
  table: 'users',
  migrateOne: async (ctx, user) => {
    // Returns patch object (optionalField will be cleared)
    return { optionalField: undefined };
  }
});
```

### Cross-Table Operations

```typescript
export const deleteOrphanedEmbeddings = migrations.define({
  table: 'embeddings',
  migrateOne: async (ctx, embedding) => {
    const chunk = await ctx.db
      .query('chunks')
      .withIndex('by_embeddingId', (q) => q.eq('embeddingId', embedding._id))
      .first();

    if (!chunk) {
      await ctx.db.delete(embedding._id);
    }
  }
});
```

### Data Transformation

```typescript
export const migratePreferences = migrations.define({
  table: 'users',
  migrateOne: async (ctx, user) => {
    if (user.preferences && typeof user.preferences === 'object') {
      // Move preferences to separate table
      await ctx.db.insert('userPreferences', {
        userId: user._id,
        ...user.preferences
      });
      return { preferences: undefined };
    }
  }
});
```

## Running Migrations

### From CLI

```bash
# Run single migration
npx convex run migrations:run '{"fn": "migrations:setDefaultPlan"}'

# Run all pending migrations
npx convex run migrations:runAll

# Run in production
npx convex run migrations:runAll --prod

# Chain with deploy
npx convex deploy --cmd 'npm run build' && npx convex run migrations:runAll --prod
```

### Programmatically

```typescript
// Run single migration
await migrations.runOne(ctx, internal.migrations.setDefaultPlan);

// Run multiple in series (skips completed)
await migrations.runSerially(ctx, [
  internal.migrations.setDefaultPlan,
  internal.migrations.validateRequiredField,
  internal.migrations.convertUnionField
]);
```

### Run to Completion (Synchronous)

```typescript
import { runToCompletion } from '@convex-dev/migrations';

export const migrateAll = internalAction({
  handler: async (ctx) => {
    await runToCompletion(
      ctx,
      components.migrations,
      internal.migrations.setDefaultPlan
    );
  }
});
```

**Note:** If action crashes, migration won't continue in background.

## Configuration Options

### Batch Size

```typescript
export const largeDocMigration = migrations.define({
  table: 'documents',
  batchSize: 10, // Default: 100
  migrateOne: async (ctx, doc) => {
    // Process large documents with smaller batches
  }
});

// Or override at runtime
await migrations.runOne(ctx, internal.migrations.largeDocMigration, {
  batchSize: 5
});
```

### Custom internalMutation

```typescript
import { internalMutation } from './functions'; // Your custom wrapper

export const migrations = new Migrations(components.migrations, {
  internalMutation // Use custom mutation for validation/triggers
});
```

### Location Prefix

```typescript
export const migrations = new Migrations(components.migrations, {
  migrationsLocationPrefix: 'migrations:' // Shorter CLI commands
});

// Now use: npx convex run migrations:run '{"fn": "setDefaultPlan"}'
// Instead of: npx convex run migrations:run '{"fn": "migrations:setDefaultPlan"}'
```

## Monitoring Status

### CLI (Live Updates)

```bash
npx convex run --component migrations lib:getStatus --watch
```

### Programmatically

```typescript
// Get recent migrations
const status = await migrations.getStatus(ctx, { limit: 10 });

// Get specific migrations
const status = await migrations.getStatus(ctx, {
  migrations: [
    internal.migrations.setDefaultPlan,
    internal.migrations.validateField
  ]
});

// Status shape
type MigrationStatus = {
  name: string;
  state: 'inProgress' | 'success' | 'failed' | 'canceled';
  cursor?: string;
  processed: number;
  // ...
};
```

## Canceling Migrations

### CLI

```bash
npx convex run --component migrations lib:cancel '{"name": "migrations:setDefaultPlan"}'
```

### Programmatically

```typescript
await migrations.cancel(ctx, internal.migrations.setDefaultPlan);
// Or by name string
await migrations.cancel(ctx, 'migrations:setDefaultPlan');
```

## Migration Workflow

### Adding a Required Field

1. **Update schema** to allow both old and new:

   ```typescript
   // Before: field didn't exist
   // After: field is optional
   plan: v.optional(v.string());
   ```

2. **Deploy** schema + migration code

3. **Run migration**:

   ```bash
   npx convex run migrations:setDefaultPlan --prod
   ```

4. **Make field required** after migration completes:

   ```typescript
   plan: v.string(); // Now required
   ```

5. **Deploy** final schema (will fail if data doesn't match)

### Removing a Field

1. **Make field optional** in schema
2. **Update code** to stop reading/writing field
3. **Run migration** to clear field values
4. **Remove field** from schema

## Behavior Notes

| Scenario            | Behavior                              |
| ------------------- | ------------------------------------- |
| Already in progress | No-op (doesn't restart)               |
| Already completed   | Skips                                 |
| Partial progress    | Resumes from cursor                   |
| Failed/canceled     | Stops series (dependencies may exist) |
| OCC conflicts       | Reduce batchSize                      |

## Resume from Specific Cursor

```typescript
// Get cursor from getStatus, then resume
await migrations.runOne(ctx, internal.migrations.myMigration, {
  cursor: 'previous_cursor_string'
});
```

## Common Patterns

### Backfill Computed Field

```typescript
export const backfillFullName = migrations.define({
  table: 'users',
  migrateOne: async (ctx, user) => {
    if (!user.fullName && user.firstName && user.lastName) {
      return { fullName: `${user.firstName} ${user.lastName}` };
    }
  }
});
```

### Normalize Data Format

```typescript
export const normalizePhoneNumbers = migrations.define({
  table: 'contacts',
  migrateOne: async (ctx, contact) => {
    if (contact.phone && !contact.phone.startsWith('+')) {
      return { phone: `+1${contact.phone.replace(/\D/g, '')}` };
    }
  }
});
```

### Soft Delete Cleanup

```typescript
export const purgeDeletedRecords = migrations.define({
  table: 'items',
  migrateOne: async (ctx, item) => {
    if (
      item.deletedAt &&
      Date.now() - item.deletedAt > 30 * 24 * 60 * 60 * 1000
    ) {
      await ctx.db.delete(item._id);
    }
  }
});
```

## Best Practices

1. **Test locally first** - Run on dev data before production
2. **Use small batch sizes** for large documents or high-conflict tables
3. **Make schema changes first** - Allow old + new formats
4. **Handle both formats in code** during migration period
5. **Don't delete data** unless necessary - mark as deprecated
6. **Monitor progress** with `--watch` flag
7. **Run migrations in series** when there are dependencies

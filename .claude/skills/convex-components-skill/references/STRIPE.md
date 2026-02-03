# Stripe Component

Payments, subscriptions, and billing integration with Stripe.

## Installation

```bash
npm install @convex-dev/stripe
```

```typescript
// convex/convex.config.ts
import stripe from '@convex-dev/stripe/convex.config';
app.use(stripe);
```

Set environment variables:

```bash
npx convex env set STRIPE_SECRET_KEY "sk_test_..."
npx convex env set STRIPE_WEBHOOK_SECRET "whsec_..."
```

## Features

- **Checkout Sessions** - One-time payments and subscription checkouts
- **Subscription Management** - Create, update, cancel subscriptions
- **Customer Management** - Automatic customer creation and linking
- **Customer Portal** - Let users manage billing
- **Seat-Based Pricing** - Update quantities for team billing
- **User/Org Linking** - Link payments to users or organizations
- **Webhook Handling** - Auto-sync Stripe data to Convex
- **Real-time Data** - Query payments, subscriptions, invoices reactively

## Webhook Setup

### 1. Register Routes

```typescript
// convex/http.ts
import { httpRouter } from 'convex/server';
import { components } from './_generated/api';
import { registerRoutes } from '@convex-dev/stripe';

const http = httpRouter();

registerRoutes(http, components.stripe, {
  webhookPath: '/stripe/webhook'
});

export default http;
```

### 2. Configure in Stripe Dashboard

1. Go to Stripe Dashboard → Developers → Webhooks
2. Add endpoint: `https://<your-deployment>.convex.site/stripe/webhook`
3. Select events:
   - `checkout.session.completed`
   - `customer.created`, `customer.updated`
   - `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`
   - `invoice.created`, `invoice.finalized`, `invoice.paid`, `invoice.payment_failed`
   - `payment_intent.succeeded`, `payment_intent.payment_failed`
4. Copy signing secret → Set as `STRIPE_WEBHOOK_SECRET`

## Basic Usage

```typescript
// convex/stripe.ts
import { action, query } from './_generated/server';
import { components } from './_generated/api';
import { StripeSubscriptions } from '@convex-dev/stripe';
import { v } from 'convex/values';

const stripeClient = new StripeSubscriptions(components.stripe, {});
```

### Create Subscription Checkout

```typescript
export const createSubscriptionCheckout = action({
  args: { priceId: v.string() },
  returns: v.object({
    sessionId: v.string(),
    url: v.union(v.string(), v.null())
  }),
  handler: async (ctx, { priceId }) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Not authenticated');

    // Get or create Stripe customer
    const customer = await stripeClient.getOrCreateCustomer(ctx, {
      userId: identity.subject,
      email: identity.email,
      name: identity.name
    });

    // Create checkout session
    return await stripeClient.createCheckoutSession(ctx, {
      priceId,
      customerId: customer.customerId,
      mode: 'subscription',
      successUrl: 'https://myapp.com/success',
      cancelUrl: 'https://myapp.com/cancel',
      subscriptionMetadata: { userId: identity.subject }
    });
  }
});
```

### Create One-Time Payment

```typescript
export const createPaymentCheckout = action({
  args: { priceId: v.string() },
  returns: v.object({
    sessionId: v.string(),
    url: v.union(v.string(), v.null())
  }),
  handler: async (ctx, { priceId }) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Not authenticated');

    const customer = await stripeClient.getOrCreateCustomer(ctx, {
      userId: identity.subject,
      email: identity.email
    });

    return await stripeClient.createCheckoutSession(ctx, {
      priceId,
      customerId: customer.customerId,
      mode: 'payment',
      successUrl: 'https://myapp.com/success',
      cancelUrl: 'https://myapp.com/cancel',
      paymentIntentMetadata: { userId: identity.subject }
    });
  }
});
```

### Customer Portal

```typescript
export const createPortalSession = action({
  args: {},
  returns: v.object({ url: v.string() }),
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error('Not authenticated');

    const customer = await stripeClient.getOrCreateCustomer(ctx, {
      userId: identity.subject,
      email: identity.email
    });

    return await stripeClient.createCustomerPortalSession(ctx, {
      customerId: customer.customerId,
      returnUrl: 'https://myapp.com/account'
    });
  }
});
```

## Query Synced Data

```typescript
// Get user's subscriptions
export const getUserSubscriptions = query({
  args: {},
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) return [];

    return await ctx.runQuery(
      components.stripe.public.listSubscriptionsByUserId,
      { userId: identity.subject }
    );
  }
});

// Get user's payments
export const getUserPayments = query({
  args: {},
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) return [];

    return await ctx.runQuery(components.stripe.public.listPaymentsByUserId, {
      userId: identity.subject
    });
  }
});

// Get user's invoices
export const getUserInvoices = query({
  args: {},
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) return [];

    return await ctx.runQuery(components.stripe.public.listInvoicesByUserId, {
      userId: identity.subject
    });
  }
});
```

## Available Queries

| Query                       | Arguments               | Description                     |
| --------------------------- | ----------------------- | ------------------------------- |
| `getCustomer`               | `stripeCustomerId`      | Get customer by Stripe ID       |
| `listSubscriptions`         | `stripeCustomerId`      | List subscriptions for customer |
| `listSubscriptionsByUserId` | `userId`                | List subscriptions for user     |
| `getSubscription`           | `stripeSubscriptionId`  | Get subscription by ID          |
| `getSubscriptionByOrgId`    | `orgId`                 | Get subscription for org        |
| `getPayment`                | `stripePaymentIntentId` | Get payment by ID               |
| `listPayments`              | `stripeCustomerId`      | List payments for customer      |
| `listPaymentsByUserId`      | `userId`                | List payments for user          |
| `listPaymentsByOrgId`       | `orgId`                 | List payments for org           |
| `listInvoices`              | `stripeCustomerId`      | List invoices for customer      |
| `listInvoicesByUserId`      | `userId`                | List invoices for user          |
| `listInvoicesByOrgId`       | `orgId`                 | List invoices for org           |

## Subscription Management

### Cancel Subscription

```typescript
export const cancelSubscription = action({
  args: { subscriptionId: v.string() },
  handler: async (ctx, { subscriptionId }) => {
    await stripeClient.cancelSubscription(ctx, {
      subscriptionId,
      cancelAtPeriodEnd: true // Cancel at end of billing period
    });
  }
});
```

### Reactivate Subscription

```typescript
export const reactivateSubscription = action({
  args: { subscriptionId: v.string() },
  handler: async (ctx, { subscriptionId }) => {
    await stripeClient.reactivateSubscription(ctx, { subscriptionId });
  }
});
```

### Update Seats

```typescript
export const updateSeats = action({
  args: { subscriptionId: v.string(), quantity: v.number() },
  handler: async (ctx, { subscriptionId, quantity }) => {
    await stripeClient.updateSubscriptionQuantity(ctx, {
      subscriptionId,
      quantity
    });
  }
});
```

## Custom Webhook Handlers

```typescript
// convex/http.ts
import { httpRouter } from 'convex/server';
import { components } from './_generated/api';
import { registerRoutes } from '@convex-dev/stripe';
import type Stripe from 'stripe';

const http = httpRouter();

registerRoutes(http, components.stripe, {
  events: {
    'customer.subscription.updated': async (ctx, event) => {
      const subscription = event.data.object;
      console.log('Subscription updated:', subscription.id);

      // Custom logic: notify user, update permissions, etc.
      if (subscription.status === 'canceled') {
        await ctx.runMutation(internal.users.revokeAccess, {
          stripeCustomerId: subscription.customer
        });
      }
    },
    'invoice.payment_failed': async (ctx, event) => {
      const invoice = event.data.object;
      // Send dunning email, notify support, etc.
    }
  },
  onEvent: async (ctx, event) => {
    // Called for ALL events - logging/analytics
    console.log('Stripe event:', event.type);
  }
});

export default http;
```

## Database Tables

The component syncs these tables automatically:

### customers

| Field              | Type    | Description        |
| ------------------ | ------- | ------------------ |
| `stripeCustomerId` | string  | Stripe customer ID |
| `email`            | string? | Customer email     |
| `name`             | string? | Customer name      |
| `metadata`         | object? | Custom metadata    |

### subscriptions

| Field                  | Type    | Description                      |
| ---------------------- | ------- | -------------------------------- |
| `stripeSubscriptionId` | string  | Stripe subscription ID           |
| `stripeCustomerId`     | string  | Customer ID                      |
| `status`               | string  | active, canceled, past_due, etc. |
| `priceId`              | string  | Price ID                         |
| `quantity`             | number? | Seat count                       |
| `currentPeriodEnd`     | number  | Period end timestamp             |
| `cancelAtPeriodEnd`    | boolean | Will cancel at period end        |
| `userId`               | string? | Linked user ID                   |
| `orgId`                | string? | Linked org ID                    |
| `metadata`             | object? | Custom metadata                  |

### payments

| Field                   | Type    | Description             |
| ----------------------- | ------- | ----------------------- |
| `stripePaymentIntentId` | string  | Payment intent ID       |
| `stripeCustomerId`      | string? | Customer ID             |
| `amount`                | number  | Amount in cents         |
| `currency`              | string  | Currency code           |
| `status`                | string  | succeeded, failed, etc. |
| `created`               | number  | Created timestamp       |
| `userId`                | string? | Linked user ID          |
| `orgId`                 | string? | Linked org ID           |

### invoices

| Field                  | Type    | Description             |
| ---------------------- | ------- | ----------------------- |
| `stripeInvoiceId`      | string  | Invoice ID              |
| `stripeCustomerId`     | string  | Customer ID             |
| `stripeSubscriptionId` | string? | Subscription ID         |
| `status`               | string  | draft, open, paid, void |
| `amountDue`            | number  | Amount due in cents     |
| `amountPaid`           | number  | Amount paid in cents    |
| `created`              | number  | Created timestamp       |

## React Integration

```tsx
import { useAction, useQuery } from 'convex/react';
import { api } from '../convex/_generated/api';

function SubscriptionButton({ priceId }: { priceId: string }) {
  const createCheckout = useAction(api.stripe.createSubscriptionCheckout);

  const handleClick = async () => {
    const { url } = await createCheckout({ priceId });
    if (url) window.location.href = url;
  };

  return <button onClick={handleClick}>Subscribe</button>;
}

function BillingPortalButton() {
  const createPortal = useAction(api.stripe.createPortalSession);

  const handleClick = async () => {
    const { url } = await createPortal();
    window.location.href = url;
  };

  return <button onClick={handleClick}>Manage Billing</button>;
}

function SubscriptionStatus() {
  const subscriptions = useQuery(api.stripe.getUserSubscriptions);

  const active = subscriptions?.find((s) => s.status === 'active');

  return active ? (
    <p>Active subscription: {active.priceId}</p>
  ) : (
    <p>No active subscription</p>
  );
}
```

## Organization Billing

Link subscriptions to organizations instead of users:

```typescript
export const createOrgSubscription = action({
  args: { priceId: v.string(), orgId: v.string() },
  handler: async (ctx, { priceId, orgId }) => {
    const org = await ctx.runQuery(api.orgs.get, { orgId });

    const customer = await stripeClient.getOrCreateCustomer(ctx, {
      orgId,
      email: org.billingEmail,
      name: org.name
    });

    return await stripeClient.createCheckoutSession(ctx, {
      priceId,
      customerId: customer.customerId,
      mode: 'subscription',
      successUrl: 'https://myapp.com/success',
      cancelUrl: 'https://myapp.com/cancel',
      subscriptionMetadata: { orgId }
    });
  }
});

// Query org subscriptions
export const getOrgSubscription = query({
  args: { orgId: v.string() },
  handler: async (ctx, { orgId }) => {
    return await ctx.runQuery(components.stripe.public.getSubscriptionByOrgId, {
      orgId
    });
  }
});
```

## Troubleshooting

**Tables empty after checkout:**

- Verify `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET` are set
- Check webhook endpoint URL is correct
- Ensure all required events are selected in Stripe

**"Not authenticated" errors:**

- Configure auth provider in `convex/auth.config.ts`
- Verify user is signed in before calling actions

**Webhooks returning 400/500:**

- Check Convex dashboard logs for errors
- Verify `STRIPE_WEBHOOK_SECRET` matches Stripe dashboard
- Confirm webhook URL format: `https://<deployment>.convex.site/stripe/webhook`

## Best Practices

1. **Use metadata** - Link subscriptions/payments to users via `subscriptionMetadata`
2. **Handle webhooks** - Don't rely on client-side success URL for fulfillment
3. **Test with Stripe CLI** - `stripe listen --forward-to localhost:3000/stripe/webhook`
4. **Use test mode** - Develop with `sk_test_` keys before production
5. **Enable all invoice events** - Include `invoice.created` and `invoice.finalized`

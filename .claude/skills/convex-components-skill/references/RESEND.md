# Resend Component

Send reliable transactional emails with Resend integration.

## Installation

```bash
npm install @convex-dev/resend
```

```typescript
// convex/convex.config.ts
import resend from '@convex-dev/resend/convex.config';
app.use(resend);
```

Set your API key:

```bash
npx convex env set RESEND_API_KEY "re_xxxxxxxxx"
```

## Features

- **Queueing** - Send unlimited emails, delivered eventually
- **Batching** - Automatically batches to Resend's `/emails/batch` endpoint
- **Durable execution** - Uses workpools for guaranteed delivery
- **Idempotency** - Exactly-once delivery, no duplicate emails from retries
- **Rate limiting** - Honors Resend API limits

## Basic Setup

```typescript
// convex/email.ts
import { components } from './_generated/api';
import { Resend } from '@convex-dev/resend';
import { internalMutation } from './_generated/server';

export const resend = new Resend(components.resend, {});

export const sendWelcomeEmail = internalMutation({
  handler: async (ctx) => {
    await resend.sendEmail(ctx, {
      from: 'App <hello@myapp.com>',
      to: 'user@example.com',
      subject: 'Welcome!',
      html: '<h1>Welcome to our app!</h1>'
    });
  }
});
```

## Configuration Options

```typescript
export const resend = new Resend(components.resend, {
  // Provide API key directly (otherwise reads RESEND_API_KEY)
  apiKey: 're_xxxxxxxxx',

  // Webhook secret (otherwise reads RESEND_WEBHOOK_SECRET)
  webhookSecret: 'whsec_...',

  // Only allow test addresses (default: true for safety)
  testMode: true,

  // Handle email status events
  onEmailEvent: internal.email.handleEmailEvent
});
```

**Important:** `testMode` defaults to `true`. Set to `false` to send to real addresses.

## Test Addresses

Resend provides test addresses that work without domain verification:

- `delivered@resend.dev` - Simulates successful delivery
- `bounced@resend.dev` - Simulates bounce
- `complained@resend.dev` - Simulates spam complaint

Labels are supported: `delivered+user-123@resend.dev`

## Sending Emails

### Basic Send

```typescript
const emailId = await resend.sendEmail(ctx, {
  from: 'App <hello@myapp.com>',
  to: 'user@example.com',
  subject: 'Your order shipped',
  html: '<p>Your order is on the way!</p>'
});
```

### With Options

```typescript
await resend.sendEmail(ctx, {
  from: 'Support <support@myapp.com>',
  to: ['user1@example.com', 'user2@example.com'],
  subject: 'Update',
  html: '<p>HTML content</p>',
  text: 'Plain text fallback',
  replyTo: ['replies@myapp.com']
  // Additional headers supported
});
```

### Check Status

```typescript
const status = await resend.status(ctx, emailId);
// Returns current delivery status
```

## React Email Integration

React Email requires Node.js runtime:

```typescript
// convex/email.ts
'use node';

import { Resend } from '@convex-dev/resend';
import { render } from '@react-email/render';
import { WelcomeEmail } from '../emails/welcome';
import { components } from './_generated/api';
import { internalAction } from './_generated/server';

export const resend = new Resend(components.resend, {
  testMode: false
});

export const sendWelcome = internalAction({
  args: { to: v.string(), name: v.string() },
  handler: async (ctx, { to, name }) => {
    const html = await render(WelcomeEmail({ name }));

    await resend.sendEmail(ctx, {
      from: 'App <hello@myapp.com>',
      to,
      subject: 'Welcome!',
      html
    });
  }
});
```

**Note:** Must use `internalAction` with `"use node"` directive for React Email.

## Webhooks for Status Updates

### 1. Set Up HTTP Endpoint

```typescript
// convex/http.ts
import { httpRouter } from 'convex/server';
import { httpAction } from './_generated/server';
import { resend } from './email';

const http = httpRouter();

http.route({
  path: '/resend-webhook',
  method: 'POST',
  handler: httpAction(async (ctx, req) => {
    return await resend.handleResendEventWebhook(ctx, req);
  })
});

export default http;
```

### 2. Configure Resend

1. Go to Resend dashboard → Webhooks
2. Add endpoint: `https://<your-project>.convex.site/resend-webhook`
3. Select events to receive
4. Copy signing secret

```bash
npx convex env set RESEND_WEBHOOK_SECRET "whsec_..."
```

### 3. Handle Events

```typescript
// convex/email.ts
import { components, internal } from './_generated/api';
import { internalMutation } from './_generated/server';
import { Resend, vOnEmailEventArgs } from '@convex-dev/resend';

export const resend = new Resend(components.resend, {
  onEmailEvent: internal.email.handleEmailEvent
});

export const handleEmailEvent = internalMutation({
  args: vOnEmailEventArgs,
  handler: async (ctx, { id, event }) => {
    // Event types: delivered, bounced, complained, opened, clicked
    console.log(`Email ${id}: ${event.type}`);

    if (event.type === 'bounced') {
      // Mark email as invalid in your database
      await ctx.db.patch(userId, { emailValid: false });
    }
  }
});
```

## Manual Sending (Advanced)

For features not supported by batch API (like attachments):

```typescript
// convex/email.ts
'use node';

import { Resend as ResendComponent } from '@convex-dev/resend';
import { Resend } from 'resend';
import { internalAction } from './_generated/server';

const resendSdk = new Resend(process.env.RESEND_API_KEY);
export const resend = new ResendComponent(components.resend, {});

export const sendWithAttachment = internalAction({
  args: { to: v.string(), pdfData: v.string() },
  handler: async (ctx, { to, pdfData }) => {
    const from = 'App <hello@myapp.com>';
    const subject = 'Your Invoice';

    const emailId = await resend.sendEmailManually(
      ctx,
      { from, to, subject },
      async (idempotencyKey) => {
        const { data, error } = await resendSdk.emails.send({
          from,
          to,
          subject,
          html: '<p>Please find your invoice attached.</p>',
          attachments: [
            {
              filename: 'invoice.pdf',
              content: pdfData
            }
          ],
          headers: {
            'Idempotency-Key': idempotencyKey
          }
        });

        if (error) {
          throw new Error(`Failed to send: ${error.message}`);
        }
        return data.id!;
      }
    );
  }
});
```

**Use `sendEmailManually` when:**

- Sending attachments
- Using features not in batch API
- Need fine-grained control

## Complete Example

```typescript
// convex/email.ts
import { components, internal } from './_generated/api';
import { internalMutation, mutation } from './_generated/server';
import { Resend, vOnEmailEventArgs } from '@convex-dev/resend';
import { v } from 'convex/values';

export const resend = new Resend(components.resend, {
  testMode: false,
  onEmailEvent: internal.email.handleEmailEvent
});

// Public mutation to request email
export const requestPasswordReset = mutation({
  args: { email: v.string() },
  handler: async (ctx, { email }) => {
    const user = await ctx.db
      .query('users')
      .withIndex('by_email', (q) => q.eq('email', email))
      .unique();

    if (!user) return; // Silent fail for security

    const token = crypto.randomUUID();
    await ctx.db.insert('passwordResets', { userId: user._id, token });

    // Schedule email send
    await ctx.scheduler.runAfter(0, internal.email.sendResetEmail, {
      to: email,
      token
    });
  }
});

// Internal mutation to actually send
export const sendResetEmail = internalMutation({
  args: { to: v.string(), token: v.string() },
  handler: async (ctx, { to, token }) => {
    const resetUrl = `https://myapp.com/reset?token=${token}`;

    await resend.sendEmail(ctx, {
      from: 'Security <security@myapp.com>',
      to,
      subject: 'Reset your password',
      html: `
        <h1>Password Reset</h1>
        <p>Click the link below to reset your password:</p>
        <a href="${resetUrl}">Reset Password</a>
        <p>This link expires in 1 hour.</p>
      `
    });
  }
});

// Handle delivery events
export const handleEmailEvent = internalMutation({
  args: vOnEmailEventArgs,
  handler: async (ctx, { id, event }) => {
    console.log(`Email ${id}: ${event.type}`);
  }
});
```

## Best Practices

1. **Start with testMode: true** - Prevents accidental sends during development
2. **Verify your domain** - Required for production email delivery
3. **Set up webhooks** - Track bounces and complaints to maintain sender reputation
4. **Use internal functions** - Send emails from `internalMutation`/`internalAction`, not public functions
5. **Handle bounces** - Remove invalid emails from your database
6. **Include unsubscribe links** - Required for marketing emails

## Troubleshooting

**Emails not sending:**

- Check `testMode` is `false` for non-test addresses
- Verify `RESEND_API_KEY` is set correctly
- Check Convex dashboard logs for errors

**Webhooks not working:**

- Verify webhook URL is correct
- Check `RESEND_WEBHOOK_SECRET` matches Resend dashboard
- Ensure HTTP route is deployed

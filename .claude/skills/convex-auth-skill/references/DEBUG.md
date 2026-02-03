# Debugging Authentication

## Quick Diagnosis

| Symptom                              | Likely Cause                     | Fix                                       |
| ------------------------------------ | -------------------------------- | ----------------------------------------- |
| `getUserIdentity()` returns `null`   | Query runs before auth validates | Use `"skip"` pattern or `<Authenticated>` |
| `isAuthenticated: false` after login | Backend misconfigured            | Check auth.config.ts deployed             |
| Token not sent to backend            | Provider misconfigured           | Check ConvexProviderWith\* setup          |
| "Invalid token" errors               | Domain/applicationID mismatch    | Verify JWT matches auth.config.ts         |

## Step 1: Check Backend Auth

Add this to the START of your function:

```typescript
console.log('server identity', await ctx.auth.getUserIdentity());
```

Check Convex Dashboard > Logs:

- **No log appears:**
  - Wrong dashboard? Check Settings > URL matches your client
  - Client not connected? Check browser console for errors
  - Code not deployed? Run `npx convex dev`

- **Log shows `null`:**
  - Client not sending token, OR
  - Backend misconfigured
  - → Proceed to Step 2

- **Log shows `{ tokenIdentifier: '...' }`:**
  - ✅ Auth working!

## Step 2: Check Frontend Auth

### Verify Token Exists

```typescript
// Clerk
import { useAuth } from '@clerk/clerk-react';
const { getToken } = useAuth();
console.log(await getToken({ template: 'convex' }));

// WorkOS
import { useAuth } from '@workos-inc/authkit-react';
const { accessToken } = useAuth();
console.log(accessToken);
```

If no token: Check you're signed in, check provider config.

### Inspect Network Traffic

1. Open DevTools > Network tab
2. Filter: `WS` for WebSocket or `Fetch/XHR` for HTTP

**WebSocket (ConvexReactClient):**

- Find `sync` connection
- Check Messages tab
- Look for `type: "Authenticate"` with `value: "<token>"`

**HTTP (ConvexHTTPClient):**

- Check request headers
- Look for `Authorization: Bearer <token>`

## Step 3: Verify Config Match

### Check Convex Dashboard

Settings > Authentication should show:

- Domain (must match JWT `iss` field)
- Application ID (must match JWT `aud` field)

If "no configured providers": Deploy auth.config.ts

### Decode JWT Token

1. Go to https://jwt.io
2. Paste the full token
3. Check PAYLOAD:
   - `iss` must match `domain` in auth.config.ts
   - `aud` must match `applicationID` in auth.config.ts

## Common Issues

### "Not authenticated" on Page Load

**Problem:** Queries run before auth completes.

**Fix:** Skip queries until authenticated:

```typescript
const { isAuthenticated } = useConvexAuth();
const data = useQuery(api.myQuery, isAuthenticated ? {} : 'skip');
```

Or use `<Authenticated>` component:

```typescript
<Authenticated>
  <MyComponent /> {/* Queries here are safe */}
</Authenticated>
```

### JWT Template Name Wrong (Clerk)

**Problem:** Clerk JWT template must be named exactly `convex`.

**Fix:** In Clerk Dashboard > JWT Templates, ensure template is named `convex` (not "Convex" or "my-template").

### Environment Variable Not Set

**Problem:** `domain: process.env.CLERK_JWT_ISSUER_DOMAIN!` is undefined.

**Fix:**

1. Set variable in Convex Dashboard > Settings > Environment Variables
2. Run `npx convex dev` to sync

### Wrong Provider Hook Used

**Problem:** Using `useAuth()` from provider instead of `useConvexAuth()`.

```typescript
// ❌ Wrong - doesn't wait for Convex validation
const { isSignedIn } = useAuth();

// ✅ Correct
const { isAuthenticated } = useConvexAuth();
```

### auth.config.ts Not Deployed

**Problem:** Changed auth.config.ts but didn't deploy.

**Fix:**

- Dev: Ensure `npx convex dev` is running
- Prod: Run `npx convex deploy`

### CORS Issues (WorkOS)

**Problem:** WorkOS AuthKit needs CORS configured.

**Fix:** WorkOS Dashboard > Authentication > Sessions > CORS > Add your domain.

## Logging Auth State

Add comprehensive logging:

```typescript
// Client-side
const { isLoading, isAuthenticated } = useConvexAuth();
console.log('Auth state:', { isLoading, isAuthenticated });

// Server-side
export const debugAuth = query({
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    console.log('Identity:', identity);
    return {
      authenticated: identity !== null,
      subject: identity?.subject,
      issuer: identity?.issuer
    };
  }
});
```

## Environment Checklist

### Development

- [ ] `npx convex dev` running
- [ ] auth.config.ts has correct domain
- [ ] Provider has development keys (pk*test*_, sk*test*_)
- [ ] Redirect URI matches dev URL (localhost:5173, etc.)

### Production

- [ ] `npx convex deploy` run after auth.config.ts changes
- [ ] Convex Dashboard has prod environment variables
- [ ] Provider has production keys (pk*live*_, sk*live*_)
- [ ] Redirect URI matches prod URL
- [ ] CORS configured (if applicable)

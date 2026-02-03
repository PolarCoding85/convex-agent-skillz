# WorkOS AuthKit Setup Guide

## Manual Setup (Existing WorkOS Account)

### 1. Sign Up for WorkOS

Go to [workos.com/sign-up](https://signin.workos.com/sign-up)

### 2. Enable AuthKit

1. Dashboard > **Authentication** > **AuthKit**
2. Click **Set up AuthKit**
3. Select **Use AuthKit's customizable hosted UI**

### 3. Configure Redirect URI

In AuthKit setup step 4, set the redirect endpoint:

- Development: `http://localhost:5173/callback` (Vite) or `http://localhost:3000/callback` (Next.js)
- Production: `https://your-domain.com/callback`

### 4. Get Credentials

From [dashboard.workos.com/get-started](https://dashboard.workos.com/get-started):

- Copy **Client ID** (format: `client_01XXXX...`)
- Copy **API Key** (format: `sk_test_...` or `sk_live_...`)

### 5. Set Convex Environment Variable

In Convex Dashboard > Settings > Environment Variables:

- Name: `WORKOS_CLIENT_ID`
- Value: Your client ID

### 6. Configure CORS (React/Vite apps only)

1. WorkOS Dashboard > **Authentication** > **Sessions**
2. Click **Cross-Origin Resource Sharing (CORS)** > **Manage**
3. Add your domains:
   - `http://localhost:5173` (development)
   - `https://your-domain.com` (production)

## Auto-Provisioning (New Projects)

For new projects, Convex can auto-provision WorkOS environments:

```bash
npm create convex@latest -- -t react-vite-authkit
cd my-app
npm run dev
```

Follow prompts to create a WorkOS team linked to your Convex team.

### What Auto-Provisioning Configures

- Creates WorkOS environment for each dev deployment
- Sets redirect URI automatically
- Configures CORS origin
- Sets local environment variables:
  - `VITE_WORKOS_CLIENT_ID` or `WORKOS_CLIENT_ID`
  - `*_WORKOS_REDIRECT_URI`
  - `WORKOS_API_KEY` (Next.js)
  - `WORKOS_COOKIE_PASSWORD` (Next.js)

### Limitations

- Only works for **development** deployments
- Production deployments require manual configuration
- One production environment per WorkOS team

## Environment Variables Reference

### React/Vite

```bash
# .env.local
VITE_WORKOS_CLIENT_ID=client_01...
VITE_WORKOS_REDIRECT_URI=http://localhost:5173/callback
```

### Next.js

```bash
# .env.local
WORKOS_CLIENT_ID=client_01...
WORKOS_API_KEY=sk_test_...
WORKOS_COOKIE_PASSWORD=generate_with_openssl_rand_base64_24
NEXT_PUBLIC_WORKOS_REDIRECT_URI=http://localhost:3000/callback
```

Generate cookie password:

```bash
openssl rand -base64 24
```

### Convex Dashboard

```
WORKOS_CLIENT_ID=client_01...
```

## Production Deployment Checklist

- [ ] Create production WorkOS environment (if separate from dev)
- [ ] Update WORKOS_CLIENT_ID in Convex prod deployment
- [ ] Update WORKOS*API_KEY to `sk_live*...`
- [ ] Update redirect URI to production domain
- [ ] Add production domain to CORS
- [ ] Run `npx convex deploy`

## Auth Config Explained

WorkOS requires two provider entries because JWTs can come from different issuers:

```typescript
// convex/auth.config.ts
const clientId = process.env.WORKOS_CLIENT_ID;

export default {
  providers: [
    // Provider 1: Standard API issuer
    {
      type: 'customJwt',
      issuer: 'https://api.workos.com/',
      algorithm: 'RS256',
      applicationID: clientId,
      jwks: `https://api.workos.com/sso/jwks/${clientId}`
    },
    // Provider 2: User management issuer
    {
      type: 'customJwt',
      issuer: `https://api.workos.com/user_management/${clientId}`,
      algorithm: 'RS256',
      jwks: `https://api.workos.com/sso/jwks/${clientId}`
    }
  ]
};
```

## Troubleshooting Commands

### Disconnect and Reconnect WorkOS Team

If you see "Platform not authorized" errors:

```bash
npx convex integration workos disconnect-team
npx convex integration workos provision-team
```

Note: You'll need a different email for the new workspace.

### Check Auth Configuration

In Convex Dashboard > Settings > Authentication, verify:

- Both provider domains are listed
- Client ID matches your WorkOS app

### Debug JWT Token

1. In browser DevTools > Network > WS > sync
2. Find `Authenticate` message
3. Copy token to https://jwt.io
4. Verify `iss` matches one of your auth.config.ts issuers
5. Verify `aud` matches your client ID

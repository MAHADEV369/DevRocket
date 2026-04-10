# Skill 13: Google OAuth2 Sign-In

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 2-3 hours
Depends On: 03

## Input Contract
- Skill 03 (authentication) completed with JWT-based auth, user model, and login/register routes
- Prisma schema has a `User` model with `id`, `email`, `password`, `name` fields
- Express app has session/cookie support or JWT token generation infrastructure
- Google Cloud Console project created with OAuth2 credentials (client ID and secret)
- `.env` file exists with `JWT_SECRET` configured

## Output Contract
- OAuth routes: `GET /auth/google` (redirect to Google), `GET /auth/google/callback` (handle callback)
- CSRF-protected state parameter for the OAuth flow
- Find-or-create user on successful Google login
- Link Google account to existing user by email match
- Redirect to frontend with JWT tokens on success
- `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` added to `src/config/env.ts` and `.env.example`
- Google avatar URL stored on user profile

## Files to Create
- `src/modules/auth/auth.oauth.routes.ts` - Express routes for Google OAuth flow
- `src/modules/auth/auth.oauth.controller.ts` - Controller for OAuth redirect and callback
- `src/modules/auth/auth.oauth.service.ts` - Business logic for Google token exchange and user management
- `src/modules/auth/auth.oauth.types.ts` - TypeScript types for OAuth-related data

## Steps

### 1. Install Google OAuth dependencies

```bash
npm install google-auth-library
```

### 2. Add Google OAuth configuration to environment schema

In `src/config/env.ts`, add the following to the zod schema:

```typescript
GOOGLE_CLIENT_ID: z.string().min(1).default('placeholder_client_id'),
GOOGLE_CLIENT_SECRET: z.string().min(1).default('placeholder_client_secret'),
GOOGLE_CALLBACK_URL: z.string().url().default('http://localhost:3000/api/auth/google/callback'),
FRONTEND_URL: z.string().url().default('http://localhost:5173'),
```

### 3. Add OAuth fields to the User model in Prisma schema

Add the following fields to the existing `User` model in `prisma/schema.prisma`:

```prisma
model User {
  // ... existing fields ...
  googleId       String?   @unique
  avatarUrl      String?
  emailVerified  Boolean   @default(false)

  @@index([googleId])
}
```

Then run the migration:

```bash
npx prisma migrate dev --name add-google-oauth-fields
npx prisma generate
```

### 4. Create OAuth types

Create `src/modules/auth/auth.oauth.types.ts`:

```typescript
export interface GoogleUserInfo {
  sub: string;
  email: string;
  email_verified: boolean;
  name: string;
  given_name: string;
  family_name: string;
  picture: string;
  locale: string;
}

export interface OAuthState {
  csrfToken: string;
  redirectUrl?: string;
  timestamp: number;
}

export interface OAuthResult {
  success: boolean;
  user?: {
    id: string;
    email: string;
    name: string;
  };
  token?: string;
  error?: string;
}
```

### 5. Create the OAuth service

Create `src/modules/auth/auth.oauth.service.ts`:

```typescript
import { OAuth2Client } from 'google-auth-library';
import { env } from '../../config/env';
import { prisma } from '../../config/database';
import { generateToken } from '../../middleware/auth';
import { GoogleUserInfo, OAuthState } from './auth.oauth.types';
import crypto from 'crypto';

const oauth2Client = new OAuth2Client(
  env.GOOGLE_CLIENT_ID,
  env.GOOGLE_CLIENT_SECRET,
  env.GOOGLE_CALLBACK_URL
);

const stateStore = new Map<string, OAuthState>();

const STATE_TTL = 10 * 60 * 1000; // 10 minutes

export function getGoogleAuthUrl(state: string): string {
  const url = oauth2Client.generateAuthUrl({
    access_type: 'offline',
    scope: [
      'https://www.googleapis.com/auth/userinfo.profile',
      'https://www.googleapis.com/auth/userinfo.email',
    ],
    state,
    prompt: 'consent',
  });
  return url;
}

export function generateState(redirectUrl?: string): string {
  const csrfToken = crypto.randomBytes(32).toString('hex');
  const state: OAuthState = {
    csrfToken,
    redirectUrl,
    timestamp: Date.now(),
  };

  // Encode state as base64 to pass through OAuth
  const encoded = Buffer.from(JSON.stringify(state)).toString('base64');

  // Store in memory for validation (use Redis in production)
  stateStore.set(csrfToken, state);

  // Clean up expired states
  for (const [key, value] of stateStore.entries()) {
    if (Date.now() - value.timestamp > STATE_TTL) {
      stateStore.delete(key);
    }
  }

  return encoded;
}

export function validateState(encodedState: string): OAuthState | null {
  try {
    const decoded = JSON.parse(
      Buffer.from(encodedState, 'base64').toString('utf-8')
    ) as OAuthState;

    const stored = stateStore.get(decoded.csrfToken);

    if (!stored) {
      return null;
    }

    if (Date.now() - stored.timestamp > STATE_TTL) {
      stateStore.delete(decoded.csrfToken);
      return null;
    }

    stateStore.delete(decoded.csrfToken);

    return stored;
  } catch {
    return null;
  }
}

export async function exchangeCodeForTokens(code: string) {
  const { tokens } = await oauth2Client.getToken(code);
  return tokens;
}

export async function getGoogleUserInfo(accessToken: string): Promise<GoogleUserInfo> {
  const url = 'https://www.googleapis.com/oauth2/v3/userinfo';
  const response = await fetch(url, {
    headers: { Authorization: `Bearer ${accessToken}` },
  });

  if (!response.ok) {
    throw new Error(`Failed to fetch Google user info: ${response.statusText}`);
  }

  return response.json() as Promise<GoogleUserInfo>;
}

export async function findOrCreateUser(googleUser: GoogleUserInfo) {
  const existingByGoogleId = await prisma.user.findUnique({
    where: { googleId: googleUser.sub },
  });

  if (existingByGoogleId) {
    const updated = await prisma.user.update({
      where: { id: existingByGoogleId.id },
      data: {
        name: googleUser.name,
        avatarUrl: googleUser.picture,
        emailVerified: true,
      },
    });

    return {
      user: updated,
      token: generateToken(updated.id, updated.email),
      isNewUser: false,
    };
  }

  const existingByEmail = await prisma.user.findUnique({
    where: { email: googleUser.email },
  });

  if (existingByEmail) {
    const linked = await prisma.user.update({
      where: { id: existingByEmail.id },
      data: {
        googleId: googleUser.sub,
        avatarUrl: googleUser.picture,
        emailVerified: true,
      },
    });

    return {
      user: linked,
      token: generateToken(linked.id, linked.email),
      isNewUser: false,
    };
  }

  const newUser = await prisma.user.create({
    data: {
      email: googleUser.email,
      name: googleUser.name,
      googleId: googleUser.sub,
      avatarUrl: googleUser.picture,
      emailVerified: true,
      password: crypto.randomBytes(32).toString('hex'),
    },
  });

  return {
    user: newUser,
    token: generateToken(newUser.id, newUser.email),
    isNewUser: true,
  };
}
```

### 6. Create the OAuth controller

Create `src/modules/auth/auth.oauth.controller.ts`:

```typescript
import { Request, Response } from 'express';
import {
  getGoogleAuthUrl,
  generateState,
  validateState,
  exchangeCodeForTokens,
  getGoogleUserInfo,
  findOrCreateUser,
} from './auth.oauth.service';
import { env } from '../../config/env';

export async function googleRedirect(req: Request, res: Response) {
  const redirectUrl = req.query.redirect as string | undefined;
  const state = generateState(redirectUrl);
  const authUrl = getGoogleAuthUrl(state);

  res.redirect(authUrl);
}

export async function googleCallback(req: Request, res: Response) {
  const { code, state: encodedState, error } = req.query;

  if (error) {
    const errorMsg = typeof error === 'string' ? error : 'unknown_error';
    res.redirect(`${env.FRONTEND_URL}/auth/error?message=${encodeURIComponent(errorMsg)}`);
    return;
  }

  if (!code || !encodedState) {
    res.redirect(`${env.FRONTEND_URL}/auth/error?message=${encodeURIComponent('Missing code or state')}`);
    return;
  }

  const state = validateState(encodedState as string);

  if (!state) {
    res.redirect(`${env.FRONTEND_URL}/auth/error?message=${encodeURIComponent('Invalid or expired state parameter')}`);
    return;
  }

  try {
    const tokens = await exchangeCodeForTokens(code as string);

    if (!tokens.access_token) {
      throw new Error('No access token received from Google');
    }

    const googleUser = await getGoogleUserInfo(tokens.access_token);

    const result = await findOrCreateUser(googleUser);

    const redirectBase = state.redirectUrl || `${env.FRONTEND_URL}/auth/callback`;
    const redirectUrl = `${redirectBase}?token=${result.token}&isNewUser=${result.isNewUser}`;

    res.redirect(redirectUrl);
  } catch (err: any) {
    console.error('Google OAuth callback error:', err.message);
    res.redirect(`${env.FRONTEND_URL}/auth/error?message=${encodeURIComponent('Authentication failed')}`);
  }
}
```

### 7. Create the OAuth routes

Create `src/modules/auth/auth.oauth.routes.ts`:

```typescript
import { Router } from 'express';
import { googleRedirect, googleCallback } from './auth.oauth.controller';

const router = Router();

router.get('/google', googleRedirect);
router.get('/google/callback', googleCallback);

export default router;
```

### 8. Register OAuth routes in the main app

In `src/index.ts` (or wherever auth routes are registered), add:

```typescript
import authOAuthRoutes from './modules/auth/auth.oauth.routes';

app.use('/api/auth', authOAuthRoutes);
```

### 9. Update .env.example

Add the following entries to `.env.example`:

```
GOOGLE_CLIENT_ID=your_google_client_id_here
GOOGLE_CLIENT_SECRET=your_google_client_secret_here
GOOGLE_CALLBACK_URL=http://localhost:3000/api/auth/google/callback
FRONTEND_URL=http://localhost:5173
```

## Verification

1. Run the Prisma migration:
   ```bash
   npx prisma migrate dev --name add-google-oauth-fields
   npx prisma generate
   ```
2. Compile the project: `npx tsc --noEmit` (no type errors)
3. Set up Google OAuth credentials:
   - Go to [Google Cloud Console](https://console.cloud.google.com/)
   - Create OAuth 2.0 Client ID credentials
   - Set authorized redirect URI to `http://localhost:3000/api/auth/google/callback`
   - Copy the Client ID and Client Secret to `.env`
4. Start the server: `npm run dev`
5. Test the OAuth flow:
   - Visit `http://localhost:3000/api/auth/google`
   - Should redirect to Google login page
   - After consent, should redirect back to callback URL
   - On success, redirects to `FRONTEND_URL/auth/callback?token=xxx&isNewUser=xxx`
6. Verify the user was created in the database with `googleId`, `avatarUrl`, and `emailVerified: true`.
7. Test linking: log in with an existing email via Google and verify the account was linked (`googleId` populated).

Expected output:
- Visiting `/api/auth/google` redirects to Google's OAuth consent screen
- Successful authentication redirects to the frontend with a JWT token
- Existing users with matching emails have their Google ID linked
- New users are created with a random password and Google profile data
- Invalid or expired state parameters redirect to an error page

## Rollback

1. Remove installed packages:
   ```bash
   npm uninstall google-auth-library
   ```
2. Revert the Prisma migration:
   ```bash
   npx prisma migrate rollback
   ```
3. Remove `googleId`, `avatarUrl`, and `emailVerified` fields from the `User` model in `prisma/schema.prisma`
4. Delete created files:
   ```bash
   rm src/modules/auth/auth.oauth.routes.ts src/modules/auth/auth.oauth.controller.ts src/modules/auth/auth.oauth.service.ts src/modules/auth/auth.oauth.types.ts
   ```
5. Remove OAuth route registration from `src/index.ts`
6. Remove Google config from `src/config/env.ts` and `.env.example`
7. Run `npx prisma generate` to update the client

## ADR-013: Google OAuth2 with State-Based CSRF Protection

**Decision**: Implement Google OAuth2 sign-in using the Authorization Code flow with server-side token exchange and a base64-encoded state parameter for CSRF protection.

**Reason**: The Authorization Code flow keeps the client secret server-side (no exposure to browsers), and server-side token exchange is more secure than the implicit flow. The state parameter encodes a CSRF token and optional redirect URL, preventing cross-site request forgery attacks. Finding-or-creating users by email allows existing users to seamlessly link their Google account.

**Consequences**:
- State is stored in memory; this should be moved to Redis for multi-instance deployments
- Users who sign up via Google get a random password, meaning they can also set a password later via a "reset password" flow
- The `emailVerified` flag is set to `true` for Google-authenticated users since Google has already verified the email
- If a user with the same email already exists, the Google account is automatically linked; this could be a security concern if an attacker creates an account with the victim's email before the victim signs up with Google (mitigate by verifying email ownership first)
- The redirect URL in the state parameter should be validated against an allowlist in production to prevent open redirect attacks

**Alternatives Considered**:
- **Google One Tap / Sign In With Google (client-side)**: Simpler UX but exposes tokens to the client side; harder to validate server-side; less control over the auth flow
- **Passport.js with Google Strategy**: Well-known but adds Passport as a dependency; our simple flow doesn't need the full Passport middleware abstraction
- **OpenID ConnectDiscovery**: More standards-compliant but overkill when we only need Google sign-in; the `google-auth-library` handles the OIDC specifics
- **Session-based OAuth**: Would require server-side sessions; our JWT-based approach is simpler for SPAs
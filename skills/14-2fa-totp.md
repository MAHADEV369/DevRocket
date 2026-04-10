# Skill 14: Two-Factor Authentication with TOTP

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 2-3 hours
Depends On: 03

## Input Contract
- Skill 03 (authentication) completed with JWT auth, login route, and user model
- Prisma schema has a `User` model with `id`, `email`, `password` fields
- Express app has authentication middleware that populates `req.user`
- bcrypt or argon2 used for password hashing in the existing auth module

## Output Contract
- `twoFactorSecret` and `twoFactorEnabled` fields added to User model in Prisma schema
- Setup endpoint (`POST /auth/2fa/setup`) generates a TOTP secret and returns QR code URI
- Verify endpoint (`POST /auth/2fa/verify`) confirms setup with a valid TOTP code
- Disable endpoint (`POST /auth/2fa/disable`) disables 2FA after password + code verification
- Login flow modified: when 2FA is enabled, login returns a partial token requiring TOTP code
- TOTP verification uses `otplib` for generation and validation
- 10 backup codes generated on setup, stored as bcrypt hashes
- All 2FA config validated via zod in `src/config/env.ts`

## Files to Create
- `src/modules/auth/auth.2fa.routes.ts` - Express routes for 2FA endpoints
- `src/modules/auth/auth.2fa.controller.ts` - Request handlers for 2FA operations
- `src/modules/auth/auth.2fa.service.ts` - Business logic for TOTP setup, verify, and backup codes
- `src/modules/auth/auth.2fa.types.ts` - TypeScript types for 2FA requests and responses

## Steps

### 1. Install TOTP and QR code dependencies

```bash
npm install otplib qrcode
npm install -D @types/qrcode
```

### 2. Add 2FA fields to User model in Prisma schema

Add the following fields to the existing `User` model in `prisma/schema.prisma`:

```prisma
model User {
  // ... existing fields ...
  twoFactorEnabled   Boolean   @default(false)
  twoFactorSecret    String?
  twoFactorBackupCodes String?  // Stored as JSON array of bcrypt hashes

  @@index([twoFactorEnabled])
}
```

Then run the migration:

```bash
npx prisma migrate dev --name add-two-factor-auth
npx prisma generate
```

### 3. Create 2FA types

Create `src/modules/auth/auth.2fa.types.ts`:

```typescript
export interface Setup2FAResponse {
  secret: string;
  otpauthUrl: string;
  qrCodeDataUrl: string;
  backupCodes: string[];
}

export interface Verify2FASetupRequest {
  code: string;
}

export interface Verify2FASetupResponse {
  verified: boolean;
  message: string;
}

export interface Disable2FARequest {
  password: string;
  code: string;
}

export interface Login2FARequest {
  tempToken: string;
  code: string;
  backupCode?: string;
}

export interface Login2FAResponse {
  token: string;
  user: {
    id: string;
    email: string;
    name: string;
    twoFactorEnabled: boolean;
  };
}
```

### 4. Create the 2FA service

Create `src/modules/auth/auth.2fa.service.ts`:

```typescript
import { authenticator } from 'otplib';
import QRCode from 'qrcode';
import bcrypt from 'bcrypt';
import crypto from 'crypto';
import { prisma } from '../../config/database';
import { generateToken } from '../../middleware/auth';
import { Setup2FAResponse } from './auth.2fa.types';

const SALT_ROUNDS = 10;
const BACKUP_CODES_COUNT = 10;

export function generateSecret(): string {
  return authenticator.generateSecret();
}

export async function generateQrCodeDataUrl(otpauthUrl: string): Promise<string> {
  return QRCode.toDataURL(otpauthUrl, {
    width: 256,
    margin: 2,
    color: {
      dark: '#000000',
      light: '#ffffff',
    },
  });
}

export function buildOtpauthUrl(secret: string, email: string, issuer: string = 'FastDepo'): string {
  return authenticator.keyuri(email, issuer, secret);
}

export function verifyTotpCode(secret: string, code: string): boolean {
  return authenticator.verify(code, secret);
}

export async function hashBackupCodes(codes: string[]): Promise<string[]> {
  const hashedCodes = await Promise.all(
    codes.map((code) => bcrypt.hash(code, SALT_ROUNDS))
  );
  return hashedCodes;
}

export function generateBackupCodes(): string[] {
  const codes: string[] = [];
  for (let i = 0; i < BACKUP_CODES_COUNT; i++) {
    const code = crypto.randomBytes(4).toString('hex').toUpperCase();
    // Format as XXXX-XXXX for readability
    codes.push(`${code.slice(0, 4)}-${code.slice(4)}`);
  }
  return codes;
}

export async function verifyBackupCode(
  storedHashes: string[],
  providedCode: string
): Promise<{ valid: boolean; remainingHashes: string[] }> {
  for (let i = 0; i < storedHashes.length; i++) {
    const match = await bcrypt.compare(providedCode, storedHashes[i]);
    if (match) {
      const remainingHashes = storedHashes.filter((_, idx) => idx !== i);
      return { valid: true, remainingHashes };
    }
  }

  return { valid: false, remainingHashes: storedHashes };
}

export async function setup2FA(userId: string): Promise<Setup2FAResponse> {
  const user = await prisma.user.findUnique({ where: { id: userId } });

  if (!user) {
    throw new Error('User not found');
  }

  if (user.twoFactorEnabled) {
    throw new Error('2FA is already enabled for this user');
  }

  const secret = generateSecret();
  const otpauthUrl = buildOtpauthUrl(secret, user.email);
  const qrCodeDataUrl = await generateQrCodeDataUrl(otpauthUrl);
  const backupCodes = generateBackupCodes();
  const hashedBackupCodes = await hashBackupCodes(backupCodes);

  // Store the secret temporarily (not yet enabled)
  // User must verify with a code before 2FA is activated
  await prisma.user.update({
    where: { id: userId },
    data: {
      twoFactorSecret: secret,
      twoFactorBackupCodes: JSON.stringify(hashedBackupCodes),
    },
  });

  return {
    secret,
    otpauthUrl,
    qrCodeDataUrl,
    backupCodes,
  };
}

export async function verify2FASetup(userId: string, code: string): Promise<{ verified: boolean }> {
  const user = await prisma.user.findUnique({ where: { id: userId } });

  if (!user) {
    throw new Error('User not found');
  }

  if (!user.twoFactorSecret) {
    throw new Error('2FA setup has not been initiated');
  }

  if (user.twoFactorEnabled) {
    throw new Error('2FA is already enabled');
  }

  const isValid = verifyTotpCode(user.twoFactorSecret, code);

  if (!isValid) {
    return { verified: false };
  }

  await prisma.user.update({
    where: { id: userId },
    data: { twoFactorEnabled: true },
  });

  return { verified: true };
}

export async function disable2FA(userId: string, password: string, code: string): Promise<{ disabled: boolean }> {
  const user = await prisma.user.findUnique({ where: { id: userId } });

  if (!user) {
    throw new Error('User not found');
  }

  if (!user.twoFactorEnabled) {
    throw new Error('2FA is not enabled');
  }

  // Verify password
  const bcrypt = await import('bcrypt');
  const passwordValid = await bcrypt.compare(password, user.password);
  if (!passwordValid) {
    throw new Error('Invalid password');
  }

  // Verify TOTP code or backup code
  const totpValid = verifyTotpCode(user.twoFactorSecret!, code);

  if (!totpValid && user.twoFactorBackupCodes) {
    const storedHashes: string[] = JSON.parse(user.twoFactorBackupCodes);
    const { valid, remainingHashes } = await verifyBackupCode(storedHashes, code);

    if (!valid) {
      throw new Error('Invalid 2FA code');
    }

    await prisma.user.update({
      where: { id: userId },
      data: { twoFactorBackupCodes: JSON.stringify(remainingHashes) },
    });
  } else if (!totpValid) {
    throw new Error('Invalid 2FA code');
  }

  await prisma.user.update({
    where: { id: userId },
    data: {
      twoFactorEnabled: false,
      twoFactorSecret: null,
      twoFactorBackupCodes: null,
    },
  });

  return { disabled: true };
}

export async function verify2FALogin(
  userId: string,
  code: string,
  isBackupCode: boolean
): Promise<{ token: string }> {
  const user = await prisma.user.findUnique({ where: { id: userId } });

  if (!user) {
    throw new Error('User not found');
  }

  if (!user.twoFactorEnabled) {
    throw new Error('2FA is not enabled for this user');
  }

  if (isBackupCode && user.twoFactorBackupCodes) {
    const storedHashes: string[] = JSON.parse(user.twoFactorBackupCodes);
    const { valid, remainingHashes } = await verifyBackupCode(storedHashes, code);

    if (!valid) {
      throw new Error('Invalid backup code');
    }

    await prisma.user.update({
      where: { id: userId },
      data: { twoFactorBackupCodes: JSON.stringify(remainingHashes) },
    });

    const token = generateToken(user.id, user.email);
    return { token };
  }

  const isValid = verifyTotpCode(user.twoFactorSecret!, code);

  if (!isValid) {
    throw new Error('Invalid 2FA code');
  }

  const token = generateToken(user.id, user.email);
  return { token };
}

export async function regenerateBackupCodes(userId: string, code: string): Promise<string[]> {
  const user = await prisma.user.findUnique({ where: { id: userId } });

  if (!user) {
    throw new Error('User not found');
  }

  if (!user.twoFactorEnabled) {
    throw new Error('2FA is not enabled');
  }

  const isValid = verifyTotpCode(user.twoFactorSecret!, code);

  if (!isValid && user.twoFactorBackupCodes) {
    const storedHashes: string[] = JSON.parse(user.twoFactorBackupCodes);
    const { valid } = await verifyBackupCode(storedHashes, code);
    if (!valid) {
      throw new Error('Invalid 2FA code');
    }
  } else if (!isValid) {
    throw new Error('Invalid 2FA code');
  }

  const backupCodes = generateBackupCodes();
  const hashedBackupCodes = await hashBackupCodes(backupCodes);

  await prisma.user.update({
    where: { id: userId },
    data: { twoFactorBackupCodes: JSON.stringify(hashedBackupCodes) },
  });

  return backupCodes;
}
```

### 5. Create the 2FA controller

Create `src/modules/auth/auth.2fa.controller.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';
import * as twoFAService from './auth.2fa.service';

export async function setup2FA(req: Request, res: Response, next: NextFunction) {
  try {
    const userId = req.user!.id;

    const result = await twoFAService.setup2FA(userId);

    res.json({
      secret: result.secret,
      otpauthUrl: result.otpauthUrl,
      qrCodeDataUrl: result.qrCodeDataUrl,
      backupCodes: result.backupCodes,
    });
  } catch (error: any) {
    if (error.message === '2FA is already enabled for this user') {
      res.status(400).json({ error: error.message });
      return;
    }
    next(error);
  }
}

export async function verify2FASetup(req: Request, res: Response, next: NextFunction) {
  try {
    const userId = req.user!.id;
    const { code } = req.body;

    if (!code) {
      res.status(400).json({ error: '2FA code is required' });
      return;
    }

    const result = await twoFAService.verify2FASetup(userId, code);

    if (!result.verified) {
      res.status(400).json({ error: 'Invalid 2FA code' });
      return;
    }

    res.json({ verified: true, message: '2FA has been enabled successfully' });
  } catch (error: any) {
    if (error.message === '2FA setup has not been initiated' || error.message === '2FA is already enabled') {
      res.status(400).json({ error: error.message });
      return;
    }
    next(error);
  }
}

export async function disable2FA(req: Request, res: Response, next: NextFunction) {
  try {
    const userId = req.user!.id;
    const { password, code } = req.body;

    if (!password || !code) {
      res.status(400).json({ error: 'Password and 2FA code are required' });
      return;
    }

    const result = await twoFAService.disable2FA(userId, password, code);

    res.json({ disabled: true, message: '2FA has been disabled' });
  } catch (error: any) {
    if (error.message === 'Invalid password' || error.message === 'Invalid 2FA code' || error.message === '2FA is not enabled') {
      res.status(400).json({ error: error.message });
      return;
    }
    next(error);
  }
}

export async function verify2FALogin(req: Request, res: Response, next: NextFunction) {
  try {
    const { tempToken, code, backupCode } = req.body;

    if (!tempToken || !code) {
      res.status(400).json({ error: 'Temporary token and code are required' });
      return;
    }

    // Decode the temp token to get the user ID
    // The temp token is a short-lived JWT with a special claim indicating 2FA is pending
    const jwt = await import('jsonwebtoken');
    const { env } = await import('../../config/env');

    let decoded: any;
    try {
      decoded = jwt.verify(tempToken, env.JWT_SECRET);
    } catch {
      res.status(401).json({ error: 'Invalid or expired temporary token' });
      return;
    }

    if (!decoded.requires2FA || !decoded.userId) {
      res.status(401).json({ error: 'Invalid temporary token' });
      return;
    }

    const isBackupCode = !!backupCode;
    const result = await twoFAService.verify2FALogin(decoded.userId, code, isBackupCode);

    res.json({
      token: result.token,
      message: '2FA verification successful',
    });
  } catch (error: any) {
    if (error.message === 'Invalid 2FA code' || error.message === 'Invalid backup code') {
      res.status(401).json({ error: error.message });
      return;
    }
    next(error);
  }
}

export async function regenerateBackupCodes(req: Request, res: Response, next: NextFunction) {
  try {
    const userId = req.user!.id;
    const { code } = req.body;

    if (!code) {
      res.status(400).json({ error: '2FA code is required' });
      return;
    }

    const backupCodes = await twoFAService.regenerateBackupCodes(userId, code);

    res.json({ backupCodes, message: 'New backup codes generated. Store these securely.' });
  } catch (error: any) {
    if (error.message === 'Invalid 2FA code' || error.message === '2FA is not enabled') {
      res.status(400).json({ error: error.message });
      return;
    }
    next(error);
  }
}
```

### 6. Create the 2FA routes

Create `src/modules/auth/auth.2fa.routes.ts`:

```typescript
import { Router } from 'express';
import { authenticate } from '../../middleware/auth';
import {
  setup2FA,
  verify2FASetup,
  disable2FA,
  verify2FALogin,
  regenerateBackupCodes,
} from './auth.2fa.controller';

const router = Router();

// Public endpoint - called during login after a partial token is obtained
router.post('/verify', verify2FALogin);

// Authenticated endpoints - require full authentication
router.post('/setup', authenticate, setup2FA);
router.post('/verify-setup', authenticate, verify2FASetup);
router.post('/disable', authenticate, disable2FA);
router.post('/backup-codes/regenerate', authenticate, regenerateBackupCodes);

export default router;
```

### 7. Modify the login flow to support 2FA

In the existing login controller (e.g., `src/modules/auth/auth.controller.ts`), modify the login handler to check for 2FA:

```typescript
import jwt from 'jsonwebtoken';
import { env } from '../../config/env';

// Inside the login handler, after password verification:
if (user.twoFactorEnabled) {
  const tempToken = jwt.sign(
    {
      userId: user.id,
      email: user.email,
      requires2FA: true,
    },
    env.JWT_SECRET,
    { expiresIn: '5m' } // Short-lived token for 2FA verification
  );

  return res.json({
    requires2FA: true,
    tempToken,
    message: '2FA verification required',
  });
}

// If 2FA is not enabled, continue with normal token generation
const token = generateToken(user.id, user.email);
res.json({ token, user: { id: user.id, email: user.email, name: user.name } });
```

### 8. Register 2FA routes in the main app

In `src/index.ts`, add:

```typescript
import auth2FARoutes from './modules/auth/auth.2fa.routes';

app.use('/api/auth/2fa', auth2FARoutes);
```

### 9. Update .env.example

Add the following entry (the issuer name for TOTP) to `.env.example`:

```
# 2FA Configuration
TOTP_ISSUER=FastDepo
```

### 10. Add TOTP issuer to env.ts schema

In `src/config/env.ts`, add:

```typescript
TOTP_ISSUER: z.string().default('FastDepo'),
```

Update the `buildOtpauthUrl` function in `auth.2fa.service.ts` to use this:

```typescript
export function buildOtpauthUrl(secret: string, email: string): string {
  return authenticator.keyuri(email, env.TOTP_ISSUER, secret);
}
```

## Verification

1. Run the Prisma migration:
   ```bash
   npx prisma migrate dev --name add-two-factor-auth
   npx prisma generate
   ```
2. Compile the project: `npx tsc --noEmit` (no type errors)
3. Start the server: `npm run dev`
4. Test the 2FA setup flow:
   ```bash
   # Step 1: Initiate setup (requires auth)
   curl -X POST -H "Authorization: Bearer <token>" http://localhost:3000/api/auth/2fa/setup

   # Step 2: Scan the QR code with an authenticator app (Google Authenticator, Authy, etc.)

   # Step 3: Verify setup with TOTP code
   curl -X POST -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     -d '{"code":"123456"}' \
     http://localhost:3000/api/auth/2fa/verify-setup

   # Step 4: Test login - should return requires2FA: true
   # Login endpoint returns { requires2FA: true, tempToken: "..." }

   # Step 5: Verify 2FA login
   curl -X POST -H "Content-Type: application/json" \
     -d '{"tempToken":"<temp_token>","code":"123456"}' \
     http://localhost:3000/api/auth/2fa/verify

   # Step 6: Disable 2FA
   curl -X POST -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     -d '{"password":"user_password","code":"123456"}' \
     http://localhost:3000/api/auth/2fa/disable
   ```
5. Verify backup codes work:
   - After setup, use one of the 10 backup codes in the 2FA verify endpoint
   - Verify the used backup code cannot be reused
6. Verify that the QR code data URL is a valid base64-encoded PNG image.

Expected output:
- `/2fa/setup` returns secret, otpauth URL, QR code data URL, and 10 backup codes
- `/2fa/verify-setup` confirms enablement with a valid TOTP code
- Login returns `{ requires2FA: true, tempToken: "..." }` when 2FA is enabled
- `/2fa/verify` with valid temp token + code returns a full JWT
- `/2fa/disable` requires both password and a valid code
- Used backup codes cannot be reused

## Rollback

1. Remove installed packages:
   ```bash
   npm uninstall otplib qrcode @types/qrcode
   ```
2. Revert the Prisma migration:
   ```bash
   npx prisma migrate rollback
   ```
3. Remove `twoFactorEnabled`, `twoFactorSecret`, `twoFactorBackupCodes` fields from the `User` model in `prisma/schema.prisma`
4. Delete created files:
   ```bash
   rm src/modules/auth/auth.2fa.routes.ts src/modules/auth/auth.2fa.controller.ts src/modules/auth/auth.2fa.service.ts src/modules/auth/auth.2fa.types.ts
   ```
5. Remove 2FA route registration from `src/index.ts`
6. Revert the login flow modification to remove the 2FA check
7. Remove `TOTP_ISSUER` from `src/config/env.ts` and `.env.example`
8. Run `npx prisma generate` to update the client

## ADR-014: TOTP-Based Two-Factor Authentication

**Decision**: Implement two-factor authentication using Time-based One-Time Password (TOTP) via the `otplib` library, with backup codes stored as bcrypt hashes for account recovery.

**Reason**: TOTP is the most widely-adopted 2FA standard, supported by all major authenticator apps (Google Authenticator, Authy, 1Password). It works offline, doesn't require SMS delivery infrastructure, and has no per-user cost. Using `otplib` provides a well-tested implementation of RFC 6238. QR code generation on the server side makes setup simple for users. Backup codes ensure users can recover access if they lose their authenticator device.

**Consequences**:
- Users must have an authenticator app installed on their device
- The login flow is modified: when 2FA is enabled, login returns a `requires2FA: true` flag with a short-lived temp token; the client must then call `/2fa/verify` with the TOTP code
- Backup codes are stored as bcrypt hashes in a JSON column; this is simple but means backup code verification requires iterating through all hashes (O(n) for 10 codes, which is acceptable)
- The `twoFactorSecret` is stored in plaintext in the database; this is standard for TOTP (the secret must be readable for verification), but the database should be encrypted at rest
- The temp token for 2FA verification has a 5-minute expiry; this is shorter than the regular JWT to limit the window for temp token theft
- QR code generation happens server-side and returns a base64 data URL; an alternative would be to construct the otpauth URI and let the client generate the QR code

**Alternatives Considered**:
- **SMS-based 2FA**: Requires a third-party SMS provider (cost per message), delivery delays, vulnerable to SIM-swap attacks, and NIST discourages SMS for 2FA
- **WebAuthn/FIDO2 (hardware keys)**: Most secure option but significantly more complex to implement; consider for a future phase
- **Email-based OTP**: Similar to SMS issues with delivery reliability; also interferes with the email channel used for notifications
- **Push notification 2FA**: Requires a dedicated mobile app; not feasible for a web application at this stage
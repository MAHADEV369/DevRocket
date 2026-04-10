# Skill 03: Authentication Module

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 3-4 hours
Depends On: 01, 02

## Input Contract
- Skill 01 scaffold complete
- Skill 02 database schema and Prisma client available
- `src/config/database.ts` exports `prisma`
- `src/config/index.ts` has `jwt` config block
- `bcryptjs` and `jsonwebtoken` available

## Output Contract
- `src/modules/auth/` with routes, controller, service, validation, types
- `src/middleware/auth.ts` with `authenticate`, `requireAdmin`, `requirePlan` middleware
- `src/services/email.service.ts` for transactional emails
- `src/utils/crypto.ts` for JWT signing and token utilities
- Auth endpoints: POST `/api/auth/register`, POST `/api/auth/login`, POST `/api/auth/refresh`, POST `/api/auth/logout`, POST `/api/auth/verify-email`, POST `/api/auth/forgot-password`, POST `/api/auth/reset-password`, GET `/api/auth/me`

## Files to Create

| File | Description |
|------|-------------|
| `src/modules/auth/types.ts` | Auth-related TypeScript types |
| `src/modules/auth/validation.ts` | Zod schemas for request validation |
| `src/modules/auth/service.ts` | Auth business logic |
| `src/modules/auth/controller.ts` | Express route handlers |
| `src/modules/auth/routes.ts` | Express router with all auth endpoints |
| `src/middleware/auth.ts` | authenticate, requireAdmin, requirePlan middleware |
| `src/services/email.service.ts` | Transactional email service |
| `src/utils/crypto.ts` | JWT signing, verification, hash utilities |

## Steps

### 1. Install dependencies

```bash
npm install jsonwebtoken zod && npm install -D @types/jsonwebtoken @types/validator
npm install validator
```

### 2. Create `src/modules/auth/types.ts`

```typescript
import type { Request } from 'express';

export interface RegisterInput {
  email: string;
  username: string;
  name?: string;
  password: string;
}

export interface LoginInput {
  email: string;
  password: string;
}

export interface TokenPayload {
  userId: string;
  role: string;
  plan: string;
}

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
}

export interface RefreshInput {
  refreshToken: string;
}

export interface ForgotPasswordInput {
  email: string;
}

export interface ResetPasswordInput {
  token: string;
  password: string;
}

export interface VerifyEmailInput {
  token: string;
}

export interface AuthenticatedRequest extends Request {
  user: TokenPayload;
}
```

### 3. Create `src/utils/crypto.ts`

```typescript
import jwt from 'jsonwebtoken';
import bcrypt from 'bcryptjs';
import crypto from 'crypto';
import { config } from '../config/index.js';
import type { TokenPayload, AuthTokens } from '../modules/auth/types.js';

const SALT_ROUNDS = 12;

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}

export function signAccessToken(payload: TokenPayload): string {
  return jwt.sign(payload, config.jwt.secret, {
    expiresIn: config.jwt.expiresIn,
  });
}

export function signRefreshToken(payload: TokenPayload): string {
  return jwt.sign(payload, config.jwt.refreshSecret, {
    expiresIn: config.jwt.refreshExpiresIn,
  });
}

export function generateTokenPair(payload: TokenPayload): AuthTokens {
  return {
    accessToken: signAccessToken(payload),
    refreshToken: signRefreshToken(payload),
  };
}

export function verifyAccessToken(token: string): TokenPayload {
  return jwt.verify(token, config.jwt.secret) as TokenPayload;
}

export function verifyRefreshToken(token: string): TokenPayload {
  return jwt.verify(token, config.jwt.refreshSecret) as TokenPayload;
}

export function generateRandomToken(): string {
  return crypto.randomBytes(32).toString('hex');
}
```

### 4. Create `src/services/email.service.ts`

```typescript
import nodemailer from 'nodemailer';
import { config } from '../config/index.js';
import { logger } from '../utils/logger.js';

let transporter: nodemailer.Transporter | null = null;

function getTransporter(): nodemailer.Transporter {
  if (!transporter) {
    transporter = nodemailer.createTransport({
      host: config.email.host,
      port: config.email.port,
      auth: {
        user: config.email.user,
        pass: config.email.pass,
      },
    });
  }
  return transporter;
}

async function sendMail(to: string, subject: string, html: string): Promise<void> {
  if (config.nodeEnv === 'test') {
    logger.info({ to, subject }, 'Email (mock)');
    return;
  }

  try {
    const info = await getTransporter().sendMail({
      from: config.email.from,
      to,
      subject,
      html,
    });
    logger.info({ messageId: info.messageId }, 'Email sent');
  } catch (error) {
    logger.error({ error }, 'Failed to send email');
    throw error;
  }
}

export async function sendVerificationEmail(email: string, token: string): Promise<void> {
  const url = `${config.corsOrigin}/verify-email?token=${token}`;
  await sendMail(
    email,
    'Verify your email address',
    `<h1>Welcome to FastDepo!</h1><p>Click <a href="${url}">here</a> to verify your email.</p>`
  );
}

export async function sendPasswordResetEmail(email: string, token: string): Promise<void> {
  const url = `${config.corsOrigin}/reset-password?token=${token}`;
  await sendMail(
    email,
    'Reset your password',
    `<h1>Password Reset</h1><p>Click <a href="${url}">here</a> to reset your password. This link expires in 1 hour.</p>`
  );
}

export async function sendWelcomeEmail(email: string, username: string): Promise<void> {
  await sendMail(
    email,
    'Welcome to FastDepo!',
    `<h1>Welcome, ${username}!</h1><p>Your account is ready. Start creating amazing AI images!</p>`
  );
}
```

### 5. Install nodemailer

```bash
npm install nodemailer && npm install -D @types/nodemailer
```

### 6. Create `src/modules/auth/validation.ts`

```typescript
import { z } from 'zod';

export const registerSchema = z.object({
  email: z.string().email('Invalid email address'),
  username: z.string().min(3, 'Username must be at least 3 characters').max(30, 'Username must be at most 30 characters').regex(/^[a-zA-Z0-9_]+$/, 'Username can only contain letters, numbers, and underscores'),
  name: z.string().max(100).optional(),
  password: z.string().min(8, 'Password must be at least 8 characters').regex(/[A-Z]/, 'Password must contain at least one uppercase letter').regex(/[a-z]/, 'Password must contain at least one lowercase letter').regex(/[0-9]/, 'Password must contain at least one number'),
});

export const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(1, 'Password is required'),
});

export const refreshSchema = z.object({
  refreshToken: z.string().min(1, 'Refresh token is required'),
});

export const forgotPasswordSchema = z.object({
  email: z.string().email('Invalid email address'),
});

export const resetPasswordSchema = z.object({
  token: z.string().min(1, 'Token is required'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

export const verifyEmailSchema = z.object({
  token: z.string().min(1, 'Token is required'),
});

export type RegisterInput = z.infer<typeof registerSchema>;
export type LoginInput = z.infer<typeof loginSchema>;
export type RefreshInput = z.infer<typeof refreshSchema>;
export type ForgotPasswordInput = z.infer<typeof forgotPasswordSchema>;
export type ResetPasswordInput = z.infer<typeof resetPasswordSchema>;
export type VerifyEmailInput = z.infer<typeof verifyEmailSchema>;
```

### 7. Create `src/modules/auth/service.ts`

```typescript
import { prisma } from '../../config/database.js';
import { hashPassword, verifyPassword, generateTokenPair, verifyRefreshToken, generateRandomToken } from '../../utils/crypto.js';
import { sendVerificationEmail, sendPasswordResetEmail, sendWelcomeEmail } from '../../services/email.service.js';
import type { TokenPayload } from './types.js';
import { AppError } from '../../utils/errors.js';

export async function register(data: { email: string; username: string; name?: string; password: string }) {
  const existing = await prisma.user.findFirst({
    where: { OR: [{ email: data.email }, { username: data.username }] },
  });
  if (existing) {
    throw new AppError(409, 'Email or username already taken');
  }

  const passwordHash = await hashPassword(data.password);
  const user = await prisma.user.create({
    data: {
      email: data.email,
      username: data.username,
      name: data.name,
      passwordHash,
    },
  });

  const verificationToken = generateRandomToken();
  await prisma.passwordReset.create({
    data: {
      userId: user.id,
      token: verificationToken,
      expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000),
    },
  });

  await sendVerificationEmail(user.email, verificationToken).catch(() => {});

  const payload: TokenPayload = { userId: user.id, role: user.role, plan: user.plan };
  const tokens = generateTokenPair(payload);

  await prisma.refreshToken.create({
    data: {
      userId: user.id,
      token: tokens.refreshToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    },
  });

  const { passwordHash: _, ...userWithoutHash } = user;
  return { user: userWithoutHash, ...tokens };
}

export async function login(email: string, password: string) {
  const user = await prisma.user.findUnique({ where: { email } });
  if (!user) {
    throw new AppError(401, 'Invalid credentials');
  }

  const valid = await verifyPassword(password, user.passwordHash);
  if (!valid) {
    throw new AppError(401, 'Invalid credentials');
  }

  const payload: TokenPayload = { userId: user.id, role: user.role, plan: user.plan };
  const tokens = generateTokenPair(payload);

  await prisma.refreshToken.create({
    data: {
      userId: user.id,
      token: tokens.refreshToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    },
  });

  const { passwordHash: _, ...userWithoutHash } = user;
  return { user: userWithoutHash, ...tokens };
}

export async function refresh(oldRefreshToken: string) {
  let payload: TokenPayload;
  try {
    payload = verifyRefreshToken(oldRefreshToken);
  } catch {
    throw new AppError(401, 'Invalid refresh token');
  }

  const stored = await prisma.refreshToken.findUnique({ where: { token: oldRefreshToken } });
  if (!stored || stored.isRevoked || stored.expiresAt < new Date()) {
    throw new AppError(401, 'Invalid or expired refresh token');
  }

  await prisma.refreshToken.update({ where: { id: stored.id }, data: { isRevoked: true } });

  const newPayload: TokenPayload = { userId: payload.userId, role: payload.role, plan: payload.plan };
  const tokens = generateTokenPair(newPayload);

  await prisma.refreshToken.create({
    data: {
      userId: payload.userId,
      token: tokens.refreshToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    },
  });

  return tokens;
}

export async function logout(userId: string, refreshToken?: string) {
  if (refreshToken) {
    await prisma.refreshToken.updateMany({
      where: { token: refreshToken, userId },
      data: { isRevoked: true },
    });
  } else {
    await prisma.refreshToken.updateMany({
      where: { userId },
      data: { isRevoked: true },
    });
  }
}

export async function verifyEmail(token: string) {
  const record = await prisma.passwordReset.findUnique({ where: { token } });
  if (!record || record.expiresAt < new Date() || record.usedAt) {
    throw new AppError(400, 'Invalid or expired verification token');
  }

  await prisma.user.update({ where: { id: record.userId }, data: { emailVerified: true } });
  await prisma.passwordReset.update({ where: { id: record.id }, data: { usedAt: new Date() } });
  await sendWelcomeEmail(record.userId, '').catch(() => {});
}

export async function forgotPassword(email: string) {
  const user = await prisma.user.findUnique({ where: { email } });
  if (!user) return;

  const token = generateRandomToken();
  await prisma.passwordReset.create({
    data: {
      userId: user.id,
      token,
      expiresAt: new Date(Date.now() + 60 * 60 * 1000),
    },
  });

  await sendPasswordResetEmail(email, token).catch(() => {});
}

export async function resetPassword(token: string, password: string) {
  const record = await prisma.passwordReset.findUnique({ where: { token } });
  if (!record || record.expiresAt < new Date() || record.usedAt) {
    throw new AppError(400, 'Invalid or expired reset token');
  }

  const passwordHash = await hashPassword(password);
  await prisma.user.update({ where: { id: record.userId }, data: { passwordHash } });
  await prisma.passwordReset.update({ where: { id: record.id }, data: { usedAt: new Date() } });
  await prisma.refreshToken.updateMany({ where: { userId: record.userId }, data: { isRevoked: true } });
}

export async function getMe(userId: string) {
  const user = await prisma.user.findUnique({ where: { id: userId } });
  if (!user) throw new AppError(404, 'User not found');
  const { passwordHash: _, ...userWithoutHash } = user;
  return userWithoutHash;
}
```

### 8. Create `src/utils/errors.ts`

```typescript
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly isOperational: boolean;

  constructor(statusCode: number, message: string, isOperational = true) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(404, `${resource} not found`);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(403, message);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(401, message);
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(400, message);
  }
}
```

### 9. Create `src/modules/auth/controller.ts`

```typescript
import type { Request, Response, NextFunction } from 'express';
import * as authService from './service.js';
import type { AuthenticatedRequest } from './types.js';

export async function register(req: Request, res: Response, next: NextFunction) {
  try {
    const result = await authService.register(req.body);
    res.status(201).json(result);
  } catch (error) {
    next(error);
  }
}

export async function login(req: Request, res: Response, next: NextFunction) {
  try {
    const result = await authService.login(req.body.email, req.body.password);
    res.json(result);
  } catch (error) {
    next(error);
  }
}

export async function refresh(req: Request, res: Response, next: NextFunction) {
  try {
    const tokens = await authService.refresh(req.body.refreshToken);
    res.json(tokens);
  } catch (error) {
    next(error);
  }
}

export async function logout(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    await authService.logout(req.user.userId, req.body.refreshToken);
    res.json({ message: 'Logged out' });
  } catch (error) {
    next(error);
  }
}

export async function verifyEmail(req: Request, res: Response, next: NextFunction) {
  try {
    await authService.verifyEmail(req.body.token);
    res.json({ message: 'Email verified successfully' });
  } catch (error) {
    next(error);
  }
}

export async function forgotPassword(req: Request, res: Response, next: NextFunction) {
  try {
    await authService.forgotPassword(req.body.email);
    res.json({ message: 'If an account with that email exists, a reset link has been sent' });
  } catch (error) {
    next(error);
  }
}

export async function resetPassword(req: Request, res: Response, next: NextFunction) {
  try {
    await authService.resetPassword(req.body.token, req.body.password);
    res.json({ message: 'Password reset successfully' });
  } catch (error) {
    next(error);
  }
}

export async function getMe(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const user = await authService.getMe(req.user.userId);
    res.json(user);
  } catch (error) {
    next(error);
  }
}
```

### 10. Create `src/modules/auth/routes.ts`

```typescript
import { Router } from 'express';
import * as controller from './controller.js';
import { validate } from '../../middleware/validate.js';
import { authenticate } from '../../middleware/auth.js';
import {
  registerSchema,
  loginSchema,
  refreshSchema,
  forgotPasswordSchema,
  resetPasswordSchema,
  verifyEmailSchema,
} from './validation.js';

const router = Router();

router.post('/register', validate(registerSchema), controller.register);
router.post('/login', validate(loginSchema), controller.login);
router.post('/refresh', validate(refreshSchema), controller.refresh);
router.post('/logout', authenticate, controller.logout);
router.post('/verify-email', validate(verifyEmailSchema), controller.verifyEmail);
router.post('/forgot-password', validate(forgotPasswordSchema), controller.forgotPassword);
router.post('/reset-password', validate(resetPasswordSchema), controller.resetPassword);
router.get('/me', authenticate, controller.getMe);

export default router;
```

### 11. Create `src/middleware/auth.ts`

```typescript
import type { NextFunction, Request, Response } from 'express';
import { verifyAccessToken } from '../utils/crypto.js';
import { AppError } from '../utils/errors.js';
import type { AuthenticatedRequest, TokenPayload } from '../modules/auth/types.js';

export function authenticate(req: Request, _res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) {
    return next(new AppError(401, 'Missing or invalid authorization header'));
  }

  const token = header.slice(7);
  try {
    const payload: TokenPayload = verifyAccessToken(token);
    (req as AuthenticatedRequest).user = payload;
    next();
  } catch {
    next(new AppError(401, 'Invalid or expired token'));
  }
}

export function requireAdmin(req: Request, _res: Response, next: NextFunction) {
  const user = (req as AuthenticatedRequest).user;
  if (user.role !== 'ADMIN') {
    return next(new AppError(403, 'Admin access required'));
  }
  next();
}

export function requirePlan(...plans: string[]) {
  return (req: Request, _res: Response, next: NextFunction) => {
    const user = (req as AuthenticatedRequest).user;
    if (!plans.includes(user.plan)) {
      return next(new AppError(403, `This feature requires ${plans.join(' or ')} plan`));
    }
    next();
  };
}
```

### 12. Register auth routes in `src/app.ts`

Add to `src/app.ts` after the health check route:

```typescript
import authRoutes from './modules/auth/routes.js';
// ...
app.use('/api/auth', authRoutes);
```

### 13. Create placeholder `validate` middleware

Create `src/middleware/validate.ts` (full implementation in Skill 04):

```typescript
import type { NextFunction, Request, Response } from 'express';
import type { ZodSchema } from 'zod';
import { AppError } from '../utils/errors.js';

export function validate(schema: ZodSchema) {
  return (req: Request, _res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      const errors = result.error.errors.map((e) => `${e.path.join('.')}: ${e.message}`);
      return next(new AppError(400, errors.join(', ')));
    }
    req.body = result.data;
    next();
  };
}
```

## Verification

```bash
npm run typecheck
# Expected: no errors

npm run dev &
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","username":"testuser","password":"Password123!"}'
# Expected: 201 with accessToken, refreshToken, user object

curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"Password123!"}'
# Expected: 200 with tokens

curl -X GET http://localhost:3000/api/auth/me \
  -H "Authorization: Bearer <access_token>"
# Expected: 200 with user object
```

## Rollback

```bash
rm -rf src/modules/auth src/middleware/auth.ts src/services/email.service.ts src/utils/crypto.ts src/utils/errors.ts src/middleware/validate.ts
# Revert app.ts to remove auth route registration
npx prisma migrate reset --force
```

## ADR-003: JWT with Refresh Token Rotation

Decision: Use JWT access tokens (15 min) with refresh token rotation (7 days). Store refresh tokens in the database and revoke on rotation.
Reason: Short-lived access tokens limit exposure if compromised. Refresh token rotation prevents replay attacks since old tokens are revoked on each refresh. Database storage enables forced logout and audit.
Consequences: Requires database round-trip on refresh. Token revocation is not instant for access tokens (they expire naturally). More complex than session-based auth.
Alternatives Considered: Session cookies only (harder for mobile apps), single long-lived JWT (no revocation possible), opaque tokens (more DB queries per request).
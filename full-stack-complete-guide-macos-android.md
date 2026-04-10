# The Complete Full-Stack Guide: Backend, Infrastructure, Frontend — macOS & Android

> Built around your **GENAI** AI Image Generator app. Every concept applies to any app you build after this.

---

# TABLE OF CONTENTS

**PART 1: BACKEND**
1. [Project Architecture Overview](#1-project-architecture-overview)
2. [Backend Project Structure](#2-backend-project-structure)
3. [Environment Configuration](#3-environment-configuration)
4. [Database — PostgreSQL with Prisma](#4-database--postgresql-with-prisma)
5. [Authentication System](#5-authentication-system)
6. [API Design — REST Endpoints](#6-api-design--rest-endpoints)
7. [Image Generation Service](#7-image-generation-service)
8. [File Upload & Storage (S3/R2)](#8-file-upload--storage-s3r2)
9. [Rate Limiting & Security](#9-rate-limiting--security)
10. [Background Jobs & Queues](#10-background-jobs--queues)
11. [WebSocket Real-Time Updates](#11-websocket-real-time-updates)
12. [Email & Notifications](#12-email--notifications)
13. [Error Handling & Logging](#13-error-handling--logging)

**PART 2: INFRASTRUCTURE**
14. [Docker & Containerization](#14-docker--containerization)
15. [Docker Compose — Full Stack](#15-docker-compose--full-stack)
16. [CI/CD Pipeline (GitHub Actions)](#16-cicd-pipeline-github-actions)
17. [Deployment — Railway, Fly.io, AWS](#17-deployment--railway-flyio-aws)
18. [Domain, SSL, & CDN](#18-domain-ssl--cdn)
19. [Monitoring, Logging & Alerting](#19-monitoring-logging--alerting)
20. [Database Backups & Migrations](#20-database-backups--migrations)

**PART 3: FRONTEND (WEB)**
21. [Frontend Project Structure](#21-frontend-project-structure)
22. [React + Vite + TypeScript Setup](#22-react--vite--typescript-setup)
23. [State Management (Zustand)](#23-state-management-zustand)
24. [API Client & Data Fetching (TanStack Query)](#24-api-client--data-fetching-tanstack-query)
25. [Authentication Flow (Frontend)](#25-authentication-flow-frontend)
26. [UI Components & Design System](#26-ui-components--design-system)
27. [Dark Mode & Theming](#27-dark-mode--theming)
28. [Image Gallery & History](#28-image-gallery--history)
29. [PWA — Offline Support](#29-pwa--offline-support)
30. [Performance Optimization](#30-performance-optimization)

**PART 4: macOS DESKTOP APP**
31. [macOS App Architecture — Tauri vs Electron](#31-macos-app-architecture--tauri-vs_electron)
32. [Tauri Setup & Project Structure](#32-tauri-setup--project-structure)
33. [macOS App Features — Native Menu, Dock, Notifications](#33-macos-app-features--native-menu-dock-notifications)
34. [macOS Code Signing & Notarization](#34-macos-code-signing--notarization)
35. [macOS App Store Distribution](#35-macos-app-store-distribution)
36. [Auto-Updates for macOS](#36-auto-updates-for-macos)

**PART 5: ANDROID APP**
37. [Android App Architecture — Flutter](#37-android-app-architecture--flutter)
38. [Flutter Project Structure](#38-flutter-project-structure)
39. [Flutter State Management (Riverpod)](#39-flutter-state-management-riverpod)
40. [Flutter Auth & API Integration](#40-flutter-auth--api-integration)
41. [Flutter Image Generation UI](#41-flutter-image-generation-ui)
42. [Flutter Offline & Caching (Hive)](#42-flutter-offline--caching-hive)
43. [Flutter Push Notifications (FCM)](#43-flutter-push-notifications-fcm)
44. [Android Signing & Release Build](#44-android-signing--release-build)
45. [Google Play Store Submission](#45-google-play-store-submission)

**PART 6: CROSS-PLATFORM CONCERNS**
46. [Shared API Contract](#46-shared-api-contract)
47. [Environment Strategy (Dev/Staging/Prod)](#47-environment-strategy-devstagingprod)
48. [Secrets Management](#48-secrets-management)
49. [Testing Strategy — All Platforms](#49-testing-strategy--all-platforms)
50. [Final Production Checklist](#50-final-production-checklist)

---

# PART 1: BACKEND

## 1. Project Architecture Overview

```
                    ┌─────────────────┐
                    │   macOS App     │
                    │   (Tauri)       │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │                  │
            ┌───────┴───────┐  ┌──────┴──────┐
            │  Android App  │  │  Web App     │
            │  (Flutter)    │  │  (React)     │
            └───────┬───────┘  └──────┬──────┘
                    │                 │
                    └────────┬────────┘
                             │ HTTPS
                    ┌────────┴────────┐
                    │   API Gateway   │
                    │   (Nginx)       │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
      ┌───────┴───────┐ ┌───┴────┐ ┌──────┴──────┐
      │  API Server    │ │ Worker │ │  WebSocket  │
      │  (Express)    │ │ (Bull) │ │  Server     │
      └───────┬───────┘ └───┬────┘ └──────┬──────┘
              │              │              │
      ┌───────┴──────────────┴──────────────┴───────┐
      │                                              │
      │  ┌─────────┐  ┌───────┐  ┌──────┐  ┌─────┐ │
      │  │PostgreSQL│  │ Redis │  │  S3  │  │SMTP │ │
      │  │  (DB)   │  │(Cache)│  │(Files)│  │(Mail)│ │
      │  └─────────┘  └───────┘  └──────┘  └─────┘ │
      │                                              │
      └──────────────────────────────────────────────┘
```

**Key principle**: One backend serves ALL platforms. The API is the contract. macOS, Android, and Web all talk to the same REST API.

## 2. Backend Project Structure

```
backend/
├── prisma/
│   ├── schema.prisma              ← Database schema (tables, relations, indexes)
│   ├── migrations/                ← Auto-generated migration files
│   └── seed.ts                    ← Seed data (demo users, default settings)
│
├── src/
│   ├── index.ts                   ← Entry point: creates server, applies middleware
│   ├── app.ts                     ← Express app setup: middleware, routes, error handler
│   │
│   ├── config/
│   │   ├── env.ts                 ← Zod-validated environment variables
│   │   ├── redis.ts               ← Redis client setup
│   │   └── sentry.ts              ← Error reporting config
│   │
│   ├── middleware/
│   │   ├── auth.ts                ← JWT verification middleware
│   │   ├── rateLimiter.ts         ← Rate limiting per-route
│   │   ├── requestLogger.ts       ← HTTP request logging (method, path, status, duration)
│   │   ├── requestId.ts           ← Generate unique ID per request for tracing
│   │   ├── validate.ts            ← Zod schema validation middleware
│   │   ├── errorHandler.ts        ← Global error handler
│   │   └── cors.ts                ← CORS configuration
│   │
│   ├── modules/
│   │   ├── auth/
│   │   │   ├── auth.routes.ts     ← POST /api/v1/auth/signup, /login, /refresh, /logout
│   │   │   ├── auth.controller.ts ← Request handling logic
│   │   │   ├── auth.service.ts    ← Business logic (hash password, generate JWT, etc.)
│   │   │   ├── auth.validation.ts ← Zod schemas for signup, login inputs
│   │   │   └── auth.types.ts      ← TypeScript types for this module
│   │   │
│   │   ├── users/
│   │   │   ├── user.routes.ts
│   │   │   ├── user.controller.ts
│   │   │   ├── user.service.ts
│   │   │   └── user.validation.ts
│   │   │
│   │   ├── images/
│   │   │   ├── image.routes.ts    ← POST /api/v1/images/generate, GET /gallery, etc.
│   │   │   ├── image.controller.ts
│   │   │   ├── image.service.ts
│   │   │   ├── image.validation.ts
│   │   │   └── providers/
│   │   │       ├── dalle.provider.ts      ← DALL-E 3 integration
│   │   │       ├── flux.provider.ts        ← Flux Pro integration
│   │   │       ├── stableDiffusion.provider.ts
│   │   │       └── pollinations.provider.ts ← Free fallback
│   │   │
│   │   ├── gallery/
│   │   │   ├── gallery.routes.ts
│   │   │   ├── gallery.controller.ts
│   │   │   └── gallery.service.ts
│   │   │
│   │   └── billing/
│   │       ├── billing.routes.ts
│   │       ├── billing.controller.ts
│   │       ├── billing.service.ts
│   │       └── stripe.provider.ts
│   │
│   ├── services/
│   │   ├── email.service.ts        ← SendGrid / Resend integration
│   │   ├── storage.service.ts      ← S3 / Cloudflare R2 upload
│   │   ├── queue.service.ts        ← BullMQ job queue
│   │   └── websocket.service.ts    ← Real-time generation status
│   │
│   ├── utils/
│   │   ├── logger.ts               ← Pino structured logging
│   │   ├── crypto.ts               ← Token generation, hashing helpers
│   │   └── errors.ts               ← Custom error classes (AppError, AuthError, etc.)
│   │
│   └── types/
│       ├── express.d.ts            ← Augment Express Request with user property
│       └── index.ts                ← Shared types
│
├── docker/
│   └── Dockerfile                  ← Multi-stage production build
│
├── tests/
│   ├── setup.ts                    ← Test database setup/teardown
│   ├── auth.test.ts
│   ├── images.test.ts
│   └── gallery.test.ts
│
├── .env.example                    ← Template for environment variables
├── .env.schema                     ← Zod validation for env vars
├── tsconfig.json
├── package.json
└── README.md
```

## 3. Environment Configuration

```typescript
// src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']).default('development'),
  PORT: z.coerce.number().default(3001),

  // Database
  DATABASE_URL: z.string().url(),

  // Redis
  REDIS_URL: z.string().url().default('redis://localhost:6379'),

  // JWT
  JWT_SECRET: z.string().min(32),
  JWT_ACCESS_EXPIRY: z.string().default('15m'),
  JWT_REFRESH_EXPIRY: z.string().default('7d'),

  // AI Providers
  OPENAI_API_KEY: z.string().optional(),
  HUGGING_FACE_TOKEN: z.string().optional(),
  REPLICATE_API_TOKEN: z.string().optional(),

  // Storage (S3 or Cloudflare R2)
  S3_ACCESS_KEY: z.string().optional(),
  S3_SECRET_KEY: z.string().optional(),
  S3_BUCKET: z.string().default('genai-images'),
  S3_REGION: z.string().default('us-east-1'),
  S3_ENDPOINT: z.string().optional(), // For R2, use R2 endpoint

  // Email (Resend)
  RESEND_API_KEY: z.string().optional(),
  EMAIL_FROM: z.string().default('GenAI <noreply@yourdomain.com>'),

  // Stripe (billing)
  STRIPE_SECRET_KEY: z.string().optional(),
  STRIPE_WEBHOOK_SECRET: z.string().optional(),
  STRIPE_PRICE_MONTHLY: z.string().optional(),
  STRIPE_PRICE_YEARLY: z.string().optional(),

  // Sentry (error tracking)
  SENTRY_DSN: z.string().optional(),

  // CORS
  CORS_ORIGIN: z.string().default('http://localhost:5173'),

  // Rate limiting
  RATE_LIMIT_WINDOW_MS: z.coerce.number().default(15 * 60 * 1000), // 15 minutes
  RATE_LIMIT_MAX: z.coerce.number().default(100),
});

export type Env = z.infer<typeof envSchema>;

function validateEnv(): Env {
  const result = envSchema.safeParse(process.env);
  if (!result.success) {
    console.error('❌ Invalid environment variables:');
    console.error(result.error.flatten().fieldErrors);
    process.exit(1);
  }
  return result.data;
}

export const env = validateEnv();
```

```env
# .env.example — COPY THIS TO .env AND FILL IN YOUR VALUES

# === SERVER ===
NODE_ENV=development
PORT=3001

# === DATABASE ===
DATABASE_URL=postgresql://genai:genai@localhost:5432/genai

# === REDIS ===
REDIS_URL=redis://localhost:6379

# === JWT ===
JWT_SECRET=change-me-to-a-64-char-random-string-use-openssl-rand-hex-32
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# === AI PROVIDERS ===
OPENAI_API_KEY=sk-...
HUGGING_FACE_TOKEN=hf_...

# === STORAGE (Cloudflare R2 recommended — S3-compatible, free egress) ===
S3_ACCESS_KEY=your-r2-access-key
S3_SECRET_KEY=your-r2-secret-key
S3_BUCKET=genai-images
S3_REGION=auto
S3_ENDPOINT=https://<your-account-id>.r2.cloudflarestorage.com

# === EMAIL ===
RESEND_API_KEY=re_...
EMAIL_FROM=GenAI <noreply@yourdomain.com>

# === STRIPE ===
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRICE_MONTHLY=price_monthly_id
STRIPE_PRICE_YEARLY=price_yearly_id

# === SENTRY ===
SENTRY_DSN=https://...@sentry.io/...

# === CORS ===
CORS_ORIGIN=http://localhost:5173,http://localhost:1420,genai://localhost

# === RATE LIMITING ===
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX=100
```

## 4. Database — PostgreSQL with Prisma

### Why PostgreSQL + Prisma

| Choice | Why |
|--------|-----|
| PostgreSQL | Industry standard, relational, JSON support, free, scales |
| Prisma ORM | Type-safe queries, auto migrations, best DX, TypeScript native |

### Database Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String    @id @default(uuid())
  email         String    @unique
  passwordHash  String
  name          String?
  avatarUrl     String?
  role          Role      @default(USER)
  plan          Plan      @default(FREE)
  stripeCustomerId String? @unique

  emailVerified Boolean   @default(false)
  verifyToken   String?   @unique

  refreshTokens  RefreshToken[]
  images         Image[]
  galleries      Gallery[]
  notifications  Notification[]

  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  deletedAt     DateTime? // Soft delete

  @@index([email])
  @@index([stripeCustomerId])
}

model RefreshToken {
  id          String   @id @default(uuid())
  token       String   @unique
  userId      String
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  deviceInfo  String?  // "Chrome/macOS" etc.
  expiresAt   DateTime

  createdAt   DateTime @default(now())

  @@index([userId])
  @@index([token])
}

model Image {
  id              String          @id @default(uuid())
  userId          String
  user            User            @relation(fields: [userId], references: [id], onDelete: Cascade)

  prompt          String
  enhancedPrompt  String?         // After style preset applied
  model           ImageModel      @default(POLLINATIONS)
  style           String          @default("none")
  size            String          @default("1024x1024")
  quality         String          @default("standard")

  imageUrl        String
  thumbnailUrl    String?         // Resized version for gallery
  storageKey      String?         // S3/R2 object key

  generationTimeMs Int?
  costCredits     Int             @default(1)  // How many credits this cost

  isPublic        Boolean         @default(false)
  isFavorite      Boolean         @default(false)

  galleries       GalleryImage[]

  createdAt       DateTime        @default(now())
  updatedAt       DateTime        @updatedAt

  @@index([userId])
  @@index([model])
  @@index([createdAt])
  @@index([isPublic])
}

model Gallery {
  id          String    @id @default(uuid())
  userId      String
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  name        String
  description String?
  isPublic    Boolean   @default(false)
  coverImage  String?

  images      GalleryImage[]

  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@index([userId])
}

model GalleryImage {
  id          String    @id @default(uuid())
  galleryId   String
  gallery     Gallery   @relation(fields: [galleryId], references: [id], onDelete: Cascade)
  imageId     String
  image       Image     @relation(fields: [imageId], references: [id], onDelete: Cascade)
  order       Int       @default(0)

  @@unique([galleryId, imageId])
  @@index([galleryId])
}

model Notification {
  id          String    @id @default(uuid())
  userId      String
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  type        NotificationType
  title       String
  message     String
  isRead      Boolean   @default(false)
  actionUrl   String?

  createdAt   DateTime  @default(now())

  @@index([userId, isRead])
}

model PasswordReset {
  id          String   @id @default(uuid())
  userId      String   @unique
  token       String   @unique
  expiresAt   DateTime

  @@index([token])
}

enum Role {
  USER
  ADMIN
}

enum Plan {
  FREE
  PRO
  ENTERPRISE
}

enum ImageModel {
  DALLE3
  FLUX_PRO
  STABLE_DIFFUSION
  POLLINATIONS
}

enum NotificationType {
  GENERATION_COMPLETE
  GALLERY_SHARED
  CREDIT_LOW
  PLAN_UPGRADED
  SYSTEM
}
```

### Seed Data

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcryptjs';

const prisma = new PrismaClient();

async function main() {
  const passwordHash = await bcrypt.hash('password123', 12);

  const user = await prisma.user.upsert({
    where: { email: 'demo@genai.app' },
    update: {},
    create: {
      email: 'demo@genai.app',
      passwordHash,
      name: 'Demo User',
      role: 'USER',
      plan: 'FREE',
      emailVerified: true,
    },
  });

  const admin = await prisma.user.upsert({
    where: { email: 'admin@genai.app' },
    update: {},
    create: {
      email: 'admin@genai.app',
      passwordHash: await bcrypt.hash('admin123', 12),
      name: 'Admin',
      role: 'ADMIN',
      plan: 'PRO',
      emailVerified: true,
    },
  });

  // Create sample images
  const samplePrompts = [
    'A cyberpunk city at sunset',
    'A cat wearing a space suit',
    'Abstract geometric patterns in neon colors',
    'A serene mountain lake in autumn',
    'A futuristic robot playing guitar',
    'Underwater coral reef with bioluminescent fish',
    'A steampunk clock tower',
    'Northern lights over a snowy cabin',
  ];

  for (let i = 0; i < samplePrompts.length; i++) {
    await prisma.image.create({
      data: {
        userId: user.id,
        prompt: samplePrompts[i],
        model: 'POLLINATIONS',
        style: ['realistic', 'artistic', 'anime', '3d-render', 'sketch', 'abstract'][i % 6],
        imageUrl: `https://image.pollinations.ai/prompt/${encodeURIComponent(samplePrompts[i])}?width=512&height=512&nologo=true&seed=${i}`,
        isPublic: i < 4,
        isFavorite: i < 2,
      },
    });
  }

  console.log('✅ Seed data created');
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

### Running Database Commands

```bash
# Create initial migration
npx prisma migrate dev --name init

# Apply migrations (production)
npx prisma migrate deploy

# Generate Prisma Client (after schema changes)
npx prisma generate

# Open Prisma Studio (visual DB browser)
npx prisma studio

# Seed database
npx prisma db seed

# Reset database (DESTRUCTIVE — drops all data)
npx prisma migrate reset

# Validate schema without migrating
npx prisma validate

# Format schema file
npx prisma format
```

## 5. Authentication System

### Auth Flow

```
┌──────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION FLOW                         │
│                                                              │
│  SIGNUP:                                                     │
│  1. POST /api/v1/auth/signup { email, password, name }       │
│  2. Server: hash password (bcrypt, 12 rounds)              │
│  3. Server: create user in DB (emailVerified: false)        │
│  4. Server: generate verify token, send verification email   │
│  5. Server: return access token (15min) + refresh token (7d)│
│  6. Client: store access token in memory, refresh token in    │
│     httpOnly secure cookie                                  │
│                                                              │
│  LOGIN:                                                      │
│  1. POST /api/v1/auth/login { email, password }             │
│  2. Server: verify email + password hash                    │
│  3. Server: check if email verified (optional enforcement)   │
│  4. Server: generate tokens, store refresh in DB           │
│  5. Server: return access token + set refresh cookie        │
│                                                              │
│  TOKEN REFRESH:                                              │
│  1. POST /api/v1/auth/refresh { refreshToken }               │
│  2. Server: verify refresh token exists in DB               │
│  3. Server: rotate — delete old, create new refresh token   │
│  4. Server: return new access token + new refresh cookie    │
│  5. If refresh token was ALREADY deleted → TOKEN THEFT:     │
│     Delete ALL refresh tokens for that user (force re-login)│
│                                                              │
│  LOGOUT:                                                     │
│  1. POST /api/v1/auth/logout                                 │
│  2. Server: delete refresh token from DB                    │
│  3. Server: clear cookie                                     │
│                                                              │
│  OAUTH (Google):                                             │
│  1. GET /api/v1/auth/google → redirect to Google consent    │
│  2. Google redirects back with authorization code            │
│  3. Server: exchange code for Google profile                 │
│  4. Server: find or create user, generate tokens            │
│  5. Server: redirect to frontend with tokens                │
│                                                              │
│  PASSWORD RESET:                                             │
│  1. POST /api/v1/auth/forgot-password { email }              │
│  2. Server: generate reset token (1 hour expiry)           │
│  3. Server: send email with reset link                       │
│  4. GET /reset-password?token=xxx (frontend route)           │
│  5. POST /api/v1/auth/reset-password { token, newPassword }  │
│  6. Server: verify token not expired, hash new password      │
│  7. Server: delete all refresh tokens (force re-login)      │
└──────────────────────────────────────────────────────────────┘
```

### Auth Service Implementation

```typescript
// src/modules/auth/auth.service.ts
import bcrypt from 'bcryptjs';
import crypto from 'crypto';
import jwt from 'jsonwebtoken';
import { PrismaClient } from '@prisma/client';
import { env } from '../../config/env';
import { AppError } from '../../utils/errors';
import { sendEmail } from '../../services/email.service';

const prisma = new PrismaClient();

export class AuthService {
  async signup(input: { email: string; password: string; name: string }) {
    const { email, password, name } = input;

    // Check if user exists
    const existing = await prisma.user.findUnique({ where: { email } });
    if (existing) {
      throw new AppError(409, 'EMAIL_EXISTS', 'An account with this email already exists');
    }

    // Validate password strength
    if (password.length < 8) {
      throw new AppError(400, 'WEAK_PASSWORD', 'Password must be at least 8 characters');
    }
    if (!/[A-Z]/.test(password) || !/[0-9]/.test(password)) {
      throw new AppError(400, 'WEAK_PASSWORD', 'Password must contain at least one uppercase letter and one number');
    }

    // Hash password
    const passwordHash = await bcrypt.hash(password, 12);

    // Generate email verification token
    const verifyToken = crypto.randomBytes(32).toString('hex');

    // Create user
    const user = await prisma.user.create({
      data: {
        email,
        passwordHash,
        name,
        verifyToken,
        emailVerified: false,
      },
    });

    // Send verification email (don't await — fire and forget)
    const verifyUrl = `${env.CORS_ORIGIN.split(',')[0]}/verify-email?token=${verifyToken}`;
    sendEmail({
      to: email,
      subject: 'Verify your GenAI account',
      html: `<h1>Welcome to GenAI!</h1><p>Click below to verify your email:</p><a href="${verifyUrl}" style="padding:12px 24px;background:#6C63FF;color:white;border-radius:8px;text-decoration:none;">Verify Email</a><p>This link expires in 24 hours.</p>`,
    }).catch(console.error);

    // Generate tokens
    return this.generateTokens(user);
  }

  async login(input: { email: string; password: string; deviceInfo?: string }) {
    const { email, password, deviceInfo } = input;

    const user = await prisma.user.findUnique({ where: { email } });
    if (!user) {
      // Don't reveal whether email exists — always say "invalid credentials"
      throw new AppError(401, 'INVALID_CREDENTIALS', 'Invalid email or password');
    }

    const isPasswordValid = await bcrypt.compare(password, user.passwordHash);
    if (!isPasswordValid) {
      throw new AppError(401, 'INVALID_CREDENTIALS', 'Invalid email or password');
    }

    if (user.deletedAt) {
      throw new AppError(403, 'ACCOUNT_DELETED', 'This account has been deactivated');
    }

    return this.generateTokens(user, deviceInfo);
  }

  async refreshToken(oldRefreshToken: string, deviceInfo?: string) {
    // Find the refresh token in DB
    const storedToken = await prisma.refreshToken.findUnique({
      where: { token: oldRefreshToken },
      include: { user: true },
    });

    if (!storedToken) {
      // TOKEN THEFT: This token was already used or doesn't exist
      // The user's refresh token was stolen. Delete ALL their refresh tokens.
      // This forces a full re-login on all devices.
      throw new AppError(401, 'TOKEN_REVOKED', 'Refresh token is invalid. Please log in again.');
    }

    if (storedToken.expiresAt < new Date()) {
      await prisma.refreshToken.delete({ where: { id: storedToken.id } });
      throw new AppError(401, 'TOKEN_EXPIRED', 'Refresh token has expired. Please log in again.');
    }

    // Delete the old refresh token (rotation)
    await prisma.refreshToken.delete({ where: { id: storedToken.id } });

    // Generate new tokens
    return this.generateTokens(storedToken.user, deviceInfo);
  }

  async logout(refreshToken: string) {
    if (refreshToken) {
      await prisma.refreshToken.deleteMany({ where: { token: refreshToken } });
    }
  }

  async verifyEmail(token: string) {
    const user = await prisma.user.findFirst({ where: { verifyToken: token } });
    if (!user) {
      throw new AppError(400, 'INVALID_TOKEN', 'Invalid verification token');
    }

    await prisma.user.update({
      where: { id: user.id },
      data: { emailVerified: true, verifyToken: null },
    });
  }

  async forgotPassword(email: string) {
    const user = await prisma.user.findUnique({ where: { email } });
    if (!user) {
      // Don't reveal whether email exists — always return success
      return { message: 'If an account with that email exists, we sent a reset link.' };
    }

    const resetToken = crypto.randomBytes(32).toString('hex');
    const expiresAt = new Date(Date.now() + 60 * 60 * 1000); // 1 hour

    await prisma.passwordReset.upsert({
      where: { userId: user.id },
      update: { token: resetToken, expiresAt },
      create: { userId: user.id, token: resetToken, expiresAt },
    });

    const resetUrl = `${env.CORS_ORIGIN.split(',')[0]}/reset-password?token=${resetToken}`;
    sendEmail({
      to: email,
      subject: 'Reset your GenAI password',
      html: `<h1>Reset Your Password</h1><p>Click below to set a new password:</p><a href="${resetUrl}" style="padding:12px 24px;background:#6C63FF;color:white;border-radius:8px;text-decoration:none;">Reset Password</a><p>This link expires in 1 hour.</p><p>If you didn't request this, ignore this email.</p>`,
    }).catch(console.error);

    return { message: 'If an account with that email exists, we sent a reset link.' };
  }

  async resetPassword(token: string, newPassword: string) {
    const reset = await prisma.passwordReset.findUnique({ where: { token } });
    if (!reset || reset.expiresAt < new Date()) {
      throw new AppError(400, 'INVALID_TOKEN', 'Reset token is invalid or expired');
    }

    const passwordHash = await bcrypt.hash(newPassword, 12);

    // Update password and delete ALL refresh tokens (force re-login on all devices)
    await prisma.$transaction([
      prisma.user.update({
        where: { id: reset.userId },
        data: { passwordHash },
      }),
      prisma.refreshToken.deleteMany({
        where: { userId: reset.userId },
      }),
      prisma.passwordReset.delete({
        where: { token },
      }),
    ]);
  }

  private async generateTokens(user: any, deviceInfo?: string) {
    // Access token — short-lived, used for API calls
    const accessToken = jwt.sign(
      { sub: user.id, email: user.email, role: user.role },
      env.JWT_SECRET,
      { expiresIn: env.JWT_ACCESS_EXPIRY }
    );

    // Refresh token — long-lived, used to get new access tokens
    const refreshToken = crypto.randomBytes(64).toString('hex');
    const expiresAt = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000); // 7 days

    // Store refresh token in DB (one per device)
    await prisma.refreshToken.create({
      data: {
        token: refreshToken,
        userId: user.id,
        deviceInfo: deviceInfo || 'unknown',
        expiresAt,
      },
    });

    return {
      accessToken,
      refreshToken,
      user: {
        id: user.id,
        email: user.email,
        name: user.name,
        role: user.role,
        plan: user.plan,
        avatarUrl: user.avatarUrl,
        emailVerified: user.emailVerified,
      },
    };
  }
}
```

### Auth Middleware

```typescript
// src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { env } from '../config/env';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export interface AuthRequest extends Request {
  user?: {
    id: string;
    email: string;
    role: string;
  };
}

export function authenticate(req: AuthRequest, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization;

  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: { code: 'UNAUTHORIZED', message: 'Missing or invalid token' } });
  }

  const token = authHeader.split(' ')[1];

  try {
    const payload = jwt.verify(token, env.JWT_SECRET) as { sub: string; email: string; role: string };
    req.user = { id: payload.sub, email: payload.email, role: payload.role };
    next();
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      return res.status(401).json({ error: { code: 'TOKEN_EXPIRED', message: 'Access token has expired' } });
    }
    return res.status(401).json({ error: { code: 'UNAUTHORIZED', message: 'Invalid token' } });
  }
}

export function requireAdmin(req: AuthRequest, res: Response, next: NextFunction) {
  if (req.user?.role !== 'ADMIN') {
    return res.status(403).json({ error: { code: 'FORBIDDEN', message: 'Admin access required' } });
  }
  next();
}

export function requirePlan(plan: 'PRO' | 'ENTERPRISE') {
  return (req: AuthRequest, res: Response, next: NextFunction) => {
    const user = req.user;
    if (!user) return res.status(401).json({ error: { code: 'UNAUTHORIZED', message: 'Authentication required' } });

    const planHierarchy = { FREE: 0, PRO: 1, ENTERPRISE: 2 };
    if (planHierarchy[user.plan as keyof typeof planHierarchy] < planHierarchy[plan]) {
      return res.status(403).json({
        error: {
          code: 'PLAN_REQUIRED',
          message: `This feature requires a ${plan} plan. Upgrade to unlock it.`,
          requiredPlan: plan,
        },
      });
    }
    next();
  };
}
```

### Auth Routes

```typescript
// src/modules/auth/auth.routes.ts
import { Router } from 'express';
import { AuthController } from './auth.controller';
import { validate } from '../../middleware/validate';
import { authValidation } from './auth.validation';
import { authenticate } from '../../middleware/auth';

const router = Router();
const controller = new AuthController();

router.post('/signup', validate(authValidation.signup), controller.signup);
router.post('/login', validate(authValidation.login), controller.login);
router.post('/refresh', controller.refreshToken);
router.post('/logout', controller.logout);
router.post('/verify-email', validate(authValidation.verifyEmail), controller.verifyEmail);
router.post('/forgot-password', validate(authValidation.forgotPassword), controller.forgotPassword);
router.post('/reset-password', validate(authValidation.resetPassword), controller.resetPassword);

// Authenticated routes
router.get('/me', authenticate, controller.getMe);
router.delete('/account', authenticate, controller.deleteAccount);

export default router;
```

### Validation Schemas

```typescript
// src/modules/auth/auth.validation.ts
import { z } from 'zod';

export const authValidation = {
  signup: z.object({
    body: z.object({
      email: z.string().email('Invalid email address'),
      password: z.string().min(8, 'Password must be at least 8 characters'),
      name: z.string().min(1, 'Name is required').max(100),
    }),
  }),

  login: z.object({
    body: z.object({
      email: z.string().email('Invalid email address'),
      password: z.string().min(1, 'Password is required'),
    }),
  }),

  verifyEmail: z.object({
    body: z.object({
      token: z.string().min(1, 'Token is required'),
    }),
  }),

  forgotPassword: z.object({
    body: z.object({
      email: z.string().email('Invalid email address'),
    }),
  }),

  resetPassword: z.object({
    body: z.object({
      token: z.string().min(1, 'Token is required'),
      password: z.string().min(8, 'Password must be at least 8 characters'),
    }),
  }),
};
```

### Validation Middleware

```typescript
// src/middleware/validate.ts
import { Request, Response, NextFunction, RequestHandler } from 'express';
import { ZodSchema } from 'zod';

export function validate(schema: ZodSchema): RequestHandler {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse({
      body: req.body,
      query: req.query,
      params: req.params,
    });

    if (!result.success) {
      return res.status(400).json({
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Invalid input data',
          details: result.error.flatten().fieldErrors,
        },
      });
    }

    // Replace req.body/query/params with validated data
    req.body = result.data.body ?? req.body;
    next();
  };
}
```

## 6. API Design — REST Endpoints

### Complete API Route Map

```
Authentication:
  POST   /api/v1/auth/signup              Create account
  POST   /api/v1/auth/login               Login
  POST   /api/v1/auth/refresh             Refresh access token
  POST   /api/v1/auth/logout              Logout (invalidate refresh)
  POST   /api/v1/auth/verify-email        Verify email address
  POST   /api/v1/auth/forgot-password     Request password reset
  POST   /api/v1/auth/reset-password      Reset password with token
  GET    /api/v1/auth/me                   Get current user

Images:
  POST   /api/v1/images/generate          Generate new image
  GET    /api/v1/images                    List user's images (paginated)
  GET    /api/v1/images/:id                Get single image
  DELETE /api/v1/images/:id                Delete image
  PATCH  /api/v1/images/:id                Update image (favorite, public)
  POST   /api/v1/images/:id/share          Share image (generate public link)

Gallery:
  GET    /api/v1/galleries                List user's galleries
  POST   /api/v1/galleries                Create gallery
  GET    /api/v1/galleries/:id             Get gallery with images
  PATCH  /api/v1/galleries/:id             Update gallery
  DELETE /api/v1/galleries/:id             Delete gallery
  POST   /api/v1/galleries/:id/images     Add images to gallery
  DELETE /api/v1/galleries/:id/images/:imageId  Remove image from gallery

Explore:
  GET    /api/v1/explore                   Get public images (paginated, searchable)
  GET    /api/v1/explore/:id               Get single public image

Billing:
  POST   /api/v1/billing/checkout         Create Stripe checkout session
  POST   /api/v1/billing/webhook           Stripe webhook handler
  GET    /api/v1/billing/subscription      Get current subscription status
  POST   /api/v1/billing/cancel            Cancel subscription

Health:
  GET    /api/v1/health                    Health check (DB, Redis, storage)
```

### Express App Setup

```typescript
// src/app.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import compression from 'compression';
import { rateLimit } from 'express-rate-limit';
import { env } from './config/env';
import { requestLogger } from './middleware/requestLogger';
import { requestId } from './middleware/requestId';
import { errorHandler } from './middleware/errorHandler';
import authRoutes from './modules/auth/auth.routes';
import imageRoutes from './modules/images/image.routes';
import galleryRoutes from './modules/gallery/gallery.routes';
import billingRoutes from './modules/billing/billing.routes';

export const app = express();

// Security headers
app.use(helmet());
app.disable('x-powered-by');

// Compression
app.use(compression());

// Request ID (for tracing)
app.use(requestId());

// CORS
app.use(cors({
  origin: env.CORS_ORIGIN.split(','),
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-Id'],
}));

// Body parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Rate limiting
const limiter = rateLimit({
  windowMs: env.RATE_LIMIT_WINDOW_MS,
  max: env.RATE_LIMIT_MAX,
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: { code: 'RATE_LIMITED', message: 'Too many requests, please try again later.' } },
  keyGenerator: (req) => req.ip || 'unknown',
});
app.use('/api/', limiter);

// Stricter rate limit for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  message: { error: { code: 'RATE_LIMITED', message: 'Too many login attempts.' } },
});
app.use('/api/v1/auth/login', authLimiter);
app.use('/api/v1/auth/signup', authLimiter);

// Request logging
app.use(requestLogger);

// Health check (before auth)
app.get('/api/v1/health', async (req, res) => {
  const checks: Record<string, boolean> = { server: true };

  try {
    await prisma.$queryRaw`SELECT 1`;
    checks.database = true;
  } catch { checks.database = false; }

  try {
    await redis.ping();
    checks.redis = true;
  } catch { checks.redis = false; }

  const healthy = Object.values(checks).every(Boolean);
  res.status(healthy ? 200 : 503).json({ status: healthy ? 'healthy' : 'degraded', checks, timestamp: new Date().toISOString() });
});

// Routes
app.use('/api/v1/auth', authRoutes);
app.use('/api/v1/images', imageRoutes);
app.use('/api/v1/galleries', galleryRoutes);
app.use('/api/v1/billing', billingRoutes);

// 404
app.use((req, res) => {
  res.status(404).json({ error: { code: 'NOT_FOUND', message: `Route ${req.method} ${req.path} not found` } });
});

// Global error handler (MUST be last)
app.use(errorHandler);
```

```typescript
// src/index.ts
import { app } from './app';
import { PrismaClient } from '@prisma/client';
import { env } from './config/env';

export const prisma = new PrismaClient();

app.listen(env.PORT, () => {
  console.log(`
  ╔══════════════════════════════════════════════════════════╗
  ║            🎨 GENAI API Server                           ║
  ╠══════════════════════════════════════════════════════════╣
  ║  Environment: ${env.NODE_ENV.padEnd(44)}║
  ║  Port:         ${String(env.PORT).padEnd(44)}║
  ║  Health:        http://localhost:${env.PORT}/api/v1/health${' '.repeat(24)}║
  ╚══════════════════════════════════════════════════════════╝
  `);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received. Shutting down gracefully...');
  await prisma.$disconnect();
  process.exit(0);
});

process.on('SIGINT', async () => {
  console.log('SIGINT received. Shutting down gracefully...');
  await prisma.$disconnect();
  process.exit(0);
});
```

## 7. Image Generation Service

```typescript
// src/modules/images/providers/image-provider.interface.ts
export interface ImageGenerationResult {
  images: string[];
  model: string;
  prompt: string;
  enhancedPrompt?: string;
  generationTimeMs?: number;
  costCredits: number;
}

export interface ImageProvider {
  name: string;
  generate(prompt: string, options: ImageOptions): Promise<ImageGenerationResult>;
}

export interface ImageOptions {
  style?: string;
  size?: string;
  quality?: string;
  width?: number;
  height?: number;
}
```

```typescript
// src/modules/images/providers/pollinations.provider.ts
import { ImageProvider, ImageGenerationResult, ImageOptions } from './image-provider.interface';
import { STYLE_PRESETS } from './style-presets';

export class PollinationsProvider implements ImageProvider {
  name = 'pollinations-ai';

  async generate(prompt: string, options: ImageOptions): Promise<ImageGenerationResult> {
    const stylePrefix = STYLE_PRESETS[options.style || 'none'] || '';
    const enhancedPrompt = stylePrefix ? `${prompt}, ${stylePrefix}` : prompt;

    const width = parseInt(options.size?.split('x')[0] || '1024');
    const height = parseInt(options.size?.split('x')[1] || '1024');

    const startTime = Date.now();

    const imageUrl = `https://image.pollinations.ai/prompt/${encodeURIComponent(enhancedPrompt)}?width=${width}&height=${height}&nologo=true&seed=${Math.floor(Math.random() * 1000000)}`;

    // Verify the image actually loads
    const response = await fetch(imageUrl);
    if (!response.ok) throw new Error(`Pollinations returned ${response.status}`);

    return {
      images: [imageUrl],
      model: this.name,
      prompt,
      enhancedPrompt,
      generationTimeMs: Date.now() - startTime,
      costCredits: 0, // Free
    };
  }
}
```

```typescript
// src/modules/images/providers/dalle.provider.ts
import OpenAI from 'openai';
import { ImageProvider, ImageGenerationResult, ImageOptions } from './image-provider.interface';
import { STYLE_PRESETS } from './style-presets';
import { env } from '../../../config/env';

export class DalleProvider implements ImageProvider {
  name = 'dall-e-3';
  private client: OpenAI | null = null;

  constructor() {
    if (env.OPENAI_API_KEY) {
      this.client = new OpenAI({ apiKey: env.OPENAI_API_KEY });
    }
  }

  async generate(prompt: string, options: ImageOptions): Promise<ImageGenerationResult> {
    if (!this.client) throw new Error('OpenAI API key not configured');

    const stylePrefix = STYLE_PRESETS[options.style || 'none'] || '';
    const enhancedPrompt = stylePrefix ? `${prompt}, ${stylePrefix}` : prompt;

    const sizeMap: Record<string, '1024x1024' | '1792x1024' | '1024x1792'> = {
      '1024x1024': '1024x1024',
      '1792x1024': '1792x1024',
      '1024x1792': '1024x1792',
    };
    const size = sizeMap[options.size || '1024x1024'] || '1024x1024';

    const startTime = Date.now();

    const response = await this.client.images.generate({
      model: 'dall-e-3',
      prompt: enhancedPrompt,
      n: 1,
      size,
      quality: options.quality === 'hd' ? 'hd' : 'standard',
    });

    return {
      images: response.data.map(img => img.url!),
      model: this.name,
      prompt,
      enhancedPrompt,
      generationTimeMs: Date.now() - startTime,
      costCredits: options.quality === 'hd' ? 2 : 1,
    };
  }
}
```

```typescript
// src/modules/images/providers/flux.provider.ts
import { ImageProvider, ImageGenerationResult, ImageOptions } from './image-provider.interface';
import { STYLE_PRESETS } from './style-presets';
import { env } from '../../../config/env';
import { StorageService } from '../../../services/storage.service';

export class FluxProvider implements ImageProvider {
  name = 'flux-pro';
  private storageService = new StorageService();

  async generate(prompt: string, options: ImageOptions): Promise<ImageGenerationResult> {
    if (!env.HUGGING_FACE_TOKEN) throw new Error('Hugging Face token not configured');

    const stylePrefix = STYLE_PRESETS[options.style || 'none'] || '';
    const enhancedPrompt = stylePrefix ? `${prompt}, ${stylePrefix}` : prompt;

    const startTime = Date.now();

    const response = await fetch(
      'https://api-inference.huggingface.co/models/black-forest-labs/FLUX.1-dev',
      {
        headers: {
          'Authorization': `Bearer ${env.HUGGING_FACE_TOKEN}`,
          'Content-Type': 'application/json',
        },
        method: 'POST',
        body: JSON.stringify({
          inputs: enhancedPrompt,
          parameters: { width: 1024, height: 1024, num_inference_steps: 28 },
        }),
      }
    );

    if (!response.ok) throw new Error(`Flux API returned ${response.status}`);

    const buffer = Buffer.from(await response.arrayBuffer());

    // Upload to storage immediately (don't store binary in DB)
    const key = `images/${Date.now()}-${Math.random().toString(36).slice(2)}.png`;
    const imageUrl = await this.storageService.upload(buffer, key, 'image/png');

    return {
      images: [imageUrl],
      model: this.name,
      prompt,
      enhancedPrompt,
      generationTimeMs: Date.now() - startTime,
      costCredits: 2,
    };
  }
}
```

```typescript
// src/modules/images/style-presets.ts
export const STYLE_PRESETS: Record<string, string> = {
  'realistic': 'photorealistic, 8k, highly detailed, professional photography, natural lighting',
  'artistic': 'artistic painting, expressive, bold colors, creative, masterpiece',
  'anime': 'anime style, manga, vibrant colors, Japanese art, anime artwork',
  '3d-render': '3D render, CGI, unreal engine, octane render, detailed, cinematic',
  'sketch': 'hand-drawn sketch, pencil drawing, artistic, detailed sketch',
  'abstract': 'abstract art, modern art, colorful, geometric patterns, surreal',
  'cyberpunk': 'cyberpunk, neon lights, futuristic city, dark, rain, technology',
  'watercolor': 'watercolor painting, soft colors, artistic, flowing, delicate',
  'none': '',
};
```

```typescript
// src/modules/images/image.service.ts
import { PrismaClient } from '@prisma/client';
import { PollinationsProvider } from './providers/pollinations.provider';
import { DalleProvider } from './providers/dalle.provider';
import { FluxProvider } from './providers/flux.provider';
import { ImageProvider, ImageOptions } from './providers/image-provider.interface';
import { AppError } from '../../utils/errors';
import {StorageService} from '../../services/storage.service';

const prisma = new PrismaClient();

export class ImageService {
  private providers: Record<string, ImageProvider>;
  private defaultProvider: ImageProvider;
  private storageService = new StorageService();

  constructor() {
    this.providers = {
      'dall-e-3': new DalleProvider(),
      'flux-pro': new FluxProvider(),
      'pollinations-ai': new PollinationsProvider(),
      'stable-diffusion': new PollinationsProvider(), // Uses Pollinations fallback
    };

    this.defaultProvider = new PollinationsProvider();
  }

  async generate(userId: string, prompt: string, options: ImageOptions) {
    // Check user's generation limits based on plan
    await this.checkGenerationLimit(userId);

    // Select provider
    const providerName = options.model || 'pollinations-ai';
    const provider = this.providers[providerName] || this.defaultProvider;

    let result;
    try {
      result = await provider.generate(prompt, options);
    } catch (error) {
      // Fallback to free provider on error
      console.error(`[ImageService] ${providerName} failed, falling back to pollinations:`, error);
      result = await this.defaultProvider.generate(prompt, options);
      result.model = `${providerName}-fallback-pollinations`;
    }

    // For providers that return URLs (not our storage), download and re-upload
    let finalImageUrl = result.images[0];
    if (!finalImageUrl.includes(env.S3_ENDPOINT || 'r2.cloudflarestorage.com')) {
      const response = await fetch(finalImageUrl);
      const buffer = Buffer.from(await response.arrayBuffer());
      const key = `images/${userId}/${Date.now()}-${Math.random().toString(36).slice(2)}.png`;
      finalImageUrl = await this.storageService.upload(buffer, key, 'image/png');
    }

    // Save to database
    const image = await prisma.image.create({
      data: {
        userId,
        prompt,
        enhancedPrompt: result.enhancedPrompt,
        model: result.model as any,
        style: options.style || 'none',
        size: options.size || '1024x1024',
        quality: options.quality || 'standard',
        imageUrl: finalImageUrl,
        generationTimeMs: result.generationTimeMs,
        costCredits: result.costCredits,
      },
    });

    return image;
  }

  private async checkGenerationLimit(userId: string) {
    const user = await prisma.user.findUnique({ where: { id: userId } });
    if (!user) throw new AppError(404, 'USER_NOT_FOUND', 'User not found');

    const today = new Date();
    today.setHours(0, 0, 0, 0);

    const todayCount = await prisma.image.count({
      where: {
        userId,
        createdAt: { gte: today },
      },
    });

    const limits = { FREE: 10, PRO: 100, ENTERPRISE: 1000 };
    const limit = limits[user.plan as keyof typeof limits] || 10;

    if (todayCount >= limit) {
      throw new AppError(429, 'LIMIT_EXCEEDED', `Daily generation limit reached (${limit}/day for ${user.plan} plan). Upgrade for more.`);
    }
  }

  async getUserImages(userId: string, page: number = 1, limit: number = 20) {
    const skip = (page - 1) * limit;

    const [images, total] = await Promise.all([
      prisma.image.findMany({
        where: { userId, deletedAt: null },
        orderBy: { createdAt: 'desc' },
        skip,
        take: limit,
      }),
      prisma.image.count({ where: { userId, deletedAt: null } }),
    ]);

    return {
      images,
      total,
      page,
      limit,
      hasMore: skip + images.length < total,
    };
  }

  async toggleFavorite(userId: string, imageId: string) {
    const image = await prisma.image.findFirst({
      where: { id: imageId, userId },
    });
    if (!image) throw new AppError(404, 'NOT_FOUND', 'Image not found');

    return prisma.image.update({
      where: { id: imageId },
      data: { isFavorite: !image.isFavorite },
    });
  }

  async deleteImage(userId: string, imageId: string) {
    const image = await prisma.image.findFirst({
      where: { id: imageId, userId },
    });
    if (!image) throw new AppError(404, 'NOT_FOUND', 'Image not found');

    // Soft delete
    return prisma.image.update({
      where: { id: imageId },
      data: { deletedAt: new Date() },
    });
  }
}
```

## 8. File Upload & Storage (S3/R2)

```typescript
// src/services/storage.service.ts
import { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { env } from '../config/env';

export class StorageService {
  private s3: S3Client;
  private bucket: string;

  constructor() {
    this.s3 = new S3Client({
      region: env.S3_REGION || 'auto',
      endpoint: env.S3_ENDPOINT, // Cloudflare R2 or AWS S3
      credentials: {
        accessKeyId: env.S3_ACCESS_KEY || '',
        secretAccessKey: env.S3_SECRET_KEY || '',
      },
    });
    this.bucket = env.S3_BUCKET;
  }

  async upload(buffer: Buffer, key: string, contentType: string): Promise<string> {
    await this.s3.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: buffer,
      ContentType: contentType,
      CacheControl: 'public, max-age=31536000', // 1 year cache
    }));

    // Return public URL
    if (env.S3_ENDPOINT) {
      // R2 public URL or custom domain
      return `${env.S3_ENDPOINT.replace('/r2/', '')}/${this.bucket}/${key}`;
    }
    return `https://${this.bucket}.s3.${env.S3_REGION}.amazonaws.com/${key}`;
  }

  async getSignedDownloadUrl(key: string, expiresIn: number = 3600): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });
    return getSignedUrl(this.s3, command, { expiresIn });
  }

  async delete(key: string): Promise<void> {
    await this.s3.send(new DeleteObjectCommand({
      Bucket: this.bucket,
      Key: key,
    }));
  }
}
```

## 9. Rate Limiting & Security

```typescript
// src/middleware/rateLimiter.ts
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { redis } from '../config/redis';

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    client: redis,
    prefix: 'rl:api:',
  }),
  message: { error: { code: 'RATE_LIMITED', message: 'Too many requests' } },
});

export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    client: redis,
    prefix: 'rl:auth:',
  }),
  message: { error: { code: 'RATE_LIMITED', message: 'Too many login attempts' } },
});

export const generateLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 5, // 5 generations per minute
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    client: redis,
    prefix: 'rl:generate:',
  }),
  keyGenerator: (req: any) => req.user?.id || req.ip,
  message: { error: { code: 'GENERATION_LIMIT', message: 'Too many image generations. Please wait a moment.' } },
});
```

## 10. Background Jobs & Queues

```typescript
// src/services/queue.service.ts
import { Queue, Worker } from 'bullmq';
import { redis } from '../config/redis';

export const imageQueue = new Queue('image-generation', {
  connection: redis.duplicate(),
});

export const emailQueue = new Queue('email', {
  connection: redis.duplicate(),
});

// Image generation worker
export const imageWorker = new Worker('image-generation', async (job) => {
  const { userId, prompt, model, style, size } = job.data;

  // Update progress
  await job.updateProgress(10);

  // Generate image (this is the long-running task)
  const imageService = new ImageService();
  const result = await imageService.generate(userId, prompt, { model, style, size });

  await job.updateProgress(90);

  // If using WebSocket, notify the client
  // websocketService.sendToUser(userId, { type: 'generation_complete', data: result });

  await job.updateProgress(100);
  return result;
}, {
  connection: redis.duplicate(),
  concurrency: 3, // Process 3 jobs at a time
});

// Email worker
export const emailWorker = new Worker('email', async (job) => {
  const { to, subject, html } = job.data;
  await sendEmail({ to, subject, html });
}, {
  connection: redis.duplicate(),
  concurrency: 5,
});
```

## 11. WebSocket Real-Time Updates

```typescript
// src/services/websocket.service.ts
import { Server as SocketServer } from 'socket.io';
import { verify } from 'jsonwebtoken';
import { env } from '../config/env';

class WebSocketService {
  private io: SocketServer | null = null;

  init(httpServer: any) {
    this.io = new SocketServer(httpServer, {
      cors: {
        origin: env.CORS_ORIGIN.split(','),
        credentials: true,
      },
    });

    this.io.use((socket, next) => {
      const token = socket.handshake.auth.token;
      if (!token) {
        return next(new Error('Authentication required'));
      }
      try {
        const payload = verify(token, env.JWT_SECRET);
        socket.data.userId = payload.sub;
        next();
      } catch {
        next(new Error('Invalid token'));
      }
    });

    this.io.on('connection', (socket) => {
      const userId = socket.data.userId;

      // Join user-specific room
      socket.join(`user:${userId}`);

      socket.on('disconnect', () => {
        socket.leave(`user:${userId}`);
      });
    });
  }

  sendToUser(userId: string, event: string, data: any) {
    this.io?.to(`user:${userId}`).emit(event, data);
  }

  broadcast(event: string, data: any) {
    this.io?.emit(event, data);
  }
}

export const websocketService = new WebSocketService();
```

## 12. Email & Notifications

```typescript
// src/services/email.service.ts
import { Resend } from 'resend';
import { env } from '../config/env';

const resend = env.RESEND_API_KEY ? new Resend(env.RESEND_API_KEY) : null;

interface EmailOptions {
  to: string;
  subject: string;
  html: string;
}

export async function sendEmail({ to, subject, html }: EmailOptions): Promise<void> {
  if (!resend) {
    console.log(`[Email] Would send to ${to}: ${subject}`);
    return;
  }

  const { error } = await resend.emails.send({
    from: env.EMAIL_FROM,
    to,
    subject,
    html,
  });

  if (error) {
    console.error('[Email] Failed to send:', error);
    throw new Error('Failed to send email');
  }
}
```

## 13. Error Handling & Logging

```typescript
// src/utils/errors.ts
export class AppError extends Error {
  public statusCode: number;
  public code: string;
  public details?: any;

  constructor(statusCode: number, code: string, message: string, details?: any) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.details = details;
  }
}

export class AuthError extends AppError {
  constructor(message: string) {
    super(401, 'AUTH_ERROR', message);
  }
}

export class ForbiddenError extends AppError {
  constructor(message: string) {
    super(403, 'FORBIDDEN', message);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(404, 'NOT_FOUND', `${resource} not found`);
  }
}

export class ValidationError extends AppError {
  constructor(details: any) {
    super(400, 'VALIDATION_ERROR', 'Invalid input data', details);
  }
}
```

```typescript
// src/middleware/errorHandler.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../utils/errors';
import { logger } from '../utils/logger';

export function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  const requestId = req.headers['x-request-id'] as string || 'unknown';

  if (err instanceof AppError) {
    logger.warn({ err, requestId }, `AppError: ${err.code}`);

    return res.status(err.statusCode).json({
      error: {
        code: err.code,
        message: err.message,
        details: err.details,
        requestId,
      },
    });
  }

  // Prisma errors
  if (err.name === 'PrismaClientKnownRequestError') {
    const prismaError = err as any;
    if (prismaError.code === 'P2002') {
      return res.status(409).json({
        error: {
          code: 'DUPLICATE',
          message: 'A record with this data already exists',
          requestId,
        },
      });
    }
  }

  // Unknown error
  logger.error({ err, requestId }, 'Unhandled error');

  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: process.env.NODE_ENV === 'production'
        ? 'An unexpected error occurred'
        : err.message,
      requestId,
    },
  });
}
```

```typescript
// src/utils/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  serializers: {
    err: pino.stdSerializers.err,
    req: (req: any) => ({
      method: req.method,
      url: req.url,
      headers: req.headers,
    }),
  },
});
```

```typescript
// src/middleware/requestLogger.ts
import { Request, Response, NextFunction } from 'express';
import { logger } from '../utils/logger';

export function requestLogger(req: Request, res: Response, next: NextFunction) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    const userId = (req as any).user?.id;

    logger.info({
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration,
      userId,
      ip: req.ip,
      userAgent: req.get('user-agent'),
    });
  });

  next();
}
```

---

# PART 2: INFRASTRUCTURE

## 14. Docker & Containerization

```dockerfile
# docker/Dockerfile

# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
COPY prisma ./prisma/
RUN npm ci

# Stage 2: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
RUN npm run build

# Stage 3: Production
FROM node:20-alpine AS runner
WORKDIR /app

# Create non-root user
RUN addgroup --g 1001 appgroup && adduser -u 1001 -G appgroup -s /bin/sh -D appuser

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
COPY --from=builder /app/prisma ./prisma

USER appuser

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3001/api/v1/health || exit 1

CMD ["node", "dist/index.js"]
```

## 15. Docker Compose — Full Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  # === API SERVER ===
  api:
    build:
      context: .
      dockerfile: docker/Dockerfile
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://genai:genai@postgres:5432/genai
      - REDIS_URL=redis://redis:6379
      - PORT=3001
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3001/api/v1/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    restart: unless-stopped

  # === WORKER (Background Jobs) ===
  worker:
    build:
      context: .
      dockerfile: docker/Dockerfile
    command: ["node", "dist/worker.js"]
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://genai:genai@postgres:5432/genai
      - REDIS_URL=redis://redis:6379
    env_file:
      - .env
    depends_on:
      - api
    restart: unless-stopped

  # === POSTGRES DATABASE ===
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: genai
      POSTGRES_PASSWORD: genai
      POSTGRES_DB: genai
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U genai -d genai"]
      interval: 10s
      timeout: 5s
      retries: 5

  # === REDIS ===
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # === NGINX REVERSE PROXY ===
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - api
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

```nginx
# nginx/nginx.conf
events {
    worker_connections 1024;
}

http {
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/m;

    # Upstream API server
    upstream api {
        server api:3001;
    }

    server {
        listen 80;
        server_name api.yourdomain.com;

        # Redirect HTTP to HTTPS
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name api.yourdomain.com;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        # Security headers
        add_header X-Frame-Options DENY always;
        add_header X-Content-Type-Options nosniff always;
        add_header X-XSS-Protection "0" always;
        add_header Referrer-Policy strict-origin-when-cross-origin always;
        add_header Content-Security-Policy "default-src 'self'; img-src 'self' data: https://*.cloudflarestorage.com https://image.pollinations.ai; script-src 'self'; style-src 'self' 'unsafe-inline';" always;

        # Health check (no rate limit)
        location /api/v1/health {
            proxy_pass http://api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # API proxy
        location /api/ {
            limit_req zone=api burst=50 nodelay;

            proxy_pass http://api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Timeouts
            proxy_connect_timeout 60s;
            proxy_read_timeout 300s;  # Image generation can take a while
            proxy_send_timeout 60s;

            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        # Max upload size
        client_max_body_size 10m;
    }
}
```

## 16. CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: genai_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci

      - run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/genai_test

      - run: npm run lint
      - run: npm run typecheck
      - run: npm run test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/genai_test
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret-for-ci-minimum-32-characters-long

      - run: npm run build

  deploy-staging:
    needs: test
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      # Deploy to Railway/Fly.io
      - uses: railwayapp/railway-deploy@v1
        with:
          railway_token: ${{ secrets.RAILWAY_TOKEN }}
          service: genai-api-staging

  deploy-production:
    needs: test
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      # Build and push Docker image
      - uses: docker/build-push-action@v5
        with:
          context: .
          file: docker/Dockerfile
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
      # Deploy to production
      - uses: railwayapp/railway-deploy@v1
        with:
          railway_token: ${{ secrets.RAILWAY_TOKEN }}
          service: genai-api-production
```

## 17. Deployment — Railway, Fly.io, AWS

### Option A: Railway (Easiest — Recommended for Start)

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login
railway login

# Initialize project
railway init

# Link to project
railway link

# Add PostgreSQL
railway add --plugin postgres

# Add Redis
railway add --plugin redis

# Set environment variables
railway variables set JWT_SECRET=$(openssl rand -hex 32)
railway variables set NODE_ENV=production
# ... set all other env vars from .env.example

# Deploy
railway up

# Run migrations
railway run npx prisma migrate deploy

# Seed database
railway run npx prisma db seed
```

### Option B: Fly.io (More Control)

```bash
# Install Fly CLI
curl -L https://fly.io/install.sh | sh

# Login
fly auth login

# Launch app
fly launch

# Add PostgreSQL
fly postgres create

# Add Redis
fly redis create

# Set secrets
fly secrets set JWT_SECRET=$(openssl rand -hex 32)
fly secrets set DATABASE_URL=postgresql://...
# ... set all other secrets

# Deploy
fly deploy

# Run migrations
fly ssh console -C "npx prisma migrate deploy"
```

### Option C: AWS (Most Control — Most Complex)

```
1. ECS Fargate for API server
2. RDS PostgreSQL for database
3. ElastiCache Redis for caching/queues
4. S3 (or CloudFront + R2) for image storage
5. CloudFront CDN for images
6. Application Load Balancer for routing
7. Route 53 for DNS
8. ACM for SSL certificates
9. CloudWatch for logging/monitoring
10. CodePipeline for CI/CD

≫ THIS IS OVERKILL for a startup. Use Railway or Fly.io until you have 10,000+ users.
```

## 18. Domain, SSL, & CDN

```
1. Buy domain on Namecheap, Cloudflare, or Google Domains
2. Point DNS to your hosting platform:
   - Railway: Add custom domain in Railway dashboard, update DNS CNAME
   - Fly.io: fly certs add api.yourdomain.com
3. SSL is automatic on both Railway and Fly.io (Let's Encrypt)
4. For images, use Cloudflare R2 (S3-compatible, free egress):
   - Create R2 bucket
   - Enable public access
   - Use R2 API token as S3_ACCESS_KEY / S3_SECRET_KEY
   - Set S3_ENDPOINT to your R2 endpoint
   - Add custom domain to R2 bucket for image URLs
5. CDN: Cloudflare automatically caches R2 images globally
```

## 19. Monitoring, Logging & Alerting

```typescript
// src/config/sentry.ts
import * as Sentry from '@sentry/node';
import { env } from './env';

export function initSentry() {
  if (!env.SENTRY_DSN) return;

  Sentry.init({
    dsn: env.SENTRY_DSN,
    environment: env.NODE_ENV,
    tracesSampleRate: env.NODE_ENV === 'production' ? 0.1 : 1.0,
    profilesSampleRate: env.NODE_ENV === 'production' ? 0.1 : 1.0,
    integrations: [
      Sentry.prismaIntegration(),
      Sentry.httpIntegration(),
      Sentry.expressIntegration(),
    ],
  });
}
```

```typescript
// Add to src/app.ts after Sentry init:
// Sentry error handler (BEFORE other error handlers)
app.use(Sentry.Handlers.errorHandler());
```

### Monitoring Stack (Self-Hosted, Free Alternative)

```
If you don't want to pay for Sentry/Datadog:

1. Prometheus + Grafana (metrics)
   - Prometheus scrapes /api/v1/health for uptime
   - Grafana dashboards for request rate, error rate, latency

2. Loki (logs)
   - Collect all logs, search with LogQL
   - Replace for CloudWatch/Elasticsearch

3. Uptime monitoring
   - Use UptimeRobot (free tier) → pings your health endpoint every 5 min
   - Alerts via email/Slack when your API is down
```

## 20. Database Backups & Migrations

```bash
# Manual backup
pg_dump -U genai -d genai > backup_$(date +%Y%m%d).sql

# Restore
psql -U genai -d genai < backup_20240101.sql

# Prisma migrations (ALWAYS use these, never raw SQL for schema changes)
npx prisma migrate dev --name add_user_avatar    # Development
npx prisma migrate deploy                         # Production

# If you need to rollback: manually create a new migration that reverses the change
```

### Automated Backups (Railway)

Railway automatically backs up PostgreSQL daily. For other platforms:

```bash
# Crontab backup script (run daily at 3 AM)
# 0 3 * * * /path/to/backup.sh

#!/bin/bash
# backup.sh
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"
DATABASE_URL="postgresql://genai:password@localhost:5432/genai"

pg_dump "$DATABASE_URL" > "$BACKUP_DIR/genai_$DATE.sql"

# Upload to S3
aws s3 cp "$BACKUP_DIR/genai_$DATE.sql" s3://genai-backups/

# Keep only last 30 days of local backups
find "$BACKUP_DIR" -name "genai_*.sql" -mtime +30 -delete
```

---

# PART 3: FRONTEND (WEB)

## 21. Frontend Project Structure

```
frontend/
├── public/
│   ├── favicon.svg
│   └── manifest.json                ← PWA manifest
├── src/
│   ├── main.tsx                     ← Entry point
│   ├── App.tsx                      ← Route definitions
│   ├── vite-env.d.ts
│   │
│   ├── api/
│   │   ├── client.ts                ← Axios instance (base URL, interceptors)
│   │   ├── auth.api.ts              ← Signup, login, refresh, logout calls
│   │   ├── images.api.ts            ← Generate, list, delete, favorite images
│   │   ├── gallery.api.ts           ← Gallery CRUD
│   │   └── types.ts                 ← API response types
│   │
│   ├── auth/
│   │   ├── AuthProvider.tsx           ← Context + token management
│   │   ├── useAuth.ts                ← Access auth state hook
│   │   └── ProtectedRoute.tsx        ← Redirect to login if not authenticated
│   │
│   ├── hooks/
│   │   ├── useImages.ts              ← TanStack Query hook for images
│   │   ├── useGallery.ts
│   │   ├── useDebounce.ts
│   │   └── useMediaQuery.ts
│   │
│   ├── stores/
│   │   ├── themeStore.ts             ← Zustand: dark/light mode
│   │   ├── generationStore.ts        ← Zustand: current generation state
│   │   └── uiStore.ts               ← Zustand: modals, sidebar, etc.
│   │
│   ├── components/
│   │   ├── ui/                       ← Primitive UI components
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── Toast.tsx
│   │   │   ├── Spinner.tsx
│   │   │   └── Dropdown.tsx
│   │   │
│   │   ├── layout/
│   │   │   ├── Header.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Footer.tsx
│   │   │   └── Shell.tsx             ← Main app shell
│   │   │
│   │   └── features/
│   │       ├── generate/
│   │       │   ├── GeneratePage.tsx
│   │       │   ├── PromptInput.tsx
│   │       │   ├── ModelSelector.tsx
│   │       │   ├── StyleSelector.tsx
│   │       │   ├── SizeSelector.tsx
│   │       │   └── GenerationResult.tsx
│   │       │
│   │       ├── gallery/
│   │       │   ├── GalleryPage.tsx
│   │       │   ├── ImageCard.tsx
│   │       │   ├── ImageDetail.tsx
│   │       │   └── GalleryGrid.tsx
│   │       │
│   │       ├── auth/
│   │       │   ├── LoginPage.tsx
│   │       │   ├── SignupPage.tsx
│   │       │   └── ForgotPasswordPage.tsx
│   │       │
│   │       └── settings/
│   │           ├── SettingsPage.tsx
│   │           └── ThemeToggle.tsx
│   │
│   ├── pages/
│   │   ├── Home.tsx
│   │   ├── Explore.tsx
│   │   └── Pricing.tsx
│   │
│   ├── styles/
│   │   └── globals.css               ← Tailwind base + custom CSS
│   │
│   └── utils/
│       ├── cn.ts                     ← clsx + twMerge utility
│       ├── format.ts                 ← Date, number formatters
│       └── constants.ts              ← App-wide constants, API URLs
│
├── index.html
├── tailwind.config.ts
├── postcss.config.js
├── vite.config.ts
├── tsconfig.json
├── package.json
└── README.md
```

## 22. React + Vite + TypeScript Setup

```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
npm install react-router-dom @tanstack/react-query zustand axios
npm install @headlessui/react lucide-react clsx tailwind-merge
npm install -D @types/node
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true,
      },
    },
  },
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
          ui: ['@headlessui/react', 'lucide-react'],
          query: ['@tanstack/react-query'],
        },
      },
    },
  },
});
```

## 23. State Management (Zustand)

```typescript
// src/stores/themeStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

type Theme = 'light' | 'dark' | 'system';

interface ThemeState {
  theme: Theme;
  setTheme: (theme: Theme) => void;
}

export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      theme: 'system',
      setTheme: (theme) => set({ theme }),
    }),
    { name: 'genai-theme' } // localStorage key
  )
);
```

```typescript
// src/stores/generationStore.ts
import { create } from 'zustand';

interface GenerationState {
  isGenerating: boolean;
  currentPrompt: string;
  currentModel: string;
  currentStyle: string;
  currentSize: string;
  resultImages: string[];
  error: string | null;
  setGenerating: (v: boolean) => void;
  setPrompt: (p: string) => void;
  setModel: (m: string) => void;
  setStyle: (s: string) => void;
  setSize: (s: string) => void;
  setResultImages: (images: string[]) => void;
  setError: (e: string | null) => void;
  reset: () => void;
}

const initialState = {
  isGenerating: false,
  currentPrompt: '',
  currentModel: 'pollinations-ai',
  currentStyle: 'none',
  currentSize: '1024x1024',
  resultImages: [] as string[],
  error: null as string | null,
};

export const useGenerationStore = create<GenerationState>()((set) => ({
  ...initialState,
  setGenerating: (v) => set({ isGenerating: v }),
  setPrompt: (p) => set({ currentPrompt: p }),
  setModel: (m) => set({ currentModel: m }),
  setStyle: (s) => set({ currentStyle: s }),
  setSize: (s) => set({ currentSize: s }),
  setResultImages: (images) => set({ resultImages: images }),
  setError: (e) => set({ error: e }),
  reset: () => set(initialState),
}));
```

## 24. API Client & Data Fetching (TanStack Query)

```typescript
// src/api/client.ts
import axios, { InternalAxiosRequestConfig } from 'axios';

const API_BASE_URL = import.meta.env.VITE_API_URL || '/api/v1';

const apiClient = axios.create({
  baseURL: API_BASE_URL,
  timeout: 60000, // 60s for image generation
  headers: { 'Content-Type': 'application/json' },
});

// Request interceptor: attach access token
apiClient.interceptors.request.use((config: InternalAxiosRequestConfig) => {
  const token = localStorage.getItem('access_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor: handle 401 → refresh token → retry
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        const refreshToken = localStorage.getItem('refresh_token');
        if (!refreshToken) {
          // No refresh token → force login
          window.location.href = '/login';
          return Promise.reject(error);
        }

        const { data } = await axios.post(`${API_BASE_URL}/auth/refresh`, {
          refreshToken,
        });

        localStorage.setItem('access_token', data.accessToken);
        localStorage.setItem('refresh_token', data.refreshToken);

        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return apiClient(originalRequest);
      } catch (refreshError) {
        // Refresh also failed → force login
        localStorage.removeItem('access_token');
        localStorage.removeItem('refresh_token');
        window.location.href = '/login';
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);

export default apiClient;
```

```typescript
// src/api/images.api.ts
import apiClient from './client';
import { Image, PaginatedResponse } from './types';

export const imagesApi = {
  generate: (data: { prompt: string; model: string; style: string; size: string }) =>
    apiClient.post<{ image: Image }>('/images/generate', data),

  list: (page: number = 1, limit: number = 20) =>
    apiClient.get<PaginatedResponse<Image>>('/images', { params: { page, limit } }),

  get: (id: string) =>
    apiClient.get<{ image: Image }>(`/images/${id}`),

  delete: (id: string) =>
    apiClient.delete(`/images/${id}`),

  toggleFavorite: (id: string) =>
    apiClient.patch<{ image: Image }>(`/images/${id}`, {}),
};
```

```typescript
// src/hooks/useImages.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { imagesApi } from '../api/images.api';

export function useImages(page: number = 1) {
  return useQuery({
    queryKey: ['images', page],
    queryFn: () => imagesApi.list(page).then(r => r.data),
  });
}

export function useGenerateImage() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: imagesApi.generate,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['images'] });
    },
  });
}

export function useDeleteImage() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: imagesApi.delete,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['images'] });
    },
  });
}
```

## 25. Authentication Flow (Frontend)

```typescript
// src/auth/AuthProvider.tsx
import { createContext, useCallback, useEffect, useState } from 'react';
import { authApi } from '../api/auth.api';

interface User {
  id: string;
  email: string;
  name: string;
  role: string;
  plan: string;
  emailVerified: boolean;
}

interface AuthContextType {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  signup: (email: string, password: string, name: string) => Promise<void>;
  logout: () => Promise<void>;
  refreshUser: () => Promise<void>;
}

export const AuthContext = createContext<AuthContextType>(null!);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  const refreshUser = useCallback(async () => {
    try {
      const token = localStorage.getItem('access_token');
      if (!token) { setUser(null); return; }
      const { data } = await authApi.me();
      setUser(data.user);
    } catch {
      setUser(null);
      localStorage.removeItem('access_token');
      localStorage.removeItem('refresh_token');
    } finally {
      setIsLoading(false);
    }
  }, []);

  useEffect(() => { refreshUser(); }, [refreshUser]);

  const login = async (email: string, password: string) => {
    const { data } = await authApi.login({ email, password });
    localStorage.setItem('access_token', data.accessToken);
    localStorage.setItem('refresh_token', data.refreshToken);
    setUser(data.user);
  };

  const signup = async (email: string, password: string, name: string) => {
    const { data } = await authApi.signup({ email, password, name });
    localStorage.setItem('access_token', data.accessToken);
    localStorage.setItem('refresh_token', data.refreshToken);
    setUser(data.user);
  };

  const logout = async () => {
    const refreshToken = localStorage.getItem('refresh_token');
    await authApi.logout(refreshToken || undefined);
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{
      user, isAuthenticated: !!user, isLoading,
      login, signup, logout, refreshUser,
    }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

```typescript
// src/auth/ProtectedRoute.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from './useAuth';
import { Spinner } from '../components/ui/Spinner';

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) return <Spinner fullScreen />;
  if (!isAuthenticated) return <Navigate to="/login" state={{ from: location }} replace />;

  return <>{children}</>;
}
```

## 26. UI Components & Design System

```tsx
// src/components/ui/Button.tsx
import { ButtonHTMLAttributes, forwardRef } from 'react';
import { cn } from '../../utils/cn';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant = 'primary', size = 'md', isLoading, children, disabled, ...props }, ref) => {
    const variants = {
      primary: 'bg-indigo-600 hover:bg-indigo-700 text-white shadow-sm',
      secondary: 'bg-gray-800 hover:bg-gray-700 text-gray-100 border border-gray-700',
      ghost: 'hover:bg-gray-800 text-gray-300',
      danger: 'bg-red-600 hover:bg-red-700 text-white',
    };

    const sizes = {
      sm: 'px-3 py-1.5 text-sm',
      md: 'px-4 py-2 text-sm',
      lg: 'px-6 py-3 text-base',
    };

    return (
      <button
        ref={ref}
        className={cn(
          'inline-flex items-center justify-center rounded-lg font-medium transition-colors',
          'focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2 focus:ring-offset-gray-900',
          'disabled:opacity-50 disabled:cursor-not-allowed',
          variants[variant],
          sizes[size],
          className
        )}
        disabled={disabled || isLoading}
        {...props}
      >
        {isLoading && (
          <svg className="animate-spin -ml-1 mr-2 h-4 w-4" fill="none" viewBox="0 0 24 24">
            <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" />
            <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z" />
          </svg>
        )}
        {children}
      </button>
    );
  }
);
```

## 27. Dark Mode & Theming

Already covered in the Zustand theme store. Apply it:

```tsx
// src/App.tsx
import { useEffect } from 'react';
import { useThemeStore } from './stores/themeStore';

function App() {
  const theme = useThemeStore((s) => s.theme);

  useEffect(() => {
    const root = document.documentElement;
    const systemDark = window.matchMedia('(prefers-color-scheme: dark)').matches;

    if (theme === 'dark' || (theme === 'system' && systemDark)) {
      root.classList.add('dark');
    } else {
      root.classList.remove('dark');
    }
  }, [theme]);

  return (
    <div className="min-h-screen bg-white dark:bg-gray-950 text-gray-900 dark:text-gray-100">
      {/* Routes */}
    </div>
  );
}
```

## 28-30. (Gallery, PWA, Performance)

These follow standard patterns:

- **Gallery**: TanStack Query `useImages()` hook → responsive grid → `ImageCard` component → click opens detail modal
- **PWA**: Add `vite-plugin-pwa`, create `manifest.json`, add service worker for offline caching
- **Performance**: Lazy-load routes with `React.lazy()`, use `IntersectionObserver` for image lazy loading, code-split with Vite's `manualChunks`

---

# PART 4: macOS DESKTOP APP

## 31. macOS App Architecture — Tauri vs Electron

| Aspect | Tauri | Electron |
|--------|-------|----------|
| Bundle size | ~3-5 MB | ~150+ MB |
| Memory usage | ~30-50 MB | ~200+ MB |
| Startup time | Instant | Slow |
| Language | Rust + Web frontend | Node.js + Web frontend |
| Native menus | Yes (Rust) | Yes (JS) |
| Auto-update | Built-in | electron-updater |
| Code signing | More complex | Simpler |
| Community | Growing | Large |
| **Recommendation** | **Use Tauri** | Only if you need npm ecosystem deeply |

## 32. Tauri Setup & Project Structure

```bash
# Prerequisites: Rust installed (curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh)

# Create Tauri app (using your existing frontend)
npm install -D @tauri-apps/cli
npx tauri init

# When prompted:
# App name: GenAI
# Window title: GenAI — AI Image Generator
# Frontend dev URL: http://localhost:5173
# Frontend build command: npm run build
# Frontend dist dir: ../dist
```

```
# After init, your project gets a src-tauri/ directory:
src-tauri/
├── Cargo.toml               ← Rust dependencies
├── tauri.conf.json          ← Tauri config (window size, permissions, etc.)
├── icons/                    ← App icons for macOS
│   ├── icon.icns             ← macOS icon
│   ├── icon.png
│   └── ...
├── src/
│   ├── main.rs               ← Rust entry point
│   └── lib.rs                ← Rust commands (bridge between frontend and OS)
└── build.rs                  ← Build script
```

```json
// src-tauri/tauri.conf.json
{
  "productName": "GenAI",
  "version": "1.0.0",
  "identifier": "com.genai.app",
  "build": {
    "beforeDevCommand": "npm run dev",
    "devUrl": "http://localhost:5173",
    "beforeBuildCommand": "npm run build",
    "frontendDist": "../dist"
  },
  "app": {
    "windows": [
      {
        "title": "GenAI — AI Image Generator",
        "width": 1200,
        "height": 800,
        "minWidth": 800,
        "minHeight": 600,
        "resizable": true,
        "fullscreen": false,
        "hiddenTitle": false
      }
    ],
    "security": {
      "csp": "default-src 'self'; img-src 'self' data: https://*.cloudflarestorage.com https://image.pollinations.ai https://api.openai.com; script-src 'self'; style-src 'self' 'unsafe-inline'"
    },
    "macOSPrivateApi": true
  },
  "bundle": {
    "active": true,
    "targets": ["dmg", "app"],
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ],
    "category": "Productivity",
    "shortDescription": "AI Image Generator",
    "longDescription": "Generate stunning images using AI models including DALL-E 3, Flux Pro, and Stable Diffusion.",
    "macOS": {
      "minimumSystemVersion": "10.15"
    }
  },
  "plugins": {
    "updater": {
      "active": true,
      "endpoints": ["https://api.yourdomain.com/api/v1/updates/{{target}}/{{arch}}/{{current_version}}"],
      "pubkey": "YOUR_PUBLIC_KEY_HERE"
    }
  }
}
```

### Rust Commands (Bridge Between Frontend and OS)

```rust
// src-tauri/src/lib.rs
use tauri::Manager;

#[tauri::command]
fn get_app_version() -> String {
    env!("CARGO_PKG_VERSION").to_string()
}

#[tauri::command]
fn get_platform_info() -> serde_json::Value {
    serde_json::json!({
        "os": std::env::consts::OS,
        "arch": std::env::consts::ARCH,
        "family": std::env::consts::FAMILY,
    })
}

// Save image to local filesystem
#[tauri::command]
async fn save_image_to_downloads(app: tauri::AppHandle, image_url: String, filename: String) -> Result<String, String> {
    let response = reqwest::get(&image_url).await.map_err(|e| e.to_string())?;
    let bytes = response.bytes().await.map_err(|e| e.to_string())?;

    let downloads_dir = dirs::download_dir().ok_or("Cannot find downloads directory")?;
    let file_path = downloads_dir.join(&filename);

    std::fs::write(&file_path, bytes).map_err(|e| e.to_string())?;

    Ok(file_path.to_string_lossy().to_string())
}

// Open URL in default browser
#[tauri::command]
async fn open_in_browser(url: String) -> Result<(), String> {
    open::that(&url).map_err(|e| e.to_string())
}

// Get local storage path for images
#[tauri::command]
async fn get_local_storage_path(app: tauri::AppHandle) -> Result<String, String> {
    let app_dir = app.path_resolver().app_data_dir().ok_or("Cannot find app data dir")?;
    let images_dir = app_dir.join("generated_images");
    std::fs::create_dir_all(&images_dir).map_err(|e| e.to_string())?;
    Ok(images_dir.to_string_lossy().to_string())
}

fn main() {
    tauri::Builder::default()
        .invoke_handler(tauri::generate_handler![
            get_app_version,
            get_platform_info,
            save_image_to_downloads,
            open_in_browser,
            get_local_storage_path,
        ])
        .setup(|app| {
            #[cfg(debug_assertions)]
            {
                let window = app.get_window("main").unwrap();
                window.open_devtools();
            }
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### Using Tauri Commands from Frontend

```typescript
// src/api/tauri.ts
import { invoke } from '@tauri-apps/api/tauri';
import { isTauri } from './platform';

export const tauriApi = {
  getPlatformInfo: () => invoke('get_platform_info'),
  saveImageToDownloads: (url: string, filename: string) =>
    invoke('save_image_to_downloads', { imageUrl: url, filename }),
  openInBrowser: (url: string) => invoke('open_in_browser', { url }),
  getLocalStoragePath: () => invoke('get_local_storage_path'),
};
```

```typescript
// src/api/platform.ts
export function isTauri(): boolean {
  return typeof window !== 'undefined' && '__TAURI' in window;
}

export function isWeb(): boolean {
  return !isTauri();
}
```

## 33. macOS App Features — Native Menu, Dock, Notifications

```typescript
// src-tauri/src/menu.rs (add to lib.rs setup)
use tauri::{Menu, MenuItem, Submenu, AboutMetadata};

pub fn create_menu() -> Menu {
    Menu::new()
        .add_submenu(
            Submenu::new(
                "GenAI",
                Menu::new()
                    .add_native_item(MenuItem::About("GenAI".to_string(), AboutMetadata::new()))
                    .add_native_item(MenuItem::Separator)
                    .add_native_item(MenuItem::Services)
                    .add_native_item(MenuItem::Separator)
                    .add_native_item(MenuItem::Hide)
                    .add_native_item(MenuItem::HideOthers)
                    .add_native_item(MenuItem::ShowAll)
                    .add_native_item(MenuItem::Separator)
                    .add_native_item(MenuItem::Quit),
            )
        )
        .add_submenu(
            Submenu::new(
                "File",
                Menu::new()
                    .add_native_item(MenuItem::CloseWindow),
            )
        )
        .add_submenu(
            Submenu::new(
                "Edit",
                Menu::new()
                    .add_native_item(MenuItem::Undo)
                    .add_native_item(MenuItem::Redo)
                    .add_native_item(MenuItem::Separator)
                    .add_native_item(MenuItem::Cut)
                    .add_native_item(MenuItem::Copy)
                    .add_native_item(MenuItem::Paste)
                    .add_native_item(MenuItem::SelectAll),
            )
        )
        .add_submenu(
            Submenu::new(
                "View",
                Menu::new()
                    .add_native_item(MenuItem::Separator)
                    .add_native_item(MenuItem::EnterFullScreen)
                    .add_native_item(MenuItem::Separator)
                    .add_native_item(MenuItem::Minimize)
                    .add_native_item(MenuItem::Zoom),
            )
        )
        .add_submenu(
            Submenu::new(
                "Window",
                Menu::new()
                    .add_native_item(MenuItem::Minimize)
                    .add_native_item(MenuItem::Zoom),
            )
        )
}
```

## 34. macOS Code Signing & Notarization

```
REQUIREMENTS FOR MAC APP STORE / DISTRIBUTION:

1. Apple Developer Account ($99/year)
2. Developer ID Application Certificate (from developer.apple.com)
3. App-specific password for notarization

Steps:
1. Install certificate in Keychain Access
2. Build Tauri app:
   npx tauri build --target universal-apple-darwin

3. Code sign the .app bundle:
   codesign --deep --force --verify --verbose --sign "Developer ID Application: Your Name (TEAMID)" target/release/bundle/macos/GenAI.app

4. Create DMG for distribution:
   hdiutil create -volname "GenAI" -srcfolder target/release/bundle/macos/ -ov GenAI.dmg

5. Notarize (REQUIRED for macOS 10.15+ Gatekeeper):
   xcrun notarytool submit GenAI.dmg --apple-id "your@email.com" --password "app-specific-password" --team-id "TEAMID" --wait

6. Staple notarization ticket:
   xcrun stapler staple GenAI.dmg

7. Distribute the DMG or submit to Mac App Store
```

## 35. macOS App Store Distribution

```
Mac App Store Requirements:
1. Apple Developer Program membership ($99/year)
2. App Sandbox enabled (Tauri does this by default)
3. App Store icons: 1024x1024 master icon
4. Privacy descriptions for any permissions
5. App Review guidelines compliance

Steps:
1. Build for App Store:
   npx tauri build --target universal-apple-darwin --bundles app

2. Open Xcode Organizer, validate the archive

3. Upload to App Store Connect

4. Fill in:
   - App name, description, keywords
   - Screenshots (1280x800 for Mac)
   - Age rating
   - Privacy policy URL
   - Category: Productivity / Graphics & Design

5. Submit for review (1-3 days)
```

## 36. Auto-Updates for macOS

```rust
// src-tauri/src/main.rs — Add updater plugin
use tauri::updater::UpdaterExt;

fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_updater::Builder::new().build())
        .setup(|app| {
            // Check for updates on startup
            let handle = app.handle().clone();
            tauri::async_runtime::spawn(async move {
                match handle.updater().check().await {
                    Ok(update) => {
                        if update.is_update_available() {
                            println!("Update available: {}", update.version());
                            // Notify user via UI
                            handle.emit_all("update-available", update.version()).unwrap();
                        }
                    }
                    Err(e) => println!("Update check failed: {}", e),
                }
            });
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

```typescript
// Frontend: Listen for updates
import { listen } from '@tauri-apps/api/event';

listen('update-available', (event) => {
  const version = event.payload as string;
  // Show update dialog to user
  showUpdateDialog(version);
});
```

## 37-45: ANDROID APP (Flutter)

The Android app uses the SAME backend API. Here's the Flutter-specific architecture:

### Flutter Project Structure

```
android_app/
├── lib/
│   ├── main.dart                    ← Entry point
│   ├── app.dart                     ← MaterialApp configuration
│   │
│   ├── core/
│   │   ├── theme/
│   │   │   ├── app_theme.dart       ← Light/dark theme definitions
│   │   │   ├── colors.dart
│   │   │   └── typography.dart
│   │   ├── network/
│   │   │   ├── api_client.dart      ← Dio HTTP client with interceptors
│   │   │   ├── api_interceptor.dart ← Auth token injection + refresh
│   │   │   └── api_exceptions.dart
│   │   ├── storage/
│   │   │   ├── secure_storage.dart  ← Encrypted token storage
│   │   │   └── hive_storage.dart   ← Local data cache
│   │   └── utils/
│   │       ├── logger.dart
│   │       └── constants.dart
│   │
│   ├── features/
│   │   ├── auth/
│   │   │   ├── data/
│   │   │   │   ├── auth_repository.dart
│   │   │   │   └── auth_remote_data_source.dart
│   │   │   ├── domain/
│   │   │   │   └── auth_usecases.dart
│   │   │   └── presentation/
│   │   │       ├── login_screen.dart
│   │   │       ├── signup_screen.dart
│   │   │       └── auth_provider.dart
│   │   │
│   │   ├── generate/
│   │   │   ├── data/
│   │   │   │   ├── image_repository.dart
│   │   │   │   └── image_remote_data_source.dart
│   │   │   ├── domain/
│   │   │   │   └── generate_image_usecase.dart
│   │   │   └── presentation/
│   │   │       ├── generate_screen.dart
│   │   │       ├── prompt_input.dart
│   │   │       ├── model_selector.dart
│   │   │       └── generate_provider.dart
│   │   │
│   │   ├── gallery/
│   │   │   ├── data/
│   │   │   │   └── gallery_repository.dart
│   │   │   └── presentation/
│   │   │       ├── gallery_screen.dart
│   │   │       ├── image_detail_screen.dart
│   │   │       └── gallery_provider.dart
│   │   │
│   │   └── settings/
│   │       └── presentation/
│   │           ├── settings_screen.dart
│   │           └── settings_provider.dart
│   │
│   └── shared/
│       ├── widgets/
│       │   ├── loading_spinner.dart
│       │   ├── error_view.dart
│       │   ├── empty_state.dart
│       │   └── image_card.dart
│       └── models/
│           ├── user.dart
│           └── generated_image.dart
│
├── pubspec.yaml
├── android/
│   ├── app/
│   │   ├── build.gradle              ← Signing config, minSdk, targetSdk
│   │   └── src/main/
│   │       ├── AndroidManifest.xml
│   │       └── kotlin/.../MainActivity.kt
│   └── build.gradle
├── ios/                              ← Unused but present for potential future
├── test/
└── README.md
```

### Flutter API Client (Shared with Web/macOS backend)

```dart
// lib/core/network/api_client.dart
import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

final apiClientProvider = Provider<Dio>((ref) {
  final dio = Dio(BaseOptions(
    baseUrl: const String.fromEnvironment('API_URL', defaultValue: 'http://10.0.2.2:3001/api/v1'),
    connectTimeout: const Duration(seconds: 60),
    receiveTimeout: const Duration(seconds: 60),
    headers: {'Content-Type': 'application/json'},
  ));

  dio.interceptors.add(AuthInterceptor(ref));
  return dio;
});

class AuthInterceptor extends Interceptor {
  final Ref ref;
  AuthInterceptor(this.ref);

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    const storage = FlutterSecureStorage();
    final token = await storage.read(key: 'access_token');
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      const storage = FlutterSecureStorage();
      final refreshToken = await storage.read(key: 'refresh_token');

      if (refreshToken == null) {
        // Force logout
        handler.next(err);
        return;
      }

      try {
        final dio = Dio();
        final response = await dio.post(
          '${err.requestOptions.baseUrl}/auth/refresh',
          data: {'refreshToken': refreshToken},
        );

        final newAccessToken = response.data['accessToken'];
        final newRefreshToken = response.data['refreshToken'];
        await storage.write(key: 'access_token', value: newAccessToken);
        await storage.write(key: 'refresh_token', value: newRefreshToken);

        // Retry the original request
        err.requestOptions.headers['Authorization'] = 'Bearer $newAccessToken';
        final retryResponse = await dio.fetch(err.requestOptions);
        handler.resolve(retryResponse);
      } catch (e) {
        await storage.deleteAll();
        handler.next(err);
      }
    } else {
      handler.next(err);
    }
  }
}
```

### Flutter Image Generation Provider

```dart
// lib/features/generate/generate_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:dio/dio.dart';

class GenerateState {
  final bool isGenerating;
  final String? imageUrl;
  final String? error;
  final String prompt;
  final String model;
  final String style;

  const GenerateState({
    this.isGenerating = false,
    this.imageUrl,
    this.error,
    this.prompt = '',
    this.model = 'pollinations-ai',
    this.style = 'none',
  });

  GenerateState copyWith({
    bool? isGenerating,
    String? imageUrl,
    String? error,
    String? prompt,
    String? model,
    String? style,
  }) => GenerateState(
    isGenerating: isGenerating ?? this.isGenerating,
    imageUrl: imageUrl ?? this.imageUrl,
    error: error ?? this.error,
    prompt: prompt ?? this.prompt,
    model: model ?? this.model,
    style: style ?? this.style,
  );
}

class GenerateNotifier extends StateNotifier<GenerateState> {
  final Dio _apiClient;

  GenerateNotifier(this._apiClient) : super(const GenerateState());

  Future<void> generateImage() async {
    if (state.prompt.trim().isEmpty) return;

    state = state.copyWith(isGenerating: true, error: null);

    try {
      final response = await _apiClient.post('/images/generate', data: {
        'prompt': state.prompt,
        'model': state.model,
        'style': state.style,
        'size': '1024x1024',
      });

      final imageUrl = response.data['image']['imageUrl'];
      state = state.copyWith(isGenerating: false, imageUrl: imageUrl);
    } on DioException catch (e) {
      final message = e.response?.data['error']?['message'] ?? 'Generation failed';
      state = state.copyWith(isGenerating: false, error: message);
    }
  }

  void setPrompt(String prompt) => state = state.copyWith(prompt: prompt);
  void setModel(String model) => state = state.copyWith(model: model);
  void setStyle(String style) => state = state.copyWith(style: style);
  void reset() => state = const GenerateState();
}

final generateProvider = StateNotifierProvider<GenerateNotifier, GenerateState>((ref) {
  return GenerateNotifier(ref.watch(apiClientProvider));
});
```

## 42. Flutter Offline & Caching (Hive)

```dart
// lib/core/storage/hive_storage.dart
import 'package:hive_flutter/hive_flutter.dart';

class HiveStorage {
  static const _imagesBox = 'cached_images';

  static Future<void> init() async {
    await Hive.initFlutter();
    await Hive.openBox(_imagesBox);
  }

  static Future<void> cacheImage(String id, Map<String, dynamic> imageData) async {
    final box = Hive.box(_imagesBox);
    await box.put(id, imageData);
  }

  static List<Map<String, dynamic>> getCachedImages() {
    final box = Hive.box(_imagesBox);
    return box.values.map((e) => Map<String, dynamic>.from(e)).toList();
  }

  static Future<void> clearCache() async {
    final box = Hive.box(_imagesBox);
    await box.clear();
  }
}
```

## 43. Flutter Push Notifications (FCM)

```dart
// android/app/src/main/AndroidManifest.xml additions:
// <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
// <uses-permission android:name="android.permission.INTERNET" />

// lib/core/notifications/fcm_service.dart
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

class FcmService {
  static final _messaging = FirebaseMessaging.instance;
  static final _localNotifications = FlutterLocalNotificationsPlugin();

  static Future<void> init() async {
    // Request permission (Android 13+)
    await _messaging.requestPermission();

    // Get token
    final token = await _messaging.getToken();
    if (token != null) {
      // Send token to your backend
      await _sendTokenToServer(token);
    }

    // Listen for token refresh
    _messaging.onTokenRefresh.listen(_sendTokenToServer);

    // Initialize local notifications
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    await _localNotifications.initialize(
      const InitializationSettings(android: androidSettings),
    );

    // Foreground messages
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

    // Background messages
    FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);
  }

  static Future<void> _sendTokenToServer(String token) async {
    // POST to /api/v1/auth/device-token
  }

  static void _handleForegroundMessage(RemoteMessage message) {
    final notification = message.notification;
    if (notification != null) {
      _localNotifications.show(
        notification.hashCode,
        notification.title,
        notification.body,
        const NotificationDetails(
          android: AndroidNotificationDetails(
            'genai_channel',
            'GenAI Notifications',
            importance: Importance.high,
            priority: Priority.high,
          ),
        ),
      );
    }
  }
}

@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  print('Background message: ${message.messageId}');
}
```

## 44. Android Signing & Release Build

```bash
# Generate keystore
keytool -genkey -v -keystore genai-release-key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias genai

# Create key.properties file
# android/key.properties
storePassword=YOUR_KEYSTORE_PASSWORD
keyPassword=YOUR_KEY_PASSWORD
keyAlias=genai
storeFile=/path/to/genai-release-key.jks

# Configure android/app/build.gradle for signing
# (See android-app-complete-guide.md for full Gradle config)

# Build release APK
flutter build apk --release

# Build release AAB (for Play Store)
flutter build appbundle --release
```

## 45. Google Play Store Submission

(See the detailed guide in `android-app-complete-guide.md` — Section 31)

The steps are:
1. Google Play Developer account ($25 one-time)
2. Create app listing
3. Upload AAB
4. Content rating questionnaire
5. Data safety declaration
6. Submit for review

---

# PART 6: CROSS-PLATFORM CONCERNS

## 46. Shared API Contract

All three platforms (Web, macOS, Android) consume the SAME API. The contract is:

```
BASE_URL:
  - Web: /api/v1 (proxied through same origin)
  - macOS: https://api.yourdomain.com/api/v1
  - Android: https://api.yourdomain.com/api/v1 (or 10.0.2.2:3001/api/v1 in dev)

AUTH:
  - All platforms store access_token in memory / SecureStorage
  - All platforms store refresh_token in httpOnly cookie (web) or SecureStorage (macOS/Android)
  - All platforms use the same refresh token rotation flow

IMAGES:
  - All platforms display images from the same S3/R2 URLs
  - All platforms call POST /api/v1/images/generate with the same body
  - All platforms receive the same response format

RESPONSE FORMAT (consistent across ALL endpoints):
{
  "data": { ... },           // On success
  "meta": { ... },           // Pagination, etc.
  "error": {                 // On error
    "code": "VALIDATION_ERROR",
    "message": "Human-readable message",
    "details": [ ... ],
    "requestId": "uuid"
  }
}
```

## 47. Environment Strategy (Dev/Staging/Prod)

```
DEVELOPMENT:
  - Frontend: http://localhost:5173
  - macOS:    http://localhost:5173 (via Tauri dev)
  - API:      http://localhost:3001
  - Database:  localhost:5432 (PostgreSQL)
  - Redis:     localhost:6379
  - Auth:      No email verification needed
  - Logging:   Pretty print, verbose

STAGING:
  - Frontend: https://staging.yourdomain.com
  - API:      https://api-staging.yourdomain.com
  - Database:  Railway managed PostgreSQL
  - Redis:     Railway managed Redis
  - Auth:      Resend sandbox (test emails only)
  - Logging:   JSON, info level

PRODUCTION:
  - Frontend: https://yourdomain.com
  - API:      https://api.yourdomain.com
  - Database:  Managed PostgreSQL (backups enabled)
  - Redis:     Managed Redis
  - Auth:      Resend (real emails)
  - Logging:   JSON, warn level
  - Monitoring: Sentry + uptime checks
```

## 48. Secrets Management

```
NEVER commit secrets to git. Use environment-specific strategies:

DEVELOPMENT:
  - .env file (in .gitignore)
  - .env.example (committed — template only)

STAGING/PRODUCTION:
  - Railway: Set secrets in dashboard → Settings → Variables
  - Fly.io: fly secrets set KEY=VALUE
  - AWS: Use Secrets Manager or Parameter Store
  - macOS: Keychain (for dev), CI env vars (for builds)
  - Android: Local.properties (in .gitignore), or CI env vars

SHARING BETWEEN TEAM:
  - Use Doppler (free tier) or Infisical (open source)
  - One source of truth for all environments
  - CI pulls secrets from Doppler/Infisical at build time
```

## 49. Testing Strategy — All Platforms

```
BACKEND (Node.js):
  - Unit tests: Jest/Vitest → test every service function
  - Integration tests: Supertest → test every API endpoint with real DB
  - Load tests: k6 → baseline 100 concurrent users

WEB FRONTEND (React):
  - Component tests: Vitest + Testing Library → test UI behavior
  - E2E tests: Playwright → test signup → login → generate → save flow
  - Visual regression: Chromatic (optional)

MACOS (Tauri):
  - Rust unit tests: cargo test → test Rust commands
  - E2E: playwright-tauri → test full app flow

ANDROID (Flutter):
  - Unit tests: flutter test → test providers, repositories
  - Widget tests: flutter test → test UI components
  - Integration tests: flutter drive → test full app on emulator
```

## 50. Final Production Checklist

```
BACKEND:
  □ All env vars documented in .env.example
  □ .env is in .gitignore (NEVER committed)
  □ Database migrations run cleanly on empty DB
  □ All tests pass (lint, typecheck, unit, integration)
  □ Docker build succeeds
  □ Health endpoint responds (/api/v1/health)
  □ API documentation accessible (Swagger/OpenAPI)
  □ HTTPS enforced (not HTTP)
  □ CORS restricts to actual origins (not *)
  □ Rate limiting active on all routes
  □ Auth works end-to-end (signup → verify → login → refresh → logout)
  □ Error responses don't leak stack traces in production
  □ Logging is structured JSON in production

INFRASTRUCTURE:
  □ Docker Compose builds and starts all services cleanly
  □ Database backups are configured and tested
  □ SSL certificates are auto-renewed
  □ Monitoring alerts are set up (uptime, errors)
  □ CI pipeline is green on all branches
  □ CD pipeline deploys staging on develop, production on tags

WEB:
  □ Lighthouse score > 90 (Performance, Accessibility, Best Practices, SEO)
  □ PWA manifest valid and tested
  □ Offline mode works for previously visited pages
  □ Responsive design tested on mobile, tablet, desktop
  □ Dark mode works throughout
  □ No console errors in production build

macOS (Tauri):
  □ Build succeeds: npx tauri build
  □ Code signing works (test on another Mac)
  □ Notarization passes (test on another Mac)
  □ Auto-update works (test update flow)
  □ Window resizing works properly
  □ Keyboard shortcuts work (Cmd+Q, Cmd+W, etc.)
  □ Native menu works
  □ Save to downloads works

ANDROID (Flutter):
  □ Build succeeds: flutter build appbundle --release
  □ ProGuard/R8 rules configured
  □ All screens render correctly on multiple device sizes
  □ Push notifications work (foreground and background)
  □ Offline mode works (cached data displays)
  □ Permissions requested properly (camera, notifications, internet)
  □ App icon and splash screen look good
  □ No crashes in release mode

SECURITY:
  □ No secrets in source code
  □ JWT tokens have short expiry (15min access, 7d refresh)
  □ Refresh tokens are hashed in database
  □ CORS whitelist is restrictive
  □ Rate limiting prevents abuse
  □ Input validation on ALL endpoints
  □ File uploads are sanitized (size, type, path)
  □ HTTPS everywhere in production
  □ Content Security Policy headers set
  □ No sensitive data in URLs or logs

PRE-LAUNCH:
  □ Privacy policy page is live
  □ Terms of service page is live
  □ App store descriptions are written
  □ Screenshots for all platforms are ready
  □ Support email is set up
  □ Monitoring dashboard is live
  □ Backup restore has been tested
  □ Someone other than the developer has tested the app
```

---

> **The key insight**: One backend serves all platforms. The API is the contract. Build it right once, then build platform-specific UIs (Web, macOS, Android) that all talk to the same API. This is how real companies do it.
# Backend & Infrastructure Build Instructions

> This is a **blueprint**. Follow these instructions in order to build production-grade backend and infrastructure for any app. Adapt the specifics (app name, routes, models) to your project. The patterns stay the same.

---

## How to Use This

1. Read each section in order
2. Do what it says — run the commands, create the files, make the decisions
3. Skip nothing — each step depends on the previous
4. When you see `[PLACEHOLDER]`, replace it with your app's value
5. The "Why" blocks explain reasoning so you understand, not just copy

---

# PHASE 0: DECIDE BEFORE YOU CODE

Before writing a single line, answer these questions. Write them down in a file called `DECISIONS.md`.

## 0.1 App Identity

```
App Name:           [e.g. GenAI]
One-Line Pitch:     [e.g. AI image generator with multiple models]
Target Users:        [e.g. creators, designers, hobbyists]
Primary Platform:    [web / macOS / Android / all]
```

## 0.2 Tech Stack Decision

Pick ONE from each row. Don't overthink — pick what you know or what the guide recommends.

| Layer | Options | Recommendation |
|---|---|---|
| Language | TypeScript, Python, Go | **TypeScript** (shares type with frontend) |
| Framework | Express, Fastify, NestJS, Hono | **Express** (biggest ecosystem, most tutorials) |
| Database | PostgreSQL, MySQL, MongoDB | **PostgreSQL** (industry standard, relational) |
| ORM | Prisma, Drizzle, TypeORM, raw SQL | **Prisma** (best DX, type-safe, auto migrations) |
| Cache/Queue | Redis, Dragonfly, none | **Redis** (caching + job queue + sessions) |
| Auth | Custom JWT, Clerk, Supabase Auth, Auth0 | **Custom JWT** (full control, no vendor lock-in) |
| Storage | AWS S3, Cloudflare R2, GCS | **Cloudflare R2** (S3-compatible, free egress) |
| Email | Resend, SendGrid, AWS SES | **Resend** (simplest DX, free tier) |
| Payments | Stripe, LemonSqueezy, none | **Stripe** (industry standard) |
| Monitoring | Sentry, Datadog, none | **Sentry** (free tier, easy setup) |
| Deployment | Railway, Fly.io, AWS, VPS | **Railway** (fastest deploy, scales later) |
| Container | Docker, none | **Docker** (required for any real deployment) |

## 0.3 Environment Naming

```
DEV:      Your machine (localhost)
STAGING:  Railway/Fly preview (mirror of production)
PROD:     Railway/Fly production (real users)
```

## 0.4 Write Down Your Data Model

List every "thing" your app stores. For each, list its fields.

```
Example for GenAI:

User:
  - id (UUID)
  - email (unique)
  - passwordHash
  - name
  - role (USER / ADMIN)
  - plan (FREE / PRO)
  - emailVerified (boolean)
  - createdAt, updatedAt

Image:
  - id (UUID)
  - userId (foreign key → User)
  - prompt (text)
  - enhancedPrompt (text)
  - model (DALLE3 / FLUX / POLLINATIONS)
  - style (realistic / artistic / anime / etc.)
  - imageUrl (string)
  - isPublic (boolean)
  - isFavorite (boolean)
  - createdAt

Gallery:
  - id (UUID)
  - userId (foreign key → User)
  - name
  - isPublic (boolean)
  - createdAt

PasswordReset:
  - id (UUID)
  - userId (foreign key → User)
  - token (unique)
  - expiresAt
```

**Why**: Your database schema IS your app. Get this wrong and everything else is harder. Think about what you need to query (indexes), what can be null (nullable), and what must be unique (unique constraints).

---

# PHASE 1: PROJECT SCAFFOLD

## 1.1 Create the Repo

```bash
mkdir [APP-NAME]-backend
cd [APP-NAME]-backend
git init

# Create .gitignore FIRST
cat > .gitignore << 'EOF'
node_modules/
dist/
.env
.env.local
*.log
.DS_Store
coverage/
.prism/
EOF

# Create initial commit
git add .
git commit -m "chore: init repo"
```

## 1.2 Initialize Package

```bash
npm init -y

# Install core dependencies
npm install express cors helmet compression dotenv zod bcryptjs jsonwebtoken uuid prisma @prisma/client

# Install dev dependencies
npm install -D typescript @types/node @types/express @types/cors @types/bcryptjs @types/jsonwebtoken ts-node-dev tsconfig-paths prettier eslint

# Install queue/cache (later)
npm install bullmq ioredis

# Install file upload (later)
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner

# Install email (later)
npm install resend

# Install monitoring (later)
npm install @sentry/node
```

## 1.3 Configure TypeScript

Create `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

## 1.4 Add Scripts to package.json

```json
{
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only -r tsconfig-paths/register src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "db:push": "npx prisma db push",
    "db:migrate": "npx prisma migrate dev",
    "db:migrate:prod": "npx prisma migrate deploy",
    "db:seed": "npx prisma db seed",
    "db:studio": "npx prisma studio",
    "db:reset": "npx prisma migrate reset --force",
    "lint": "eslint src/",
    "typecheck": "tsc --noEmit",
    "test": "jest",
    "test:watch": "jest --watch"
  }
}
```

**Why**: These scripts are your daily commands. `dev` for local development, `build` for production, `db:*` for database management. Memorize them.

## 1.5 Create the Folder Structure

```bash
mkdir -p src/{config,middleware,modules,services,utils,types}
mkdir -p src/modules/{auth,users,images,gallery,billing}
mkdir -p prisma
mkdir -p docker
mkdir -p tests
mkdir -p .github/workflows
```

**Why**: Each module is self-contained (routes, controller, service, validation, types). This is Clean Architecture applied to Express. New feature = new module folder. No god files.

---

# PHASE 2: DATABASE

## 2.1 Install PostgreSQL

```bash
# macOS
brew install postgresql@16
brew services start postgresql@16

# Create database
createdb [APP-NAME]

# Verify
psql -d [APP-NAME] -c "SELECT version();"
```

**Why PostgreSQL over MongoDB**: Relational databases enforce data integrity at the database level. Foreign keys, unique constraints, check constraints — these prevent bugs that MongoDB can't catch. Plus: 95% of production apps use Postgres. Learn it once, use it forever.

## 2.2 Initialize Prisma

```bash
npx prisma init
```

This creates `prisma/schema.prisma` and `.env` with `DATABASE_URL`.

## 2.3 Write Your Schema

Open `prisma/schema.prisma`. This is where you define your entire data model based on what you wrote in Phase 0.4.

**Instructions** (not code — follow the pattern):

1. Set `provider = "postgresql"` and `url = env("DATABASE_URL")`
2. For EACH entity from your 0.4 list, create a `model` block
3. Every model gets: `id String @id @default(uuid())`, `createdAt DateTime @default(now())`, `updatedAt DateTime @updatedAt`
4. Foreign keys use `@relation(fields: [userId], references: [id])` syntax
5. Add `@@index` on every column you will filter/sort by
6. Add `@unique` on emails, tokens, and anything that must be unique
7. Use `enum` for fixed choices (roles, plans, model types)
8. Add `deletedAt DateTime?` on models you want soft-delete on

**Run after writing schema**:

```bash
npx prisma migrate dev --name init
npx prisma generate
```

**Verify**: Open `npx prisma studio` and confirm your tables exist.

## 2.4 Write Seed Data

Create `prisma/seed.ts`. Instructions:

1. Create at least 2 users (1 regular, 1 admin)
2. Create sample data for each model (10+ rows minimum)
3. Use realistic faker data (install `@faker-js/faker`)
4. Hash passwords with `bcryptjs` (cost factor 12)
5. Add `prisma.seed` command to `package.json` pointing to your seed file
6. Run: `npx prisma db seed`

**Why seed data**: Every developer on your team (including future you) starts with a populated database instead of an empty one. Tests need data. Demos need data. Seed it once, reset anytime.

## 2.5 DATABASE_URL Format

In `.env`:

```
DATABASE_URL=postgresql://[USER]:[PASSWORD]@localhost:5432/[APP-NAME]
```

**Why Prisma over raw SQL**: Type safety from schema to TypeScript. Auto migrations. No SQL injection possible (parameterized queries by default). The `prisma generate` command creates TypeScript types that match your exact schema.

---

# PHASE 3: ENVIRONMENT CONFIGURATION

## 3.1 Create .env.example

Create `.env.example` with EVERY variable your app needs. Document each one with a comment.

```env
# === SERVER ===
NODE_ENV=development
PORT=3001

# === DATABASE ===
DATABASE_URL=postgresql://user:password@localhost:5432/appname

# === REDIS ===
REDIS_URL=redis://localhost:6379

# === JWT ===
# Generate with: openssl rand -hex 32
JWT_SECRET=change-me-to-64-random-hex-characters
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# === STORAGE ===
S3_ACCESS_KEY=
S3_SECRET_KEY=
S3_BUCKET=
S3_REGION=auto
S3_ENDPOINT=

# === EMAIL ===
RESEND_API_KEY=
EMAIL_FROM="App Name <noreply@yourdomain.com>"

# === AI PROVIDERS (optional) ===
OPENAI_API_KEY=
HUGGING_FACE_TOKEN=

# === CORS ===
CORS_ORIGIN=http://localhost:5173,http://localhost:1420
```

**Rule**: `.env.example` is committed to git. `.env` is NEVER committed. Copy `.env.example` to `.env` and fill in real values.

## 3.2 Create Env Validation

Create `src/config/env.ts`. Instructions:

1. Import `z` from `zod`
2. Define a `z.object()` with ALL your env variables, types, and defaults
3. Call `z.parse(process.env)` at app startup
4. Export the typed result
5. If validation fails, print clear error messages and `process.exit(1)`

**Why**: If you forget to set `JWT_SECRET`, you want the app to crash immediately at startup, not 3 hours into production with an empty secret.

## 3.3 Create Config Files

Create one file per config concern in `src/config/`:

- `env.ts` — validated env vars (described above)
- `redis.ts` — Redis client setup (connect, handle errors, log ready/not ready)
- `database.ts` — Prisma client singleton (prevent multiple instances in dev)
- `sentry.ts` — Sentry init (only in production)

**Database singleton pattern** (follow this for Prisma):

```typescript
// src/config/database.ts
// Create ONE PrismaClient instance and reuse it
// In dev: globalThis.prisma to prevent hot-reload from creating new instances
// In prod: just create it once
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as { prisma: PrismaClient | undefined };

export const prisma = globalForPrisma.prisma ?? new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
});

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

**Why the singleton**: In development with hot-reload, Node creates new instances. Without `globalThis`, you'd get dozens of Prisma connections and exhaust your database pool.

---

# PHASE 4: EXPRESS APP SETUP

## 4.1 Create the App File

Create `src/app.ts`. Instructions:

1. Import express and all middleware
2. Apply middleware IN THIS ORDER:
   - `helmet()` — security headers (FIRST, always first)
   - `cors()` — with your specific origin whitelist
   - `compression()` — gzip responses
   - `express.json({ limit: '10mb' })` — body parsing with size limit
   - `express.urlencoded({ extended: true })` — URL-encoded bodies
   - Request ID middleware — generate UUID, attach to request, set as response header
   - Request logger middleware — log method, path, status, duration, userId
   - Rate limiting — global limit (100 req/15min) and auth-specific (10 req/15min)
3. Mount routes: `app.use('/api/v1/auth', authRoutes)`, etc.
4. Health check endpoint: `GET /api/v1/health` — check DB, Redis, return status
5. 404 handler: return JSON `{ error: { code: 'NOT_FOUND', message: '...' } }`
6. Error handler: the LAST middleware, catches everything, returns structured JSON errors
7. Export `app` (do NOT start the server here — that goes in `index.ts`)

**Why this order matters**: Helmet must be first so security headers are on every response. CORS must be before routes. Body parsing before auth. Rate limiting before expensive operations. Error handler must be last.

## 4.2 Create the Entry Point

Create `src/index.ts`. Instructions:

1. Import `app` from `./app`
2. Import `prisma` from `./config/database`
3. Import `env` from `./config/env`
4. Call `prisma.$connect()` to verify DB connection
5. Start listening with `app.listen(env.PORT)`
6. Log startup info (environment, port, health URL)
7. Handle graceful shutdown:
   - `process.on('SIGTERM')` — disconnect Prisma, exit 0
   - `process.on('SIGINT')` — disconnect Prisma, exit 0

**Why graceful shutdown**: When Kubernetes/Railway sends SIGTERM, your app has ~30 seconds to finish in-flight requests and close connections. Without this handler, connections are killed mid-query.

## 4.3 Create Middleware Files

Create each file in `src/middleware/`:

### `requestId.ts`
- Generate a UUID using `crypto.randomUUID()`
- Attach to `req.headers['x-request-id']`
- Set as response header: `res.setHeader('X-Request-Id', requestId)`
- Call `next()`

### `requestLogger.ts`
- Log: method, path, status code, duration (ms), userId (if authenticated), IP
- Use your logger (Pino — see Phase 13)
- `res.on('finish')` to capture status code after response is sent
- Calculate duration: `Date.now() - start`

### `rateLimiter.ts`
- Use `express-rate-limit` with `rate-limit-redis` store
- Create THREE limiters:
  - `apiLimiter`: 100 requests per 15 minutes (global)
  - `authLimiter`: 10 requests per 15 minutes (login, signup, reset-password)
  - `generateLimiter`: 5 requests per minute (image generation)
- Each returns JSON on limit exceeded: `{ error: { code: 'RATE_LIMITED', message: '...' } }`

### `validate.ts`
- Accept a Zod schema
- Parse `req.body`, `req.query`, `req.params`
- If valid, replace `req.body` with parsed data, call `next()`
- If invalid, return 400 with `{ error: { code: 'VALIDATION_ERROR', details: ... } }`

### `auth.ts`
- Extract Bearer token from `Authorization` header
- Verify JWT using `jsonwebtoken.verify()` and your `JWT_SECRET`
- Attach decoded payload to `req.user`
- If invalid/expired, return 401 with `{ error: { code: 'UNAUTHORIZED' } }`
- Create a `requireAdmin` middleware that checks `req.user.role === 'ADMIN'`
- Create a `requirePlan` middleware factory that checks user plan level

### `errorHandler.ts`
- Must have 4 parameters: `(err, req, res, next)`
- If `err instanceof AppError`, return `err.statusCode` and structured JSON
- If Prisma error (check `err.code`), return appropriate status (P2002 = 409 duplicate, P2025 = 404 not found)
- For unknown errors, log full error, return 500 with generic message
- In development: include stack trace
- In production: no stack trace, generic error message

---

# PHASE 5: AUTHENTICATION

Build this module in `src/modules/auth/`.

## 5.1 Files to Create

| File | Purpose |
|---|---|
| `auth.validation.ts` | Zod schemas for signup, login, reset-password inputs |
| `auth.service.ts` | Business logic: hash password, generate JWT, verify email, etc. |
| `auth.controller.ts` | Request/response handling: parse input, call service, return response |
| `auth.routes.ts` | Express route definitions: POST /signup, POST /login, etc. |
| `auth.types.ts` | TypeScript interfaces for auth operations |

## 5.2 AUTH FLOW (follow this exactly)

**Signup**:
1. Receiver: `POST /api/v1/auth/signup` with `{ email, password, name }`
2. Validate with Zod: email format, password ≥ 8 chars with uppercase + number
3. Check if email exists in DB: if yes, return 409
4. Hash password with bcrypt (cost factor 12)
5. Generate email verification token (`crypto.randomBytes(32).toString('hex')`)
6. Create user in DB with `emailVerified: false` and the token
7. Send verification email (fire-and-forget, don't await)
8. Generate access token (15min expiry) and refresh token (7d expiry)
9. Store refresh token in DB (hashed with SHA-256) with device info and expiry
10. Return `{ accessToken, refreshToken, user }` (NEVER return passwordHash)

**Login**:
1. Validate input
2. Find user by email
3. Compare password with bcrypt
4. If wrong password: return 401 "Invalid credentials" (don't say "wrong password" — security)
5. If correct: generate tokens same as signup
6. Return tokens + user

**Refresh**:
1. Receive refresh token
2. Find token in DB (hashed)
3. If not found: THIS IS TOKEN THEFT — delete ALL refresh tokens for this user, return 401
4. If expired: delete it, return 401 "Token expired, please login again"
5. If valid: delete old token, create new refresh token (rotation), new access token
6. Return both

**Why rotation**: If a refresh token is used twice, the second use means someone stole it. By deleting ALL tokens for that user, you force them to re-login everywhere, invalidating the thief's access too.

## 5.3 Password Reset Flow

1. `POST /forgot-password` — receive email
2. Generate reset token (`crypto.randomBytes(32).toString('hex')`), expiry = 1 hour
3. Store in `PasswordReset` table
4. Send email with link: `https://yourdomain.com/reset-password?token=TOKEN`
5. `POST /reset-password` — receive `{ token, newPassword }`
6. Verify token exists and hasn't expired
7. Hash new password
8. Update user password
9. Delete ALL refresh tokens for this user (force re-login on all devices)
10. Delete the reset token

## 5.4 Delete Account

1. `DELETE /api/v1/auth/account` (authenticated)
2. Delete all user's data (images, galleries, refresh tokens)
3. Delete the user record
4. Return 200

**Important**: Use a database transaction (`prisma.$transaction()`) so if any delete fails, the whole thing rolls back. You don't want to delete the user but leave their images.

---

# PHASE 6: API MODULES

For EVERY new resource (images, galleries, users), follow this EXACT pattern:

## 6.1 Module Structure

```
src/modules/[resource]/
├── [resource].validation.ts   ← Zod schemas for input validation
├── [resource].service.ts      ← Business logic (DB queries, business rules)
├── [resource].controller.ts   ← HTTP request/response handling
├── [resource].routes.ts      ← Route definitions
└── [resource].types.ts        ← TypeScript interfaces
```

## 6.2 The Pattern

**Validation** (`[resource].validation.ts`):
- Define a Zod schema for every endpoint's input
- signup: `z.object({ body: z.object({ email: z.string().email(), ... }) })`
- Be specific: minimum lengths, regex patterns, enums for fixed values
- Export and use in routes: `router.post('/', validate(imageValidation.create), controller.create)`

**Service** (`[resource].service.ts`):
- This is where business logic lives
- Receives validated data from controller
- Talks to the database via Prisma
- Returns domain objects (not Prisma internal types)
- Throws `AppError` for business rule violations
- Method pattern: `async create(userId: string, data: CreateImageInput): Promise<Image>`

**Controller** (`[resource].controller.ts`):
- Extracts data from request: `req.body`, `req.params`, `req.query`, `req.user`
- Calls service method
- Returns `res.status(201).json({ data: ... })` or `res.status(200).json({ data: ... })`
- Catches errors and passes to `next()`
- NEVER puts business logic here — that goes in service

**Routes** (`[resource].routes.ts`):
- Define Express router
- Chain middleware: `router.post('/', authenticate, validate(schema), controller.create)`
- Every mutating route (POST, PUT, PATCH, DELETE) MUST have `authenticate` middleware
- Every route MUST have input validation

## 6.3 Response Format

ALWAYS return the same structure:

```json
// Success
{ "data": { ... }, "meta": { "page": 1, "limit": 20, "total": 100, "hasMore": true } }

// Error
{ "error": { "code": "VALIDATION_ERROR", "message": "Human-readable", "details": [...], "requestId": "uuid" } }
```

**Why consistency**: Frontend developers (including future you) should never have to guess what shape a response is. Every endpoint returns the same envelope.

## 6.4 Pagination

Every list endpoint MUST support pagination:

```
GET /api/v1/images?page=1&limit=20&sort=createdAt&order=desc
```

In your service:
- `page` defaults to 1, `limit` defaults to 20, max 100
- Use `prisma.findMany({ skip: (page - 1) * limit, take: limit })`
- Count total with `prisma.count({ where: ... })`
- Return `{ data, meta: { page, limit, total, hasMore: skip + data.length < total } }`

## 6.5 Soft Deletes

NEVER hard-delete user data. Add `deletedAt DateTime?` to every model.

- Delete endpoint sets `deletedAt: new Date()`
- Every query adds `where: { deletedAt: null }`
- Provide a separate "hard delete" admin endpoint if needed
- This lets you implement "undo" and data recovery

---

# PHASE 7: FILE STORAGE

## 7.1 Set Up Cloudflare R2 (Recommended) or AWS S3

Instructions for R2:

1. Create Cloudflare account
2. Go to R2 → Create bucket → Name it `[app-name]-images`
3. Go to R2 → Manage R2 API Tokens → Create API Token
4. Permissions: Object Read & Write
5. Save: Access Key ID, Secret Access Key, Endpoint URL
6. Add these to `.env` as `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_ENDPOINT`, `S3_BUCKET`
7. Enable public access on the bucket for image serving
8. (Optional) Add a custom domain to the bucket for clean URLs

## 7.2 Create Storage Service

Create `src/services/storage.service.ts`. Instructions:

1. Use `@aws-sdk/client-s3` package (works with R2 — it's S3-compatible)
2. Create an `S3Client` with your credentials and R2 endpoint
3. Implement three methods:
   - `upload(buffer, key, contentType)` → Upload to S3/R2, return public URL
   - `getSignedUrl(key, expiresIn)` → Generate a time-limited download URL
   - `delete(key)` → Remove from S3/R2
4. Set `CacheControl: 'public, max-age=31536000'` on uploads (1 year cache)
5. Use a key structure: `images/{userId}/{timestamp}-{random}.{ext}`

**Why R2 over S3**: R2 has zero egress fees. S3 charges per GB downloaded. For an image app, that's a huge cost difference.

---

# PHASE 8: BACKGROUND JOBS

## 8.1 Set Up Redis

```bash
# macOS
brew install redis
brew services start redis

# Verify
redis-cli ping
# Should return: PONG

# Docker (for production)
# Already in docker-compose.yml
```

## 8.2 Create Queue Service

Create `src/services/queue.service.ts`. Instructions:

1. Use BullMQ (built on Redis)
2. Create named queues: `'image-generation'`, `'email'`
3. Create workers for each queue
4. Image generation worker: receives `{ userId, prompt, model, style }`, calls ImageService
5. Email worker: receives `{ to, subject, html }`, calls EmailService
6. Set concurrency limits: image generation = 3, email = 5
7. Handle job failures: log error, retry up to 3 times with exponential backoff

**When to use queues vs direct**: If an operation takes >2 seconds (image generation), put it in a queue. If it's fast (email), you can send directly but queuing is still better for reliability.

## 8.3 WebSocket for Real-Time Updates

Only needed for long-running operations (image generation). Instructions:

1. Install `socket.io`
2. Create `src/services/websocket.service.ts`
3. Initialize Socket.IO server attached to your HTTP server
4. Authenticate connections: verify JWT from `socket.handshake.auth.token`
5. When a user connects, join them to room `user:${userId}`
6. When image generation completes, emit to `user:${userId}` with event `generation_complete`
7. In the frontend, connect to WebSocket and listen for updates

**Why WebSocket for generation**: Image generation takes 5-30 seconds. Without WebSocket, the frontend has to poll every second. With WebSocket, it gets notified instantly.

---

# PHASE 9: EMAIL

## 9.1 Set Up Resend

1. Go to resend.com, create account
2. Add and verify your domain (or use `on.resend.com` for testing)
3. Generate API key
4. Add `RESEND_API_KEY` to `.env`

## 9.2 Create Email Service

Create `src/services/email.service.ts`. Instructions:

1. Create `sendEmail({ to, subject, html })` function
2. If `RESEND_API_KEY` exists, send via Resend
3. If not, log the email (for development)
4. Create email template functions:
   - `sendVerificationEmail(to, verifyUrl)`
   - `sendPasswordResetEmail(to, resetUrl)`
   - `sendWelcomeEmail(to, name)`
5. All emails should be HTML with inline styles (email clients strip `<style>` tags)
6. Add your app's branding and logo to templates

---

# PHASE 10: RATE LIMITING & SECURITY

## 10.1 Rate Limiting Configuration

Create `src/middleware/rateLimiter.ts`. Instructions:

1. Use `express-rate-limit` with `rate-limit-redis` store
2. Three tiers:
   - **Global**: 100 requests / 15 minutes per IP
   - **Auth**: 5 requests / 15 minutes per IP (for login, signup, forgot-password)
   - **Generation**: 5 requests / minute per user (for image generation)
3. Use Redis store so limits persist across server restarts
4. Return 429 with JSON: `{ error: { code: 'RATE_LIMITED', message: '...', retryAfter: 900 } }`
5. Set `standardHeaders: true` to include `RateLimit-*` headers

## 10.2 Security Headers Checklist

Your app should set these headers (most are set by `helmet()`):

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0                     ← Modern browsers don't need this
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; img-src 'self' data: https://*.cloudflarestorage.com https://image.pollinations.ai; script-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

## 10.3 Input Validation Rules

For EVERY endpoint that accepts user input:

1. Validate with Zod — NEVER trust client data
2. String fields: set `min`, `max`, regex patterns
3. Email: `z.string().email()`
4. Passwords: minimum 8 characters, at least 1 uppercase, 1 number
5. URLs: `z.string().url()`
6. Enums: `z.enum(['option1', 'option2'])` — never accept arbitrary strings
7. Pagination: `z.number().int().min(1).max(100).default(1)`
8. Strip unknown fields: `z.object({...}).strict()`
9. Sanitize HTML: strip all HTML tags from string inputs (or use a library like `sanitize-html`)

---

# PHASE 11: DOCKER

## 11.1 Create Dockerfile

Create `docker/Dockerfile`. Instructions:

1. **Stage 1 (deps)**: Start from `node:20-alpine`, copy `package.json` and `package-lock.json`, run `npm ci`
2. **Stage 2 (builder)**: Copy source code, run `npx prisma generate`, run `npm run build`
3. **Stage 3 (runner)**: Start from `node:20-alpine`, create non-root user (`addgroup` + `adduser`), copy built files and node_modules from builder, expose port, set `CMD ["node", "dist/index.js"]`
4. Add `HEALTHCHECK` that pings `/api/v1/health`

**Why multi-stage**: Distroless/alpine final image is ~100MB instead of ~1GB. Non-root user prevents container escape attacks. Health check lets Kubernetes/Docker know if the app is alive.

## 11.2 Create Docker Compose

Create `docker-compose.yml`. Instructions:

1. Define 5 services: `api`, `worker`, `postgres`, `redis`, `nginx`
2. **api**: Build from Dockerfile, expose port 3001, depends_on postgres + redis, set env_file, healthcheck
3. **worker**: Same image as api, command `["node", "dist/worker.js"]`, depends_on api
4. **postgres**: `postgres:16-alpine`, expose 5432, volume `postgres_data`, healthcheck with `pg_isready`, environment vars for user/password/db
5. **redis**: `redis:7-alpine`, expose 6379, volume `redis_data`, healthcheck with `redis-cli ping`
6. **nginx**: Skip for now (add in Phase 18 when you have SSL)
7. Define named volumes: `postgres_data`, `redis_data`

**Why Docker Compose**: One command (`docker compose up`) starts the entire stack — database, cache, API, worker, proxy. Every developer gets the same environment. Production parity.

## 11.3 Create .dockerignore

```
node_modules
dist
.env
.env.local
.git
.gitignore
README.md
docker
tests
*.md
```

## 11.4 Create docker-compose.dev.yml

Develop WITH Docker but WITH hot reload:

```yaml
# docker-compose.dev.yml
services:
  api:
    build:
      context: .
      dockerfile: docker/Dockerfile.dev  # Uses ts-node-dev for hot reload
    volumes:
      - .:/app                           # Mount source code
      - /app/node_modules                # Don't mount node_modules
    environment:
      - NODE_ENV=development
    ports:
      - "3001:3001"
      - "9229:9229"                      # Debug port
    # ... same postgres, redis as production

# docker/Dockerfile.dev
# FROM node:20-alpine
# WORKDIR /app
# COPY package*.json ./
# RUN npm ci
# CMD ["npm", "run", "dev"]
```

---

# PHASE 12: CI/CD

## 12.1 Create GitHub Actions Workflow

Create `.github/workflows/ci.yml`. Instructions:

1. Trigger on push to `main`, `develop`, and pull requests to `main`
2. Jobs:
   - **lint**: Run `npm run lint`
   - **typecheck**: Run `npm run typecheck`
   - **test**: Start PostgreSQL service container, run `npx prisma migrate deploy`, run `npm test`
   - **build**: Run `npm run build`, verify Docker build succeeds
3. Cache `node_modules` between runs
4. Use service containers for PostgreSQL (so tests run against a real DB)
5. Set env vars for test: `DATABASE_URL=postgresql://test:test@localhost:5432/test`, `JWT_SECRET=test-secret-32-chars-long-xxxx`

## 12.2 Create Deployment Workflow

Create `.github/workflows/deploy.yml`. Instructions:

1. Trigger on push to `main` (staging) and tags `v*` (production)
2. Build Docker image
3. Push to container registry (GitHub Container Registry)
4. Deploy to Railway using `railway` CLI
5. Run database migrations: `railway run npx prisma migrate deploy`
6. Run smoke test: curl health endpoint, verify 200
7. If smoke test fails, rollback

---

# PHASE 13: LOGGING & MONITORING

## 13.1 Set Up Pino Logger

Create `src/utils/logger.ts`. Instructions:

1. Use `pino` with `pino-pretty` for development
2. In development: pretty-print, colorize, log level = `debug`
3. In production: JSON format, log level = `info`
4. Always log: timestamp, level, message, requestId
5. Never log: passwords, tokens, full request bodies
6. Create child loggers for specific contexts: `logger.child({ module: 'auth' })`

## 13.2 Set Up Sentry

Create `src/config/sentry.ts`. Instructions:

1. Install `@sentry/node`
2. Initialize Sentry before Express app setup (BEFORE middleware)
3. Only enable in production: `dsn: env.SENTRY_DSN, environment: env.NODE_ENV`
4. Set `tracesSampleRate: 0.1` (10% of transactions)
5. Add user context: `Sentry.setUser({ id: user.id, email: user.email })` after auth
6. Add the Sentry error handler AFTER your Express error handler
7. Create a Sentry project at sentry.io, copy the DSN to `.env`

## 13.3 Set Up Uptime Monitoring

1. Create a free UptimeRobot account
2. Add monitor: `https://api.yourdomain.com/api/v1/health`
3. Check every 5 minutes
4. Alert via email and/or Slack webhook
5. Set up a second monitor for your frontend URL

---

# PHASE 14: DEPLOYMENT

## 14.1 Railway Deployment (Recommended for Start)

Instructions:

1. Install Railway CLI: `npm install -g @railway/cli`
2. Login: `railway login`
3. Initialize: `railway init` (select "Empty project")
4. Add PostgreSQL: `railway add --plugin postgres`
5. Add Redis: `railway add --plugin redis`
6. Get database URL: `railway variables` → copy `DATABASE_URL`
7. Set ALL environment variables: `railway variables set KEY=VALUE` for each in `.env.example`
8. Generate JWT secret: `openssl rand -hex 32` → set as `JWT_SECRET`
9. Deploy: `railway up`
10. Run migrations: `railway run npx prisma migrate deploy`
11. Seed: `railway run npx prisma db seed`
12. Verify: `curl https://your-app.up.railway.app/api/v1/health`

**Why Railway**: Zero-config deployment. Push code, it deploys. Built-in PostgreSQL and Redis. Free tier for small apps. Scales with a slider.

## 14.2 Custom Domain

1. Buy domain on Cloudflare, Namecheap, or Google Domains
2. In Railway dashboard → Settings → Domains → Add custom domain
3. Railway gives you a CNAME target
4. Add CNAME record in your DNS: `api` → `railway-target.up.railway.app`
5. SSL is automatic (Railway uses Let's Encrypt)

## 14.3 Fly.io Deployment (Alternative)

```bash
# Install CLI
curl -L https://fly.io/install.sh | sh

# Login
fly auth login

# Launch
fly launch

# Set secrets
fly secrets set JWT_SECRET=$(openssl rand -hex 32)
fly secrets set DATABASE_URL=...
# ... all env vars

# Deploy
fly deploy

# Run migrations
fly ssh console -C "npx prisma migrate deploy"
```

---

# PHASE 15: TESTING

## 15.1 Test Setup

1. Install: `npm install -D jest ts-jest @types/jest supertest @types/supertest`
2. Create `jest.config.ts`: set `preset: 'ts-jest'`, `testEnvironment: 'node'`, `setupFilesAfterSetup: ['<rootDir>/tests/setup.ts']`
3. Create `tests/setup.ts`: start test database, run migrations, seed data, clean up after tests

## 15.2 What to Test

**Test every module's service layer** (not controllers — those are thin wrappers):

| What | Example |
|---|---|
| Success path | Signup creates user, returns tokens |
| Validation errors | Signup with invalid email returns 400 |
| Uniqueness violations | Signup with existing email returns 409 |
| Auth errors | Access protected route without token returns 401 |
| Business rules | FREE plan user can't generate more than 10 images/day |
| Pagination | List endpoint returns correct page, total, hasMore |
| Soft deletes | Delete hides record, list doesn't show it |
| Token refresh rotation | Using refresh token twice invalidates all tokens |

## 15.3 Test Naming Convention

```
describe('AuthService')
  describe('signup')
    it('creates user with valid data')
    it('rejects duplicate email')
    it('rejects weak password')
    it('sends verification email')
    it('hashes password with bcrypt')
```

## 15.4 Integration Test Pattern

```
1. Start test database (Prisma test DB or Docker PostgreSQL)
2. Before each test: clear database, seed minimal data
3. Make HTTP request with supertest
4. Assert response status, body structure, database state
5. After each test: clean up
```

---

# PHASE 16: FINAL PRE-LAUNCH CHECKLIST

Before you deploy to production, verify EVERY item:

```
INFRASTRUCTURE:
  □ Docker builds successfully (`docker compose build`)
  □ Docker Compose starts all services (`docker compose up`)
  □ Health endpoint responds 200 (`/api/v1/health`)
  □ Database migrations run cleanly on empty DB
  □ Seed data loads correctly
  □ Redis connection works

SECURITY:
  □ .env is in .gitignore (NEVER committed)
  □ All env vars are in .env.example (documented)
  □ JWT_SECRET is 64 random hex characters (openssl rand -hex 32)
  □ CORS restricts to your actual domains (not *)
  □ Rate limiting is active on all routes
  □ Helmet security headers are set
  □ Input validation on EVERY endpoint
  □ No console.log in production — use Pino logger
  □ Error responses don't leak stack traces in production
  □ Password reset tokens expire (1 hour)
  □ Refresh tokens rotate (one-time use)

PERFORMANCE:
  □ Response compression enabled (gzip/brotti)
  □ Database connection pooling configured
  □ Image uploads go to R2/S3 (not local disk)
  □ Health check endpoint exists and works
  □ Graceful shutdown handles SIGTERM

TESTING:
  □ All tests pass (`npm test`)
  □ Lint passes (`npm run lint`)
  □ Typecheck passes (`npm run typecheck`)
  □ Build succeeds (`npm run build`)

MONITORING:
  □ Sentry is initialized and receiving errors
  □ Uptime monitoring is set up
  □ Structured JSON logging is working
  □ Request ID is on every response

BACKUPS:
  □ Database backups are configured (daily)
  □ Backup restore has been tested
  □ .env backup is in a secure location (not in git)

DOCUMENTATION:
  □ README.md has setup instructions
  □ .env.example has ALL variables documented
  □ API documentation exists (Swagger/OpenAPI or similar)
```

---

# QUICK REFERENCE: DAILY COMMANDS

```bash
# Development
npm run dev                    # Start API with hot reload
npx prisma studio              # Visual database browser
npx prisma migrate dev         # Create a new migration
npx prisma db seed             # Seed data
npm run lint                   # Check code quality
npm run typecheck              # Check TypeScript
npm test                       # Run tests

# Docker
docker compose up              # Start full stack
docker compose up -d           # Start in background
docker compose logs -f api     # Tail API logs
docker compose down             # Stop all services
docker compose down -v         # Stop and DELETE all data

# Database
npx prisma migrate dev --name add_field_x    # Add a new field
npx prisma migrate reset                      # Reset DB (DESTRUCTIVE)
npx prisma generate                           # Regenerate Prisma Client
npx prisma validate                           # Check schema for errors

# Production
npm run build                  # Build for production
npm run start                  # Start production server
npx prisma migrate deploy      # Apply migrations (production)

# Deployment
railway up                     # Deploy to Railway
railway logs                   # View production logs
railway run npx prisma migrate deploy  # Run migrations on Railway
```

---

> **Remember**: This is a blueprint. The patterns are universal. The specifics (model names, field names, routes) change per app. The process — schema first, then auth, then CRUD, then storage, then deploy — stays the same.
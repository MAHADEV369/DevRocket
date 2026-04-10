# Build Instructions for All 27 Modules

> Follow each section in order. Each tells you exactly what to create, where to put it, what goes inside it, and why. When you see `[APP-NAME]`, replace with your app name.

---

# 1. Backend Template Code (Express + TypeScript + Prisma + Auth)

## What to Build

A production-ready Express API with TypeScript, Prisma ORM, and a complete authentication system.

## Folder Structure to Create

```
backend/
├── src/
│   ├── index.ts                     ← Server entry point
│   ├── app.ts                       ← Express app setup
│   ├── config/
│   │   ├── database.ts             ← Prisma client singleton
│   │   └── redis.ts                ← Redis client singleton
│   ├── middleware/
│   │   ├── auth.ts                  ← JWT verification
│   │   ├── rateLimiter.ts          ← Rate limiting
│   │   ├── validate.ts             ← Zod validation
│   │   ├── requestId.ts            ← Unique request ID
│   │   ├── requestLogger.ts        ← HTTP request logging
│   │   ├── errorHandler.ts         ← Global error handler
│   │   └── cors.ts                 ← CORS configuration
│   ├── modules/
│   │   ├── auth/
│   │   │   ├── auth.routes.ts      ← POST /signup, /login, /refresh, /logout
│   │   │   ├── auth.controller.ts  ← Request handling
│   │   │   ├── auth.service.ts     ← Business logic
│   │   │   ├── auth.validation.ts  ← Zod schemas
│   │   │   └── auth.types.ts       ← TypeScript types
│   │   ├── users/
│   │   │   ├── user.routes.ts
│   │   │   ├── user.controller.ts
│   │   │   ├── user.service.ts
│   │   │   └── user.validation.ts
│   │   └── images/
│   │       ├── image.routes.ts
│   │       ├── image.controller.ts
│   │       ├── image.service.ts
│   │       └── image.validation.ts
│   ├── services/
│   │   ├── email.service.ts
│   │   ├── storage.service.ts
│   │   └── queue.service.ts
│   └── utils/
│       ├── logger.ts               ← Pino logger
│       ├── crypto.ts               ← Token generation helpers
│       └── errors.ts               ← Custom error classes
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
├── docker/
│   └── Dockerfile
├── tests/
│   ├── setup.ts
│   ├── auth.test.ts
│   └── images.test.ts
├── .env.example
├── .gitignore
├── tsconfig.json
├── package.json
└── README.md
```

## How to Build It

### Step 1: Initialize

```bash
mkdir backend && cd backend
git init
npm init -y
npm install express cors helmet compression dotenv zod bcryptjs jsonwebtoken uuid @prisma/client bullmq ioredis @aws-sdk/client-s3 @aws-sdk/s3-request-presigner resend @sentry/node pino pino-pretty
npm install -D typescript @types/node @types/express @types/cors @types/bcryptjs @types/jsonwebtoken ts-node-dev tsconfig-paths prisma jest ts-jest @types/jest supertest @types/supertest eslint
npx tsc --init
npx prisma init
```

### Step 2: `src/app.ts` — Express setup

Create this file. It must:

1. Import express and all middleware packages
2. Apply middleware IN THIS EXACT ORDER:
   - `helmet()` — always first, security headers
   - CORS with your specific origin whitelist
   - `compression()` — response compression
   - `express.json({ limit: '10mb' })` — body parsing with size limit
   - `express.urlencoded({ extended: true, limit: '10mb' })`
   - Request ID middleware — generate UUID, attach to request, set response header `X-Request-Id`
   - Request logger middleware — method, path, status, duration, userId
   - Rate limiting middleware — global then per-route
3. Mount routes: `app.use('/api/v1/auth', authRoutes)`, etc.
4. Health check at `GET /api/v1/health` — check DB connectivity, Redis connectivity, return JSON status
5. 404 handler for unmatched routes — return JSON `{ error: { code: 'NOT_FOUND', message: 'Route not found' } }`
6. Global error handler — 4 params `(err, req, res, next)`, return structured JSON error
7. DO NOT call `app.listen()` here — export `app` only

**Why this order**: Helmet first means security headers on every response including errors. CORS before routes so preflight works. Body parsing before auth so auth middleware can read the body. Rate limiting after body parsing so limit counts correctly. Error handler must be last.

### Step 3: `src/index.ts` — Server entry

Create this file. It must:

1. Import `app` from `./app`
2. Import `prisma` from `./config/database`
3. Import `env` from `./config/env` (this validates env vars and crashes if missing)
4. Call `await prisma.$connect()` to verify database connection
5. Call `app.listen(env.PORT)` to start the server
6. Log startup info: environment, port, health check URL
7. Handle graceful shutdown:
   - `process.on('SIGTERM')` — disconnect Prisma, `process.exit(0)`
   - `process.on('SIGINT')` — disconnect Prisma, `process.exit(0)`
8. Handle unhandled promise rejections: log error, exit 1
9. Handle uncaught exceptions: log error, exit 1

### Step 4: `src/config/database.ts` — Prisma singleton

Create this file. It must:

1. Import `PrismaClient` from `@prisma/client`
2. Create ONE instance: `new PrismaClient({ log: [...] })`
3. In development, log queries and errors. In production, log only errors.
4. Store in `globalThis` to prevent hot-reload from creating duplicate instances
5. Export the instance

**Why globalThis**: In development with ts-node-dev, every file change restarts the process. Without globalThis, you'd create a new PrismaClient on each restart, exhausting database connections.

### Step 5: `src/config/redis.ts` — Redis client

Create this file. It must:

1. Import Redis from `ioredis`
2. Create a Redis client with `env.REDIS_URL`
3. Handle `error` event — log but don't crash
4. Handle `connect` event — log "Redis connected"
5. Export the client
6. For BullMQ queues, create separate Redis connections (use `redis.duplicate()`)

**Why duplicate connections**: BullMQ requires dedicated connections. If the queue connection dies, your main cache connection shouldn't die too.

---

# 2. Database Schema & Migrations

## What to Build

A complete Prisma schema with every model your app needs, proper indexes, and migration history.

## How to Build It

### `prisma/schema.prisma`

Open this file. Set `provider = "postgresql"` and `url = env("DATABASE_URL")`.

Then define every model following these rules:

**For EVERY model, include these fields**:
- `id String @id @default(uuid())` — UUID primary key (never auto-increment — UUIDs are globally unique, safe for distributed systems, and don't leak record counts)
- `createdAt DateTime @default(now())` — creation timestamp
- `updatedAt DateTime @updatedAt` — auto-updated timestamp
- `deletedAt DateTime?` — soft delete (if the model needs it)

**For User model specifically**:
- `email String @unique` — unique index for login lookups
- `passwordHash String` — bcrypt hash, NEVER store plaintext
- `name String?` — nullable because OAuth users might not have a name initially
- `role Role @default(USER)` — enum: USER, ADMIN
- `plan Plan @default(FREE)` — enum: FREE, PRO, ENTERPRISE
- `emailVerified Boolean @default(false)` — verified flag
- `verifyToken String? @unique` — email verification token
- `stripeCustomerId String? @unique` — for billing
- `avatarUrl String?` — profile picture
- Relations: `refreshTokens RefreshToken[]`, `images Image[]`

**For Image model**:
- `userId String` — foreign key
- `user User @relation(...)` — relation
- `prompt String` — user's original prompt
- `enhancedPrompt String?` — prompt after style preset applied
- `model ImageModel` — enum: DALLE3, FLUX_PRO, STABLE_DIFFUSION, POLLINATIONS
- `style String @default("none")` — style preset name
- `size String @default("1024x1024")` — image dimensions
- `imageUrl String` — full URL to stored image
- `thumbnailUrl String?` — resized version for gallery
- `storageKey String?` — S3/R2 object key for deletion
- `isPublic Boolean @default(false)` — public gallery visibility
- `isFavorite Boolean @default(false)` — user favorite flag
- `generationTimeMs Int?` — how long generation took
- `costCredits Int @default(1)` — how many credits this cost
- `deletedAt DateTime?` — soft delete

**Indexes to add**:
- `@@index([userId])` — every query filters by user
- `@@index([model])` — filter by model
- `@@index([createdAt])` — sort by date
- `@@index([isPublic])` — public gallery queries
- `@@index([userId, createdAt])` — composite: user's images sorted by date

**For Gallery model**:
- `userId String`
- `name String`
- `description String?`
- `isPublic Boolean @default(false)`
- `coverImage String?`
- Relations: `images GalleryImage[]`
- Index on `userId`

**GalleryImage** (join table):
- `galleryId String`
- `imageId String`
- `order Int @default(0)` — display order
- `@@unique([galleryId, imageId])` — prevent duplicates

**For PasswordReset**:
- `userId String @unique`
- `token String @unique`
- `expiresAt DateTime`
- Index on `token`

**For RefreshToken**:
- `id String @id @default(uuid())`
- `token String @unique` — the actual refresh token
- `userId String`
- `user User @relation(...)`
- `deviceInfo String?` — browser/device info
- `expiresAt DateTime`
- `createdAt DateTime @default(now())`
- `@@index([userId])` — look up all tokens for a user
- `@@index([token])` — look up by token value

### After writing schema

```bash
npx prisma migrate dev --name init
npx prisma generate
npx prisma studio   # Verify your tables look correct
```

### Migration Rules

1. NEVER edit migration files after they're committed — always create new ones
2. Use `npx prisma migrate dev --name describe_what_changed` for development
3. Use `npx prisma migrate deploy` for production
4. Test migrations on a clean database: `npx prisma migrate reset && npx prisma migrate deploy`
5. If you need to rollback, create a NEW migration that reverses the change — don't delete old migrations

---

# 3. Docker + Docker Compose

## What to Build

A multi-stage Dockerfile for the API, and a Docker Compose file that runs the entire stack locally.

## `docker/Dockerfile`

Create this file with three stages:

**Stage 1 — deps**:
- Base: `node:20-alpine`
- Workdir: `/app`
- Copy `package.json` and `package-lock.json`
- Run `npm ci`
- Copy `prisma/schema.prisma` and run `npx prisma generate`

**Stage 2 — builder**:
- Base: `node:20-alpine`
- Copy `node_modules` from deps stage
- Copy all source code
- Run `npm run build`

**Stage 3 — runner** (production):
- Base: `node:20-alpine`
- Create non-root user: `addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -s /bin/sh -D appuser`
- Copy `dist/` from builder
- Copy `node_modules/` from builder
- Copy `package.json`
- Copy `prisma/` folder (needed for migrations at runtime)
- Set `USER appuser`
- Expose the port from env
- Add `HEALTHCHECK`: `wget --no-verbose --tries=1 --spider http://localhost:3001/api/v1/health || exit 1`
- CMD: `["node", "dist/index.js"]`

**Why multi-stage**: The deps stage caches npm packages. The builder stage compiles TypeScript. The runner stage only has what's needed to run — no dev dependencies, no source code. Final image is ~150MB instead of ~1GB.

## `docker-compose.yml`

Create this file with these services:

**api service**:
- Build from `docker/Dockerfile`
- Ports: `3001:3001`
- Environment: load from `.env` file
- Depends on: `postgres` (condition: service_healthy), `redis` (condition: service_healthy)
- Healthcheck: curl/wget the health endpoint every 30 seconds
- Restart: unless-stopped

**worker service**:
- Same image as api
- Command: `["node", "dist/worker.js"]`
- Same environment vars
- Depends on: api
- Restart: unless-stopped

**postgres service**:
- Image: `postgres:16-alpine`
- Ports: `5432:5432`
- Environment: POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
- Volume: `postgres_data:/var/lib/postgresql/data`
- Healthcheck: `pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}`

**redis service**:
- Image: `redis:7-alpine`
- Ports: `6379:6379`
- Volume: `redis_data:/data`
- Healthcheck: `redis-cli ping`

**Volumes**: `postgres_data` and `redis_data`

### `.dockerignore`

```
node_modules
dist
.env
.env.local
.git
README.md
docs
tests
*.md
```

---

# 4. CI/CD GitHub Actions

## What to Build

Two workflow files: one for testing (on every push/PR) and one for deployment (on merge to main and on tags).

## `.github/workflows/ci.yml`

Create this file. It must:

1. Trigger on: push to any branch, pull requests to main
2. Jobs:
   - **lint**: checkout → setup Node 20 → `npm ci` → `npm run lint`
   - **typecheck**: checkout → setup Node 20 → `npm ci` → `npm run typecheck`
   - **test**: checkout → setup Node 20 → start PostgreSQL service container → `npm ci` → run Prisma migration → `npm test`

The test job needs a PostgreSQL service container:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    env:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: appname_test
    ports:
      - 5432:5432
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

Set test environment variables: `DATABASE_URL=postgresql://test:test@localhost:5432/appname_test`, `JWT_SECRET=test-secret-minimum-32-characters-long`, `NODE_ENV=test`

3. Cache `node_modules` between runs using `actions/cache`
4. All three jobs run in parallel

## `.github/workflows/deploy.yml`

Create this file. It must:

1. Trigger on: push to main (→ staging), tags v* (→ production)
2. Jobs:
   - **deploy-staging** (only on main push):
     - checkout → setup Node 20 → `npm ci` → `npm run build`
     - Deploy to Railway staging: `railway up` using Railway GitHub Action
     - Run migrations: `railway run npx prisma migrate deploy`
     - Smoke test: curl health endpoint

   - **deploy-production** (only on v* tags):
     - checkout → setup Node 20 → `npm ci` → `npm run build`
     - Build and push Docker image to GitHub Container Registry
     - Deploy to Railway production
     - Run migrations
     - Smoke test

---

# 5. Nginx Config

## What to Build

A reverse proxy configuration that handles SSL termination, security headers, and request routing.

## `nginx/nginx.conf`

Create this file. It must:

1. Set `worker_connections 1024` in the events block
2. Define upstream `api` pointing to `api:3001` (Docker service name)
3. HTTP server on port 80 that redirects all traffic to HTTPS
4. HTTPS server on port 443 with:
   - SSL certificate paths
   - SSL protocols: TLSv1.2, TLSv1.3
   - SSL ciphers: `HIGH:!aNULL:!MD5`
   - Security headers (add_header):
     - X-Frame-Options: DENY
     - X-Content-Type-Options: nosniff
     - X-XSS-Protection: "0"
     - Referrer-Policy: strict-origin-when-cross-origin
     - Content-Security-Policy: default-src 'self'; img-src 'self' data: https://*.cloudflarestorage.com https://image.pollinations.ai; script-src 'self'; style-src 'self' 'unsafe-inline'
     - Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
5. Location blocks:
   - `/api/v1/health` — proxy pass with NO rate limiting
   - `/api/` — proxy pass with rate limiting (burst=50)
   - Timeouts: proxy_connect_timeout 60s, proxy_read_timeout 300s (for image generation which is slow)
   - WebSocket support: proxy_set_header Upgrade, Connection "upgrade"
   - Proxy headers: Host, X-Real-IP, X-Forwarded-For, X-Forwarded-Proto
   - `client_max_body_size 10m` — allow file uploads

---

# 6. Env Validation (Zod)

## What to Build

A module that validates ALL environment variables at startup, crashes with clear error messages if anything is missing, and exports a typed config object.

## `src/config/env.ts`

Create this file. It must:

1. Import `z` from `zod`
2. Define a Zod schema covering EVERY environment variable your app needs:
   - `NODE_ENV`: enum of `development`, `staging`, `production` with default `development`
   - `PORT`: coerce to number, default 3001
   - `DATABASE_URL`: string, url format
   - `REDIS_URL`: string, url format, default `redis://localhost:6379`
   - `JWT_SECRET`: string, min 32 characters
   - `JWT_ACCESS_EXPIRY`: string, default `15m`
   - `JWT_REFRESH_EXPIRY`: string, default `7d`
   - `OPENAI_API_KEY`: optional string
   - `HUGGING_FACE_TOKEN`: optional string
   - `S3_ACCESS_KEY`: optional string
   - `S3_SECRET_KEY`: optional string
   - `S3_BUCKET`: string, default `appname-images`
   - `S3_REGION`: string, default `auto`
   - `S3_ENDPOINT`: optional string
   - `RESEND_API_KEY`: optional string
   - `EMAIL_FROM`: string, default `App Name <noreply@yourdomain.com>`
   - `STRIPE_SECRET_KEY`: optional string
   - `STRIPE_WEBHOOK_SECRET`: optional string
   - `CORS_ORIGIN`: string, default `http://localhost:5173`
   - `RATE_LIMIT_WINDOW_MS`: coerce to number, default 900000 (15 min)
   - `RATE_LIMIT_MAX`: coerce to number, default 100
3. Call `schema.parse(process.env)` — this crashes if validation fails
4. Export the parsed result as `env`
5. In development, also print which env vars are missing/set

**Why this matters**: If someone forgets `JWT_SECRET`, the app should crash immediately at startup with a clear message, not run with an empty secret that lets anyone forge tokens.

---

# 7. Error Handling Patterns

## What to Build

A custom error class hierarchy and a global Express error handler that returns consistent JSON errors.

## `src/utils/errors.ts`

Create this file with these error classes:

1. `AppError` extends `Error`:
   - Properties: `statusCode`, `code` (machine-readable), `message` (human-readable), `details` (optional, for validation errors)
   - Constructor takes all four
   - This is the BASE class — all other errors extend this

2. `AuthError extends AppError`:
   - Always `statusCode: 401`
   - Code: `UNAUTHORIZED`

3. `ForbiddenError extends AppError`:
   - Always `statusCode: 403`
   - Code: `FORBIDDEN`

4. `NotFoundError extends AppError`:
   - Always `statusCode: 404`
   - Constructor takes resource name, message becomes `"${resource} not found"`

5. `ValidationError extends AppError`:
   - Always `statusCode: 400`
   - Code: `VALIDATION_ERROR`
   - Takes Zod error details as the `details` field

6. `ConflictError extends AppError`:
   - Always `statusCode: 409`
   - For duplicate records

7. `RateLimitError extends AppError`:
   - Always `statusCode: 429`
   - Code: `RATE_LIMITED`

## `src/middleware/errorHandler.ts`

Create this file. It must:

1. Export a function with 4 params: `(err, req, res, next)`
2. Extract `requestId` from `req.headers['x-request-id']`
3. If `err instanceof AppError`, return structured JSON:
   ```json
   { "error": { "code": err.code, "message": err.message, "details": err.details, "requestId": "uuid" } }
   ```
   With `res.status(err.statusCode)`
4. If it's a Prisma error (check `err.code` starts with 'P'):
   - P2002 → 409 Conflict (duplicate)
   - P2025 → 404 Not Found
   - Other Prisma errors → 500
5. If it's a JWT error:
   - TokenExpiredError → 401
   - JsonWebTokenError → 401
6. If it's none of the above:
   - Log the full error with logger
   - Return 500 with `{ error: { code: 'INTERNAL_ERROR', message: 'An unexpected error occurred', requestId } }`
   - In development mode, include `err.stack` in the response
   - In production mode, never include the stack

**Why**: One error handler for the entire app. Every route just throws errors and they're caught here. Consistent response format means the frontend always knows what shape errors are.

---

# 8. Rate Limiting, Security Middleware

## What to Build

Three rate limiters and security configuration.

## `src/middleware/rateLimiter.ts`

Create this file with three rate limiters using `express-rate-limit` with Redis store (`rate-limit-redis`):

1. **apiLimiter**: 100 requests per 15 minutes per IP
   - Use `standardHeaders: true` (sends RateLimit headers)
   - Use `legacyHeaders: false`
   - Store in Redis with prefix `rl:api:`
   - Message: `{ error: { code: 'RATE_LIMITED', message: 'Too many requests, please try again later.' } }`

2. **authLimiter**: 5 requests per 15 minutes per IP
   - Store in Redis with prefix `rl:auth:`
   - Apply to: POST /signup, POST /login, POST /forgot-password, POST /reset-password
   - Message: `{ error: { code: 'RATE_LIMITED', message: 'Too many authentication attempts.' } }`

3. **generateLimiter**: 5 requests per minute per USER (not IP)
   - Key generator: use `req.user?.id || req.ip`
   - Store in Redis with prefix `rl:gen:`
   - Apply to: POST /images/generate
   - Message: `{ error: { code: 'GENERATION_LIMIT', message: 'Too many image generations. Please wait a moment.' } }`

## Security Headers in `src/app.ts`

Add these AFTER helmet() in your middleware chain:

```typescript
// Remove X-Powered-By header (also done by helmet, but belt and suspenders)
app.disable('x-powered-by');

// Add Content Security Policy
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    "default-src 'self'; img-src 'self' data: https://*.cloudflarestorage.com https://image.pollinations.ai; script-src 'self'; style-src 'self' 'unsafe-inline';"
  );
  next();
});

// HSTS (HTTP Strict Transport Security) — also set by helmet if enabled
// helmet handles this, but verify it's set to at least 1 year with includeSubDomains
```

**Why Redis for rate limiting**: In-memory rate limiting resets when the server restarts. Redis persists across restarts. Also, if you run multiple instances behind a load balancer, they must share the same rate limit counters — only Redis makes this possible.

---

# 9. Deployment Configs (Railway/Fly)

## What to Build

Configuration files for deploying to Railway or Fly.io.

## Railway Configuration

Railway works without a config file. But you should create:

### `railway.toml`

```toml
[build]
builder = "DOCKERFILE"
dockerfilePath = "docker/Dockerfile"

[deploy]
startCommand = "node dist/index.js"
healthcheckPath = "/api/v1/health"
healthcheckTimeout = 300
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 10
```

### Steps to deploy on Railway:

1. Install CLI: `npm install -g @railway/cli`
2. Login: `railway login`
3. Link: `railway link` (create new project)
4. Add PostgreSQL: `railway add --plugin postgres`
5. Add Redis: `railway add --plugin redis`
6. Set ALL environment variables from `.env.example`:
   ```bash
   railway variables set JWT_SECRET=$(openssl rand -hex 32)
   railway variables set NODE_ENV=production
   # ... set each variable
   ```
7. Deploy: `railway up`
8. Run migrations: `railway run npx prisma migrate deploy`
9. Seed: `railway run npx prisma db seed`
10. Check health: `curl https://your-app.up.railway.app/api/v1/health`

## Fly.io Configuration

### `fly.toml`

```toml
app = "your-app-name"
primary_region = "sjc"

[build]
  dockerfile = "docker/Dockerfile"

[env]
  NODE_ENV = "production"
  PORT = "3001"

[deploy]
  release_command = "npx prisma migrate deploy"

[checks]
  [checks.health]
    port = 3001
    type = "http"
    interval = "30s"
    timeout = "5s"
    grace_period = "10s"
    method = "get"
    path = "/api/v1/health"
```

### Steps to deploy on Fly.io:

1. Install CLI: `curl -L https://fly.io/install.sh | sh`
2. Login: `fly auth login`
3. Launch: `fly launch` (it reads fly.toml)
4. Add PostgreSQL: `fly postgres create` then `fly postgres attach <db-name>`
5. Add Redis: `fly redis create` then `fly redis attach <redis-name>`
6. Set secrets: `fly secrets set JWT_SECRET=$(openssl rand -hex 32)` for each variable
7. Deploy: `fly deploy`
8. Check: `fly open /api/v1/health`

---

# 10. `.env.example`

## What to Build

A comprehensive environment variable template that documents every variable.

## Create `.env.example`

```env
# ═══════════════════════════════════════════════════════════
# [APP-NAME] API — Environment Variables
# ═══════════════════════════════════════════════════════════
# Copy this file to .env and fill in your values.
# NEVER commit .env to git.
# Generate secrets with: openssl rand -hex 32
# ═══════════════════════════════════════════════════════════

# === SERVER ===
NODE_ENV=development                    # development | staging | production
PORT=3001                               # API server port

# === DATABASE ===
DATABASE_URL=postgresql://user:password@localhost:5432/appname
# Format: postgresql://USER:PASSWORD@HOST:PORT/DATABASE

# === REDIS ===
REDIS_URL=redis://localhost:6379        # For caching, queues, and rate limiting

# === JWT AUTH ===
# Generate with: openssl rand -hex 32
JWT_SECRET=change-me-to-64-random-hex-characters-minimum
JWT_ACCESS_EXPIRY=15m                   # Access token lifetime (short)
JWT_REFRESH_EXPIRY=7d                   # Refresh token lifetime (long)

# === FILE STORAGE (Cloudflare R2 or AWS S3) ===
# Cloudflare R2: https://dash.cloudflare.com → R2 → Create bucket → API tokens
S3_ACCESS_KEY=
S3_SECRET_KEY=
S3_BUCKET=appname-images
S3_REGION=auto                           # 'auto' for Cloudflare R2
S3_ENDPOINT=                             # R2 endpoint URL, e.g. https://xxx.r2.cloudflarestorage.com

# === AI PROVIDERS (optional — app works without these using free Pollinations.ai) ===
OPENAI_API_KEY=                          # https://platform.openai.com/api-keys
HUGGING_FACE_TOKEN=                      # https://huggingface.co/settings/tokens

# === EMAIL (optional — required for email verification & password reset) ===
RESEND_API_KEY=                          # https://resend.com/api-keys
EMAIL_FROM="App Name <noreply@yourdomain.com>"

# === STRIPE BILLING (optional — required for paid plans) ===
STRIPE_SECRET_KEY=                       # https://dashboard.stripe.com/apikeys
STRIPE_WEBHOOK_SECRET=                  # From Stripe Dashboard → Webhooks
STRIPE_PRICE_MONTHLY=                   # Stripe Price ID for monthly plan
STRIPE_PRICE_YEARLY=                    # Stripe Price ID for yearly plan

# === SENTRY (optional — error tracking in production) ===
SENTRY_DSN=                             # https://sentry.io → Project → Settings → DSN

# === CORS ===
# Comma-separated list of allowed origins
CORS_ORIGIN=http://localhost:5173,http://localhost:1420
# 5173 = Vite dev server, 1420 = Tauri dev

# === RATE LIMITING ===
RATE_LIMIT_WINDOW_MS=900000              # 15 minutes in milliseconds
RATE_LIMIT_MAX=100                       # Max requests per window per IP
```

---

# 11. README for the Repo

## What to Build

A README that explains what this repo is, how to set it up, and how to run it.

## Create `README.md`

The README must contain these sections:

1. **Title and description** — App name, what it does, what stack it uses
2. **Architecture diagram** — ASCII art or Mermaid diagram showing: clients → API → database/Redis/storage
3. **Prerequisites** — Node 20+, PostgreSQL 16+, Redis 7+, Docker (optional)
4. **Quick Start** — 5 commands to get running: clone, install, env, db push, dev
5. **Available Scripts** — Table of all npm scripts with descriptions
6. **Project Structure** — Tree of folder structure with one-line descriptions
7. **API Endpoints** — Complete table of all endpoints: method, path, description, auth required
8. **Authentication** — JWT flow explanation: signup → login → access token → refresh → rotate
9. **Environment Variables** — Link to `.env.example` with note to never commit `.env`
10. **Database** — How to run migrations, seed data, reset, and open Prisma Studio
11. **Docker** — How to start/stop the full stack with Docker Compose
12. **Testing** — How to run tests, what's covered
13. **Deployment** — Railway/Fly instructions
14. **License** — MIT

---

# 12. WebSocket Real Implementation

## What to Build

A Socket.IO server that connects users to private rooms and pushes real-time events (image generation complete, notifications, etc.).

## `src/services/websocket.service.ts`

Create this file. It must:

1. Import `Server` from `socket.io`
2. Create a singleton class `WebSocketService` with a private `io` property
3. Implement `init(httpServer)` method:
   - Create Socket.IO server with CORS config (match your API origins)
   - Use JWT auth middleware: verify token from `socket.handshake.auth.token`
   - If valid: attach userId to `socket.data.userId`
   - If invalid: disconnect with error message
   - On connection: join room `user:${userId}`
   - On disconnect: leave room `user:${userId}`
   - Log connections and disconnections
4. Implement `sendToUser(userId, event, data)` method:
   - Emit event to `user:${userId}` room
5. Implement `broadcast(event, data)` method:
   - Emit to all connected sockets
6. Export a singleton instance

## How to Initialize

In `src/index.ts`, after creating the HTTP server but before calling `listen`:

1. Import `websocketService` from the service file
2. Pass the raw HTTP server to `websocketService.init(server)`
3. This means you need to create the server manually: `const server = http.createServer(app)` then use `server.listen` instead of `app.listen`

## How to Use from Services

In your image generation service, when a generation completes:

```typescript
websocketService.sendToUser(userId, 'generation_complete', { imageId: image.id, imageUrl: image.imageUrl });
```

In your notification service:
```typescript
websocketService.sendToUser(userId, 'notification', { type: 'info', title: 'Image Ready', message: 'Your image has been generated!' });
```

## Frontend Connection

On any frontend (web, macOS, Android):

```typescript
import { io } from 'socket.io-client';
const socket = io('https://api.yourdomain.com', { auth: { token: accessToken } });
socket.on('generation_complete', (data) => { /* update UI */ });
socket.on('notification', (data) => { /* show notification */ });
```

**Why WebSocket over polling**: Image generation takes 5-30 seconds. Polling every second wastes resources and adds latency. WebSocket delivers the result instantly when ready.

---

# 13. Worker Entry Point

## What to Build

A separate process that runs background jobs (email sending, image processing) using BullMQ.

## `src/worker.ts`

Create this file. It must:

1. Import environment config (validates env vars at startup)
2. Import and initialize Prisma client (for database access)
3. Import Redis connection
4. Import all queue definitions from `src/services/queue.service.ts`
5. Import all worker definitions
6. Log startup: "Worker started. Processing jobs..."
7. Handle graceful shutdown:
   - `process.on('SIGTERM')` — close all workers, disconnect Prisma, exit 0
   - `process.on('SIGINT')` — same as SIGTERM

## `src/services/queue.service.ts`

Create this file. It must:

1. Define named queues using BullMQ:
   - `imageGenerationQueue` — for image generation jobs
   - `emailQueue` — for sending emails
   - `notificationQueue` — for push notifications
2. Define workers for each queue:
   - **Image generation worker**: concurrency 3, processes jobs by calling the image service
   - **Email worker**: concurrency 5, processes jobs by calling the email service
   - **Notification worker**: concurrency 5, processes jobs by sending WebSocket events
3. Each worker must:
   - Log job start and completion
   - Handle errors: retry up to 3 times with exponential backoff (10s, 30s, 90s)
   - Mark job as completed with the result
   - Mark job as failed if all retries exhausted

## How to Enqueue a Job

From your API controller, instead of processing synchronously:

```typescript
// Instead of:
const result = await imageService.generate(userId, prompt, options);
// Do:
await imageGenerationQueue.add('generate', { userId, prompt, ...options }, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 10000 },
});
```

Then the worker picks it up and processes it in the background.

**Why a separate worker**: If image generation takes 20 seconds and you have 10 concurrent requests, your API workers are all blocked. With a separate worker process, the API returns immediately (job queued) and workers process jobs independently. You can scale workers independently from API servers.

---

# 14. Stripe/Billing Module

## What to Build

A billing module that handles Stripe checkout sessions, webhooks, and subscription status.

## `src/modules/billing/`

Create these files:

### `billing.routes.ts`
- `POST /api/v1/billing/checkout` — Create Stripe checkout session (authenticated)
- `POST /api/v1/billing/webhook` — Stripe webhook handler (NO auth — Stripe signs it)
- `GET /api/v1/billing/subscription` — Get current subscription status (authenticated)
- `POST /api/v1/billing/cancel` — Cancel subscription (authenticated)
- `POST /api/v1/billing/portal` — Create Stripe customer portal session (authenticated)

### `billing.service.ts`

Create these methods:

1. **createCheckoutSession(userId, priceId)**:
   - Look up user, get or create Stripe customer
   - Create `stripe.checkout.sessions.create()` with:
     - mode: 'subscription'
     - payment_method_types: ['card']
     - line_items: [{ price: priceId, quantity: 1 }]
     - success_url: `${FRONTEND_URL}/billing?success=true`
     - cancel_url: `${FRONTEND_URL}/billing?canceled=true`
     - metadata: { userId }
   - Return session URL

2. **handleWebhook(event)**:
   - Use `stripe.webhooks.constructEvent()` with raw body and signature header
   - Switch on event type:
     - `checkout.session.completed` → update user plan to PRO
     - `customer.subscription.updated` → update plan status
     - `customer.subscription.deleted` → downgrade user to FREE
     - `invoice.payment_failed` → send notification email
   - Each handler updates the user's `plan` field in the database

3. **getSubscription(userId)**:
   - Query Stripe for the customer's active subscriptions
   - Return: plan name, status, current period end, cancel at period end

4. **cancelSubscription(userId)**:
   - Get Stripe subscription ID from user record
   - Call `stripe.subscriptions.update(subId, { cancel_at_period_end: true })`
   - User keeps access until the end of their billing period

### `billing.controller.ts`

Standard controller pattern: extract input from request, call service, return response.

### Webhook Security

CRITICAL: The webhook endpoint must verify Stripe's signature:

1. Use `express.raw({ type: 'application/json' })` as middleware BEFORE `express.json()` on the webhook route only
2. Get the `Stripe-Signature` header
3. Use `stripe.webhooks.constructEvent(rawBody, signature, webhookSecret)` to verify
4. If verification fails, return 400
5. If verification succeeds, process the event and return 200
6. Always return 200 quickly — Stripe retries if it doesn't get 200 within 10 seconds

---

# 15. Database Migration Rollback Scripts

## What to Build

A strategy and scripts for reversing database migrations when something goes wrong.

## Rules

1. NEVER edit committed migration files — always create new ones
2. Prisma doesn't have automatic rollback — you create a NEW migration that reverses the change
3. Test every migration on a clean database before deploying to production

## How to Handle Rollbacks

### Strategy 1: Create a Reverse Migration

If you added a column `tags TEXT` and need to remove it:

```bash
npx prisma migrate dev --name remove_tags_column
```

Then edit the generated SQL to:
```sql
ALTER TABLE "images" DROP COLUMN "tags";
```

### Strategy 2: Use Prisma Migrate Reset (Development Only)

```bash
npx prisma migrate reset --force
```

This drops the database, recreates it, and runs all migrations. **NEVER run this in production.**

### Strategy 3: Manual Database Backup Before Migrations (Production)

Create a script `scripts/backup-db.sh`:

1. Use `pg_dump` to create a backup: `pg_dump $DATABASE_URL > backups/backup_$(date +%Y%m%d_%H%M%S).sql`
2. Compress it: `gzip backups/backup_*.sql`
3. Upload to S3/R2: `aws s3 cp backups/ s3://db-backups/ --recursive`
4. Delete local backups older than 7 days

Create `scripts/restore-db.sh`:

1. Download from S3: `aws s3 cp s3://db-backups/$BACKUP_FILE .`
2. Decompress: `gunzip $BACKUP_FILE`
3. Restore: `psql $DATABASE_URL < $RESTORED_FILE`

### Strategy 4: Safe Migration Deployment

Before running migrations in production:

1. Backup the database
2. Run `npx prisma migrate deploy` (applies only pending migrations)
3. Verify health endpoint
4. If health endpoint fails, restore from backup

Create `scripts/deploy-migrations.sh`:

```bash
#!/bin/bash
set -e
echo "Backing up database..."
./scripts/backup-db.sh
echo "Running migrations..."
npx prisma migrate deploy
echo "Verifying health..."
curl -f http://localhost:3001/api/v1/health || { echo "Health check failed! Rolling back..."; exit 1; }
echo "Migration complete!"
```

---

# 16. Nginx SSL Config

## What to Build

HTTPS configuration for Nginx with Let's Encrypt or custom certificates.

## SSL Certificate Generation (Development)

```bash
# Generate self-signed certificate for local development
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/ssl/key.pem \
  -out nginx/ssl/cert.pem \
  -subj "/CN=localhost"
```

## SSL Certificate for Production

Use Let's Encrypt with Certbot:

```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx

# Generate certificate
sudo certbot --nginx -d api.yourdomain.com

# Auto-renewal (certbot adds a cron job)
sudo certbot renew --dry-run
```

## `nginx/ssl.conf` (included in main nginx.conf)

Create this file with SSL best practices:

1. `ssl_certificate /etc/nginx/ssl/cert.pem;`
2. `ssl_certificate_key /etc/nginx/ssl/key.pem;`
3. `ssl_protocols TLSv1.2 TLSv1.3;`
4. `ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;`
5. `ssl_prefer_server_ciphers off;`
6. `ssl_session_cache shared:SSL:10m;`
7. `ssl_session_timeout 10m;`
8. `ssl_session_tickets off;` — disables session tickets for forward secrecy
9. OCSP stapling: `ssl_stapling on; ssl_stapling_verify on; resolver 8.8.8.8 8.8.4.4 valid=300s;`

## HTTP to HTTPS Redirect

In your nginx.conf, add a server block for port 80:

```
server {
    listen 80;
    server_name api.yourdomain.com;
    return 301 https://$server_name$request_uri;
}
```

---

# 17. API Documentation (OpenAPI/Swagger)

## What to Build

Auto-generated API documentation that stays in sync with your code.

## How to Add Swagger

### Step 1: Install

```bash
npm install swagger-ui-express swagger-jsdoc
npm install -D @types/swagger-ui-express @types/swagger-jsdoc
```

### Step 2: Create `src/config/swagger.ts`

1. Import `swaggerJsdoc` from `swagger-jsdoc`
2. Define options:
   - `definition`: API info (title, version, description, servers array)
   - `definition.security`: Bearer auth scheme
   - `apis`: glob pattern pointing to your route files (`['src/modules/**/*.ts']`)
3. Export `specs = swaggerJsdoc(options)`

### Step 3: Add JSDoc Comments to Routes

On EVERY route file, add JSDoc comments above each route:

```typescript
/**
 * @openapi
 * /api/v1/auth/signup:
 *   post:
 *     summary: Create a new account
 *     tags: [Auth]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required: [email, password, name]
 *             properties:
 *               email:
 *                 type: string
 *                 format: email
 *               password:
 *                 type: string
 *                 minLength: 8
 *               name:
 *                 type: string
 *     responses:
 *       201:
 *         description: Account created, tokens returned
 *       409:
 *         description: Email already exists
 *       400:
 *         description: Validation error
 */
```

### Step 4: Mount Swagger UI

In `src/app.ts`:

```typescript
import swaggerUi from 'swagger-ui-express';
import { specs } from './config/swagger';

app.use('/api/docs', swaggerUi.serve, swaggerUi.setup(specs));
```

Now visit `http://localhost:3001/api/docs` to see interactive API documentation.

### Step 5: Add to Health Check

Include in your health endpoint response whether docs are available:

```json
{ "status": "healthy", "docs": "http://localhost:3001/api/docs" }
```

---

# 18. Load Testing Config

## What to Build

A k6 load testing script that establishes performance baselines.

## Install k6

```bash
# macOS
brew install k6

# Linux
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415B36442BEAE31B4F3D
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt update && sudo apt install k6
```

## Create `tests/load/basic.js`

This script must test:

1. **Health check** — GET /api/v1/health (baseline, should be <50ms)
2. **Signup** — POST /api/v1/auth/signup (baseline, should be <500ms)
3. **Login** — POST /api/v1/auth/login (baseline, should be <200ms)
4. **List images** — GET /api/v1/images (with auth, should be <100ms)
5. **Generate image** — POST /api/v1/images/generate (with auth, should be <10000ms)

Structure:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '30s', target: 20 },   // Ramp up to 20 users
    { duration: '60s', target: 20 },    // Stay at 20 users
    { duration: '30s', target: 50 },    // Ramp up to 50 users
    { duration: '60s', target: 50 },    // Stay at 50 users
    { duration: '30s', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],    // 95% of requests under 500ms
    http_req_failed_rate: ['rate<0.01'], // Less than 1% failure rate
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3001';
```

For each endpoint, make the request, check status code, and assert response time.

## Create `tests/load/auth-flow.js`

A more realistic test that simulates a user journey:
1. Signup a new user
2. Login
3. Generate an image
4. List images
5. Check generated image
6. Logout

## Running Load Tests

```bash
# Local
k6 run tests/load/basic.js

# Against staging
BASE_URL=https://api-staging.yourdomain.com k6 run tests/load/basic.js

# With more users
k6 run --vus 100 --duration 5m tests/load/basic.js
```

---

# 19. PWA Service Worker

## What to Build

A Progressive Web App configuration for the web frontend that enables offline access, installability, and caching.

## `frontend/vite.config.ts` Modifications

1. Install PWA plugin: `npm install -D vite-plugin-pwa`
2. Add to Vite config:

```typescript
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: 'autoUpdate',
      includeAssets: ['favicon.svg', 'apple-touch-icon.png'],
      manifest: {
        name: '[APP-NAME]',
        short_name: '[APP-NAME]',
        description: '[One-line description]',
        theme_color: '#6C63FF',
        background_color: '#0F0F23',
        display: 'standalone',
        scope: '/',
        start_url: '/',
        icons: [
          { src: '/icon-192x192.png', sizes: '192x192', type: 'image/png' },
          { src: '/icon-512x512.png', sizes: '512x512', type: 'image/png' },
          { src: '/icon-512x512.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' },
        ],
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\..*/i,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache',
              expiration: { maxEntries: 100, maxAgeSeconds: 60 * 5 },
              cacheableResponse: { statuses: [0, 200] },
            },
          },
          {
            urlPattern: /^https:\/\/.*cloudflarestorage\.com\/.*/i,
            handler: 'CacheFirst',
            options: {
              cacheName: 'image-cache',
              expiration: { maxEntries: 200, maxAgeSeconds: 60 * 60 * 24 * 30 },
              cacheableResponse: { statuses: [0, 200] },
            },
          },
        ],
      },
    }),
  ],
});
```

## What this does

- **manifest**: Makes the app installable on phones and desktops
- **NetworkFirst for API**: Always try network first, fall back to cache if offline
- **CacheFirst for images**: Cache generated images for 30 days (they don't change)
- **autoUpdate**: Service worker updates automatically when you deploy new code
- **globPatterns**: Pre-cache all static assets on install

## Create App Icons

You need these sizes:
- `icon-192x192.png` — Android home screen
- `icon-512x512.png` — Android splash screen, PWA requirement
- `apple-touch-icon.png` (180x180) — iOS home screen
- `favicon.svg` — Browser tab

---

# 20. Monitoring Dashboards

## What to Build

Monitoring setup using Sentry for errors + a simple health dashboard.

## Sentry Setup

1. Create account at sentry.io
2. Create a new project: choose Node.js
3. Copy the DSN to your `.env` as `SENTRY_DSN`

### `src/config/sentry.ts`

1. Import `@sentry/node`
2. Create `initSentry()` function:
   - Only initialize if `SENTRY_DSN` is set and `NODE_ENV` is not `test`
   - Set `dsn`, `environment`, `tracesSampleRate: 0.1` (10% of transactions)
   - Add integrations: `prismaIntegration()`, `httpIntegration()`, `expressIntegration()`
3. Call this function BEFORE creating the Express app in `src/index.ts`
4. Add Sentry error handler AFTER your custom error handler in `src/app.ts`

### Set User Context

After authentication, in your auth middleware:

```typescript
Sentry.setUser({ id: req.user.id, email: req.user.email });
```

This way, when an error occurs, you see exactly which user triggered it.

## Simple Health Dashboard

Create `src/routes/health.ts` with a detailed health endpoint that returns:

```json
{
  "status": "healthy",
  "timestamp": "2024-01-01T00:00:00Z",
  "uptime": 3600,
  "version": "1.0.0",
  "checks": {
    "database": { "status": "up", "responseTime": "5ms" },
    "redis": { "status": "up", "responseTime": "2ms" },
    "storage": { "status": "up" }
  },
  "stats": {
    "users": 1234,
    "images": 5678,
    "requests24h": 9012
  }
}
```

To check each:
- **Database**: Run `SELECT 1` and measure response time
- **Redis**: Run `redis.ping()` and measure response time
- **Storage**: Attempt to list objects in S3/R2 bucket (or just check if credentials are configured)

## Uptime Monitoring (External)

1. Sign up for UptimeRobot (free tier: 50 monitors)
2. Add monitor: `https://api.yourdomain.com/api/v1/health`
3. Check every 5 minutes
4. Alert via email + Slack webhook
5. For Slack: create a webhook URL in Slack, configure UptimeRobot to post to it

---

# 21. OAuth (Google) Implementation

## What to Build

"Sign in with Google" flow using OAuth2 authorization code grant.

## Setup

1. Go to Google Cloud Console → APIs & Services → Credentials
2. Create OAuth 2.0 Client ID
3. Add authorized redirect URI: `https://api.yourdomain.com/api/v1/auth/google/callback`
4. Copy Client ID and Client Secret to `.env`:
   ```
   GOOGLE_CLIENT_ID=your-client-id
   GOOGLE_CLIENT_SECRET=your-client-secret
   ```

## Add to Auth Module

### `auth.routes.ts` additions:
- `GET /api/v1/auth/google` → redirect to Google consent screen
- `GET /api/v1/auth/google/callback` → handle Google redirect

### `auth.service.ts` additions:

1. **getGoogleAuthUrl()**: 
   - Build Google OAuth URL: `https://accounts.google.com/o/oauth2/v2/auth?client_id=...&redirect_uri=...&response_type=code&scope=openid email profile`
   - Generate a `state` parameter (random UUID) for CSRF protection
   - Store state in Redis with 10-minute expiry
   - Return the URL

2. **handleGoogleCallback(code, state)**:
   - Verify state matches what was stored in Redis (prevent CSRF)
   - Exchange code for tokens: POST to `https://oauth2.googleapis.com/token` with code, client_id, client_secret, redirect_uri, grant_type=authorization_code
   - Parse the id_token (JWT) to get user info: email, name, picture
   - Find or create user in database:
     - If email exists: link Google account to existing user, log them in
     - If email doesn't exist: create new user with Google info, set emailVerified=true
   - Generate access + refresh tokens
   - Return tokens

### Frontend Integration

1. Frontend redirects to `GET /api/v1/auth/google`
2. User signs in on Google
3. Google redirects to your callback URL
4. Backend exchanges code, finds/creates user, generates tokens
5. Backend redirects to frontend with tokens: `https://yourdomain.com/auth/callback?accessToken=...&refreshToken=...`
6. Frontend stores tokens and redirects to dashboard

---

# 22. 2FA/TOTP

## What to Build

Time-based One-Time Password (TOTP) two-factor authentication using the Authenticator app pattern.

## Install

```bash
npm install otplib qrcode
npm install -D @types/qrcode
```

## Add to User Model

Add to the Prisma schema:
```
twoFactorSecret  String?            ← TOTP secret key (encrypted)
twoFactorEnabled Boolean @default(false)
twoFactorBackupCodes String?       ← JSON array of backup codes, hashed
```

Run: `npx prisma migrate dev --name add_two_factor`

## Add to Auth Module

### `auth.routes.ts` additions:
- `POST /api/v1/auth/2fa/setup` (authenticated) → Generate TOTP secret and QR code
- `POST /api/v1/auth/2fa/verify` (authenticated) → Verify setup code and enable 2FA
- `POST /api/v1/auth/2fa/disable` (authenticated, requires current password) → Disable 2FA

### `auth.service.ts` additions:

1. **setup2FA(userId)**:
   - Generate TOTP secret: `authenticator.generateSecret()`
   - Store secret temporarily in Redis (not in DB yet — not enabled)
   - Generate otpauth URI: `otpauth://totp/[APP-NAME]:{email}?secret={secret}&issuer=[APP-NAME]`
   - Generate QR code from the URI: `QRCode.toDataURL(otpauthUri)`
   - Return: `{ secret, qrCodeUrl, manualEntryKey }`
   - User scans QR code with Google Authenticator / Authy

2. **verify2FASetup(userId, code)**:
   - Get stored secret from Redis
   - Verify code: `authenticator.verify({ token: code, secret })`
   - If valid: store secret in User table (twoFactorSecret), set twoFactorEnabled=true
   - Generate 10 backup codes: `Array.from({ length: 10 }, () => randomBytes(4).toString('hex'))`
   - Hash backup codes with bcrypt (store hashed)
   - Return: `{ backupCodes: [...plainCodes] }` — show these to user ONCE, they can't be recovered

3. **verify2FALogin(userId, code)**:
   - Get user's twoFactorSecret from database
   - Verify code: `authenticator.verify({ token: code, secret })`
   - If invalid: also check backup codes (compare against hashed codes)
   - If backup code match: remove that code from the array
   - Return: valid or invalid

### Modify Login Flow

After successful password check, if `user.twoFactorEnabled`:
1. Return a special response: `{ requiresTwoFactor: true, tempToken: '...' }`
2. Store temp token in Redis with 5-minute expiry
3. Frontend shows 2FA code input
4. User submits code to `POST /api/v1/auth/2fa/login` with `tempToken` and `code`
5. Verify temp token, verify 2FA code
6. If valid: return regular access + refresh tokens

---

# 23. Gallery/Sharing Endpoints

## What to Build

CRUD endpoints for galleries (collections of images) and a sharing mechanism.

## `src/modules/gallery/`

### `gallery.routes.ts`
- `GET /api/v1/galleries` (authenticated) → List user's galleries
- `POST /api/v1/galleries` (authenticated) → Create gallery
- `GET /api/v1/galleries/:id` (authenticated) → Get gallery with images
- `PATCH /api/v1/galleries/:id` (authenticated) → Update gallery name/description
- `DELETE /api/v1/galleries/:id` (authenticated) → Delete gallery (soft delete)
- `POST /api/v1/galleries/:id/images` (authenticated) → Add images to gallery
- `DELETE /api/v1/galleries/:id/images/:imageId` (authenticated) → Remove image from gallery

### `gallery.service.ts`

1. **create(userId, data)**:
   - Validate: name is required, maxLength 100
   - Create gallery in database with userId
   - Return gallery

2. **list(userId, page, limit)**:
   - Find galleries where userId matches and deletedAt is null
   - Include count of images in each gallery
   - Return paginated results

3. **getById(galleryId, userId)**:
   - Find gallery by ID
   - If gallery is not public AND userId doesn't match owner: throw ForbiddenError
   - Include images (with thumbnails)
   - Return gallery with images

4. **addImages(galleryId, userId, imageIds)**:
   - Verify gallery belongs to user
   - Verify all images belong to user
   - Create GalleryImage records for each
   - Return updated gallery

5. **removeImage(galleryId, userId, imageId)**:
   - Verify gallery belongs to user
   - Delete the GalleryImage join record
   - Return updated gallery

6. **share(galleryId, userId)**:
   - Generate a unique share token: `crypto.randomBytes(8).toString('hex')`
   - Update gallery: `shareToken = token, isPublic = true`
   - Return share URL: `${FRONTEND_URL}/gallery/${token}`

---

# 24. Notification System

## What to Build

An in-app notification system that stores notifications in the database and pushes them via WebSocket.

## Add to Prisma Schema

```
model Notification {
  id          String            @id @default(uuid())
  userId      String
  user        User              @relation(fields: [userId], references: [id], onDelete: Cascade)
  type        NotificationType
  title       String
  message     String
  actionUrl   String?           ← URL to navigate when clicked
  isRead      Boolean           @default(false)
  createdAt   DateTime          @default(now())

  @@index([userId, isRead])
  @@index([userId, createdAt])
}

enum NotificationType {
  GENERATION_COMPLETE
  GALLERY_SHARED
  CREDIT_LOW
  PLAN_UPGRADED
  WELCOME
  SYSTEM
}
```

## `src/modules/notifications/`

### `notification.service.ts`

1. **create(userId, type, title, message, actionUrl?)**:
   - Create notification in database
   - Push to user via WebSocket: `websocketService.sendToUser(userId, 'notification', notification)`
   - Return notification

2. **list(userId, page, limit)**:
   - Find notifications where userId matches
   - Order by createdAt DESC
   - Return paginated

3. **markAsRead(notificationId, userId)**:
   - Verify notification belongs to user
   - Update isRead to true

4. **markAllAsRead(userId)**:
   - Update all unread notifications for this user to isRead: true

5. **deleteOldNotifications()**:
   - Called by a scheduled job (WorkManager)
   - Delete notifications older than 90 days where isRead = true

### `notification.routes.ts`
- `GET /api/v1/notifications` (authenticated) → List user's notifications
- `PATCH /api/v1/notifications/:id/read` (authenticated) → Mark as read
- `PATCH /api/v1/notifications/read-all` (authenticated) → Mark all as read
- `GET /api/v1/notifications/unread-count` (authenticated) → Get count of unread

### When to Create Notifications

- Image generation completes → `create(userId, 'GENERATION_COMPLETE', 'Image Ready', 'Your image is ready to view', `/gallery/${imageId}`)`
- Gallery is shared with you → `create(userId, 'GALLERY_SHARED', 'New Shared Gallery', 'Someone shared a gallery with you', `/gallery/${shareToken}`)`
- Credits running low → `create(userId, 'CREDIT_LOW', 'Credits Running Low', 'You have 2 generations left today', '/settings/plan')`
- Plan upgraded → `create(userId, 'PLAN_UPGRADED', 'Welcome to Pro!', 'Your Pro plan is now active', '/settings')`

---

# 25. Explore (Public Gallery) Endpoints

## What to Build

A public feed of images that anyone can browse without authentication.

## `src/modules/explore/`

### `explore.routes.ts`
- `GET /api/v1/explore` (public, NO auth) → Browse public images with pagination
- `GET /api/v1/explore/:id` (public) → View a single public image

### `explore.service.ts`

1. **list(page, limit, sortBy, model, style)**:
   - Find images where isPublic = true AND deletedAt = null
   - Apply filters: model (optional), style (optional)
   - Sort options: `newest`, `popular` (most favorited), `random`
   - Include user name only (not email)
   - Return paginated results with total count

2. **getById(imageId)**:
   - Find image where id matches AND isPublic = true AND deletedAt = null
   - Include user name and avatar
   - Increment a view count (add `viewCount Int @default(0)` to Image model)
   - Return image details

### Don't Include in Public Responses

- Never return: email, passwordHash, role, plan, stripeCustomerId
- Only return: id, prompt, enhancedPrompt, model, style, imageUrl (or thumbnailUrl for list), createdAt, user.name

### Caching Strategy

1. Cache the explore list response in Redis for 60 seconds (it's public, doesn't change every second)
2. Cache key format: `explore:page:{page}:sort:{sortBy}:model:{model}:style:{style}`
3. On every new image creation, invalidate the explore cache

---

# 26. Database Indexing Strategy

## What to Build

A plan for which indexes to add and why, based on query patterns.

## Indexing Rules

1. **Add an index for every column in a WHERE clause that filters many rows**
2. **Add an index for every column in an ORDER BY that sorts many rows**
3. **Add composite indexes for common filter+sort combinations**
4. **Don't over-index** — each index slows down writes and takes disk space
5. **Always index foreign keys** — JOIN queries need them

## Indexes to Add to Your Prisma Schema

### Users table
```prisma
@@index([email])          // Already @unique, so this index exists automatically
// No additional indexes needed — queries are always by primary key or email
```

### Images table
```prisma
@@index([userId])                    // Every query filters by user
@@index([userId, createdAt])          // User's images sorted by date (most common query)
@@index([model])                       // Filter by model type
@@index([createdAt])                   // Sort by newest globally
@@index([isPublic, createdAt])        // Explore page: public images sorted by date
@@index([isPublic, model])            // Explore page: filter by model
@@index([isFavorite])                  // User's favorites
```

### RefreshTokens table
```prisma
@@index([token])       // Already @unique, automatic
@@index([userId])      // Delete all tokens for a user on logout
@@index([expiresAt])   // Cleanup expired tokens
```

### Notifications table
```prism@@index([userId, isRead])        // Get unread notifications for a user
@@index([userId, createdAt])      // List notifications sorted by date
@@index([createdAt])               // Cleanup old notifications
```

### Galleries table
```prisma
@@index([userId])       // User's galleries
@@index([isPublic])     // Public galleries
```

## How to Apply Indexes

1. Add `@@index([...])` to your Prisma schema
2. Run `npx prisma migrate dev --name add_indexes`
3. Prisma generates the migration SQL
4. For LARGE tables (millions of rows), create indexes concurrently:
   ```sql
   CREATE INDEX CONCURRENTLY "Image_userId_createdAt_idx" ON "Image"("userId", "createdAt");
   ```
   (Run this manually in production to avoid locking the table)

## Verify Indexes Are Used

```sql
-- Connect to your database
EXPLAIN ANALYZE SELECT * FROM "Image" WHERE "userId" = 'abc123' ORDER BY "createdAt" DESC LIMIT 20;

-- Look for "Index Scan" instead of "Seq Scan"
-- If you see "Seq Scan", your index is NOT being used
```

---

# 27. Seed Script (Comprehensive)

## What to Build

A complete seed script that creates realistic test data including users, images, galleries, and notifications.

## `prisma/seed.ts`

Create this file. It must create:

### Users (5 total)
1. **Admin user**: email=admin@app.com, password=Admin123!, role=ADMIN, plan=ENTERPRISE, emailVerified=true
2. **Pro user**: email=pro@app.com, password=ProUser123!, role=USER, plan=PRO, emailVerified=true
3. **Regular user**: email=user@app.com, password=User123!, role=USER, plan=FREE, emailVerified=true
4. **Unverified user**: email=unverified@app.com, password=Unverified123!, role=USER, plan=FREE, emailVerified=false
5. **Inactive user**: email=inactive@app.com, password=Inactive123!, role=USER, plan=FREE, emailVerified=true

Use `bcrypt.hash(password, 12)` for all passwords.

### Images (30 total)
- 8 images for admin (mix of models: 3 DALL-E, 2 Flux, 3 Pollinations)
- 10 images for pro user (mix of models and styles)
- 10 images for regular user (all Pollinations — free tier)
- 2 images public (isPublic: true) for explore
- Vary: prompts (use realistic AI art prompts), styles, sizes, dates (spread over last 30 days)
- Set isFavorite: true on 3-4 images per user
- Use Pollinations URLs for image URLs: `https://image.pollinations.ai/prompt/${encodeURIComponent(prompt)}?width=512&height=512&nologo=true&seed=${i}`

### Galleries (3 total)
- Gallery "My Favorites" for pro user with 4 favorite images
- Gallery "Landscapes" for regular user with 5 landscape images
- Gallery "Best Work" for admin with 6 best images, isPublic: true

### Notifications (10 total)
- 3 notifications for regular user (welcome, credit_low, generation_complete)
- 4 notifications for pro user (welcome, plan_upgraded, gallery_shared, generation_complete)
- 3 notifications for admin (system notifications)

### Run the seed

```bash
npx prisma db seed
```

Add to `package.json`:
```json
{
  "prisma": {
    "seed": "ts-node --compiler-options {\"module\":\"CommonJS\"} prisma/seed.ts"
  }
}
```

### Seed Script Structure

```typescript
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcryptjs';

const prisma = new PrismaClient();

async function main() {
  console.log('Seeding database...');

  // 1. Clean existing data (in order of dependencies)
  await prisma.galleryImage.deleteMany();
  await prisma.notification.deleteMany();
  await prisma.image.deleteMany();
  await prisma.gallery.deleteMany();
  await prisma.refreshToken.deleteMany();
  await prisma.passwordReset.deleteMany();
  await prisma.user.deleteMany();

  // 2. Create users
  const adminUser = await prisma.user.create({ ... });
  const proUser = await prisma.user.create({ ... });
  const regularUser = await prisma.user.create({ ... });
  // ... etc

  // 3. Create images for each user
  const adminImages = await Promise.all([
    prisma.image.create({ data: { userId: adminUser.id, prompt: 'A cyberpunk city...', model: 'DALLE3', ... } }),
    // ... more images
  ]);

  // 4. Create galleries
  const favoritesGallery = await prisma.gallery.create({ ... });
  await prisma.galleryImage.createMany({ data: [...] });

  // 5. Create notifications
  await prisma.notification.createMany({ data: [...] });

  console.log('✅ Seed complete!');
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

---

# SUMMARY: Build Order

Follow this order when implementing. Each phase depends on the previous:

```
Phase 1:  Project scaffold + TypeScript config       [1]
Phase 2:  Database schema + Prisma + seed data       [2, 27]
Phase 3:  Environment validation (Zod)               [6]
Phase 4:  Express app + middleware                    [1, 4, 7, 8]
Phase 5:  Auth module (signup, login, refresh)        [1]
Phase 6:  Image CRUD module                           [1]
Phase 7:  Gallery + Explore modules                   [23, 25]
Phase 8:  Storage service (S3/R2)                     [7]
Phase 9:  Queue service + Worker                       [13]
Phase 10: WebSocket                                    [12]
Phase 11: Email (Resend)                              [9]
Phase 12: Notification system                          [24]
Phase 13: Billing (Stripe)                             [14]
Phase 14: OAuth (Google)                              [21]
Phase 15: 2FA (TOTP)                                  [22]
Phase 16: Nginx + SSL                                 [5, 16]
Phase 17: Docker + Docker Compose                      [3]
Phase 18: CI/CD GitHub Actions                         [4]
Phase 19: Deployment configs                           [9]
Phase 20: Logging + Monitoring                         [20]
Phase 21: API documentation                            [17]
Phase 22: PWA service worker                           [19]
Phase 23: Load testing                                 [18]
Phase 24: Database indexing                            [26]
Phase 25: Migration rollback scripts                   [15]
Phase 26: .env.example                                 [10]
Phase 27: README                                       [11]
```

Total: 27 modules. ~4-6 weeks for one developer to build all of this to production quality.
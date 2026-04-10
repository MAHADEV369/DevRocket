# Skill 28: .env.example and README Documentation

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 1 hour
Depends On: 01

## Input Contract
- Skill 01 complete: working Express + TypeScript project scaffolded
- All subsequent skills implemented (02-27): full application with auth, images, gallery, storage, queue, websocket, email, billing, Docker, CI/CD, etc.
- All environment variables known from implemented features
- All scripts and commands known from package.json and previous skills

## Output Contract
- `.env.example` with ALL environment variables documented with comments, grouped by category, with example values
- `README.md` with project overview, architecture diagram (ASCII), quick start guide, available scripts, project structure, API endpoints table, auth flow explanation, environment variables reference, database commands, Docker commands, testing commands, and deployment instructions

## Files to Create

| File | Description |
|------|-------------|
| `.env.example` | Full environment variables template with comments |
| `README.md` | Complete project documentation |

## Steps

### Step 1: Create .env.example

`.env.example`:

```env
# =============================================================================
# Application
# =============================================================================
NODE_ENV=development                  # development | production | test
PORT=3000                             # Server port
API_PREFIX=/api                       # API route prefix
APP_URL=http://localhost:3000         # Public app URL (for email links, CORS)
CORS_ORIGIN=http://localhost:5173     # Allowed CORS origins (comma-separated)

# =============================================================================
# Database (PostgreSQL)
# =============================================================================
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/fastdepo?schema=public
# Full PostgreSQL connection string
# Format: postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=SCHEMA

# =============================================================================
# Redis
# =============================================================================
REDIS_URL=redis://localhost:6379      # Redis connection string
# Used for: session storage, queue backend, rate limiting, WebSocket state

# =============================================================================
# Authentication (JWT)
# =============================================================================
JWT_SECRET=change-me-to-a-64-char-random-string-in-production
# Secret key for signing access tokens. Must be 64+ characters in production.
# Generate with: openssl rand -hex 64

JWT_REFRESH_SECRET=change-me-to-a-different-64-char-random-string
# Secret key for signing refresh tokens. Must differ from JWT_SECRET.

JWT_EXPIRES_IN=15m                   # Access token expiration (e.g., 15m, 1h, 7d)
JWT_REFRESH_EXPIRES_IN=7d             # Refresh token expiration

# =============================================================================
# OAuth (Google)
# =============================================================================
GOOGLE_CLIENT_ID=your-google-client-id.apps.googleusercontent.com
# Google OAuth 2.0 client ID from Google Cloud Console

GOOGLE_CLIENT_SECRET=your-google-client-secret
# Google OAuth 2.0 client secret

GOOGLE_CALLBACK_URL=http://localhost:3000/api/auth/google/callback
# Google OAuth redirect URI (must match Google Cloud Console)

# =============================================================================
# Two-Factor Authentication (TOTP)
# =============================================================================
TOTP_ISSUER=FastDepo                  # Name shown in authenticator apps
TOTP_SECRET_LENGTH=32                 # Length of generated TOTP secrets

# =============================================================================
# Email (SMTP)
# =============================================================================
SMTP_HOST=smtp.resend.com             # SMTP server hostname
SMTP_PORT=465                         # SMTP port (465 for SSL, 587 for TLS)
SMTP_USER=your-smtp-username          # SMTP authentication username
SMTP_PASS=your-smtp-password          # SMTP authentication password
SMTP_FROM=noreply@yourdomain.com       # Default sender email address
SMTP_SECURE=true                      # Use SSL/TLS (true for port 465)

# =============================================================================
# AI Image Providers
# =============================================================================
OPENAI_API_KEY=sk-your-openai-api-key
# OpenAI API key for DALL-E image generation
# Get from: https://platform.openai.com/api-keys

FLUX_API_KEY=your-flux-api-key        # Flux provider API key (if used)
FLUX_API_URL=https://api.flux.ai/v1   # Flux API base URL

# Pollinations is free and requires no API key
# POLLINATIONS_API_URL defaults to https://image.pollinations.ai

# =============================================================================
# Storage (AWS S3 / compatible)
# =============================================================================
STORAGE_PROVIDER=s3                   # s3 | local (local for development only)
AWS_S3_BUCKET=fastdepo-images         # S3 bucket name
AWS_S3_REGION=us-east-1               # S3 bucket region
AWS_ACCESS_KEY_ID=your-access-key     # AWS access key with S3 permissions
AWS_SECRET_ACCESS_KEY=your-secret-key # AWS secret key
AWS_S3_PUBLIC_URL=https://fastdepo-images.s3.amazonaws.com
# Public URL for S3 objects (can be CloudFront URL in production)

# Local storage (only used when STORAGE_PROVIDER=local)
LOCAL_STORAGE_PATH=./uploads          # Path for local file storage

# =============================================================================
# Billing (Stripe)
# =============================================================================
STRIPE_SECRET_KEY=sk_test_your-stripe-secret-key
# Stripe secret key (use sk_test_ for dev, sk_live_ for production)

STRIPE_PUBLISHABLE_KEY=pk_test_your-stripe-publishable-key
# Stripe publishable key (for frontend Stripe.js integration)

STRIPE_WEBHOOK_SECRET=whsec_your-webhook-secret
# Stripe webhook signing secret for endpoint verification

STRIPE_PRICE_ID_PRO=price_your-pro-price-id
# Stripe Price ID for the Pro subscription plan

STRIPE_PRICE_ID_ENTERPRISE=price_your-enterprise-price-id
# Stripe Price ID for the Enterprise subscription plan (if applicable)

# =============================================================================
# Rate Limiting
# =============================================================================
RATE_LIMIT_WINDOW_MS=900000           # Rate limit window (15 min in ms)
RATE_LIMIT_MAX_REQUESTS=100           # Max requests per window per IP

# =============================================================================
# Image Generation Limits
# =============================================================================
MAX_IMAGE_GENERATIONS_PER_DAY=50      # Free tier daily limit
MAX_IMAGE_GENERATIONS_PRO=500          # Pro tier daily limit
MAX_IMAGE_WIDTH=2048                   # Maximum image width in pixels
MAX_IMAGE_HEIGHT=2048                  # Maximum image height in pixels

# =============================================================================
# Queue (Bull)
# =============================================================================
QUEUE_CONCURRENCY=5                   # Number of concurrent queue workers
QUEUE_ATTEMPTS=3                      # Max retry attempts for failed jobs
QUEUE_BACKOFF_DELAY=5000              # Delay between retry attempts (ms)

# =============================================================================
# WebSocket
# =============================================================================
WS_HEARTBEAT_INTERVAL=30000           # WebSocket heartbeat interval (ms)
WS_MAX_CONNECTIONS=1000                # Max concurrent WebSocket connections

# =============================================================================
# Logging
# =============================================================================
LOG_LEVEL=info                         # debug | info | warn | error
LOG_FORMAT=dev                         # dev | json (use json in production)

# =============================================================================
# Monitoring (Sentry - optional)
# =============================================================================
SENTRY_DSN=                            # Sentry DSN for error tracking (leave empty to disable)
SENTRY_TRACES_SAMPLE_RATE=0.1          # Percentage of transactions to trace (0.0-1.0)
SENTRY_ENVIRONMENT=development         # environment label in Sentry

# =============================================================================
# Docker (used by docker-compose)
# =============================================================================
POSTGRES_USER=postgres                 # PostgreSQL user (Docker)
POSTGRES_PASSWORD=postgres             # PostgreSQL password (Docker)
POSTGRES_DB=fastdepo                   # PostgreSQL database name (Docker)
REDIS_PASSWORD=                         # Redis password (leave empty for local dev)

# =============================================================================
# Test Environment
# =============================================================================
TEST_DATABASE_URL=postgresql://postgres:postgres@localhost:5432/fastdepo_test?schema=public
# Separate test database to avoid polluting development data
```

### Step 2: Create README.md

`README.md`:

```markdown
# FastDepo - AI Image Generator

Full-stack application for generating images using AI (DALL-E, Flux, Pollinations), with user authentication, image galleries, billing, and multi-platform support.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Applications                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   React   │  │  Flutter │  │  Tauri   │  │   PWA    │       │
│  │   Web     │  │  Android │  │  macOS   │  │  Mobile  │       │
│  └─────┬─────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘       │
│        │              │              │              │             │
│        └──────────────┴──────┬───────┴──────────────┘             │
└──────────────────────────────┼───────────────────────────────────┘
                                │
                          ┌─────▼─────┐
                          │  Nginx    │
                          │ (SSL/CORS)│
                          └─────┬─────┘
                                │
┌───────────────────────────────┼───────────────────────────────────┐
│                       Backend Services                           │
│                          ┌─────▼─────┐                           │
│                          │  Express  │                           │
│                          │   API     │                           │
│                          └─────┬─────┘                           │
│                                │                                  │
│       ┌────────────┬──────────┼──────────┬────────────┐          │
│       │            │          │          │            │          │
│  ┌────▼────┐ ┌─────▼────┐ ┌──▼───┐ ┌───▼────┐ ┌────▼────┐    │
│  │   Auth   │ │  Images  │ │Gallery│ │Billing │ │WebSocket│    │
│  │ Module   │ │  Module  │ │Module │ │(Stripe)│ │  (WS)   │    │
│  └────┬─────┘ └─────┬────┘ └───┬───┘ └────┬───┘ └────┬────┘    │
│       │              │          │          │          │          │
│  ┌────▼──────────────▼──────────▼──────────▼──────────▼────┐    │
│  │                    Prisma ORM                            │    │
│  └─────────────────────────┬───────────────────────────────┘    │
│                            │                                      │
│  ┌────────────┐     ┌─────▼─────┐     ┌────────────┐            │
│  │   Redis    │     │ PostgreSQL │     │  S3/Local  │            │
│  │  (Queue/   │     │  (Data)    │     │  (Images)  │            │
│  │   Cache)   │     │            │     │            │            │
│  └────────────┘     └────────────┘     └────────────┘            │
│                                                                   │
│  ┌────────────┐     ┌────────────┐     ┌────────────┐            │
│  │   Bull     │     │   Email    │     │  Cron      │            │
│  │  (Worker)  │     │ (Resend)   │     │  (Jobs)    │            │
│  └────────────┘     └────────────┘     └────────────┘            │
└───────────────────────────────────────────────────────────────────┘
```

## Quick Start

### Prerequisites

- Node.js >= 18
- PostgreSQL >= 14
- Redis >= 7
- Docker & Docker Compose (optional, for containerized setup)

### 1. Clone and Install

```bash
git clone https://github.com/your-org/fastdepo.git
cd fastdepo
npm install
```

### 2. Configure Environment

```bash
cp .env.example .env
# Edit .env with your actual values
```

Minimum required values to get started:
- `DATABASE_URL` - PostgreSQL connection string
- `JWT_SECRET` and `JWT_REFRESH_SECRET` - Generate with `openssl rand -hex 64`
- `REDIS_URL` - Redis connection string

### 3. Setup Database

```bash
# Run migrations
npx prisma migrate dev

# Seed with sample data
npx prisma db seed
```

### 4. Start Development Server

```bash
npm run dev
```

The API will be available at `http://localhost:3000/api`.

### 5. Start Frontend (separate terminal)

```bash
cd client
npm install
npm run dev
```

The frontend will be available at `http://localhost:5173`.

## Available Scripts

| Command | Description |
|---------|-------------|
| `npm run dev` | Start backend in development mode with hot reload |
| `npm run build` | Build backend TypeScript to `dist/` |
| `npm start` | Start production server |
| `npm run lint` | Run ESLint on all source files |
| `npm run typecheck` | Run TypeScript type checking |
| `npm test` | Run all tests |
| `npm run test:watch` | Run tests in watch mode |
| `npm run test:coverage` | Run tests with coverage report |
| `npm run test:e2e` | Run end-to-end tests (Playwright) |
| `npx prisma migrate dev` | Create and apply a new migration |
| `npx prisma migrate deploy` | Apply migrations in production |
| `npx prisma studio` | Open Prisma database browser |
| `npx prisma db seed` | Seed database with sample data |

### Frontend Scripts (in `client/`)

| Command | Description |
|---------|-------------|
| `npm run dev` | Start Vite dev server |
| `npm run build` | Build production bundle |
| `npm run preview` | Preview production build |
| `npm run lint` | Run ESLint |
| `npm run typecheck` | Run TypeScript checking |

### Flutter App Scripts (in `app/`)

| Command | Description |
|---------|-------------|
| `flutter run` | Run app in debug mode |
| `flutter build apk` | Build debug APK |
| `flutter build appbundle` | Build release AAB for Play Store |
| `flutter analyze` | Run static analysis |
| `flutter test` | Run Flutter tests |

## Project Structure

```
fastdepo/
├── src/                          # Backend source
│   ├── config/                   # Configuration modules
│   │   ├── index.ts              # Central config (env vars)
│   │   ├── database.ts           # Prisma client singleton
│   │   └── redis.ts              # Redis client singleton
│   ├── modules/                  # Feature modules
│   │   ├── auth/                 # Authentication
│   │   │   ├── types.ts          # Type definitions
│   │   │   ├── validation.ts     # Zod schemas
│   │   │   ├── service.ts        # Business logic
│   │   │   ├── controller.ts     # Route handlers
│   │   │   └── routes.ts         # Express router
│   │   ├── images/               # Image generation & CRUD
│   │   │   ├── providers/        # AI provider implementations
│   │   │   └── ...
│   │   ├── gallery/              # Gallery management
│   │   ├── billing/              # Stripe billing
│   │   ├── notifications/        # Push notifications
│   │   └── email/                # Email service
│   ├── middleware/                # Express middleware
│   │   ├── auth.ts               # JWT authentication
│   │   ├── rateLimiter.ts        # Rate limiting
│   │   └── errorHandler.ts       # Global error handler
│   ├── utils/                    # Shared utilities
│   │   ├── errors.ts             # Custom error classes
│   │   ├── crypto.ts             # JWT & token utilities
│   │   └── logger.ts             # Winston logger
│   ├── queue/                    # Bull queue processors
│   ├── websockets/                # WebSocket handlers
│   ├── app.ts                    # Express app setup
│   └── server.ts                 # Server entry point
├── prisma/                       # Database
│   ├── schema.prisma             # Database schema
│   ├── seed.ts                   # Seed data
│   └── migrations/               # Migration files
├── client/                       # React frontend
│   ├── src/
│   │   ├── api/                  # API client & endpoints
│   │   ├── auth/                 # Auth context & provider
│   │   ├── hooks/                # TanStack Query hooks
│   │   ├── stores/               # Zustand state stores
│   │   ├── components/           # UI components
│   │   │   ├── ui/               # Base UI components
│   │   │   └── features/         # Feature components
│   │   ├── pages/                # Route pages
│   │   └── styles/              # Global styles
│   ├── src-tauri/                # Tauri desktop app
│   └── ...
├── app/                          # Flutter Android app
│   ├── lib/
│   │   ├── core/                 # Core (network, storage, theme)
│   │   └── features/             # Feature modules (auth, generate, gallery)
│   └── ...
├── docker-compose.yml            # Production Docker composition
├── docker-compose.dev.yml        # Development Docker overrides
├── Dockerfile                    # Multi-stage production build
├── railway.toml                  # Railway deployment config
├── fly.toml                      # Fly.io deployment config
├── .env.example                  # Environment variables template
└── README.md                     # This file
```

## API Endpoints

### Authentication

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/auth/register` | Register new user | No |
| POST | `/api/auth/login` | Login with email/password | No |
| POST | `/api/auth/refresh` | Refresh access token | No |
| POST | `/api/auth/logout` | Logout and invalidate tokens | Yes |
| GET | `/api/auth/me` | Get current user profile | Yes |
| POST | `/api/auth/verify-email` | Verify email address | No |
| POST | `/api/auth/forgot-password` | Request password reset | No |
| POST | `/api/auth/reset-password` | Reset password with token | No |
| GET | `/api/auth/google` | Initiate Google OAuth | No |
| GET | `/api/auth/google/callback` | Google OAuth callback | No |
| POST | `/api/auth/2fa/enable` | Enable TOTP 2FA | Yes |
| POST | `/api/auth/2fa/verify` | Verify TOTP code | Yes |
| POST | `/api/auth/2fa/disable` | Disable TOTP 2FA | Yes |

### Images

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/images/generate` | Generate image from prompt | Yes |
| GET | `/api/images` | List user's images (paginated) | Yes |
| GET | `/api/images/:id` | Get single image | Yes |
| DELETE | `/api/images/:id` | Delete image | Yes |
| PATCH | `/api/images/:id/favorite` | Toggle favorite status | Yes |
| GET | `/api/images/:id/share` | Get shareable link | Yes |

### Gallery

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/gallery` | Create gallery | Yes |
| GET | `/api/gallery` | List user's galleries | Yes |
| GET | `/api/gallery/:id` | Get gallery with images | Yes |
| PATCH | `/api/gallery/:id` | Update gallery | Yes |
| DELETE | `/api/gallery/:id` | Delete gallery | Yes |
| POST | `/api/gallery/:id/images` | Add image to gallery | Yes |
| DELETE | `/api/gallery/:id/images/:imageId` | Remove image from gallery | Yes |
| GET | `/api/gallery/explore` | Browse public galleries | No |

### Billing

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/billing/plans` | List available plans | No |
| POST | `/api/billing/checkout` | Create Stripe checkout session | Yes |
| POST | `/api/billing/portal` | Create Stripe portal session | Yes |
| GET | `/api/billing/subscription` | Get current subscription | Yes |
| POST | `/api/billing/webhook` | Stripe webhook endpoint | No |

### Notifications

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| GET | `/api/notifications` | List user notifications | Yes |
| PATCH | `/api/notifications/:id/read` | Mark notification as read | Yes |
| PATCH | `/api/notifications/read-all` | Mark all as read | Yes |

## Authentication Flow

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Client  │     │   API    │     │ Database │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
     │  POST /login   │                │
     │───────────────>│                │
     │                │  Find user     │
     │                │───────────────>│
     │                │  User found    │
     │                │<───────────────│
     │                │                │
     │                │  Verify password (bcrypt)     │
     │                │                │
     │                │  Create tokens│
     │                │───────────────>│ (save refresh token)
     │  { accessToken, refreshToken }  │
     │<───────────────│                │
     │                │                │
     │  GET /api/images (with Bearer)  │
     │───────────────>│                │
     │                │  Verify JWT    │
     │                │  (no DB call)  │
     │  200 OK        │                │
     │<───────────────│                │
     │                │                │
     │  401 (expired) │                │
     │<───────────────│                │
     │                │                │
     │  POST /refresh │                │
     │───────────────>│                │
     │                │  Verify refresh token        │
     │                │───────────────>│
     │                │  Valid         │
     │                │<───────────────│
     │                │  Rotate tokens │
     │                │───────────────>│ (invalidate old, save new)
     │  { accessToken, refreshToken }  │
     │<───────────────│                │
     │                │                │
     │  POST /logout  │                │
     │───────────────>│                │
     │                │  Delete refresh token        │
     │                │───────────────>│
     │  200 OK        │                │
     │<───────────────│                │
```

## Environment Variables

See [`.env.example`](.env.example) for a complete list of environment variables with descriptions and example values.

Key variables:

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `REDIS_URL` | Yes | Redis connection string |
| `JWT_SECRET` | Yes | Access token signing secret (64+ chars) |
| `JWT_REFRESH_SECRET` | Yes | Refresh token signing secret (64+ chars) |
| `OPENAI_API_KEY` | Yes | For DALL-E image generation |
| `STORAGE_PROVIDER` | Yes | `s3` or `local` |
| `AWS_S3_BUCKET` | If S3 | S3 bucket name for image storage |
| `SMTP_HOST` | Yes | SMTP server for sending emails |
| `STRIPE_SECRET_KEY` | If billing | Stripe API key |

## Database Commands

```bash
# Create a new migration after schema changes
npx prisma migrate dev --name describe_the_change

# Apply all pending migrations
npx prisma migrate deploy

# Reset database (WARNING: deletes all data)
npx prisma migrate reset

# Seed database with sample data
npx prisma db seed

# Open Prisma Studio (visual database browser)
npx prisma studio

# Generate Prisma client after schema changes
npx prisma generate

# View migration status
npx prisma migrate status
```

## Docker Commands

```bash
# Start all services (API, worker, PostgreSQL, Redis)
docker-compose up -d

# Start with development overrides (hot reload)
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# View logs
docker-compose logs -f api

# Stop all services
docker-compose down

# Stop and remove volumes (WARNING: deletes all data)
docker-compose down -v

# Rebuild images after dependency changes
docker-compose build --no-cache

# Run database migrations in Docker
docker-compose exec api npx prisma migrate deploy

# Seed database in Docker
docker-compose exec api npx prisma db seed

# Open a shell in the API container
docker-compose exec api sh

# Check service health
docker-compose ps
```

## Testing Commands

```bash
# Run all unit and integration tests
npm test

# Run tests in watch mode
npm run test:watch

# Run tests with coverage report
npm run test:coverage

# Run E2E tests (requires frontend and backend running)
npm run test:e2e

# Run specific test file
npx vitest run src/modules/auth/__tests__/auth.service.test.ts

# Run Flutter tests
cd app && flutter test

# Run frontend type checking
cd client && npm run typecheck

# Run backend type checking
npm run typecheck

# Lint all code
npm run lint
cd client && npm run lint
```

## Deployment

### Railway

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login and deploy
railway login
railway init
railway add --plugin postgresql
railway add --plugin redis
railway variables set JWT_SECRET=$(openssl rand -hex 64)
railway variables set JWT_REFRESH_SECRET=$(openssl rand -hex 64)
# ... set all required env variables
railway up
railway run npx prisma migrate deploy
railway run npx prisma db seed
```

### Fly.io

```bash
# Install Fly CLI
curl -L https://fly.io/install.sh | sh

# Launch and deploy
fly auth login
fly launch --dockerfile Dockerfile --name fastdepo --region sjc
fly postgres create
fly postgres attach fastdepo-db
fly redis create
fly redis attach fastdepo-redis
fly secrets set JWT_SECRET=$(openssl rand -hex 64) JWT_REFRESH_SECRET=$(openssl rand -hex 64) ...
fly deploy
```

See `skills/22-deployment.md` for detailed deployment instructions.

## License

Private - All rights reserved.
```

## Verification

```bash
# 1. Verify .env.example has no actual secrets
grep -i "password\|secret\|key" .env.example | grep -v "your-\|change-me\|sk_test\|pk_test\|placeholder"
# Expected: No output (no real secrets)

# 2. Verify .env.example is tracked in git but .env is not
git ls-files .env.example  # Should show the file
git ls-files .env          # Should show nothing
cat .gitignore | grep ".env"  # Should show ".env" entry

# 3. Verify README links work
# Check that any relative links in README resolve correctly:
ls skills/22-deployment.md    # Should exist
ls .env.example                # Should exist

# 4. Verify all environment variables in code are documented in .env.example
# Search for process.env references in src/ and cross-reference with .env.example
grep -rn "process.env" src/ --include="*.ts" | sed 's/.*process.env\.\([A-Z_]*\).*/\1/' | sort -u
# Compare output with variables in .env.example

# 5. Verify README renders correctly
# Use a markdown renderer to preview:
npx markdown-it README.md > /dev/null
# Or preview on GitHub

# 6. Count environment variables in .env.example
grep -c "^[A-Z_]" .env.example
# Expected: 40+ variables documented

# 7. Verify no sensitive data in README
grep -i "password=\|secret=\|api_key=\|token=" README.md
# Expected: Only placeholder/example values
```

## Rollback

```bash
# Remove the created files
rm .env.example
rm README.md

# Restore from git if needed
git checkout .env.example README.md
```

These files are documentation only, so rollback is straightforward — delete the files or restore previous versions from git.

## ADR-028: Comprehensive .env.example and README as Living Documentation

**Decision**: Create a single comprehensive `.env.example` with all environment variables grouped by category, and a `README.md` that serves as the single source of truth for project documentation including architecture, setup, commands, and API reference.

**Reason**: A complete `.env.example` eliminates the "which env vars do I need?" problem for new developers. Grouping by category makes it scannable. The README consolidates all essential project information in one place, reducing onboarding time and preventing knowledge from being siloed in team members' heads. Including an ASCII architecture diagram gives immediate visual context.

**Consequences**:
- Both files must be kept up-to-date when adding new features or env vars
- README will grow large — considered acceptable because developers can search it
- .env.example must never contain real secrets (only placeholder values)
- The architecture diagram must be updated when services change
- CI should validate that env vars in code match .env.example

**Alternatives Considered**:
- Separate docs/ directory with multiple markdown files: Better organization but harder to find the "quick start"
- OpenAPI documentation only: Doesn't cover setup, deployment, or architecture
- Notion/Confluence wiki: External, can go stale, not version-controlled
- Minimal README pointing to wiki: Fragments information, harder to maintain consistently
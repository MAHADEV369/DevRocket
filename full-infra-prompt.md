# The Only Prompt You Need to Build Full App Infra — The Right Way

---

## COPY-PASTE THE PROMPT BELOW

---

```
You are a senior full-stack architect and DevOps engineer. Build my complete application infrastructure from scratch, end-to-end, with production-grade quality. Do NOT skip steps. Do NOT leave placeholders. Do NOT leave TODOs. Every file must be complete and runnable.

## PROJECT OVERVIEW

- **App Name**: [YOUR_APP_NAME]
- **One-line Description**: [ONE LINE DESCRIPTION OF WHAT THE APP DOES]
- **Target Platform**: [web / mobile / desktop / all]
- **Frontend Framework**: [React / Next.js / Flutter / Vue / Svelte]
- **Backend Framework**: [Node.js+Express / FastAPI / NestJS / Django / Go+Gin]
- **Database**: [PostgreSQL / MongoDB / MySQL / SQLite / Firestore]
- **Auth Provider**: [Firebase Auth / Supabase Auth / Auth0 / Custom JWT / Clerk]
- **Deployment Target**: [Vercel + Railway / AWS / GCP / Fly.io / Docker VPS]
- **Repo URL**: [YOUR_REPO_URL or "new repo"]

## PHASE 0 — REPO & MONOREPO SCAFFOLD

1. Initialize repo with proper `.gitignore` (node, python, os, env, build, IDE)
2. Set up monorepo structure:

```
/
├── apps/
│   ├── web/          # Frontend app
│   ├── mobile/       # Mobile app (if applicable)
│   └── api/          # Backend API
├── packages/
│   ├── shared/       # Shared types, utils, constants
│   ├── ui/           # Shared UI components (if web+mobile share)
│   └── config/       # Shared ESLint, TSConfig, etc.
├── infra/            # IaC (Docker, Terraform, etc.)
├── scripts/          # Build, deploy, seed scripts
├── .github/          # CI/CD workflows
│   └── workflows/
├── docs/             # API docs, architecture decisions
├── docker-compose.yml
├── turbo.json        # or nx.json
├── package.json      # Root workspace config
├── .env.example      # Template for all env vars
├── .env.schema       # ENV validation schema
└── README.md
```

3. Configure workspace tooling: Turborepo or Nx (pick one)
4. Set up TypeScript across ALL packages — strict mode, path aliases
5. Create root `package.json` with workspaces, scripts for `dev`, `build`, `lint`, `test`, `typecheck`, `format`, `db:push`, `db:seed`, `db:reset`
6. Configure ESLint + Prettier at root level with consistent rules
7. Configure Husky pre-commit hooks: lint-staged (eslint --fix, prettier --write), typecheck
8. Configure commitlint (conventional commits enforced)
9. Create `.env.example` with ALL required env vars documented
10. Create `.env.schema` with Zod validation that runs on app startup
11. Add `editorconfig` + `.vscode/settings.json` + `.vscode/extensions.json`

## PHASE 1 — DATABASE & DATA LAYER

12. Set up database connection with connection pooling (pg-pool / Prisma / Drizzle / mongoose)
13. Create database migration system (Prisma migrate / Drizzle Kit / Alembic / Knex)
14. Define ALL database schemas/models:

```
For EVERY entity, define:
- Table/collection name
- All columns with types, constraints (NOT NULL, UNIQUE, DEFAULT, CHECK)
- Primary keys (UUID or auto-increment — pick one convention and STICK TO IT)
- Foreign keys with ON DELETE behavior (CASCADE, SET NULL, RESTRICT)
- Indexes (unique, composite, partial) — think about query patterns
- Enum types for status/state fields
- Timestamps: created_at, updated_at, deleted_at (soft delete)
- Audit fields: created_by, updated_by
- Version/optimistic locking field if needed
```

15. Create seed scripts with realistic faker data (at least 10 users, 50+ records per entity)
16. Create migration rollback scripts
17. Implement soft-delete pattern globally (deleted_at column, default scope filters)
18. Set up database-level constraints that enforce business rules (no app-layer-only validation)
19. Implement DB health check endpoint (`GET /api/health` → check DB connectivity)
20. Create `packages/shared/` types that mirror DB schemas (auto-generate if using Prisma/Drizzle)

## PHASE 2 — AUTHENTICATION & AUTHORIZATION

21. Implement auth system (pick one):
    - **Custom JWT**: access token (15min) + refresh token (7d) + rotation
    - **Session-based**: httpOnly secure cookie sessions
    - **OAuth**: Google, GitHub, Apple sign-in
22. Password hashing: bcrypt/argon2 with proper salt rounds
23. Implement signup flow: email verification, password strength validation (zxcvbn), rate limiting
24. Implement login flow: brute-force protection (rate limit 5 attempts/window), account lockout after 10 failures
25. Implement logout: invalidate refresh tokens server-side
26. Implement password reset: secure token (crypto.randomBytes), 1-hour expiry, single-use
27. Implement email verification: send on signup, resend endpoint, block unverified users from protected routes
28. Implement OAuth2 flows (Google, GitHub minimum) with proper CSRF state parameter
29. Implement refresh token rotation: one-time-use refresh tokens, detect theft (reuse detection → revoke all user tokens)
30. Implement RBAC middleware: roles (admin, user, etc.) with route-level guards
31. Implement resource-level authorization: users can only access their own resources (tenant isolation)
32. Create auth middleware that attaches `req.user` to all protected routes
33. Implement CSRF protection for cookie-based auth (double-submit cookie or CSRF token)
34. Add security headers middleware (Helmet.js equivalent): CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy
35. Store all tokens securely: refresh tokens in DB (hashed), access tokens stateless (JWT), never in localStorage for web
36. Implement session device management: list active sessions, revoke by device
37. Add auth rate limiting on ALL auth endpoints (login, signup, reset, resend-verification)
38. Implement 2FA/TOTP as optional (generate QR, verify code, backup codes)

## PHASE 3 — API DESIGN & IMPLEMENTATION

39. Structure API routes with versioning: `/api/v1/...`
40. Implement proper REST conventions:
    - `GET /resources` → list (with pagination, filtering, sorting)
    - `POST /resources` → create
    - `GET /resources/:id` → get one
    - `PATCH /resources/:id` → update
    - `DELETE /resources/:id` → soft delete
41. Implement pagination: cursor-based for large datasets, offset-based for small. Include `total`, `hasMore`, `nextCursor`/`nextOffset`
42. Implement filtering: query parameter-based (`?status=active&role=admin`)
43. Implement sorting: `?sort=created_at:desc,name:asc`
44. Implement field selection: `?fields=id,name,email` (avoid over-fetching)
45. Implement consistent error response format:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable message",
    "details": [
      { "field": "email", "message": "Invalid email format" }
    ],
    "requestId": "uuid-for-tracing"
  }
}
```

46. Implement Zod/Joi/Valibot validation on EVERY endpoint — validate body, query, params, headers
47. Create request ID middleware (generate UUID per request, include in all logs and error responses)
48. Implement response format:

```json
{
  "data": { ... },
  "meta": { "page": 1, "limit": 20, "total": 100 }
}
```

49. Implement file upload with:
    - Multer/Formidable with size limits (configurable)
    - MIME type validation (check magic bytes, not just extension)
    - Virus scanning (ClamAV integration or skip for dev)
    - Upload to S3/R2/GCS with signed URLs for retrieval
    - Image processing (resize, compress, thumbnail generation) using Sharp
50. Implement rate limiting per-route AND global:
    - Global: 100 req/min per IP
    - Auth routes: 5 req/min per IP
    - Create/Write routes: 30 req/min per user
    - Use Redis-backed rate limiter (not in-memory for production)
51. Implement API request logging middleware: method, path, status, duration, userId, requestId, error (if any)
52. Create OpenAPI/Swagger documentation auto-generated from routes/schemas
53. Implement API versioning strategy (header-based or URL-based)
54. Implement CORS properly: whitelist specific origins, credentials: true, specific methods/headers
55. Implement request body size limits globally and per-route
56. Implement response compression (gzip/brotli)
57. Add request timeout middleware (30s default, configurable)

## PHASE 4 — BUSINESS LOGIC (Service Layer)

58. Implement clean architecture layers:
    ```
    Route → Controller → Service → Repository/Model
    ```
    - Route: HTTP mapping, middleware attachment
    - Controller: Request parsing, response formatting, error catching
    - Service: Business logic, validation, orchestration
    - Repository: Data access, query building, DB operations
59. EVERY service method must be in a separate file/class — no god services
60. Implement transaction wrapping for multi-step DB operations (use DB transactions, not eventual consistency where ACID is needed)
61. Implement idempotency keys for POST endpoints that create resources (prevent duplicate submissions)
62. Implement background job queue (BullMQ / Agenda / Celery) for:
    - Email sending
    - File processing
    - Webhook delivery
    - Scheduled/cron tasks
63. Implement event emitter pattern for side effects (emit event → listener handles async work)
64. Implement optimistic concurrency control where applicable (version field)
65. Implement caching layer (Redis) for:
    - Session data
    - Rate limit counters
    - Frequently accessed, rarely changed data (invalidate on write)
66. Create health check that verifies: DB, Redis, external APIs, disk space, memory
67. Implement graceful shutdown: stop accepting new requests, finish in-flight, close DB connections, notify load balancer
68. Add Winston/Pino structured JSON logging with log levels configurable via ENV
69. Create a notification/email service abstraction (SendGrid / Resend / SES / SMTP) with delivery tracking
70. Implement webhook system for external integrations (with retry, exponential backoff, dead letter queue)

## PHASE 5 — FRONTEND ARCHITECTURE

71. Set up frontend with React/Next.js/Flutter (per choice) with TypeScript STRICT mode
72. Implement folder structure:

```
src/
├── app/ or pages/       # Route-level components
├── components/
│   ├── ui/              # Primitive UI (Button, Input, Modal, Toast)
│   ├── layout/          # Header, Sidebar, Footer, Shell
│   ├── forms/           # Reusable form fields with validation
│   └── features/        # Feature-specific components
├── hooks/               # Custom React hooks
├── lib/                 # Third-party config (axios instance, dayjs, etc.)
├── services/            # API client functions (one per resource)
├── stores/              # State management (Zustand / Redux / Riverpod)
├── types/               # TypeScript types (auto-generated from OpenAPI preferred)
├── utils/               # Pure utility functions
├── constants/           # App-wide constants
└── styles/              # Global CSS, theme config
```

73. Set up state management:
    - Server state: TanStack Query (React Query) for ALL API data
    - Client state: Zustand (minimal, only UI state that isn't from server)
    - URL state: useSearchParams for filters, pagination, modals
    - Form state: React Hook Form + Zod resolver
74. Implement API client:
    - Axios instance with base URL, timeout, interceptors
    - Request interceptor: attach auth token (from secure storage)
    - Response interceptor: handle 401 → refresh token → retry, handle errors → toast
    - One service file per API resource with typed methods
75. Implement auth flow on frontend:
    - Login/signup forms with validation
    - Auth state persistence (secure cookie or in-memory + refresh)
    - Protected route wrapper
    - Redirect to login on 401 with return URL
    - Auto-refresh tokens before expiry
76. Implement error handling:
    - Global error boundary (React) or equivalent
    - Toast notifications for API errors
    - Error page (404, 500, 403)
    - Network error handling (offline detection, retry)
77. Implement loading states:
    - Suspense boundaries with skeletons
    - Optimistic updates for mutations (TanStack Query)
    - Progressive loading for image-heavy pages
78. Implement responsive design:
    - Mobile-first CSS
    - Breakpoint system (Tailwind breakpoints or custom)
    - Touch-friendly interactions (min 44px tap targets)
79. Implement accessibility:
    - Semantic HTML
    - ARIA labels on interactive elements
    - Keyboard navigation
    - Focus management (modals, route changes)
    - Color contrast ratios (WCAG AA minimum)
80. Implement dark mode:
    - CSS custom properties or Tailwind dark:
    - Persist preference (localStorage + system preference detection)
    - No flash of wrong theme (script in <head>)

## PHASE 6 — REAL-TIME & NOTIFICATIONS

81. Implement WebSocket server (Socket.IO / ws / SignalR):
    - Authenticated connections (verify JWT on handshake)
    - Room-based subscriptions (user joins their own room, project rooms)
    - Event types: notification, data-update, presence
82. Implement push notification system:
    - Web: Service Worker + Push API (VAPID keys)
    - Mobile: FCM (Firebase Cloud Messaging) or APNS
    - Store device tokens per user
    - Batch notification sending
83. Implement in-app notification system:
    - Notification model (type, title, body, link, read, createdAt)
    - Real-time badge count via WebSocket
    - Notification preferences (per-user, per-type opt-in/out)
84. Implement email notification templates:
    - Transactional (welcome, password reset, verification)
    - Notification digests (daily/weekly)
    - Use React Email / MJML for responsive templates

## PHASE 7 — TESTING

85. Set up test infrastructure:
    - Jest or Vitest (frontend + API)
    - Playwright or Cypress (E2E)
    - Testing Library (React component tests)
    - MSW (Mock Service Worker) for API mocking in frontend tests
86. Unit tests:
    - ALL service functions
    - ALL utility functions
    - ALL validation schemas
    - Target: 80%+ coverage on service/utils layers
87. Integration tests:
    - API endpoint tests with real DB (test database, not mocking)
    - Test: success case, validation errors, auth errors, rate limiting
    - Use setup/teardown for test data
88. E2E tests:
    - Critical user flows: signup → verify → login → CRUD → logout
    - OAuth flow
    - Password reset flow
    - File upload flow
89. Load/performance tests:
    - Use k6 or Artillery
    - Baseline: handle 100 concurrent users, <200ms p95 response time
90. Write test data factories (Fishery / custom) for consistent test data creation
91. Add pre-push hook that runs: lint, typecheck, unit tests (fail fast)

## PHASE 8 — SECURITY HARDENING

92. Implement security middleware stack (in order):
    - Helmet (security headers)
    - CORS (whitelisted origins)
    - Rate limiting (global + per-route)
    - CSRF protection (double-submit cookie)
    - Request sanitization (strip HTML/JS from inputs)
    - Compression
    - Request ID
    - Logging
    - Auth middleware
93. Implement input sanitization:
    - Escape HTML entities in all user input on output
    - Validate and sanitize ALL inputs server-side (never trust client)
    - Prevent SQL injection (use parameterized queries ALWAYS)
    - Prevent NoSQL injection (sanitize MongoDB queries)
    - Prevent command injection (never pass user input to exec/spawn)
94. Implement file upload security:
    - Whitelist allowed MIME types
    - Check file magic bytes, not just extension
    - Limit file size
    - Store files outside web root or in cloud storage
    - Serve files with `Content-Disposition: attachment` for downloads
    - Never use user-provided filenames directly
95. Configure dependency auditing:
    - `npm audit` / `pip-audit` in CI
    - Snyk or Dependabot for vulnerability scanning
    - Automerge patch-level security updates
96. Implement secrets management:
    - NEVER commit .env files
    - Use `.env.example` as template only
    - Reference secrets from environment, not hardcoded
    - In production: use cloud secret manager (AWS Secrets Manager, GCP Secret Manager, Doppler, Infisical)
97. Add Content Security Policy headers:
    - Strict CSP that prevents XSS
    - nonce-based for inline scripts
98. Implement API key management if building a public API:
    - Key generation, rotation, revocation
    - Usage tracking and quota enforcement
99. Add brute-force protection beyond rate limiting:
    - Exponential backoff on failed auth attempts
    - Account lockout after threshold
    - IP blocking for patterns
    - CAPTCHA after repeated failures (hCaptcha preferred — privacy-focused)
100. Security response headers checklist:
    - Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
    - X-Content-Type-Options: nosniff
    - X-Frame-Options: DENY
    - X-XSS-Protection: 0 (disabled — modern browsers handle this)
    - Referrer-Policy: strict-origin-when-cross-origin
    - Permissions-Policy: camera=(), microphone=(), geolocation=()

## PHASE 9 — DOCKER & CONTAINERIZATION

101. Create multi-stage Dockerfile for API:

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine AS runner
WORKDIR /app
RUN addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -s /bin/sh -D appuser
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
USER appuser
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

102. Create Dockerfile for frontend (multi-stage: build → nginx serve)
103. Create docker-compose.yml with ALL services:
    - api (backend)
    - web (frontend)
    - db (PostgreSQL/MongoDB)
    - redis (cache + queue)
    - worker (background job processor)
    - nginx (reverse proxy + SSL termination)
104. Create docker-compose.dev.yml for development (hot reload, volumes, debug ports)
105. Create docker-compose.test.yml for CI testing
106. Create .dockerignore (node_modules, .git, .env, build artifacts)
107. Add healthcheck to EVERY container in docker-compose
108. Create Makefile or taskfile.yml with common commands:

```makefile
dev          # Start all services locally
build        # Build all containers
test         # Run all tests
lint         # Run linters
db-reset     # Reset database
db-seed      # Seed database
logs         # Tail all logs
down         # Stop all services
```

## PHASE 10 — CI/CD PIPELINE

109. Create GitHub Actions workflow (`.github/workflows/ci.yml`):
    - Trigger: push to any branch, PRs to main
    - Steps: checkout → setup Node/Python → install deps → lint → typecheck → unit test → integration test → build
    - Cache: node_modules, build caches
    - Matrix: test on multiple Node versions (18, 20)
110. Create deployment workflow (`.github/workflows/deploy.yml`):
    - Trigger: push to main (staging), tag v* (production)
    - Build Docker images
    - Push to container registry (Docker Hub, ECR, GCR)
    - Deploy to staging/production
    - Run smoke tests
    - Rollback on failure
111. Create PR checks workflow:
    - Lint, typecheck, test
    - Preview deployment (Vercel preview / Railway PR env)
    - Bundle size check (limit size increase)
112. Create dependency review workflow:
    - Auto-approve and merge Dependabot PRs for patch updates
    - Block PRs that introduce known vulnerabilities
113. Implement environment-based configuration:
    - Development: local .env, hot reload, verbose logging, mock services
    - Staging: mirrors production, uses staging DB, real services
    - Production: minimal logging, optimized builds, real services

## PHASE 11 — MONITORING & OBSERVABILITY

114. Implement structured logging:
    - JSON format for production
    - Include: timestamp, level, message, requestId, userId, duration, error.stack
    - Use Pino (Node) or structlog (Python) for performance
115. Set up health and readiness endpoints:
    - `/health` → liveness (process is running)
    - `/ready` → readiness (DB connected, Redis connected, ready for traffic)
116. Integrate APM (choose one):
    - Sentry: error tracking, performance monitoring
    - Datadog: full observability
    - New Relic: APM + infra monitoring
    - Or: OpenTelemetry → Jaeger/Prometheus/Grafana (self-hosted)
117. Set up application metrics:
    - Request count, error rate, response time (p50, p95, p99)
    - Active users, signup rate
    - DB connection pool usage
    - Queue depth, job failure rate
    - Memory/CPU usage
118. Set up alerting:
    - Error rate > 1% → alert
    - Response time p95 > 500ms → alert
    - Health check failure → alert
    - Disk/CPU/Memory > 80% → alert
    - Use PagerDuty / Opsgenie / Slack webhook
119. Implement distributed tracing (OpenTelemetry):
    - Trace requests across services
    - Propagate trace context in headers
120. Set up log aggregation:
    - Forward all logs to centralized system (Loki, CloudWatch Logs, Datadog Logs)
    - Add correlation IDs across services

## PHASE 12 — PERFORMANCE OPTIMIZATION

121. Database optimization:
    - Add indexes on ALL frequently queried columns (check slow query log)
    - Use EXPLAIN ANALYZE on all queries
    - Implement query result caching (Redis)
    - Configure connection pooling (pg-pool with 10-20 connections)
    - Set up read replicas if read-heavy
122. API performance:
    - Enable response compression (gzip/brotli)
    - Implement HTTP caching headers (ETag, Cache-Control, Last-Modified)
    - Use pagination on ALL list endpoints (never return unbounded data)
    - Implement field projection (select only needed columns)
    - Add request deduplication for concurrent identical requests
123. Frontend performance:
    - Code splitting per route (lazy loading)
    - Image optimization (WebP, lazy loading, responsive srcset)
    - Font optimization (preload, font-display: swap)
    - Bundle analysis and size budget (<200KB initial JS)
    - Implement service worker for offline support and caching
    - Server-side rendering or static generation where beneficial
    - Prefetch/preload critical resources
124. CDN setup:
    - Serve static assets via CDN (CloudFront, Cloudflare, Vercel)
    - Cache static assets aggressively (1 year with content hash)
    - API responses: short cache with stale-while-revalidate

## PHASE 13 — DOCUMENTATION

125. Create `README.md` with:
    - Project description
    - Prerequisites (Node version, DB version)
    - Quick start (clone → env → install → db:push → dev)
    - Available scripts
    - Environment variables reference
    - Architecture overview diagram (Mermaid)
    - API documentation link
126. Create `CONTRIBUTING.md` with:
    - Branch naming convention
    - Commit message format
    - PR process
    - Code review guidelines
    - Testing requirements
127. Create API documentation:
    - OpenAPI spec auto-generated from code
    - Swagger UI hosted at `/api/docs`
    - Include request/response examples for every endpoint
    - Document all error codes and their meanings
128. Create architecture decision records (ADRs) in `docs/adr/`:
    - Numbered decisions (ADR-001, ADR-002, etc.)
    - Format: Context → Decision → Consequences
129. Create runbook in `docs/runbook.md`:
    - How to restart services
    - How to roll back deployments
    - How to investigate common errors
    - How to run manual migrations
    - Emergency contacts and escalation

## PHASE 14 — FINAL PRODUCTION CHECKLIST

130. Verify ALL environment variables are documented in `.env.example`
131. Verify `.env` is in `.gitignore` (NEVER committed)
132. Verify database migrations run cleanly on empty database (test with `db:push && db:seed`)
133. Verify all tests pass (`test`, `lint`, `typecheck`)
134. Verify Docker builds succeed (`docker compose build`)
135. Verify the app starts cleanly (`docker compose up` → all services healthy)
136. Verify health endpoints respond (`/health`, `/ready`)
137. Verify API documentation is accessible (`/api/docs`)
138. Verify HTTPS is configured (not HTTP in production)
139. Verify CORS is restrictive (not `*`)
140. Verify rate limiting is active on all routes
141. Verify authentication works end-to-end (signup → verify → login → refresh → logout)
142. Verify authorization works (user cannot access other user's resources)
143. Verify file uploads are secured (size limit, type check, no path traversal)
144. Verify error responses don't leak stack traces or internal details in production
145. Verify logging is structured JSON in production, pretty print in development
146. Verify graceful shutdown (SIGTERM → finish requests → close connections → exit)
147. Verify backup strategy for database (automated daily backups, tested restore)
148. Verify monitoring and alerting is active (Sentry, metrics, uptime checks)
149. Verify CI pipeline is green on all branches
150. Verify deployment pipeline works (push to main → staging auto-deploy, tag → prod deploy)

## MANDATORY TECHNOLOGY CONVENTIONS

- **ALL code in TypeScript/strict** (no `any`, no `// @ts-ignore`)
- **No console.log in production** — use structured logger
- **No hardcoded secrets** — all from environment
- **No TODO/FIXME** in final code — implement or document as issue
- **No placeholder data** — all endpoints return real data or proper 501
- **No `eval()`, `new Function()`, or dynamic code execution**
- **No SQL string concatenation** — parameterized queries only
- **Every API endpoint has validation** (input schema + output schema)
- **Every API endpoint has tests** (unit + integration minimum)
- **Every database write uses transactions** where multiple tables are affected
- **Every error is caught and handled** — no unhandled promise rejections
- **Every service has a health check** that verifies its dependencies
- **Git history must be clean** — conventional commits, no force-push to main

## APP-SPECIFIC ENTITIES & FEATURES

[DESCRIBE YOUR APP'S SPECIFIC ENTITIES, FEATURES, AND BUSINESS RULES HERE]
[EXAMPLE: "TaskSync has Tasks (title, description, status, priority, dueDate, assignee), Projects (name, description, members), and Comments (body, author, task). Tasks belong to Projects. Users can be Project members with roles (owner, admin, member). Tasks can have subtasks, attachments, and time tracking."]

---

EXECUTE EVERY STEP ABOVE. DO NOT SKIP ANY. DO NOT LEAVE INCOMPLETE. BUILD IT RIGHT, BUILD IT ONCE.
```

---

## Quick Reference: Fill In These Before Using

| Placeholder | What to Put | Example |
|---|---|---|
| `[YOUR_APP_NAME]` | Your app's name | `TaskSync` |
| `[ONE LINE DESCRIPTION]` | What it does | `Cross-platform task manager with real-time sync` |
| `[Target Platform]` | Where it runs | `web + mobile` |
| `[Frontend Framework]` | UI tech | `React + Next.js` |
| `[Backend Framework]` | API tech | `Node.js + Express` |
| `[Database]` | Data store | `PostgreSQL` |
| `[Auth Provider]` | How users log in | `Custom JWT + Google OAuth` |
| `[Deployment Target]` | Where it lives | `Vercel + Railway` |
| `[Repo URL]` | Git repo | `github.com/you/tasksync` |
| `[APP-SPECIFIC ENTITIES]` | Your business logic | `See example in prompt` |

## Why This Works

This prompt works because it:

1. **Eliminates ambiguity** — every decision is explicit
2. **Prevents shortcuts** — 150 numbered steps, nothing left to "I'll do it later"
3. **Enforces production standards** from day one (security, testing, monitoring)
4. **Makes the AI accountable** — "DO NOT SKIP, DO NOT LEAVE INCOMPLETE"
5. **Covers the full stack** — DB → API → Auth → Frontend → DevOps → Monitoring
6. **Includes the things everyone forgets** — error schemas, rate limiting, soft deletes, graceful shutdown, health checks
7. **Separates concerns** — clean architecture layers, not spaghetti
8. **Security-first** — not bolted on, built in (steps 92-100)

Trust the process. Run the prompt. Review each phase output before moving to the next. Fix issues when they appear, not later.
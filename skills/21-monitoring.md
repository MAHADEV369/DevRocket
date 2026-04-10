# Skill 21: Monitoring and Alerting with Sentry + UptimeRobot

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 1-2 hours
Depends On: 01

## Input Contract
- Express application with routes in `src/routes/`
- Authentication middleware that sets `req.user` after login
- PostgreSQL, Redis, and file storage dependencies
- Sentry account (free tier works) with DSN
- UptimeRobot account (free tier: 50 monitors)

## Output Contract
- Sentry SDK initialized with traces, integrations, and proper error handler
- Sentry user context set after authentication
- Enhanced `/api/v1/health` endpoint with DB, Redis, and storage checks plus stats
- UptimeRobot monitoring configuration instructions
- Error tracking and performance monitoring operational

## Files to Create
- `src/config/sentry.ts` — Sentry initialization and configuration
- `src/middleware/sentryUser.ts` — Middleware to set Sentry user context after auth
- `src/routes/health.ts` — Enhanced health endpoint (replace existing if present)

## Steps

### Step 1: Install Sentry Dependencies

```bash
npm install @sentry/node @sentry/profiling-node
```

### Step 2: Create `src/config/sentry.ts`

```typescript
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

const SENTRY_DSN = process.env.SENTRY_DSN || '';
const ENVIRONMENT = process.env.NODE_ENV || 'development';
const RELEASE = process.env.SENTRY_RELEASE || process.env.npm_package_version || 'unknown';

export function initSentry() {
  if (!SENTRY_DSN) {
    console.warn('[Sentry] SENTRY_DSN not set — Sentry is disabled');
    return;
  }

  Sentry.init({
    dsn: SENTRY_DSN,
    environment: ENVIRONMENT,
    release: RELEASE,
    tracesSampleRate: ENVIRONMENT === 'production' ? 0.2 : 1.0,
    profilesSampleRate: ENVIRONMENT === 'production' ? 0.1 : 1.0,
    integrations: [
      nodeProfilingIntegration(),
      Sentry.postgresIntegration(),
      Sentry.redisIntegration(),
      Sentry.expressIntegration(),
      Sentry.httpIntegration(),
    ],
    beforeSend(event, hint) {
      const error = hint.originalException;
      if (error && typeof error === 'object' && 'name' in error) {
        const errorName = (error as Error).name;
        const ignoredErrors = [
          'UnauthorizedError',
          'TokenExpiredError',
          'JsonWebTokenError',
          'ValidationError',
        ];
        if (ignoredErrors.includes(errorName)) {
          return null;
        }
      }
      return event;
    },
  });

  console.log(`[Sentry] Initialized — env=${ENVIRONMENT} release=${RELEASE}`);
}

export { Sentry };
```

Key configuration decisions:
- `tracesSampleRate: 0.2` in production — captures 20% of traces for performance monitoring without overwhelming quota
- `profilesSampleRate: 0.1` — 10% profiling in production
- `beforeSend` — filters out known non-critical errors (JWT errors, validation errors) that shouldn't alert
- Integrations for PostgreSQL, Redis, Express, and HTTP for automatic instrumentation

### Step 3: Create `src/middleware/sentryUser.ts`

```typescript
import { Request, Response, NextFunction } from 'express';
import { Sentry } from '../config/sentry';

export function sentryUserMiddleware(req: Request, _res: Response, next: NextFunction) {
  if (req.user) {
    Sentry.setUser({
      id: req.user.id,
      email: req.user.email,
    });
  } else {
    Sentry.setUser(null);
  }
  next();
}
```

### Step 4: Update Express App — `src/server.ts`

Add Sentry initialization and handlers to the Express application. The order matters:

```typescript
import express from 'express';
import { initSentry, Sentry } from './config/sentry';
import { sentryUserMiddleware } from './middleware/sentryUser';
// ... other imports ...

// 1. Initialize Sentry FIRST (before any request handling)
initSentry();

const app = express();

// 2. Sentry request handler — must be the first middleware
app.use(Sentry.Handlers.requestHandler());

// 3. Sentry tracing handler — must be before routes
app.use(Sentry.Handlers.tracingHandler());

// 4. Body parsing and other middleware
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// 5. Public routes (health, auth) — before auth middleware
import healthRoutes from './routes/health';
app.use('/api/v1', healthRoutes);

// 6. Auth middleware — sets req.user
// app.use(authMiddleware);

// 7. Sentry user context — after auth middleware
app.use(sentryUserMiddleware);

// 8. Protected routes
// app.use('/api/v1/images', imageRoutes);

// 9. Sentry error handler — must be BEFORE other error handlers
app.use(Sentry.Handlers.errorHandler());

// 10. Custom error handler — after Sentry
app.use((err: Error, req: express.Request, res: express.Response, next: express.NextFunction) => {
  const statusCode = (err as any).statusCode || 500;
  const message = statusCode === 500 ? 'Internal Server Error' : err.message;

  res.status(statusCode).json({
    message,
    statusCode,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
  });
});

export default app;
```

### Step 5: Create Enhanced Health Endpoint — `src/routes/health.ts`

This replaces or extends the basic health endpoint with detailed dependency checks:

```typescript
import { Router, Request, Response } from 'express';
import { Redis } from 'ioredis';
import { Pool } from 'pg';
import fs from 'fs';
import path from 'path';
import os from 'os';

const router = Router();

interface HealthCheckResult {
  status: 'ok' | 'degraded' | 'down';
  latency: number;
  details?: Record<string, any>;
}

async function checkDatabase(db: Pool): Promise<HealthCheckResult> {
  const start = Date.now();
  try {
    const result = await db.query('SELECT NOW() as now, pg_is_in_recovery() as replication');
    return {
      status: 'ok',
      latency: Date.now() - start,
      details: {
        serverTime: result.rows[0].now,
        inRecovery: result.rows[0].replication,
      },
    };
  } catch (error) {
    return {
      status: 'down',
      latency: Date.now() - start,
      details: {
        error: error instanceof Error ? error.message : 'Unknown database error',
      },
    };
  }
}

async function checkRedis(redis: Redis): Promise<HealthCheckResult> {
  const start = Date.now();
  try {
    const pong = await redis.ping();
    const info = await redis.info('memory');
    const usedMemoryMatch = info.match(/used_memory_human:(\S+)/);
    const connectedClientsMatch = info.match(/connected_clients:(\d+)/);

    return {
      status: pong === 'PONG' ? 'ok' : 'degraded',
      latency: Date.now() - start,
      details: {
        usedMemory: usedMemoryMatch?.[1] || 'unknown',
        connectedClients: connectedClientsMatch?.[1] || 'unknown',
      },
    };
  } catch (error) {
    return {
      status: 'down',
      latency: Date.now() - start,
      details: {
        error: error instanceof Error ? error.message : 'Unknown Redis error',
      },
    };
  }
}

function checkStorage(): HealthCheckResult {
  const start = Date.now();
  const uploadDir = process.env.UPLOAD_DIR || path.join(process.cwd(), 'uploads');

  try {
    if (!fs.existsSync(uploadDir)) {
      fs.mkdirSync(uploadDir, { recursive: true });
    }

    const testFile = path.join(uploadDir, `.health_check_${Date.now()}`);
    fs.writeFileSync(testFile, 'health-check');
    fs.unlinkSync(testFile);

    const stats = fs.statSync(uploadDir);
    return {
      status: 'ok',
      latency: Date.now() - start,
      details: {
        path: uploadDir,
        writable: true,
      },
    };
  } catch (error) {
    return {
      status: 'down',
      latency: Date.now() - start,
      details: {
        path: uploadDir,
        writable: false,
        error: error instanceof Error ? error.message : 'Unknown storage error',
      },
    };
  }
}

function getSystemStats() {
  const uptime = process.uptime();
  const memoryUsage = process.memoryUsage();
  const cpuUsage = process.cpuUsage();

  return {
    uptime: Math.floor(uptime),
    uptimeHuman: formatUptime(uptime),
    memory: {
      rss: formatBytes(memoryUsage.rss),
      heapTotal: formatBytes(memoryUsage.heapTotal),
      heapUsed: formatBytes(memoryUsage.heapUsed),
      external: formatBytes(memoryUsage.external),
      rssMB: Math.round(memoryUsage.rss / 1024 / 1024),
      heapUsedMB: Math.round(memoryUsage.heapUsed / 1024 / 1024),
    },
    cpu: {
      user: cpuUsage.user,
      system: cpuUsage.system,
    },
    node: {
      version: process.version,
      env: process.env.NODE_ENV || 'unknown',
    },
    os: {
      platform: os.platform(),
      arch: os.arch(),
      cpus: os.cpus().length,
      totalMemory: formatBytes(os.totalmem()),
      freeMemory: formatBytes(os.freemem()),
      loadAvg: os.loadavg().map((v) => v.toFixed(2)),
    },
  };
}

function formatBytes(bytes: number): string {
  if (bytes === 0) return '0 B';
  const k = 1024;
  const sizes = ['B', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return `${parseFloat((bytes / Math.pow(k, i)).toFixed(2))} ${sizes[i]}`;
}

function formatUptime(seconds: number): string {
  const days = Math.floor(seconds / 86400);
  const hours = Math.floor((seconds % 86400) / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const parts: string[] = [];
  if (days > 0) parts.push(`${days}d`);
  if (hours > 0) parts.push(`${hours}h`);
  parts.push(`${minutes}m`);
  return parts.join(' ');
}

// Dependencies will be injected via route setup
let dbPool: Pool | null = null;
let redisClient: Redis | null = null;

export function setHealthDependencies(pool: Pool, redis: Redis) {
  dbPool = pool;
  redisClient = redis;
}

// GET /health — simple health check for load balancers
router.get('/health', (_req: Request, res: Response) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// GET /health/detailed — detailed health check with dependency status
router.get('/health/detailed', async (_req: Request, res: Response) => {
  const checks: Record<string, HealthCheckResult> = {};

  if (dbPool) {
    checks.database = await checkDatabase(dbPool);
  }

  if (redisClient) {
    checks.redis = await checkRedis(redisClient);
  }

  checks.storage = checkStorage();

  const allStatuses = Object.values(checks).map((c) => c.status);
  const overall: 'ok' | 'degraded' | 'down' =
    allStatuses.includes('down') ? 'down' :
    allStatuses.includes('degraded') ? 'degraded' : 'ok';

  const statusCode = overall === 'down' ? 503 : 200;

  res.status(statusCode).json({
    status: overall,
    timestamp: new Date().toISOString(),
    checks,
    system: getSystemStats(),
  });
});

export default router;
```

### Step 6: Wire Up Health Dependencies in `src/server.ts`

```typescript
import healthRouter, { setHealthDependencies } from './routes/health';
// ... after creating db pool and redis client ...

const db = new Pool({ connectionString: process.env.DATABASE_URL });
const redis = new Redis(process.env.REDIS_URL);

setHealthDependencies(db, redis);

app.use('/api/v1', healthRouter);
```

### Step 7: Add Environment Variables

Add to `.env.example`:

```
# Sentry (optional — leave empty to disable)
SENTRY_DSN=https://xxxxx@o123456.ingest.sentry.io/123456
SENTRY_RELEASE=1.0.0
```

### Step 8: Create Sentry Performance Transaction Helper (Optional)

Create `src/utils/trace.ts` for manual transaction tracking in important operations:

```typescript
import { Sentry } from '../config/sentry';

export async function traceAsync<T>(
  name: string,
  op: string,
  fn: () => Promise<T>
): Promise<T> {
  const transaction = Sentry.startTransaction({ name, op });
  try {
    const result = await fn();
    transaction.setStatus('ok');
    return result;
  } catch (error) {
    transaction.setStatus('internal_error');
    Sentry.captureException(error);
    throw error;
  } finally {
    transaction.finish();
  }
}

export function traceSync<T>(
  name: string,
  op: string,
  fn: () => T
): T {
  const transaction = Sentry.startTransaction({ name, op });
  try {
    const result = fn();
    transaction.setStatus('ok');
    return result;
  } catch (error) {
    transaction.setStatus('internal_error');
    Sentry.captureException(error);
    throw error;
  } finally {
    transaction.finish();
  }
}
```

### Step 9: Set Up UptimeRobot

**Manual Setup (UptimeRobot Dashboard):**

1. Log in to https://uptimerobot.com
2. Create the following monitors:

| Monitor Name | URL | Interval | Method |
|---|---|---|---|
| API Health | `https://api.example.com/api/v1/health` | 5 min | HTTP(s) - expect 200 |
| API Detailed Health | `https://api.example.com/api/v1/health/detailed` | 5 min | HTTP(s) - expect 200 |
| API Signup | `https://api.example.com/api/v1/auth/signup` | 5 min | HTTP(s) - expect 400/405 (no body) |

3. Configure alert contacts:
   - Email alert on downtime
   - Slack/webhook for team notifications

**API Setup (alternative, programmatic):**

Create `scripts/uptime-robot-setup.sh`:

```bash
#!/bin/bash
set -e

API_KEY=${UPTIMEROBOT_API_KEY:?"Set UPTIMEROBOT_API_KEY environment variable"}
BASE_URL="https://api.uptimerobot.com/v2/newMonitor"

create_monitor() {
  local name="$1"
  local url="$2"
  local expected="$3"

  curl -s -X POST "$BASE_URL" \
    -H "Content-Type: application/json" \
    -d "{
      \"api_key\": \"$API_KEY\",
      \"format\": \"json\",
      \"type\": \"1\",
      \"name\": \"$name\",
      \"url\": \"$url\",
      \"interval\": \"300\",
      \"http_method\": \"1\",
      \"alert_contacts\": \"\"
    }" | python3 -m json.tool

  echo "Created monitor: $name -> $url"
}

echo "Setting up UptimeRobot monitors..."
create_monitor "API Health" "https://${DOMAIN:-api.example.com}/api/v1/health" "200"
create_monitor "API Detailed Health" "https://${DOMAIN:-api.example.com}/api/v1/health/detailed" "200"
echo "Done."
```

```bash
chmod +x scripts/uptime-robot-setup.sh
```

### Step 10: Add Environment Variables to Docker Compose

Update `docker-compose.yml` to include Sentry environment variables:

```yaml
services:
  api:
    environment:
      # ... existing env vars ...
      SENTRY_DSN: ${SENTRY_DSN:-}
      SENTRY_RELEASE: ${SENTRY_RELEASE:-}
```

## Verification

1. **Verify Sentry initialization:**
   ```bash
   SENTRY_DSN=https://test@test.ingest.sentry.io/123 npm start
   ```
   Expected: `[Sentry] Initialized — env=development release=unknown` in console output.

2. **Test error capture:**
   - Trigger an unhandled error (e.g., hit a non-existent route or a route that throws)
   - Check the Sentry dashboard at https://sentry.io
   - Expected: Error appears with stack trace, request context, and user info (if authenticated)

3. **Test health endpoint:**
   ```bash
   curl http://localhost:3000/api/v1/health
   ```
   Expected: `{"status":"ok","timestamp":"..."}`

4. **Test detailed health endpoint:**
   ```bash
   curl http://localhost:3000/api/v1/health/detailed
   ```
   Expected: JSON with `status`, `checks` (database, redis, storage), and `system` stats:
   ```json
   {
     "status": "ok",
     "timestamp": "...",
     "checks": {
       "database": { "status": "ok", "latency": 5, "details": {...} },
       "redis": { "status": "ok", "latency": 2, "details": {...} },
       "storage": { "status": "ok", "latency": 1, "details": {...} }
     },
     "system": {
       "uptime": "...", "memory": {...}, "cpu": {...}, "node": {...}, "os": {...}
     }
   }
   ```

5. **Test degraded/down state:**
   - Stop PostgreSQL or Redis
   - Hit `/api/v1/health/detailed`
   - Expected: `503` status code with `checks.database.status: "down"` or `checks.redis.status: "down"`

6. **Verify Sentry user context:**
   - Log in through the API
   - Trigger an error
   - Expected: Sentry event includes `user.id` and `user.email`

7. **Verify Sentry filtering:**
   - Send an invalid JWT (should trigger `JsonWebTokenError`)
   - Expected: Error does NOT appear in Sentry (filtered by `beforeSend`)

## Rollback

1. Uninstall Sentry:
   ```bash
   npm uninstall @sentry/node @sentry/profiling-node
   ```

2. Remove Sentry files:
   ```bash
   rm src/config/sentry.ts src/middleware/sentryUser.ts src/utils/trace.ts
   ```

3. Remove Sentry imports and middleware from `src/server.ts`:
   - Remove `initSentry()` call
   - Remove `Sentry.Handlers.requestHandler()`
   - Remove `Sentry.Handlers.tracingHandler()`
   - Remove `Sentry.Handlers.errorHandler()`
   - Remove `sentryUserMiddleware`
   - Restore original error handler

4. Remove Sentry environment variables from `docker-compose.yml` and `.env.example`.

5. Simplify health endpoint back to basic:
   ```typescript
   router.get('/health', (_req, res) => res.json({ status: 'ok' }));
   ```
   Remove `src/routes/health.ts` detailed checks and `setHealthDependencies`.

6. Delete UptimeRobot monitors via dashboard or API.

7. Clean up UptimeRobot setup script:
   ```bash
   rm scripts/uptime-robot-setup.sh
   ```

## ADR-021: Sentry for Error Tracking + UptimeRobot for Uptime Monitoring

**Decision:** Use Sentry for error tracking and performance monitoring with selective trace sampling, and UptimeRobot as an external uptime monitor hitting the detailed health endpoint.

**Reason:** Sentry provides rich error context (stack traces, breadcrumbs, user info, release tracking) with minimal integration effort. The Node.js SDK auto-instruments Express, PostgreSQL, and Redis, capturing performance traces without manual spans. Filtering known non-critical errors (JWT, validation) in `beforeSend` prevents alert fatigue. UptimeRobot provides free external monitoring from multiple locations, detecting outages that internal health checks cannot (DNS failure, firewall blocks, certificate expiry).

**Consequences:**
- Sentry free tier: 5,000 events/month, 10,000 transactions/month — sufficient for small workloads
- 20% trace sampling in production balances visibility with quota usage
- `beforeSend` filtering means some errors are silently dropped — review this list periodically
- UptimeRobot 5-minute intervals mean up to 5 minutes of undetected downtime
- Detailed health endpoint exposes system info (memory, CPU, versions) — consider auth-protecting it in production

**Alternatives Considered:**
- **Datadog:** Comprehensive but expensive ($23/host/month minimum)
- **New Relic:** Good APM but complex pricing and heavier agent
- **Prometheus + Grafana:** Self-hosted, powerful, but significant operational overhead for a single service
- **Rollbar:** Strong error tracking but no APM/performance on free tier
- **Log-based monitoring (ELK):** Requires reading logs to detect issues, no proactive alerting
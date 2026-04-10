# Skill 15: Docker & Docker Compose

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 2-3 hours
Depends On: 01

## Input Contract
- Skill 01 completed: working Express + TypeScript project with `src/` structure
- `package.json` with scripts: `build`, `start`, `dev`
- PostgreSQL and Redis configured in `src/config/database.ts` and `src/config/redis.ts`
- Environment variables documented in `.env.example`
- Project builds successfully with `npm run build`

## Output Contract
- Multi-stage Dockerfile producing a minimal production image
- `docker-compose.yml` with api, worker, postgres, redis services
- `docker-compose.dev.yml` for hot reload during development
- `.dockerignore` excluding unnecessary files
- Healthcheck endpoints configured for each service
- All services start and communicate correctly

## Files to Create
- `Dockerfile` - Multi-stage build (deps → build → runner)
- `docker-compose.yml` - Production-style multi-service orchestration
- `docker-compose.dev.yml` - Development overrides with hot reload
- `.dockerignore` - Exclude node_modules, .git, docs, etc.
- `scripts/wait-for-it.sh` - Service dependency wait script

## Steps

### 1. Create .dockerignore

```
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
!.env.example
README.md
docs
coverage
.nyc_output
tests
*.md
!skills/*.md
.vscode
.idea
dist
*.tgz
.DS_Store
docker-compose*.yml
Dockerfile
```

### 2. Create the Multi-Stage Dockerfile

```dockerfile
# ---- Stage 1: Dependencies ----
FROM node:20-alpine AS deps

RUN apk add --no-cache libc6-compat

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --only=production && \
    cp -R node_modules /prod_node_modules && \
    npm ci

# ---- Stage 2: Build ----
FROM node:20-alpine AS build

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN npm run build

# ---- Stage 3: Runner ----
FROM node:20-alpine AS runner

RUN apk add --no-cache tini curl

WORKDIR /app

ENV NODE_ENV=production

RUN addgroup -g 1001 -S nodejs && \
    adduser -S appuser -u 1001

COPY --from=deps /prod_node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./

USER appuser

EXPOSE 3000

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
```

### 3. Create docker-compose.yml

```yaml
version: "3.9"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: fastdepo-api
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgres://fastdepo:fastdepo_secret@postgres:5432/fastdepo
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - fastdepo-net

  worker:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: fastdepo-worker
    restart: unless-stopped
    command: ["node", "dist/workers/worker.js"]
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://fastdepo:fastdepo_secret@postgres:5432/fastdepo
      REDIS_URL: redis://redis:6379
      WORKER_MODE: "true"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    networks:
      - fastdepo-net

  postgres:
    image: postgres:16-alpine
    container_name: fastdepo-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: fastdepo
      POSTGRES_PASSWORD: fastdepo_secret
      POSTGRES_DB: fastdepo
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U fastdepo -d fastdepo"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - fastdepo-net

  redis:
    image: redis:7-alpine
    container_name: fastdepo-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 5s
    networks:
      - fastdepo-net

volumes:
  postgres_data:
  redis_data:

networks:
  fastdepo-net:
    driver: bridge
```

### 4. Create docker-compose.dev.yml for Hot Reload

```yaml
version: "3.9"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: deps
    container_name: fastdepo-api-dev
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      NODE_ENV: development
      PORT: 3000
      DATABASE_URL: postgres://fastdepo:fastdepo_secret@postgres:5432/fastdepo
      REDIS_URL: redis://redis:6379
    command: npx nodemon --watch src --ext ts --exec "npx ts-node src/index.ts"
    ports:
      - "3000:3000"
      - "9229:9229"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  worker:
    build:
      context: .
      dockerfile: Dockerfile
      target: deps
    container_name: fastdepo-worker-dev
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      NODE_ENV: development
      DATABASE_URL: postgres://fastdepo:fastdepo_secret@postgres:5432/fastdepo
      REDIS_URL: redis://redis:6379
      WORKER_MODE: "true"
    command: npx nodemon --watch src --ext ts --exec "npx ts-node src/workers/worker.ts"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    ports:
      - "5432:5432"

  redis:
    ports:
      - "6379:6379"
```

### 5. Create scripts/wait-for-it.sh

Download or create a minimal wait-for-it script that polls a host:port until available. This is used in entrypoint scripts for services that need database connectivity before starting.

```bash
#!/bin/sh
# Minimal wait-for-it implementation
set -e

host="$1"
port="$2"
shift 2
cmd="$@"

max_attempts=30
attempt=0

until nc -z "$host" "$port" >/dev/null 2>&1; do
  attempt=$((attempt + 1))
  if [ $attempt -ge $max_attempts ]; then
    echo "Service $host:$port not available after $max_attempts attempts"
    exit 1
  fi
  echo "Waiting for $host:$port... (attempt $attempt/$max_attempts)"
  sleep 2
done

echo "$host:$port is available"
exec $cmd
```

Make it executable: `chmod +x scripts/wait-for-it.sh`

### 6. Add Docker Scripts to package.json

```json
{
  "scripts": {
    "docker:build": "docker compose build",
    "docker:up": "docker compose up -d",
    "docker:down": "docker compose down",
    "docker:dev": "docker compose -f docker-compose.yml -f docker-compose.dev.yml up",
    "docker:dev:down": "docker compose -f docker-compose.yml -f docker-compose.dev.yml down",
    "docker:logs": "docker compose logs -f api",
    "docker:logs:worker": "docker compose logs -f worker"
  }
}
```

### 7. Configure the Health Endpoint

Ensure `src/routes/health.ts` (or equivalent) exposes `GET /api/v1/health` returning:

```typescript
router.get('/health', async (req: Request, res: Response) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});
```

This endpoint must respond with HTTP 200 for Docker healthchecks to pass.

### 8. Environment Variable Handling

Ensure the app reads connection strings from environment variables:

```typescript
// src/config/database.ts
export const databaseConfig = {
  url: process.env.DATABASE_URL || 'postgres://localhost:5432/fastdepo',
};

// src/config/redis.ts
export const redisConfig = {
  url: process.env.REDIS_URL || 'redis://localhost:6379',
};
```

## Verification

```bash
# 1. Build images
docker compose build

# 2. Start all services
docker compose up -d

# 3. Verify all containers are healthy
docker compose ps
# Expected: All 4 services show status "healthy" or "running"

# 4. Test the API
curl http://localhost:3000/api/v1/health
# Expected: {"status":"ok","timestamp":"..."}

# 5. Test PostgreSQL connectivity
docker compose exec api node -e "
  const { Pool } = require('pg');
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });
  pool.query('SELECT NOW()').then(r => { console.log(r.rows); pool.end(); });
"
# Expected: Current timestamp row

# 6. Test Redis connectivity
docker compose exec api node -e "
  const redis = require('redis');
  const client = redis.createClient({ url: process.env.REDIS_URL });
  client.connect().then(() => client.ping()).then(console.log).then(() => client.quit());
"
# Expected: PONG

# 7. Check healthchecks
docker inspect --format='{{.State.Health.Status}}' fastdepo-api
docker inspect --format='{{.State.Health.Status}}' fastdepo-postgres
docker inspect --format='{{.State.Health.Status}}' fastdepo-redis
# Expected: "healthy" for each

# 8. Verify dev overrides work
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
# Expected: Services start with volume mounts and hot reload

# 9. Teardown
docker compose down -v
```

## Rollback

1. `docker compose down -v` — stop and remove all containers, networks, and volumes
2. Delete `Dockerfile`, `docker-compose.yml`, `docker-compose.dev.yml`, `.dockerignore`, `scripts/wait-for-it.sh`
3. Remove Docker scripts from `package.json`
4. No application code changes were required beyond environment variable configuration, so the app still runs locally without Docker

## ADR-015: Multi-Stage Docker Build with Alpine

**Decision:** Use a three-stage Docker build (deps → build → runner) with Alpine Linux base images.

**Reason:** Multi-stage builds minimize the final image size by separating build-time dependencies from runtime dependencies. Alpine Linux reduces the base image footprint to ~50MB vs ~350MB for Debian-based images. The deps stage caches npm install, the build stage compiles TypeScript, and the runner stage copies only production artifacts.

**Consequences:**
- Final production image is ~120MB vs ~1GB for a single-stage build
- Builds are faster on subsequent runs due to Docker layer caching
- Alpine musl libc may cause issues with some native modules (mitigated by installing libc6-compat)
- The non-root user (appuser) improves container security

**Alternatives Considered:**
- Single-stage Dockerfile: Simpler but produces 5-10x larger images
- Distroless base image: More secure but harder to debug; no shell for healthchecks
- Debian-based images: Better compatibility but much larger footprint
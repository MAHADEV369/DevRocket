# Skill 22: Deployment to Railway and Fly.io

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 1-2 hours
Depends On: 15, 16

## Input Contract
- Skill 15 complete: multi-stage Dockerfile, docker-compose.yml, docker-compose.dev.yml
- Skill 16 complete: CI/CD pipeline with test and build stages
- Docker image builds successfully locally
- PostgreSQL migrations run without errors
- Redis connection verified
- All environment variables documented in `.env.example`
- Health check endpoint `/api/health` responding with 200

## Output Contract
- `railway.toml` with Railway deployment configuration
- `fly.toml` with Fly.io deployment configuration
- Step-by-step Railway deployment guide (CLI, PostgreSQL addon, Redis addon, env vars, deploy, migrate, seed)
- Step-by-step Fly.io deployment guide (launch, secrets, deploy)
- Custom domain setup documentation with SSL and DNS configuration
- Health check verification steps for both platforms

## Files to Create

| File | Description |
|------|-------------|
| `railway.toml` | Railway deployment configuration |
| `fly.toml` | Fly.io deployment configuration |
| `scripts/deploy-railway.sh` | Automated Railway deployment script |
| `scripts/deploy-fly.sh` | Automated Fly.io deployment script |
| `docs/deployment-guide.md` | Full deployment guide with domain and SSL setup |

## Steps

### Step 1: Create railway.toml

```toml
[build]
builder = "dockerfile"
dockerfilePath = "Dockerfile"

[deploy]
startCommand = "node dist/server.js"
healthcheckPath = "/api/health"
healthcheckTimeout = 300
restartPolicyType = "on_failure"
restartPolicyMaxRetries = 10

[[deploy.env]]
name = "NODE_ENV"
value = "production"

[[deploy.env]]
name = "PORT"
value = "3000"
```

### Step 2: Create fly.toml

```toml
app = "fastdepo"
primary_region = "sjc"

[build]
dockerfile = "Dockerfile"

[deploy]
release_command = "npx prisma migrate deploy"

[env]
NODE_ENV = "production"
PORT = "3000"

[http_service]
internal_port = 3000
force_https = true
auto_stop_machines = "stop"
auto_start_machines = true
min_machines_running = 1

[http_service.concurrency]
type = "connections"
hard_limit = 250
soft_limit = 200

[[http_service.checks]]
grace_period = "10s"
interval = "30s"
method = "GET"
path = "/api/health"
timeout = "5s"

[[vm]]
memory = "512mb"
cpu_kind = "shared"
cpus = 1
```

### Step 3: Railway Deployment (Manual CLI Steps)

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login to Railway
railway login

# Initialize project (run from project root)
railway init

# Create the project
railway add

# Add PostgreSQL addon
railway add --plugin postgresql

# Add Redis addon
railway add --plugin redis

# Get the database URL from PostgreSQL addon
railway variables
# Note: DATABASE_URL will be auto-set by the PostgreSQL addon
# Note: REDIS_URL will be auto-set by the Redis addon

# Set required environment variables
railway variables set JWT_SECRET="<generate-64-char-secret>"
railway variables set JWT_REFRESH_SECRET="<generate-64-char-secret>"
railway variables set JWT_EXPIRES_IN="15m"
railway variables set JWT_REFRESH_EXPIRES_IN="7d"
railway variables set NODE_ENV="production"
railway variables set CORS_ORIGIN="https://yourdomain.com"
railway variables set OPENAI_API_KEY="<your-key>"
railway variables set SMTP_HOST="smtp.resend.com"
railway variables set SMTP_PORT="465"
railway variables set SMTP_USER="<your-user>"
railway variables set SMTP_PASS="<your-password>"
railway variables set EMAIL_FROM="noreply@yourdomain.com"
railway variables set STORAGE_PROVIDER="s3"
railway variables set AWS_S3_BUCKET="<bucket>"
railway variables set AWS_S3_REGION="<region>"
railway variables set AWS_ACCESS_KEY_ID="<key>"
railway variables set AWS_SECRET_ACCESS_KEY="<secret>"
railway variables set STRIPE_SECRET_KEY="<key>"
railway variables set STRIPE_WEBHOOK_SECRET="<secret>"
railway variables set STRIPE_PRICE_ID_PRO="<price-id>"
railway variables set APP_URL="https://api.yourdomain.com"

# Deploy the application
railway up

# Run database migrations
railway run npx prisma migrate deploy

# Seed the database (first time only)
railway run npx prisma db seed

# Check deployment status
railway status

# View logs
railway logs
```

### Step 4: Fly.io Deployment (Manual CLI Steps)

```bash
# Install Fly CLI
curl -L https://fly.io/install.sh | sh

# Login to Fly.io
fly auth login

# Launch the app (creates fly.toml if not present)
fly launch --dockerfile Dockerfile --name fastdepo --region sjc

# When prompted:
# - Do not create a PostgreSQL database yet (we'll use a separate command)
# - Do not create a Redis database yet

# Create PostgreSQL database
fly postgres create
# Name: fastdepo-db
# Region: sjc (or closest)
# Select: Development (single node) for staging, Production for prod

# Attach the database to the app
fly postgres attach fastdepo-db

# Create Redis
fly redis create
# Name: fastdepo-redis
# Region: sjc

# Attach Redis to the app
fly redis attach fastdepo-redis

# Set secrets (these become environment variables)
fly secrets set \
  JWT_SECRET="<generate-64-char-secret>" \
  JWT_REFRESH_SECRET="<generate-64-char-secret>" \
  JWT_EXPIRES_IN="15m" \
  JWT_REFRESH_EXPIRES_IN="7d" \
  CORS_ORIGIN="https://yourdomain.com" \
  OPENAI_API_KEY="<your-key>" \
  SMTP_HOST="smtp.resend.com" \
  SMTP_PORT="465" \
  SMTP_USER="<your-user>" \
  SMTP_PASS="<your-password>" \
  EMAIL_FROM="noreply@yourdomain.com" \
  STORAGE_PROVIDER="s3" \
  AWS_S3_BUCKET="<bucket>" \
  AWS_S3_REGION="<region>" \
  AWS_ACCESS_KEY_ID="<key>" \
  AWS_SECRET_ACCESS_KEY="<secret>" \
  STRIPE_SECRET_KEY="<key>" \
  STRIPE_WEBHOOK_SECRET="<secret>" \
  STRIPE_PRICE_ID_PRO="<price-id>" \
  APP_URL="https://api.yourdomain.com" \
  NODE_ENV="production"

# Deploy the application
fly deploy

# Monitor the deployment
fly status
fly logs
```

### Step 5: Create Deploy Scripts

Create `scripts/deploy-railway.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Deploying to Railway ==="

if ! command -v railway &>/dev/null; then
  echo "Railway CLI not found. Install: npm install -g @railway/cli"
  exit 1
fi

railway up --detached

echo "Waiting for deployment to be ready..."
sleep 30

echo "Running database migrations..."
railway run npx prisma migrate deploy

echo "Verifying health check..."
HEALTH_URL=$(railway domain 2>/dev/null | head -1)
if [ -n "$HEALTH_URL" ]; then
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "https://${HEALTH_URL}/api/health" || true)
  if [ "$HTTP_CODE" = "200" ]; then
    echo "Health check passed (HTTP 200)"
  else
    echo "WARNING: Health check returned HTTP $HTTP_CODE"
  fi
fi

echo "=== Railway deployment complete ==="
```

Create `scripts/deploy-fly.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Deploying to Fly.io ==="

if ! command -v fly &>/dev/null; then
  echo "Fly CLI not found. Install: https://fly.io/docs/hands-on/install-flycli/"
  exit 1
fi

fly deploy

echo "Waiting for app to be healthy..."
sleep 15

echo "Verifying health check..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "https://fastdepo.fly.dev/api/health" || true)
if [ "$HTTP_CODE" = "200" ]; then
  echo "Health check passed (HTTP 200)"
else
  echo "WARNING: Health check returned HTTP $HTTP_CODE"
fi

echo "Checking VM status..."
fly status

echo "=== Fly.io deployment complete ==="
```

### Step 6: Custom Domain Setup with SSL

For **Railway**:

```bash
# Add a custom domain via Railway dashboard or CLI
# Go to your project > Settings > Domains
# Add domain: api.yourdomain.com

# In your DNS provider, add:
# Type:  CNAME
# Name:  api
# Value: <your-railway-app>.up.railwayway.app

# SSL is auto-provisioned by Railway via Let's Encrypt
# Verify with:
curl -s -o /dev/null -w "%{http_code}" https://api.yourdomain.com/api/health
```

For **Fly.io**:

```bash
# Add custom domain
fly domains add api.yourdomain.com

# Get the DNS instructions
fly certs create api.yourdomain.com

# Fly will show you the required DNS records:
# Type:  A
# Name:  api
# Value: <fly-app-ipv4>

# OR Type:  AAAA
# Name:  api
# Value: <fly-app-ipv6>

# Type:  CNAME (alternative)
# Name:  api
# Value: fastdepo.fly.dev

# Verify certificate issuance
fly certs check api.yourdomain.com

# SSL is auto-managed by Fly.io via Let's Encrypt
```

### Step 7: DNS Configuration Guide

Add to `docs/deployment-guide.md`:

```markdown
## DNS Configuration

### MX Records (if using custom email domain)
| Type | Name  | Value                    | Priority |
|------|-------|--------------------------|----------|
| MX   | @     | mail.yourdomain.com      | 10       |

### Application DNS
| Type  | Name | Value                              | TTL  |
|-------|------|-------------------------------------|------|
| A     | api  | <hosting-provider-ip>              | 300  |
| AAAA  | api  | <hosting-provider-ipv6>             | 300  |
| CNAME | www  | api.yourdomain.com                  | 3600 |
| TXT   | @    | v=spf1 include:_spf.resend.com ~all | 3600 |

### SSL Notes
- Railway: Auto-provisioned via Let's Encrypt, auto-renewed
- Fly.io: Auto-provisioned via Let's Encrypt, auto-renewed
- Both platforms enforce HTTPS redirects
```

### Step 8: Health Check Verification

For **Railway**:

```bash
# Check via Railway CLI
railway logs --filter "health"

# Direct curl check
curl -f https://<your-railway-domain>/api/health | jq .

# Expected response:
# {
#   "status": "ok",
#   "timestamp": "2025-01-15T12:00:00.000Z",
#   "uptime": 12345,
#   "database": "connected",
#   "redis": "connected"
# }
```

For **Fly.io**:

```bash
# Check via Fly CLI
fly status

# Direct curl check
curl -f https://fastdepo.fly.dev/api/health | jq .

# Check machine health
fly machines list

# View application metrics
fly metrics
```

### Step 9: Post-Deployment Verification Checklist

```markdown
## Post-Deployment Verification

- [ ] Health check returns 200 on `/api/health`
- [ ] Database migration ran successfully (check logs)
- [ ] Redis connection established
- [ ] CORS accepts requests from frontend domain
- [ ] JWT tokens are issued correctly (test login endpoint)
- [ ] Image generation works end-to-end
- [ ] Stripe webhooks reachable at `/api/billing/webhook`
- [ ] Email sending works (test forgot-password flow)
- [ ] SSL certificate is valid (check in browser)
- [ ] Custom domain resolves correctly
- [ ] WebSocket connections work (if applicable)
```

## Verification

```bash
# 1. Verify Railway deployment
railway status
curl -f https://<railway-domain>/api/health

# 2. Verify Fly.io deployment
fly status
curl -f https://<fly-domain>/api/health

# 3. Verify SSL certificates
echo | openssl s_client -connect api.yourdomain.com:443 -servername api.yourdomain.com 2>/dev/null | openssl x509 -noout -dates

# 4. Verify database migration
railway run npx prisma migrate status
fly ssh console -C "npx prisma migrate status"

# 5. Run smoke tests against production
curl -f -X POST https://api.yourdomain.com/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"smoketest@example.com","password":"TestPass123!","name":"Smoke Test"}'

# 6. Expected: HTTP 201 with user object and tokens
```

## Rollback

### Railway Rollback
```bash
# List recent deployments
railway status

# Rollback to previous deployment (via Railway dashboard)
# Go to: Project > Deployments > Select previous deployment > Redeploy

# Or via CLI
railway rollback
```

### Fly.io Rollback
```bash
# List recent releases
fly releases --limit 5

# Rollback to previous release
fly rollback <version-number>

# If database migration needs rollback:
fly ssh console -C "npx prisma migrate resolve --rolled-back <migration-name>"
```

### Database Migration Rollback
```bash
# If a deployment includes a breaking migration:
# 1. Rollback the application first
# 2. Then handle the database

# Railway: Run a custom migration rollback
railway run npx prisma migrate resolve --rolled-back <migration-name>

# Fly.io:
fly ssh console -C "npx prisma migrate resolve --rolled-back <migration-name>"

# In extreme cases, restore from database backup:
# Railway: railway run pg_dump $DATABASE_URL > backup.sql
# Fly.io: fly postgres db dump -a fastdepo-db
```

## ADR-022: Dual-Platform Deployment Strategy

**Decision**: Support both Railway and Fly.io as first-class deployment targets with config files for each.

**Reason**: Railway provides the simplest developer experience for rapid prototyping, while Fly.io offers more control over regions, machine sizing, and cost optimization. Supporting both gives teams flexibility based on their priorities.

**Consequences**:
- Two deployment configurations to maintain
- Both platforms use standard Dockerfile, so the core deployment artifact is shared
- Environment variable management differs between platforms (Railway variables vs Fly secrets)
- Monitoring and log access differ between platforms
- Teams should pick ONE platform for production and use the other for staging/DR

**Alternatives Considered**:
- AWS ECS/Fargate: More control but significantly more configuration overhead
- Vercel/Railway only: Simpler but less flexible for scaling needs
- Kubernetes: Overkill for this project size, too much operational burden
- Single platform only: Limits flexibility and creates vendor lock-in risk
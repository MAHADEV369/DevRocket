# Skill 16: CI/CD GitHub Actions

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 2-3 hours
Depends On: 01, 15

## Input Contract
- Repository hosted on GitHub
- Dockerfile and docker-compose.yml from Skill 15
- `package.json` with `lint`, `typecheck`, `test`, and `build` scripts
- TypeScript project with Express server
- PostgreSQL and Redis as dependencies (available as GitHub Actions service containers)
- Tests that require a running database

## Output Contract
- `.github/workflows/ci.yml` — Continuous integration on every push/PR
- `.github/workflows/deploy.yml` — Staging and production deployment pipelines
- CI pipeline: lint → typecheck → test (with PostgreSQL service container)
- Deploy pipeline: Docker build+push, run migrations, deploy, smoke test
- Staging deploys on `develop` branch push, production deploys on `v*` tag push

## Files to Create
- `.github/workflows/ci.yml` — Lint, typecheck, and test pipeline
- `.github/workflows/deploy.yml` — Build, push, deploy pipeline for staging and production

## Steps

### Step 1: Create `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

  typecheck:
    name: TypeCheck
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: TypeCheck
        run: npm run typecheck

  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
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
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      NODE_ENV: test
      DATABASE_URL: postgresql://testuser:testpassword@localhost:5432/testdb
      REDIS_URL: redis://localhost:6379
      JWT_SECRET: test-secret-for-ci-only

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run migrations
        run: npm run db:migrate

      - name: Run tests
        run: npm test -- --coverage --forceExit

      - name: Upload coverage
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7

  security:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: npm audit
        run: npm audit --audit-level=high
        continue-on-error: true

      - name: Check for license issues
        run: npx license-checker --failOn "GPL-3.0;AGPL-3.0"
        continue-on-error: true
```

### Step 2: Create `.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches: [develop]
    tags: ["v*"]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
      image_digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    name: Deploy to Staging
    needs: build-and-push
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /opt/app
            docker compose pull
            docker compose up -d

      - name: Run database migrations
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd /opt/app
            docker compose exec api npm run db:migrate

      - name: Smoke test — health check
        run: |
          for i in $(seq 1 10); do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://${{ secrets.STAGING_HOST }}/api/v1/health || true)
            if [ "$STATUS" = "200" ]; then
              echo "Health check passed"
              exit 0
            fi
            echo "Attempt $i: status=$STATUS, retrying..."
            sleep 5
          done
          echo "Health check failed after 10 attempts"
          exit 1

      - name: Smoke test — signup endpoint
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            -X POST https://${{ secrets.STAGING_HOST }}/api/v1/auth/signup \
            -H "Content-Type: application/json" \
            -d '{"email":"smoke@test.com","password":"Test1234!"}' || true)
          echo "Signup endpoint status: $STATUS"
          if [ "$STATUS" != "201" ] && [ "$STATUS" != "409" ]; then
            echo "Unexpected signup response"
            exit 1
          fi

  deploy-production:
    name: Deploy to Production
    needs: build-and-push
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Verify tag format
        run: |
          TAG="${GITHUB_REF#refs/tags/}"
          if [[ ! "$TAG" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "Tag $TAG does not match semantic versioning (vX.Y.Z)"
            exit 1
          fi
          echo "Deploying version: $TAG"

      - name: Deploy to production server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USER }}
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            cd /opt/app
            docker compose pull
            docker compose up -d

      - name: Run database migrations
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USER }}
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            cd /opt/app
            docker compose exec api npm run db:migrate

      - name: Wait for service readiness
        run: sleep 10

      - name: Smoke test — health check
        run: |
          for i in $(seq 1 15); do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://${{ secrets.PRODUCTION_HOST }}/api/v1/health || true)
            if [ "$STATUS" = "200" ]; then
              echo "Production health check passed"
              exit 0
            fi
            echo "Attempt $i: status=$STATUS, retrying..."
            sleep 5
          done
          echo "Production health check failed"
          exit 1

      - name: Notify deployment success
        if: success()
        run: |
          echo "Production deployment of ${GITHUB_REF#refs/tags/} completed successfully"

      - name: Rollback on failure
        if: failure()
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USER }}
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            cd /opt/app
            docker compose down
            docker compose up -d --no-pull || true
            echo "Rollback attempted — manual intervention may be required"
```

### Step 3: Add Required npm Scripts

Ensure `package.json` has these scripts (add missing ones):

```json
{
  "scripts": {
    "lint": "eslint src/ --ext .ts",
    "typecheck": "tsc --noEmit",
    "test": "jest --runInBand",
    "build": "tsc",
    "db:migrate": "node dist/db/migrate.js"
  }
}
```

### Step 4: Configure GitHub Repository Secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret | Description |
|--------|-------------|
| `STAGING_HOST` | Staging server hostname or IP |
| `STAGING_USER` | SSH user for staging |
| `STAGING_SSH_KEY` | SSH private key for staging |
| `PRODUCTION_HOST` | Production server hostname or IP |
| `PRODUCTION_USER` | SSH user for production |
| `PRODUCTION_SSH_KEY` | SSH private key for production |

### Step 5: Create GitHub Environments

Go to **Settings → Environments** and create:

1. **staging** — No required reviewers, `develop` branch only
2. **production** — Add required reviewers for approval, `v*` tags only

Optional: Add deployment branches protection rules to restrict which branches can deploy to each environment.

### Step 6: Verify Workflow Files Syntax

```bash
# Install actionlint for workflow validation
brew install actionlint
actionlint .github/workflows/ci.yml
actionlint .github/workflows/deploy.yml
```

## Verification

1. **Push a branch and check CI triggers:**
   ```bash
   git checkout -b test/ci-verification
   git push origin test/ci-verification
   ```
   Then check the **Actions** tab. Expected: `lint`, `typecheck`, `test`, and `security` jobs all run.

2. **Verify test job uses PostgreSQL service:**
   - Go to the test job logs
   - Expected: PostgreSQL and Redis service containers are initialized before the test steps

3. **Push to `develop` and verify staging deployment:**
   ```bash
   git push origin develop
   ```
   Expected: `build-and-push` → `deploy-staging` pipeline runs, smoke tests pass.

4. **Tag a release and verify production deployment:**
   ```bash
   git tag v0.1.0
   git push origin v0.1.0
   ```
   Expected: `build-and-push` → `deploy-production` pipeline runs with review gate.

5. **Check container registry:**
   Go to `github.com/<org>/<repo>/pkgs` — Expected: Docker image is published with correct tags.

## Rollback

1. Delete workflow files:
   ```bash
   rm -rf .github/workflows/ci.yml .github/workflows/deploy.yml
   ```

2. Remove GitHub secrets (STAGING_HOST, STAGING_USER, STAGING_SSH_KEY, PRODUCTION_HOST, PRODUCTION_USER, PRODUCTION_SSH_KEY) from repository settings.

3. Delete GitHub environments (staging, production) from repository settings.

4. Remove any CI-specific npm scripts from `package.json` if they were added solely for CI.

5. Cancel any running workflow runs in the Actions tab.

## ADR-016: GitHub Actions with Service Containers for CI

**Decision:** Use GitHub Actions with PostgreSQL and Redis as service containers for CI, and conditional deployment jobs triggered by branch and tag patterns.

**Reason:** Service containers provide ephemeral, isolated database instances per test run without polluting the host. Conditional jobs (`if: github.ref == 'refs/heads/develop'` and `if: startsWith(github.ref, 'refs/tags/v')`) allow a single workflow file to handle both staging and production deployments with different environments and approval gates.

**Consequences:**
- Every PR and push gets a fresh database — no test state leakage
- Staging deploys automatically on `develop` push — fast feedback loop
- Production deploys require a `v*` tag and optionally a manual approval gate
- SSH-based deployment requires managing SSH keys as secrets
- No built-in rollback mechanism — the rollback step is a best-effort attempt

**Alternatives Considered:**
- **GitLab CI:** Would require migrating away from GitHub — not viable
- **Docker-in-Docker for testing:** More complex, slower startup than service containers
- **Heroku/GCP/AWS deploy actions:** Vendor-specific; SSH approach is portable to any VPS
- **Kubernetes rollout:** Overkill for current scale; can migrate later
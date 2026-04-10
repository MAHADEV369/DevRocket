# CI/CD Pipelines Guide

Complete guide to setting up CI/CD pipelines for FastDepo using GitHub Actions.

---

## 1. GitHub Actions Workflow Structure

All workflows are stored in `.github/workflows/`. Create this directory structure:

```
.github/
├── workflows/
│   ├── ci.yml                    # Continuous integration
│   ├── deploy-staging.yml        # Deploy to staging
│   ├── deploy-production.yml     # Deploy to production
│   └── preview.yml               # PR preview deployments
└── actions/                      (optional: reusable composite actions)
```

---

## 2. ci.yml — Continuous Integration

This workflow runs on every push and pull request. It ensures code quality, type safety, and test coverage.

```yaml
# .github/workflows/ci.yml
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

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Run Prettier check
        run: npx prettier --check 'src/**/*.{ts,tsx}'

  typecheck:
    name: Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Generate Prisma client
        run: npx prisma generate

      - name: TypeScript type check
        run: npx tsc --noEmit

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: [lint, typecheck]

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: fastdepo_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run database migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/fastdepo_test

      - name: Run tests
        run: npm test -- --coverage
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/fastdepo_test
          TEST_DATABASE_URL: postgresql://postgres:postgres@localhost:5432/fastdepo_test
          JWT_SECRET: ci-test-secret-key
          JWT_REFRESH_SECRET: ci-test-refresh-secret-key

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/lcov.info
          token: ${{ secrets.CODECOV_TOKEN }}

      - name: Check coverage thresholds
        run: |
          LINE_COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$LINE_COVERAGE < 60" | bc -l) )); then
            echo "Line coverage $LINE_COVERAGE% is below 60% threshold"
            exit 1
          fi
          echo "Line coverage $LINE_COVERAGE% meets threshold"

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [test]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Build frontend
        run: npm run build
        env:
          VITE_API_URL: /api

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: frontend-build
          path: dist/
          retention-days: 5

  e2e:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: [build]

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: fastdepo_e2e
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Setup test database
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/fastdepo_e2e

      - name: Seed test data
        run: npm run db:seed
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/fastdepo_e2e

      - name: Run E2E tests
        run: npx playwright test
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/fastdepo_e2e
          E2E_BASE_URL: http://localhost:5173

      - name: Upload E2E test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: e2e-results
          path: playwright-report/

      - name: Upload test screenshots
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: e2e-screenshots
          path: test-results/
```

---

## 3. deploy-staging.yml

Deploys to the Railway staging environment on every push to `develop`.

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy Staging

on:
  push:
    branches: [develop]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run database migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}

      - name: Deploy to Railway (Staging)
        uses: railwayapp/railway-deploy-action@v1
        with:
          railway-token: ${{ secrets.RAILWAY_STAGING_TOKEN }}
          railway-environment: staging
          railway-service: fastdepo-api

      - name: Run smoke tests
        run: |
          sleep 15
          STAGING_URL=${{ secrets.STAGING_URL }}
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$STAGING_URL/api/health")
          if [ "$STATUS" != "200" ]; then
            echo "Smoke test failed: health endpoint returned $STATUS"
            exit 1
          fi
          echo "Staging deployment successful"

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: |
            {
              "text": "❌ Staging deployment failed",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "❌ *Staging deployment failed*\nBranch: `${{ github.ref_name }}`\nCommit: `${{ github.sha }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View logs>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 4. deploy-production.yml

Deploys to production when a version tag (`v*`) is pushed. Uses Docker build, GHCR push, and Railway deploy.

```yaml
# .github/workflows/deploy-production.yml
name: Deploy Production

on:
  push:
    tags:
      - 'v*'

jobs:
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract version from tag
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}/fastdepo-api
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            NODE_ENV=production

  run-migrations:
    name: Run Database Migrations
    runs-on: ubuntu-latest
    needs: [build-and-push]
    environment: production

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci

      - name: Run production migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build-and-push, run-migrations]
    environment: production

    steps:
      - name: Deploy to Railway (Production)
        uses: railwayapp/railway-deploy-action@v1
        with:
          railway-token: ${{ secrets.RAILWAY_PRODUCTION_TOKEN }}
          railway-environment: production
          railway-service: fastdepo-api

      - name: Wait for deployment
        run: sleep 20

      - name: Run smoke tests
        run: |
          PRODUCTION_URL=${{ secrets.PRODUCTION_URL }}
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$PRODUCTION_URL/api/health")
          if [ "$STATUS" != "200" ]; then
            echo "Production smoke test failed: health endpoint returned $STATUS"
            exit 1
          fi

          # Test auth endpoint responds
          AUTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" -X POST "$PRODUCTION_URL/api/auth/login" -H "Content-Type: application/json" -d '{"email":"health@check.com","password":"test"}')
          if [ "$AUTH_STATUS" != "401" ] && [ "$AUTH_STATUS" != "400" ]; then
            echo "Auth endpoint unexpected status: $AUTH_STATUS"
          fi

          echo "✅ Production deployment successful"

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: |
            {
              "text": "🚀 Production deployment succeeded",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "🚀 *Production deployment succeeded*\nVersion: `${{ github.ref_name }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View logs>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: |
            {
              "text": "🚨 Production deployment FAILED",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "🚨 *Production deployment FAILED*\nVersion: `${{ github.ref_name }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View logs>\n\n⚠️ *Rollback may be needed*"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 5. Environment Variables and Secrets

### Required GitHub Secrets

Set these in **Settings > Secrets and variables > Actions**:

| Secret | Description | Used In |
|--------|-------------|---------|
| `DATABASE_URL` | Production database URL | prod deploy |
| `STAGING_DATABASE_URL` | Staging database URL | staging deploy |
| `PRODUCTION_DATABASE_URL` | Production database URL | prod migrations |
| `RAILWAY_STAGING_TOKEN` | Railway API token for staging | staging deploy |
| `RAILWAY_PRODUCTION_TOKEN` | Railway API token for production | prod deploy |
| `STAGING_URL` | Staging API base URL | staging smoke tests |
| `PRODUCTION_URL` | Production API base URL | prod smoke tests |
| `SLACK_WEBHOOK_URL` | Slack incoming webhook URL | notifications |
| `CODECOV_TOKEN` | Codecov upload token | coverage upload |
| `GITHUB_TOKEN` | Auto-provided by GitHub | GHCR push |

### Required GitHub Variables

| Variable | Description |
|----------|-------------|
| `DOCKER_IMAGE_NAME` | `ghcr.io/<org>/fastdepo-api` |

---

## 6. Docker Build Optimization

### Multi-Stage Dockerfile

```dockerfile
# Dockerfile

# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
COPY prisma ./prisma/
RUN npm ci --only=production && npx prisma generate

# Stage 2: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Production
FROM node:20-alpine AS production
WORKDIR /app

# Install dumb-init for signal handling
RUN apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only what's needed
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/prisma ./prisma
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./

USER appuser

EXPOSE 3000

ENV NODE_ENV=production
ENV PORT=3000

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

### `.dockerignore`

```
node_modules
.git
.github
.vscode
coverage
playwright-report
test-results
dist
*.md
.env*
.dockerignore
Dockerfile
docker-compose*.yml
```

### Docker Build Caching

The GitHub Actions workflows above use `cache-from: type=gha` and `cache-to: type=gha,mode=max` for Docker layer caching in GHCR builds.

For local Docker builds:

```bash
# Build with caching
docker build \
  --cache-from ghcr.io/your-org/fastdepo-api:latest \
  -t fastdepo-api:latest \
  -t fastdepo-api:$(git describe --tags) \
  .

# Run locally
docker run -p 3000:3000 \
  -e DATABASE_URL=postgresql://postgres:postgres@host.docker.internal:5432/fastdepo \
  fastdepo-api:latest
```

---

## 7. Branch Protection Rules

Configure in **Settings > Branches > Branch protection rules** for `main`:

### Required Checks

- ✅ Require status checks to pass before merging
  - `Lint`
  - `Type Check`
  - `Test`
  - `Build`
  - `E2E Tests`
- ✅ Require branches to be up to date before merging

### Required Reviews

- ✅ Require pull request reviews before merging
  - Required approving reviews: 1
  - Dismiss stale reviews on new commits
- ✅ Require review from Code Owners (if `.github/CODEOWNERS` exists)

### Restrictions

- ✅ Require signed commits
- ✅ Require linear history (no merge commits)
- ✅ Do not allow force pushes
- ✅ Do not allow deletions

### CODEOWNERS File

```
# .github/CODEOWNERS
# Global owners
* @trident

# Backend
/src/services/ @trident
/src/repositories/ @trident
/prisma/ @trident

# Frontend
/src/components/ @trident
/src/pages/ @trident
/src/api/ @trident

# Infrastructure
/docker-compose*.yml @trident
/.github/workflows/ @trident
/Dockerfile @trident
```

---

## 8. Semantic Versioning with Git Tags

### Creating a Release

```bash
# Patch release (1.0.0 -> 1.0.1)
npm version patch -m "chore: release v%s"
git push --follow-tags

# Minor release (1.0.0 -> 1.1.0)
npm version minor -m "chore: release v%s"
git push --follow-tags

# Major release (1.0.0 -> 2.0.0)
npm version major -m "chore: release v%s"
git push --follow-tags
```

### Automated Version Bumping (Alternative)

Add to `package.json` scripts:

```json
{
  "scripts": {
    "release:patch": "npm version patch -m 'chore(release): v%s'",
    "release:minor": "npm version minor -m 'chore(release): v%s'",
    "release:major": "npm version major -m 'chore(release): v%s'"
  }
}
```

The `npm version` command updates `package.json`, creates a commit, and creates a tag. Pushing the tag triggers `deploy-production.yml`.

### Conventional Commits

Enforce conventional commit messages:

```bash
npm install -D @commitlint/cli @commitlint/config-conventional husky
npx husky init
```

`.husky/commit-msg`:

```bash
npx --no -- commitlint --edit $1
```

`commitlint.config.js`:

```javascript
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'build', 'ci', 'chore', 'revert',
    ]],
  },
};
```

---

## 9. Rollback Procedures

### Railway Rollback

```bash
# List deployments
railway status

# Rollback to previous deployment
railway rollback

# Or rollback to a specific deployment
railway rollback --deployment <deployment-id>
```

### Docker/GHCR Rollback

```bash
# Re-deploy a previous image
docker pull ghcr.io/your-org/fastdepo-api:v1.0.0
# Update Railway/Railway to use this specific image tag
```

### Database Rollback

```bash
# NEVER use prisma migrate dev in production

# Option 1: Create a reverse migration
npx prisma migrate dev --name revert_add_avatar_url
# Then edit the generated migration SQL to DROP the column

# Option 2: Manual SQL rollback (use with extreme caution)
psql $DATABASE_URL -c "ALTER TABLE \"User\" DROP COLUMN \"avatarUrl\";"
```

### Emergency Rollback Script

```bash
#!/bin/bash
# scripts/rollback.sh
set -e

PREVIOUS_TAG=${1:?Usage: rollback.sh <previous-tag>}

echo "Rolling back to $PREVIOUS_TAG..."

# Tag the rollback
git tag -f "$PREVIOUS_TAG.rollback" "$PREVIOUS_TAG"
git push origin "$PREVIOUS_TAG.rollback"

echo "Rollback tag created. The deployment pipeline will redeploy $PREVIOUS_TAG."
echo "Verify manually: curl -s $PRODUCTION_URL/api/health"
```

---

## 10. Database Migration in CI

### Migrations in Staging

The staging deployment runs migrations automatically via Railway integration. The `deploy-staging.yml` runs `prisma migrate deploy` before deploying.

### Migrations in Production

The `deploy-production.yml` has a dedicated `run-migrations` job that:

1. Runs **after** the Docker image is built and pushed
2. Runs **before** the production deploy
3. Uses `prisma migrate deploy` (production-safe — applies pending migrations only)

### `prisma migrate deploy` vs `prisma migrate dev`

| Command | Use In | Behavior |
|---------|--------|----------|
| `prisma migrate dev` | Development only | Creates new migrations, resets DB if needed |
| `prisma migrate deploy` | Production | Applies pending migrations, never resets |
| `prisma db push` | Prototyping only | Pushes schema without migrations, no history |

### Migration Safety in CI

```yaml
# In deploy-production.yml, the migration step includes a backup:

- name: Backup database before migration
  run: |
    TIMESTAMP=$(date +%Y%m%d%H%M%S)
    pg_dump "$PRODUCTION_DATABASE_URL" | gzip > "/tmp/backup_$TIMESTAMP.sql.gz"

- name: Upload backup to S3
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1

- run: aws s3 cp "/tmp/backup_$TIMESTAMP.sql.gz" s3://fastdepo-db-backups/

- name: Run production migrations
  run: npx prisma migrate deploy
  env:
    DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
```

---

## 11. Preview Deployments for PRs

```yaml
# .github/workflows/preview.yml
name: Preview Deployment

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  preview:
    name: Deploy Preview
    runs-on: ubuntu-latest
    if: github.event.action != 'closed'

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Deploy preview to Railway
        id: deploy
        uses: railwayapp/railway-deploy-action@v1
        with:
          railway-token: ${{ secrets.RAILWAY_PREVIEW_TOKEN }}
          railway-environment: preview
          railway-service: fastdepo-api-preview

      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });
            const botComment = comments.find(c => c.user.login === 'github-actions[bot]' && c.body.includes('Preview Deployment'));
            const body = `## 🚀 Preview Deployment\n\nURL: ${{ steps.deploy.outputs.url }}\n\nThis preview will be updated on each push to this PR.`;
            if (botComment) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
                body,
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.issue.number,
                body,
              });
            }

  cleanup:
    name: Cleanup Preview
    runs-on: ubuntu-latest
    if: github.event.action == 'closed'

    steps:
      - name: Destroy preview environment
        uses: railwayapp/railway-deploy-action@v1
        with:
          railway-token: ${{ secrets.RAILWAY_PREVIEW_TOKEN }}
          railway-environment: preview
          command: destroy
```

---

## 12. Monitoring Pipeline Success

### GitHub Actions Status Badges

Add to your README:

```markdown
![CI](https://github.com/your-org/fastdepo/workflows/CI/badge.svg)
![Deploy Staging](https://github.com/your-org/fastdepo/workflows/Deploy%20Staging/badge.svg)
![Deploy Production](https://github.com/your-org/fastdepo/workflows/Deploy%20Production/badge.svg)
```

### Slack/Discord Notification on Failure

Already included in the deployment workflows above. To add notifications to the CI workflow:

```yaml
# Add to ci.yml after the test job
  notify-on-failure:
    name: Notify on Failure
    runs-on: ubuntu-latest
    needs: [lint, typecheck, test, e2e]
    if: failure()

    steps:
      - name: Slack notification
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: |
            {
              "text": "⚠️ CI Pipeline Failed",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "⚠️ *CI Pipeline Failed*\nBranch: `${{ github.ref_name }}`\nCommit: `${{ github.sha }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View logs>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Discord Notification (Alternative)

```yaml
  - name: Discord notification
    if: failure()
    uses: sarisia/actions-status-discord@v1
    with:
      webhook: ${{ secrets.DISCORD_WEBHOOK_URL }}
      title: "CI Pipeline Failed"
      description: "Branch: ${{ github.ref_name }}\nCommit: ${{ github.sha }}"
      color: 0xff0000
```

### Cron-Based Smoke Tests

```yaml
# .github/workflows/smoke-test.yml
name: Smoke Test

on:
  schedule:
    - cron: '*/15 * * * *'  # Every 15 minutes
  workflow_dispatch:

jobs:
  smoke:
    runs-on: ubuntu-latest
    steps:
      - name: Health check
        run: |
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${{ secrets.PRODUCTION_URL }}/api/health")
          if [ "$STATUS" != "200" ]; then
            echo "Health check failed with status $STATUS"
            exit 1
          fi
          echo "Health check passed"

      - name: Slack alert
        if: failure()
        uses: slackapi/slack-github-action@v1.27.0
        with:
          payload: |
            {
              "text": "🔴 Production health check failed!",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "🔴 *Production health check failed!*\nChecked at: $(date -u)"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```
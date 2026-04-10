# Repository Review: GENAI Design & Build Instructions

**Reviewer**: Expert Software Engineer (10+ years building production systems)
**Date**: 2025-01-15
**Repository**: GENAI Design & Build Repository
**Scope**: 32,393 lines of documentation across 44 files (13 root + 30 skill files + 1 meta)

---

## Executive Summary

**Rating: 6.5/10**

A comprehensive but incomplete blueprint repository. Contains excellent structural thinking (skill dependency graph, input/output contracts) but lacks execution verification, suffers from early-stage bloat (three overlapping guides), and has not been proven by actually following the instructions to build anything.

---

## What's Good (7/10)

### 1. Skill Dependency Architecture (8.5/10)
The skill file structure with explicit `Input Contract`, `Output Contract`, `Verification`, and `Rollback` sections is genuinely excellent. This is how professional onboarding documentation should work.

**Example** (skills/03-auth.md):
```markdown
## Input Contract
- ✅ Express app in src/app.ts
- ✅ Prisma client in src/config/database.ts
- ✅ User model in prisma/schema.prisma
- ✅ JWT_SECRET in .env
```

This clarity prevents the "where do I start?" problem that plagues most documentation.

### 2. Comprehensive Coverage (8/10)
Backend, infrastructure, frontend, mobile (Flutter + Expo), testing, CI/CD, database — it's all here. The stack choices (Express, PostgreSQL, Prisma, Docker, Railway) are sensible and well-reasoned.

### 3. Error Handling Patterns (8/10)
The AppError hierarchy with structured JSON responses is production-grade thinking:
```json
{ "error": { "code": "VALIDATION_ERROR", "message": "...", "requestId": "uuid" }}
```

### 4. Infrastructure as Code (7.5/10)
Docker Compose with healthchecks, proper multi-stage Dockerfile, GitHub Actions CI/CD — infrastructure is properly thought through.

---

## What's Missing or Problematic (4/10)

### 1. Zero Verified Execution (4/10)
No one has actually followed these instructions to build anything. The skills exist as documentation but have never been executed. This is the biggest problem.

**Evidence**: No `package.json`, no `src/` directory with actual code, no `docker-compose up` that actually works.

**Impact**: Skills may have bugs, missing dependencies, or contradictions that only become apparent when building. An expert knows that "documentation is fake until you've walked through it."

### 2. Three Overlapping Guides Cause Confusion (5/10)
The repo has three guides covering the same ground:

| Guide | Approach | Best For |
|-------|----------|----------|
| `backend-infra-build-instructions.md` | Narrative walkthrough | Understanding |
| `all-27-modules-build-instructions.md` | Detailed instructions | Building |
| `full-stack-complete-guide-macos-android.md` | Copy-paste code | Reference |

While the README attempts to clarify when to use which, it's still cognitive overhead. An expert would choose **one format** as primary and use the others as supplementary references only.

**Recommendation**: Delete `all-27-modules-build-instructions.md` and `full-stack-complete-guide-macos-android.md`. Keep `backend-infra-build-instructions.md` as the single authoritative guide. The skill files already contain all the implementation details.

### 3. No DECISIONS.md File (4/10)
The repository references `DECISIONS.md` in Phase 0 but it was never created. This file should contain:
- Why Express over Fastify/NestJS
- Why Prisma over Drizzle/TypeORM
- Why Railway over Fly/AWS
- JWT vs session vs Clerk

Without this, future maintainers (including future you) don't know why certain choices were made.

### 4. Incomplete Branch Skills (4/10)
The manifest references branch skills (B-auth-clerk, B-auth-supabase, B-deploy-aws, B-deploy-flyio) but none exist as actual files:
```json
"branches": {
  "auth": { "default": "03-auth (Custom JWT)", "alternatives": ["B-auth-clerk (Clerk)", "B-auth-supabase (Supabase Auth)"] }
}
```

These create expectation but deliver nothing. Either build them or remove the references.

### 5. No Testing Code (3/10)
While `testing-strategy.md` and `skills/27-testing.md` describe testing patterns, there are:
- No actual `test/*.test.ts` files
- No Jest/Vitest config in the repo
- No test database setup script
- No `npm test` command that actually runs

### 6. Gallery Module Is Incomplete (5/10)
`skills/06-gallery.md` mentions public sharing but doesn't show:
- How `shareToken` is generated or stored
- How the public explore endpoint works
- How the share link redirects users

The gallery appears halfway built.

### 7. No Worker Entry Point Verified (5/10)
Multiple files reference `dist/worker.js`:
```bash
# docker-compose.yml
command: ["node", "dist/worker.js"]
```

But `skills/08-queue-worker.md` creates `src/worker.ts` and assumes transpilation — there should be a `package.json` script showing the build command that produces `dist/worker.js`.

### 8. Security Gaps (6/10)
- No rate limiting examples in skill files (just says "use express-rate-limit")
- No actual CSRF protection code
- No input sanitization examples (XSS prevention)
- No security headers middleware in any skill file (just references helmet via comment)

An expert expects seeing the actual security code, not just "use X library."

---

## Specific Technical Issues

### Issue 1: Auth Flow Contradiction
- `skills/03-auth.md` says "Rotate refresh tokens (one-time use)"
- But doesn't explicitly handle the race condition where two simultaneous refresh requests both fail

### Issue 2: Database Connection Singleton
- `skills/02-database.md` uses `globalThis` pattern correctly
- But `skills/08-queue-worker.md` creates a new Prisma client without using the same singleton
- This will cause connection pool exhaustion at scale

### Issue 3: WebSocket Missing Entry Point
- `skills/09-websocket.md` describes initializing Socket.IO
- But doesn't show how to run the HTTP server with WebSocket support in production
- The `index.ts` in any skill doesn't show the combined server setup

### Issue 4: Environment Variables Inconsistency
- Some skills reference `JWT_SECRET`
- Some reference `process.env.JWT_SECRET`
- No standard across skills for environment access

### Issue 5: No Database Seed Verification
- `skills/28-env-readme.md` mentions seeding but doesn't verify:
- What happens if seeding fails?
- What's the expected data after seeding?
- How to reset corrupted seed data?

### Issue 6: Docker Multi-Stage Build Missing
- `skills/15-docker.md` describes multi-stage Dockerfile
- But doesn't explain how to handle Prisma migrations at runtime
- No explanation of `npx prisma generate` vs `npx prisma migrate deploy` in Dockerfile

---

## What's Missing Entirely

| Item | Priority | Impact |
|------|----------|--------|
| DECISIONS.md | Critical | Can't understand why decisions were made |
| Actual test files | Critical | Can't run tests |
| Worker entry point build script | High | Can't deploy worker to production |
| Gallery shareToken implementation | Medium | Feature incomplete |
| Security code examples | Medium | Security gaps |
| Branch skill files | Low | Broken promises |
| Admin dashboard | Low | Missing UI |

---

## Web Research: Current Best Practices (2025-2026)

### Express Best Practices (from web search)

| Best Practice | Your Repo Status | Gap |
|---|---|---|
| Multi-stage Dockerfile | ✅ Present (skills/15-docker.md) | Matches |
| Modular architecture (routes/controllers/services) | ✅ Present | Good |
| Centralized error handling | ✅ Present (AppError hierarchy) | Good |
| Helmet, rate limiting, input validation | ⚠️ Referenced but no code | Missing |
| TypeScript | ✅ Present | Good |
| Health checks | ✅ Present | Good |
| Compression | ✅ Present | Good |

### Prisma Best Practices (from web search)

| Best Practice | Your Repo Status | Gap |
|---|---|---|
| Connection pooling config in schema | ❌ Missing | Add `connection_limit` and `pool_timeout` |
| Prisma in production dependencies | ❌ Not specified | Should be in `dependencies`, not `devDependencies` |
| Migration safety (pgfence for production) | ❌ Missing | No pre-deploy safety checks |
| Transaction API for multi-table ops | ⚠️ Mentioned | Not shown in gallery/image create |
| Indexing strategy | ✅ Present (skills/26-db-indexing.md) | Good |
| Prisma migrate deploy in CI/CD | ✅ Present (skills/16-ci-cd.md) | Matches |

### Missing from Prisma (according to 2025 best practices):

1. **Connection pooling in schema**:
```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  connection_limit = 10
  pool_timeout = 20
}
```

2. **Prisma in production dependencies**:
Your skills say `npm install -D prisma` but production best practice is:
```json
"dependencies": {
  "@prisma/client": "^5.0.0"
},
"devDependencies": {
  "prisma": "^5.0.0"
}
```

3. **Migration safety tool (pgfence)**:
No pre-deploy safety check for dangerous migrations like `ALTER COLUMN TYPE` without `CONCURRENTLY`.

---

## Recommendations for 8/10

1. **Delete two guides**: Remove `all-27-modules-build-instructions.md` and `full-stack-complete-guide-macos-android.md`. Keep only `backend-infra-build-instructions.md`.

2. **Build it**: Actually follow the skills to create the codebase. You'll find real bugs.

3. **Create DECISIONS.md**: Fill in the rationale for all key choices.

4. **Add actual test files**: Not just descriptions — real Jest tests.

5. **Build branch skills**: Either create them or remove the manifest references.

6. **Write the worker build script**: Show exactly how `dist/worker.js` is created.

7. **Verify everything**: Run `docker-compose up`, run migrations, run tests.

---

## Final Assessment

This repository shows excellent structural thinking and would score 8/10 as a **concept**. But as **executable documentation**, it scores only 6.5/10 because nothing has been verified to actually work.

The gap between "documented" and "deployed" is where most projects fail. An expert knows this better than anyone.

**Next step**: Actually build skill 01 from scratch and verify you have a working Express server with Prisma connected to PostgreSQL before adding more complexity.
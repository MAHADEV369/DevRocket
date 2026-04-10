# Skill 26: Database Indexing Strategy

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 1-2 hours
Depends On: 02

## Input Contract
- Skill 02 complete: Prisma schema with all models defined and migrations applied
- `prisma/schema.prisma` contains User, Image, Gallery, GalleryImage, RefreshToken, PasswordReset, Notification, and billing models
- PostgreSQL database is running and populated with seed data
- All API endpoints and query patterns are known from Skills 03-13

## Output Contract
- All Prisma schema indexes added for query patterns
- Composite indexes for multi-column queries (userId+createdAt, userId+isPublic, etc.)
- Individual indexes on foreign keys and frequently filtered columns
- Concurrent index creation migration for production safety
- EXPLAIN ANALYZE verification for all indexed queries
- Index review checklist document

## Files to Create

| File | Description |
|------|-------------|
| `prisma/schema.prisma` | Updated with all @@index and @@unique declarations |
| `prisma/migrations/add_performance_indexes/migration.sql` | Raw SQL migration with CONCURRENT index creation |
| `scripts/verify-indexes.ts` | Script to run EXPLAIN ANALYZE on all query patterns |
| `docs/index-review-checklist.md` | Index review checklist and maintenance guide |

## Steps

### Step 1: Analyze Query Patterns

Review all API endpoints and their database queries. Document the query patterns that need indexes:

| Query Pattern | Model | Columns | Type |
|---------------|-------|---------|------|
| User lookup by email | User | email | unique |
| User images list (paginated) | Image | userId, createdAt | composite |
| User images by favorite status | Image | userId, isFavorite | composite |
| Public images for explore | Image | isPublic, createdAt | composite |
| Images by provider | Image | provider, createdAt | composite |
| Gallery lookup by user | Gallery | userId, createdAt | composite |
| Public galleries for explore | Gallery | isPublic, createdAt | composite |
| Gallery images by gallery | GalleryImage | galleryId, position | composite |
| Refresh token lookup | RefreshToken | token | unique |
| Password reset token lookup | PasswordReset | token | unique |
| User notifications | Notification | userId, isRead, createdAt | composite |
| Stripe customer lookup | User | stripeCustomerId | single |
| Subscription status check | User | stripeSubscriptionStatus | single |
| Image by share token | Image | shareToken | unique |

### Step 2: Update Prisma Schema with Indexes

Add indexes to `prisma/schema.prisma`. The following additions go inside each model block:

**User model indexes:**

```prisma
model User {
  // ... existing fields ...

  @@unique([email])
  @@index([stripeCustomerId])
  @@index([stripeSubscriptionStatus])
  @@index([createdAt])
}
```

**Image model indexes:**

```prisma
model Image {
  // ... existing fields ...

  @@index([userId, createdAt])      // User's images sorted by date
  @@index([userId, isFavorite])     // User's favorite images
  @@index([isPublic, createdAt])     // Public explore feed
  @@index([provider, createdAt])    // Images by provider
  @@index([galleryId])              // Images in a gallery
  @@index([createdAt])              // Global date sorting
  @@unique([shareToken])             // Share link lookup
}
```

**Gallery model indexes:**

```prisma
model Gallery {
  // ... existing fields ...

  @@index([userId, createdAt])      // User's galleries sorted by date
  @@index([isPublic, createdAt])    // Public explore galleries
  @@index([createdAt])              // Global date sorting
}
```

**GalleryImage model indexes:**

```prisma
model GalleryImage {
  // ... existing fields ...

  @@index([galleryId, position])    // Gallery images in order
  @@index([imageId])                // Find which galleries contain an image
}
```

**RefreshToken model indexes:**

```prisma
model RefreshToken {
  // ... existing fields ...

  @@unique([token])                  // Token lookup for auth
  @@index([userId])                 // Find all tokens for a user
  @@index([expiresAt])              // Cleanup expired tokens
}
```

**PasswordReset model indexes:**

```prisma
model PasswordReset {
  // ... existing fields ...

  @@unique([token])                  // Token lookup for reset
  @@index([userId])                 // Find resets for a user
  @@index([expiresAt])              // Cleanup expired resets
}
```

**Notification model indexes:**

```prisma
model Notification {
  // ... existing fields ...

  @@index([userId, isRead, createdAt])  // User's notifications by read status
  @@index([userId, createdAt])           // User's notifications sorted by date
  @@index([isRead])                       // Unread count aggregation
}
```

### Step 3: Create Concurrent Index Migration

For production databases with existing data, create indexes concurrently to avoid locking:

`prisma/migrations/add_performance_indexes/migration.sql`:

```sql
-- Concurrent index creation for production safety
-- These cannot run inside a transaction, so each is separate

-- User indexes
CREATE INDEX CONCURRENTLY IF NOT EXISTS "User_stripeCustomerId_idx" ON "User"("stripeCustomerId");
CREATE INDEX CONCURRENTLY IF NOT EXISTS "User_stripeSubscriptionStatus_idx" ON "User"("stripeSubscriptionStatus");
CREATE INDEX CONCURRENTLY IF NOT EXISTS "User_createdAt_idx" ON "User"("createdAt");

-- Image indexes
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Image_userId_createdAt_idx" ON "Image"("userId", "createdAt" DESC);
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Image_userId_isFavorite_idx" ON "Image"("userId", "isFavorite");
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Image_isPublic_createdAt_idx" ON "Image"("isPublic", "createdAt" DESC);
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Image_provider_createdAt_idx" ON "Image"("provider", "createdAt" DESC);
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Image_galleryId_idx" ON "Image"("galleryId");
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Image_createdAt_idx" ON "Image"("createdAt" DESC);

-- Gallery indexes
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Gallery_userId_createdAt_idx" ON "Gallery"("userId", "createdAt" DESC);
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Gallery_isPublic_createdAt_idx" ON "Gallery"("isPublic", "createdAt" DESC);
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Gallery_createdAt_idx" ON "Gallery"("createdAt" DESC);

-- GalleryImage indexes
CREATE INDEX CONCURRENTLY IF NOT EXISTS "GalleryImage_galleryId_position_idx" ON "GalleryImage"("galleryId", "position");
CREATE INDEX CONCURRENTLY IF NOT EXISTS "GalleryImage_imageId_idx" ON "GalleryImage"("imageId");

-- RefreshToken indexes
CREATE INDEX CONCURRENTLY IF NOT EXISTS "RefreshToken_userId_idx" ON "RefreshToken"("userId");
CREATE INDEX CONCURRENTLY IF NOT EXISTS "RefreshToken_expiresAt_idx" ON "RefreshToken"("expiresAt");

-- PasswordReset indexes
CREATE INDEX CONCURRENTLY IF NOT EXISTS "PasswordReset_userId_idx" ON "PasswordReset"("userId");
CREATE INDEX CONCURRENTLY IF NOT EXISTS "PasswordReset_expiresAt_idx" ON "PasswordReset"("expiresAt");

-- Notification indexes
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Notification_userId_createdAt_idx" ON "Notification"("userId", "createdAt" DESC);
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Notification_userId_isRead_createdAt_idx" ON "Notification"("userId", "isRead", "createdAt" DESC);
CREATE INDEX CONCURRENTLY IF NOT EXISTS "Notification_isRead_idx" ON "Notification"("isRead");
```

**Important**: `CONCURRENTLY` cannot run inside a transaction. Prisma migrations run inside transactions by default. To apply this migration in production, you have two options:

Option A - Apply manually:
```bash
# Connect to production database and run indexes one at a time
psql $DATABASE_URL -f prisma/migrations/add_performance_indexes/migration.sql
```

Option B - Use Prisma's `migrate diff` for the schema sync:
```bash
# After running indexes manually, mark the migration as applied
npx prisma migrate resolve --applied add_performance_indexes
```

### Step 4: Create Verification Script

`scripts/verify-indexes.ts`:

```ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

interface ExplainResult {
  'QUERY PLAN': string;
}

async function runExplainAnalyze(query: string, label: string): Promise<void> {
  console.log(`\n=== ${label} ===`);
  console.log(`Query: ${query.substring(0, 100)}...`);

  try {
    const result = await prisma.$queryRawUnsafe<ExplainResult[]>(
      `EXPLAIN ANALYZE ${query}`
    );

    const planLines = result.map((r) => r['QUERY PLAN']);
    const usesIndex = planLines.some(
      (line) =>
        line.includes('Index Scan') || line.includes('Index Only Scan')
    );
    const isSeqScan = planLines.some((line) => line.includes('Seq Scan'));

    if (isSeqScan && !usesIndex) {
      console.log('⚠️  WARNING: Sequential scan detected (no index used)');
    } else if (usesIndex) {
      console.log('✅ Index scan detected');
    } else {
      console.log('ℹ️  Check query plan manually');
    }

    planLines.forEach((line) => console.log(`  ${line}`));
  } catch (error) {
    console.log(`❌ Error: ${error}`);
  }
}

async function verifyIndexes(): Promise<void> {
  console.log('=== Database Index Verification ===\n');

  // User queries
  await runExplainAnalyze(
    `SELECT * FROM "User" WHERE email = 'test@example.com'`,
    'User lookup by email'
  );

  await runExplainAnalyze(
    `SELECT * FROM "User" WHERE "stripeCustomerId" = 'cus_xxx'`,
    'User lookup by Stripe customer ID'
  );

  // Image queries
  await runExplainAnalyze(
    `SELECT * FROM "Image" WHERE "userId" = 'user1' ORDER BY "createdAt" DESC LIMIT 20`,
    'User images by date'
  );

  await runExplainAnalyze(
    `SELECT * FROM "Image" WHERE "userId" = 'user1' AND "isFavorite" = true`,
    'User favorite images'
  );

  await runExplainAnalyze(
    `SELECT * FROM "Image" WHERE "isPublic" = true ORDER BY "createdAt" DESC LIMIT 20`,
    'Public explore images'
  );

  await runExplainAnalyze(
    `SELECT * FROM "Image" WHERE "provider" = 'dalle' ORDER BY "createdAt" DESC`,
    'Images by provider'
  );

  await runExplainAnalyze(
    `SELECT * FROM "Image" WHERE "shareToken" = 'abc123'`,
    'Image share token lookup'
  );

  // Gallery queries
  await runExplainAnalyze(
    `SELECT * FROM "Gallery" WHERE "userId" = 'user1' ORDER BY "createdAt" DESC`,
    'User galleries by date'
  );

  await runExplainAnalyze(
    `SELECT * FROM "Gallery" WHERE "isPublic" = true ORDER BY "createdAt" DESC LIMIT 20`,
    'Public explore galleries'
  );

  // GalleryImage queries
  await runExplainAnalyze(
    `SELECT * FROM "GalleryImage" WHERE "galleryId" = 'gal1' ORDER BY "position" ASC`,
    'Gallery images in order'
  );

  await runExplainAnalyze(
    `SELECT * FROM "GalleryImage" WHERE "imageId" = 'img1'`,
    'Find galleries containing an image'
  );

  // RefreshToken queries
  await runExplainAnalyze(
    `SELECT * FROM "RefreshToken" WHERE token = 'token123'`,
    'Refresh token lookup'
  );

  await runExplainAnalyze(
    `SELECT * FROM "RefreshToken" WHERE "expiresAt" < NOW()`,
    'Expired token cleanup'
  );

  // Notification queries
  await runExplainAnalyze(
    `SELECT * FROM "Notification" WHERE "userId" = 'user1' AND "isRead" = false ORDER BY "createdAt" DESC`,
    'Unread notifications'
  );

  await runExplainAnalyze(
    `SELECT COUNT(*) FROM "Notification" WHERE "userId" = 'user1' AND "isRead" = false`,
    'Unread notification count'
  );

  // Summary: list all indexes
  console.log('\n\n=== All Indexes in Database ===');
  const indexes = await prisma.$queryRawUnsafe<{
    tablename: string;
    indexname: string;
    indexdef: string;
  }[]>(`
    SELECT
      t.relname AS tablename,
      i.relname AS indexname,
      pg_get_indexdef(i.oid) AS indexdef
    FROM pg_index idx
    JOIN pg_class t ON t.oid = idx.indrelid
    JOIN pg_class i ON i.oid = idx.indexrelid
    JOIN pg_namespace n ON n.oid = t.relnamespace
    WHERE n.nspname = 'public'
    ORDER BY t.relname, i.relname
  `);

  let currentTable = '';
  indexes.forEach((idx) => {
    if (idx.tablename !== currentTable) {
      currentTable = idx.tablename;
      console.log(`\n${currentTable}:`);
    }
    console.log(`  ${idx.indexname}: ${idx.indexdef}`);
  });

  await prisma.$disconnect();
}

verifyIndexes().catch(console.error);
```

### Step 5: Apply Migrations

```bash
# Development: Apply via Prisma (indexes are small, fast create)
npx prisma migrate dev --name add_performance_indexes

# Production: Apply concurrently (no locking)
# Step 1: Create the migration SQL file
npx prisma migrate diff \
  --from-migrations prisma/migrations \
  --to-schema-datamodel prisma/schema.prisma \
  --script > prisma/migrations/add_performance_indexes/migration.sql

# Step 2: Edit the generated migration.sql to add CONCURRENTLY
# (See Step 3 for the modified version)

# Step 3: Apply to production database manually
psql $DATABASE_URL < prisma/migrations/add_performance_indexes/migration.sql

# Step 4: Mark migration as applied in Prisma
npx prisma migrate resolve --applied add_performance_indexes
```

### Step 6: Verify Index Usage

```bash
# Run the verification script
npx ts-node scripts/verify-indexes.ts

# Expected: All queries should show "Index Scan" in the EXPLAIN ANALYZE output
# No sequential scans should appear on large tables

# Manual verification of specific slow queries:
psql $DATABASE_URL -c "
  EXPLAIN ANALYZE
  SELECT * FROM \"Image\"
  WHERE \"userId\" = 'test-user-id'
  ORDER BY \"createdAt\" DESC
  LIMIT 20;
"

# Check index sizes
psql $DATABASE_URL -c "
  SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
  FROM pg_stat_user_indexes
  ORDER BY pg_relation_size(indexrelid) DESC;
"
```

### Step 7: Create Index Review Checklist

`docs/index-review-checklist.md`:

```markdown
# Index Review Checklist

## When to Review
- After adding new database queries
- After modifying existing query patterns
- After significant data growth (>10x original size)
- Monthly as part of routine maintenance

## Checklist

### New Query Verification
- [ ] Identify all WHERE clause columns — add single indexes
- [ ] Identify multi-column WHERE clauses — add composite indexes
- [ ] Identify ORDER BY patterns — include sort column in index
- [ ] Check EXPLAIN ANALYZE — no sequential scans on tables > 1000 rows
- [ ] Verify index is being used — check `pg_stat_user_indexes.idx_scan`

### Existing Index Maintenance
- [ ] Check for unused indexes (`idx_scan = 0` in pg_stat_user_indexes)
- [ ] Review index bloat (`pg_stat_user_indexes` — compare idx_scan to table size)
- [ ] Monitor index size growth (should be proportional to data growth)
- [ ] Consider dropping indexes with < 10 scans/month on production

### Composite Index Rules
- [ ] Equality columns first, range columns second
- [ ] Most selective column first
- [ ] Cover commonly selected columns (include in index for index-only scans)
- [ ] Limit composite indexes to 3-4 columns max

### Production Safety
- [ ] Always use CREATE INDEX CONCURRENTLY for existing data
- [ ] Monitor disk space during index creation
- [ ] Run during low-traffic periods if table has > 1M rows
- [ ] Verify application queries after index changes

### Monitoring Queries

```sql
-- Find unused indexes
SELECT schemaname, tablename, indexname
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER by pg_relation_size(indexrelid) DESC;

-- Find index bloat
SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;

-- Check index usage ratio
SELECT schemaname, tablename, indexname,
       idx_scan as index_scans,
       seq_scan as table_scans,
       CASE WHEN seq_scan > 0
            THEN round(idx_scan::numeric / seq_scan, 2)
            ELSE NULL
       END as scan_ratio
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_scan DESC;
```
```

### Step 8: Add indexes for new models from later skills

If billing models (from Skill 12) have been added, also include:

```prisma
model Subscription {
  // ... existing fields ...
  @@index([userId])
  @@index([stripeSubscriptionId])
  @@index([status])
  @@index([currentPeriodEnd])
}

model Payment {
  // ... existing fields ...
  @@index([userId, createdAt])
  @@index([stripePaymentIntentId])
  @@index([status])
}
```

## Verification

```bash
# 1. Generate and apply the migration
npx prisma migrate dev --name add_performance_indexes

# 2. Run the verification script
npx ts-node scripts/verify-indexes.ts

# 3. Expected: All queries use Index Scan, no Sequential Scan on data tables

# 4. Check index count
psql $DATABASE_URL -c "SELECT count(*) FROM pg_indexes WHERE schemaname = 'public';"
# Expected: 20+ indexes (depending on model count)

# 5. Verify specific critical queries
psql $DATABASE_URL -c "EXPLAIN ANALYZE SELECT * FROM \"Image\" WHERE \"userId\" = 'test' ORDER BY \"createdAt\" DESC LIMIT 20;"
# Expected: "Index Scan using Image_userId_createdAt_idx"

psql $DATABASE_URL -c "EXPLAIN ANALYZE SELECT * FROM \"Image\" WHERE \"isPublic\" = true ORDER BY \"createdAt\" DESC LIMIT 20;"
# Expected: "Index Scan using Image_isPublic_createdAt_idx"

psql $DATABASE_URL -c "EXPLAIN ANALYZE SELECT * FROM \"RefreshToken\" WHERE token = 'test';"
# Expected: "Index Scan using RefreshToken_token_key"
```

## Rollback

```bash
# To remove all performance indexes (NOT unique constraints)

# Development:
npx prisma migrate dev --name remove_performance_indexes
# Then remove all @@index declarations from schema.prisma

# Production: Drop indexes concurrently
psql $DATABASE_URL <<'SQL'
DROP INDEX CONCURRENTLY IF EXISTS "User_stripeCustomerId_idx";
DROP INDEX CONCURRENTLY IF EXISTS "User_stripeSubscriptionStatus_idx";
DROP INDEX CONCURRENTLY IF EXISTS "User_createdAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Image_userId_createdAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Image_userId_isFavorite_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Image_isPublic_createdAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Image_provider_createdAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Image_galleryId_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Image_createdAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Gallery_userId_createdAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Gallery_isPublic_createdAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Gallery_createdAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "GalleryImage_galleryId_position_idx";
DROP INDEX CONCURRENTLY IF EXISTS "GalleryImage_imageId_idx";
DROP INDEX CONCURRENTLY IF EXISTS "RefreshToken_userId_idx";
DROP INDEX CONCURRENTLY IF EXISTS "RefreshToken_expiresAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "PasswordReset_userId_idx";
DROP INDEX CONCURRENTLY IF EXISTS "PasswordReset_expiresAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Notification_userId_createdAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Notification_userId_isRead_createdAt_idx";
DROP INDEX CONCURRENTLY IF EXISTS "Notification_isRead_idx";
SQL

# Then remove @@index declarations from schema.prisma
# And run: npx prisma migrate resolve --applied remove_performance_indexes
```

## ADR-026: Prisma @@index vs Raw SQL for Index Management

**Decision**: Use Prisma `@@index` declarations in schema.prisma for index definition, but use raw SQL migrations with `CREATE INDEX CONCURRENTLY` for production deployment.

**Reason**: Prisma `@@index` provides a single source of truth for index definitions alongside the schema. However, Prisma migrations create indexes within transactions, which locks the table. For production databases with existing data, `CONCURRENTLY` is required to avoid downtime. We define indexes in Prisma schema for documentation and dev migrations, but use raw SQL for production.

**Consequences**:
- Developers can see all indexes in `schema.prisma` (documentation value)
- Dev environment uses Prisma's automatic index creation (simpler)
- Production requires manual SQL execution for concurrent index creation
- Migration state must be manually marked as applied after SQL execution
- Need to keep Prisma schema and raw SQL in sync

**Alternatives Considered**:
- Only raw SQL migrations: Loses the documentation value of Prisma schema; indexes are hidden in migration files
- Only Prisma @@index: Table locks during production migration; unacceptable for large tables
- Third-party tool (pg_index_manager): Overkill for this project; adds another dependency
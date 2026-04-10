# Prisma Migrations Guide

Complete guide to managing database schema changes with Prisma Migrations for FastDepo.

---

## 1. How Prisma Migrations Work

Prisma migrations compare your `schema.prisma` file against the actual database state. When you run `prisma migrate dev`, Prisma:

1. Reads the current `schema.prisma`
2. Compares it against the last migrated state (stored in `_prisma_migrations` table)
3. Calculates the SQL diff needed
4. Generates a new migration folder in `prisma/migrations/` with the SQL
5. Applies the migration to the database
6. Updates the `_prisma_migrations` table

Each migration is a folder containing:
```
prisma/migrations/
└── 20260410120000_add_user_avatar_url/
    └── migration.sql
```

The `migration.sql` file contains the exact SQL that will be applied. You can edit it before applying.

---

## 2. Creating Your First Migration

### Initial Setup

```bash
# Install Prisma
npm install prisma @prisma/client
npx prisma init
```

This creates `prisma/schema.prisma` and `.env`. Configure your database URL:

```env
# .env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/fastdepo?schema=public"
```

### Initial Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id           String    @id @default(cuid())
  email        String    @unique
  name         String
  passwordHash String    @map("password_hash")
  avatarUrl    String?
  createdAt    DateTime  @default(now()) @map("created_at")
  updatedAt    DateTime  @updatedAt @map("updated_at")
  images       Image[]
  refreshTokens RefreshToken[]

  @@map("users")
}

model Image {
  id        String   @id @default(cuid())
  prompt    String
  model     String
  imageUrl  String   @map("image_url")
  userId    String   @map("user_id")
  createdAt DateTime @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@map("images")
}

model RefreshToken {
  id        String   @id @default(cuid())
  token     String   @unique
  userId    String   @map("user_id")
  expiresAt DateTime @map("expires_at")
  createdAt DateTime @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([token])
  @@map("refresh_tokens")
}
```

### Create the Initial Migration

```bash
npx prisma migrate dev --name init
```

This creates:

```
prisma/migrations/
└── 20260410120000_init/
    └── migration.sql
```

The generated `migration.sql`:

```sql
-- CreateTable
CREATE TABLE "users" (
    "id" TEXT NOT NULL,
    "email" TEXT NOT NULL,
    "name" TEXT NOT NULL,
    "password_hash" TEXT NOT NULL,
    "avatar_url" TEXT,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,

    CONSTRAINT "users_pkey" PRIMARY KEY ("id")
);

-- CreateTable
CREATE TABLE "images" (
    "id" TEXT NOT NULL,
    "prompt" TEXT NOT NULL,
    "model" TEXT NOT NULL,
    "image_url" TEXT NOT NULL,
    "user_id" TEXT NOT NULL,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT "images_pkey" PRIMARY KEY ("id")
);

-- CreateTable
CREATE TABLE "refresh_tokens" (
    "id" TEXT NOT NULL,
    "token" TEXT NOT NULL,
    "user_id" TEXT NOT NULL,
    "expires_at" TIMESTAMP(3) NOT NULL,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT "refresh_tokens_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE UNIQUE INDEX "users_email_key" ON "users"("email");

-- CreateIndex
CREATE UNIQUE INDEX "refresh_tokens_token_key" ON "refresh_tokens"("token");

-- CreateIndex
CREATE INDEX "images_user_id_idx" ON "images"("user_id");

-- CreateIndex
CREATE INDEX "refresh_tokens_user_id_idx" ON "refresh_tokens"("user_id");

-- CreateIndex
CREATE INDEX "refresh_tokens_token_idx" ON "refresh_tokens"("token");

-- AddForeignKey
ALTER TABLE "images" ADD CONSTRAINT "images_user_id_fkey" FOREIGN KEY ("user_id") REFERENCES "users"("id") ON DELETE CASCADE ON UPDATE CASCADE;

-- AddForeignKey
ALTER TABLE "refresh_tokens" ADD CONSTRAINT "refresh_tokens_user_id_fkey" FOREIGN KEY ("user_id") REFERENCES "users"("id") ON DELETE CASCADE ON UPDATE CASCADE;
```

### Generate the Prisma Client

```bash
npx prisma generate
```

This must be run after every migration to update the TypeScript client types.

---

## 3. Making Schema Changes

### Add a Column

```prisma
// prisma/schema.prisma - add avatarUrl to User
model User {
  id           String    @id @default(cuid())
  email        String    @unique
  name         String
  passwordHash String    @map("password_hash")
  avatarUrl    String?   @map("avatar_url")   // Add this
  createdAt    DateTime  @default(now()) @map("created_at")
  updatedAt    DateTime  @updatedAt @map("updated_at")
  images       Image[]
  refreshTokens RefreshToken[]

  @@map("users")
}
```

```bash
npx prisma migrate dev --name add_user_avatar_url
```

Generated SQL:

```sql
-- AlterTable
ALTER TABLE "users" ADD COLUMN "avatar_url" TEXT;
```

### Add a Table

```prisma
// prisma/schema.prisma - add Notification model
model Notification {
  id        String   @id @default(cuid())
  userId    String   @map("user_id")
  title     String
  message   String
  read      Boolean  @default(false)
  createdAt DateTime @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId, read])
  @@map("notifications")
}

// Update User model to include relation
model User {
  // ... existing fields
  notifications Notification[]
}
```

```bash
npx prisma migrate dev --name add_notification_model
```

Generated SQL:

```sql
-- CreateTable
CREATE TABLE "notifications" (
    "id" TEXT NOT NULL,
    "user_id" TEXT NOT NULL,
    "title" TEXT NOT NULL,
    "message" TEXT NOT NULL,
    "read" BOOLEAN NOT NULL DEFAULT false,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT "notifications_pkey" PRIMARY KEY ("id")
);

-- CreateIndex
CREATE INDEX "notifications_user_id_read_idx" ON "notifications"("user_id", "read");

-- AddForeignKey
ALTER TABLE "notifications" ADD CONSTRAINT "notifications_user_id_fkey" FOREIGN KEY ("user_id") REFERENCES "users"("id") ON DELETE CASCADE ON UPDATE CASCADE;
```

### Add a Relation

```prisma
// Add a Collection model and relate it to User and Image
model Collection {
  id          String   @id @default(cuid())
  name        String
  description String?
  userId      String   @map("user_id")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  user   User          @relation(fields: [userId], references: [id], onDelete: Cascade)
  images CollectionImage[]

  @@map("collections")
}

model CollectionImage {
  collectionId String @map("collection_id")
  imageId      String @map("image_id")

  collection Collection @relation(fields: [collectionId], references: [id], onDelete: Cascade)
  image      Image      @relation(fields: [imageId], references: [id], onDelete: Cascade)

  @@id([collectionId, imageId])
  @@map("collection_images")
}

// Update User model
model User {
  // ... existing fields
  collections Collection[]
}

// Update Image model
model Image {
  // ... existing fields
  collections CollectionImage[]
}
```

```bash
npx prisma migrate dev --name add_collections
```

### Add an Index

```prisma
// Add compound index on images for user + date filtering
model Image {
  id        String   @id @default(cuid())
  prompt    String
  model     String
  imageUrl  String   @map("image_url")
  userId    String   @map("user_id")
  createdAt DateTime @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([userId, createdAt])  // Add compound index
  @@map("images")
}
```

```bash
npx prisma migrate dev --name add_image_user_date_index
```

Generated SQL:

```sql
-- CreateIndex
CREATE INDEX "images_user_id_created_at_idx" ON "images"("user_id", "created_at");
```

---

## 4. Prisma Migrate Dev vs Prisma DB Push

### When to Use `prisma migrate dev`

- **Any environment that needs migration history** (development, staging, production)
- When you want to track and version database changes
- When multiple developers are collaborating on schema changes
- For production deployments

```bash
# Creates a migration file and applies it
npx prisma migrate dev --name descriptive_name

# Resets the database and applies all migrations from scratch
npx prisma migrate dev --name reset_test --create-only
```

### When to Use `prisma db push`

- **Rapid prototyping** when you don't need to track changes
- Exploring schema options before committing to a migration
- **Never in production**
- Prototyping in a throwaway development database

```bash
# Push schema changes directly to the database (no migration file)
npx prisma db push

# Accept data loss (drops columns/tables)
npx prisma db push --accept-data-loss
```

### Comparison

| Feature | `migrate dev` | `db push` |
|---------|--------------|------------|
| Creates migration files | Yes | No |
| Tracks history | Yes | No |
| Reversible | Yes (via new migration) | No |
| Safe for production | Yes (via `migrate deploy`) | No |
| Speed | Slower (writes files) | Faster (direct apply) |
| Collaborative | Yes | No |
| Use in CI/CD | Yes | No |

### Recommended Workflow

1. Prototype with `db push` on a local/dev database
2. Once satisfied, switch to `migrate dev` to create the proper migration
3. Commit the migration folder to git
4. Deploy with `migrate deploy` in staging/production

---

## 5. Running Migrations in Production

### The ONLY Safe Command

```bash
npx prisma migrate deploy
```

This command:
- Applies only pending migrations (never re-applies)
- Never resets the database
- Fails if a migration has already been applied
- Is idempotent (safe to run multiple times)

### NEVER Run in Production

```bash
# NEVER - can reset your database
npx prisma migrate dev

# NEVER - can cause data loss
npx prisma migrate reset

# NEVER - no migration history, no rollback path
npx prisma db push
```

### Production Migration Process

```bash
# 1. Generate the Prisma client
npx prisma generate

# 2. Review pending migrations
npx prisma migrate status

# 3. Apply pending migrations
npx prisma migrate deploy

# 4. Verify
npx prisma migrate status
```

### CI/CD Integration

```yaml
# In your deployment workflow
- name: Run database migrations
  run: npx prisma migrate deploy
  env:
    DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
```

---

## 6. Migration Rollback Strategies

Prisma does NOT support automatic rollback. Instead, create reverse migrations.

### Strategy 1: Create a New Reverse Migration

If you added `avatarUrl` and need to remove it:

```bash
# 1. Create an empty migration
npx prisma migrate dev --name remove_user_avatar_url --create-only
```

Then edit the generated `migration.sql`:

```sql
-- prisma/migrations/20260410140000_remove_user_avatar_url/migration.sql

-- Reverse the previous migration
ALTER TABLE "users" DROP COLUMN "avatar_url";
```

Update your `schema.prisma` to remove the field:

```prisma
model User {
  id           String    @id @default(cuid())
  email        String    @unique
  name         String
  passwordHash String    @map("password_hash")
  // avatarUrl removed
  createdAt    DateTime  @default(now()) @map("created_at")
  updatedAt    DateTime  @updatedAt @map("updated_at")
  images       Image[]
  refreshTokens RefreshToken[]

  @@map("users")
}
```

Then apply:

```bash
npx prisma migrate dev
```

### Strategy 2: Manual SQL Rollback (Emergency)

For immediate rollback without going through Prisma:

```bash
# Connect to the database
psql "$DATABASE_URL"

# Run manual SQL
ALTER TABLE "users" DROP COLUMN "avatar_url";
```

Then fix the Prisma state:

```bash
# Mark the original migration as rolled back (advanced)
# Or create a baseline migration
```

### Strategy 3: Blue-Green Deployment

Deploy both the old and new versions simultaneously:

1. The new version adds the column (backward-compatible)
2. Deploy new code that uses the column
3. After verification, remove old code paths

This only works for additive changes (adding columns, tables). Breaking changes need careful planning.

---

## 7. Production Migration Safety

### Always Backup First

```bash
# Before any production migration
pg_dump "$DATABASE_URL" | gzip > "backup_$(date +%Y%m%d%H%M%S).sql.gz"
```

### Use Transactions

Prisma migrations run within transactions by default. For large data changes, you may need to split them:

```sql
-- This migration uses transactions by default
ALTER TABLE "users" ADD COLUMN "avatar_url" TEXT;
```

For destructive operations that need explicit transaction handling:

```sql
-- prisma/migrations/20260410150000_add_user_avatar_url/migration.sql

-- Prisma supports transactional migrations
-- Add column with default value for existing rows
ALTER TABLE "users" ADD COLUMN "avatar_url" TEXT;

-- Update existing rows if needed
UPDATE "users" SET "avatar_url" = '' WHERE "avatar_url" IS NULL;
```

### CREATE INDEX CONCURRENTLY

For large tables, creating an index locks the table. Use `CONCURRENTLY` to avoid this:

```bash
# Create empty migration
npx prisma migrate dev --name add_concurrent_index --create-only
```

Edit the SQL:

```sql
-- prisma/migrations/20260410160000_add_concurrent_index/migration.sql

-- IMPORTANT: CONCURRENTLY cannot run inside a transaction
-- Prisma wraps migrations in transactions by default, so we need to disable that

-- By default, Prisma wraps this in a transaction. To use CONCURRENTLY,
-- we need to add a comment to disable the transaction wrapper.
-- Prisma versions 5.10+ support this:

CREATE INDEX CONCURRENTLY "images_user_id_created_at_idx" ON "images"("user_id", "created_at");
```

For Prisma versions that don't support `CONCURRENTLY` in migrations, run the index creation separately:

```bash
# Run outside of Prisma migrations
psql "$DATABASE_URL" -c "CREATE INDEX CONCURRENTLY \"images_user_id_created_at_idx\" ON \"images\"(\"user_id\", \"created_at\");"
```

Then mark it as applied:

```bash
# Dummy migration to track the state
npx prisma migrate dev --name track_concurrent_index --create-only
```

### Migration Safety Checklist

Before running any production migration:

- [ ] Backup the database
- [ ] Test the migration locally with production-like data
- [ ] Review the generated SQL
- [ ] Check for column drops that could cause data loss
- [ ] Verify rollback plan exists
- [ ] Run during low-traffic period
- [ ] Have monitoring alerts ready
- [ ] Verify application code supports both old and new schema when possible

---

## 8. Seed Scripts

### `prisma/seed.ts`

```typescript
import { PrismaClient } from '@prisma/client';
import { hash } from 'bcryptjs';
import { faker } from '@faker-js/faker';

const prisma = new PrismaClient();

async function main() {
  console.log('🌱 Seeding database...');

  // Clean existing data (in order respecting foreign keys)
  await prisma.collectionImage.deleteMany();
  await prisma.collection.deleteMany();
  await prisma.image.deleteMany();
  await prisma.refreshToken.deleteMany();
  await prisma.notification.deleteMany();
  await prisma.user.deleteMany();

  // Create admin user
  const admin = await prisma.user.create({
    data: {
      email: 'admin@fastdepo.com',
      name: 'Admin User',
      passwordHash: await hash('adminpassword123', 12),
      avatarUrl: faker.image.avatar(),
    },
  });

  // Create test users
  const users = [];
  for (let i = 0; i < 10; i++) {
    const user = await prisma.user.create({
      data: {
        email: faker.internet.email(),
        name: faker.person.fullName(),
        passwordHash: await hash('password123', 12),
        avatarUrl: faker.image.avatar(),
      },
    });
    users.push(user);
  }

  // Create images for each user
  const models = ['stable-diffusion-xl', 'dall-e-3', 'flux-schnell'];
  for (const user of [admin, ...users]) {
    const imageCount = faker.number.int({ min: 3, max: 8 });
    for (let i = 0; i < imageCount; i++) {
      await prisma.image.create({
        data: {
          prompt: faker.lorem.sentence({ min: 5, max: 15 }),
          model: faker.helpers.arrayElement(models),
          imageUrl: `https://cdn.fastdepo.com/images/${faker.string.uuid()}.png`,
          userId: user.id,
        },
      });
    }
  }

  // Create collections for admin
  const collection1 = await prisma.collection.create({
    data: {
      name: 'Landscapes',
      description: 'Beautiful natural scenes',
      userId: admin.id,
    },
  });

  // Add some admin images to collection
  const adminImages = await prisma.image.findMany({
    where: { userId: admin.id },
    take: 3,
  });

  for (const image of adminImages) {
    await prisma.collectionImage.create({
      data: {
        collectionId: collection1.id,
        imageId: image.id,
      },
    });
  }

  // Create notifications
  for (const user of [admin, ...users.slice(0, 5)]) {
    await prisma.notification.createMany({
      data: [
        {
          userId: user.id,
          title: 'Welcome to FastDepo!',
          message: 'Start creating amazing images with AI.',
          read: false,
        },
        {
          userId: user.id,
          title: 'New model available',
          message: 'Try the new DALL-E 3 model for your generations.',
          read: faker.datatype.boolean(),
        },
      ],
    });
  }

  // Create refresh tokens for admin
  await prisma.refreshToken.create({
    data: {
      token: faker.string.uuid(),
      userId: admin.id,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30 days
    },
  });

  console.log(`✅ Seed completed:
  - ${1 + users.length} users
  - Images for each user
  - ${1} collection with images
  - Notifications for users
  - Refresh token for admin
  `);
}

main()
  .catch((e) => {
    console.error('❌ Seed failed:', e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

### Configure Seed Command

In `package.json`:

```json
{
  "prisma": {
    "seed": "npx tsx prisma/seed.ts"
  }
}
```

Install dependencies:

```bash
npm install -D tsx @faker-js/faker bcryptjs @types/bcryptjs
```

Run the seed:

```bash
# Development
npx prisma db seed

# Reset and seed
npx prisma migrate reset

# Seed with specific database URL
DATABASE_URL="postgresql://..." npx prisma db seed
```

---

## 9. Schema Evolution Examples

### Adding User.avatarUrl

1. Update `schema.prisma`:

```prisma
model User {
  id           String    @id @default(cuid())
  email        String    @unique
  name         String
  passwordHash String    @map("password_hash")
  avatarUrl    String?   @map("avatar_url")   // New field
  createdAt    DateTime  @default(now()) @map("created_at")
  updatedAt    DateTime  @updatedAt @map("updated_at")
  images       Image[]
  refreshTokens RefreshToken[]
  notifications Notification[]

  @@map("users")
}
```

2. Create migration:

```bash
npx prisma migrate dev --name add_user_avatar_url
```

3. Update TypeScript code to use the new field:

```typescript
// In auth service
const user = await prisma.user.update({
  where: { id: userId },
  data: { avatarUrl: uploadedUrl },
});
```

### Adding Notification Model

Already shown in section 3. Follow the same pattern: update schema → create migration → update code.

### Changing Column Type (e.g., String to Int)

Prisma cannot automatically change column types. You need a multi-step migration:

1. Add a new column:

```prisma
model Image {
  // ...
  likesCountNew Int? @map("likes_count_new")
  // Keep old column temporarily
  likesCount String @default("0") @map("likes_count")
}
```

```bash
npx prisma migrate dev --name add_likes_count_new_int
```

2. Migrate data:

```sql
-- In the migration or manually
UPDATE "images" SET "likes_count_new" = CAST("likes_count" AS INTEGER) WHERE "likes_count_new" IS NULL;
```

3. Remove old column, rename new column:

```prisma
model Image {
  // ...
  likesCount Int @default(0) @map("likes_count")
}
```

```bash
npx prisma migrate dev --name finalize_likes_count_int
```

Edit the migration SQL:

```sql
ALTER TABLE "images" DROP COLUMN "likes_count";
ALTER TABLE "images" RENAME COLUMN "likes_count_new" TO "likes_count";
ALTER TABLE "images" ALTER COLUMN "likes_count" SET NOT NULL;
ALTER TABLE "images" ALTER COLUMN "likes_count" SET DEFAULT 0;
```

---

## 10. Troubleshooting Common Migration Errors

### Error P3005: Database schema is not empty

```
Error: P3005
The database schema is not empty. Read more about how to baseline an existing production database: https://pris.ly/d/migrate-baseline
```

**Fix**: Baseline the database to tell Prisma it's already in sync:

```bash
npx prisma migrate resolve --applied "20260410120000_init"
npx prisma migrate resolve --applied "20260410120001_add_user_avatar_url"
# Apply for each existing migration
```

Or baseline all at once:

```bash
npx prisma migrate diff --from-migrations prisma/migrations --to-schema-datamodel prisma/schema.prisma --script
```

### Error P3006: Migration failed to apply

```
Error: P3006
Migration `20260410120000_init` failed to apply.
```

**Fix**: Check the migration SQL for errors, then either:

1. Fix the SQL and re-apply:

```bash
# In development only
npx prisma migrate reset
npx prisma migrate dev
```

2. Mark the migration as rolled back:

```bash
npx prisma migrate resolve --rolled-back "20260410120000_init"
```

3. Mark the migration as applied (if you manually applied it):

```bash
npx prisma migrate resolve --applied "20260410120000_init"
```

### Error P2011: Null constraint violation

```
Error: P2011
Null constraint violation on the `passwordHash` field.
```

**Fix**: Either:
- Add a default value to the field: `@default("")`
- Make the field optional: `passwordHash String?`
- Provide the value in your seed data or application code

For existing data, add the column as nullable first, then populate, then add the NOT NULL constraint:

```sql
-- Step 1: Add nullable column
ALTER TABLE "users" ADD COLUMN "status" TEXT;

-- Step 2: Set default values
UPDATE "users" SET "status" = 'active';

-- Step 3: Add NOT NULL constraint
ALTER TABLE "users" ALTER COLUMN "status" SET NOT NULL;
```

### Error: Migration is locked

If a migration is stuck in "applying" state:

```bash
# Check current state
npx prisma migrate status

# Unlock if stuck
# Connect to database and clear the lock
psql "$DATABASE_URL" -c "DELETE FROM _prisma_migrations WHERE finished_at IS NULL;"
```

### Drift Detection

If someone manually changed the database:

```bash
npx prisma migrate diff \
  --from-migrations prisma/migrations \
  --to-schema-datamodel prisma/schema.prisma \
  --script
```

This shows any differences between the migration history and your current schema.

To fix drift:

```bash
# Option 1: Create a migration for the drift
npx prisma migrate dev --name fix_drift

# Option 2: Reset and reapply (development only!)
npx prisma migrate reset
```

---

## 11. Prisma Studio for Visual Database Inspection

### Running Prisma Studio

```bash
# Local database
npx prisma studio

# Specific database URL
DATABASE_URL="postgresql://..." npx prisma studio
```

Opens a web UI at `http://localhost:5555` where you can:
- Browse all tables and rows
- Edit records inline
- Filter and sort data
- Add and delete records

### Prisma Studio in Production (Caution!)

```bash
# Tunnel to production database
ssh -L 5555:localhost:5555 production-server

# Then on the production server
DATABASE_URL="$PRODUCTION_DATABASE_URL" npx prisma studio --port 5555 --browser none
```

Access at `http://localhost:5555` through the SSH tunnel.

**Warning**: Never expose Prisma Studio publicly. Only use through SSH tunnels.

---

## 12. Resetting Development Database Safely

```bash
# Reset development database (WARNING: destroys all data)
npx prisma migrate reset

# This will:
# 1. Drop all tables
# 2. Re-apply all migrations
# 3. Run the seed script
```

### Reset Without Seed

```bash
npx prisma migrate reset --skip-seed
```

### Force Reset (Skip Confirmation)

```bash
# Use with caution!
npx prisma migrate reset --force
```

### Save Current Data Before Reset

```bash
# Export data before resetting
pg_dump "$DATABASE_URL" > backup.sql

# Reset
npx prisma migrate reset

# Import data (may need manual adjustment for schema changes)
psql "$DATABASE_URL" < backup.sql
```

---

## 13. Multiple Environments

### Environment Configuration

Create separate `.env` files for each environment:

```
.env                # Default (development)
.env.test           # Test environment
.env.staging        # Staging environment  
.env.production     # Production (never commit this!)
```

`.env`:
```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/fastdepo"
```

`.env.test`:
```
DATABASE_URL="postgresql://postgres:postgres@localhost:5433/fastdepo_test"
```

`.env.staging`:
```
DATABASE_URL="postgresql://fastdepo:password@staging-db.fastdepo.com:5432/fastdepo"
```

### `.gitignore` Configuration

```gitignore
# Environment files
.env
.env.local
.env.*.local
.env.staging
.env.production

# Keep these tracked
!.env.example
```

### Running Migrations per Environment

```bash
# Development
npx prisma migrate dev --name add_feature

# Test
DATABASE_URL="postgresql://postgres:postgres@localhost:5433/fastdepo_test" npx prisma migrate deploy

# Staging
DATABASE_URL="$STAGING_DATABASE_URL" npx prisma migrate deploy

# Production
DATABASE_URL="$PRODUCTION_DATABASE_URL" npx prisma migrate deploy
```

### Environment-Specific Migrations (Advanced)

For environment-specific data or indexes:

```sql
-- prisma/migrations/20260410170000_add_production_indexes/migration.sql

-- Only add these indexes in production where the dataset is large
-- These are safe to run in development too (they just won't be used as much)

CREATE INDEX IF NOT EXISTS "images_model_idx" ON "images"("model");
CREATE INDEX IF NOT EXISTS "notifications_user_read_idx" ON "notifications"("user_id", "read") WHERE "read" = false;
```

### CI/CD Environment Matrix

| Environment | DATABASE_URL Source | Migration Command | Purpose |
|-------------|--------------------|-------------------|---------|
| Local dev | `.env` file | `prisma migrate dev` | Create migrations |
| CI test | GitHub Actions service container | `prisma migrate deploy` | Run tests |
| Staging | Railway/staging secret | `prisma migrate deploy` | Pre-production testing |
| Production | Railway/production secret | `prisma migrate deploy` | Live deployment |

### Script for Multi-Environment Management

```bash
#!/bin/bash
# scripts/migrate.sh
set -e

ENV=${1:-development}

case $ENV in
  development)
    echo "Running development migration..."
    npx prisma migrate dev --name "${2:-update}"
    ;;
  test)
    echo "Running test migrations..."
    DATABASE_URL="$TEST_DATABASE_URL" npx prisma migrate deploy
    ;;
  staging)
    echo "Running staging migrations..."
    DATABASE_URL="$STAGING_DATABASE_URL" npx prisma migrate deploy
    ;;
  production)
    echo "⛔ Running PRODUCTION migrations..."
    read -p "Are you sure? (yes/no) " -r
    echo
    if [[ $REPLY =~ ^yes$ ]]; then
      DATABASE_URL="$PRODUCTION_DATABASE_URL" npx prisma migrate deploy
      echo "✅ Production migrations applied"
    else
      echo "Aborted"
      exit 1
    fi
    ;;
  *)
    echo "Usage: ./migrate.sh [development|test|staging|production] [migration_name]"
    exit 1
    ;;
esac
```

```bash
# Usage
./scripts/migrate.sh development add_user_bio
./scripts/migrate.sh test
./scripts/migrate.sh production
```

---

## Quick Reference

| Command | Use When | Safe for Prod |
|---------|----------|--------------|
| `prisma migrate dev --name X` | Creating new migrations | No |
| `prisma migrate deploy` | Applying pending migrations | Yes |
| `prisma migrate status` | Checking migration state | Yes |
| `prisma migrate reset` | Resetting dev database | No |
| `prisma db push` | Rapid prototyping | No |
| `prisma db seed` | Populating data | Development only |
| `prisma generate` | Updating client types | Yes |
| `prisma studio` | Visual database browsing | Development only |
| `prisma migrate diff` | Comparing schema states | Yes |
| `prisma migrate resolve` | Fixing migration state | Advanced only |
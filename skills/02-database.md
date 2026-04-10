# Skill 02: Database Schema & Migrations

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 2-3 hours
Depends On: 01

## Input Contract
- Skill 01 scaffold complete
- PostgreSQL database server running and accessible
- `DATABASE_URL` set in `.env`

## Output Contract
- `prisma/schema.prisma` with all models (User, Image, Gallery, GalleryImage, RefreshToken, PasswordReset, Notification)
- `prisma/seed.ts` with realistic seed data
- Migrations generated and applied
- `src/config/database.ts` with Prisma client singleton
- `prisma/migrations/.gitkeep` present

## Files to Create

| File | Description |
|------|-------------|
| `prisma/schema.prisma` | Full Prisma schema with all models |
| `prisma/seed.ts` | Realistic seed data for development |
| `src/config/database.ts` | Prisma client singleton |
| `prisma/migrations/.gitkeep` | Ensure migrations dir is tracked |

## Steps

### 1. Initialize Prisma

```bash
npx prisma init --datasource-provider postgresql
```

### 2. Write `prisma/schema.prisma`

Replace the generated file:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id             String           @id @default(cuid())
  email          String           @unique
  username       String           @unique
  name           String?
  passwordHash   String
  role           Role             @default(USER)
  plan           Plan             @default(FREE)
  emailVerified  Boolean          @default(false)
  avatarUrl      String?
  bio            String?
  createdAt      DateTime         @default(now())
  updatedAt      DateTime         @updatedAt

  images           Image[]
  galleries        Gallery[]
  refreshTokens    RefreshToken[]
  passwordResets   PasswordReset[]
  notifications    Notification[]

  @@map("users")
}

model Image {
  id           String    @id @default(cuid())
  userId       String
  prompt       String
  negativePrompt String?
  provider    String    // "dalle", "flux", "pollinations"
  providerId  String?   // ID from the provider
  url          String
  thumbnailUrl String?
  width        Int?
  height       Int?
  isFavorite   Boolean   @default(false)
  isPublic     Boolean   @default(false)
  metadata     Json?
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt

  user          User           @relation(fields: [userId], references: [id], onDelete: Cascade)
  galleryImages GalleryImage[]

  @@index([userId])
  @@index([provider])
  @@index([isFavorite])
  @@map("images")
}

model Gallery {
  id          String    @id @default(cuid())
  userId      String
  title       String
  description String?
  coverUrl    String?
  isPublic    Boolean   @default(false)
  shareToken  String?   @unique
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  user          User           @relation(fields: [userId], references: [id], onDelete: Cascade)
  galleryImages GalleryImage[]

  @@index([userId])
  @@index([shareToken])
  @@map("galleries")
}

model GalleryImage {
  id        String   @id @default(cuid())
  galleryId String
  imageId   String
  position  Int      @default(0)
  createdAt DateTime @default(now())

  gallery Gallery @relation(fields: [galleryId], references: [id], onDelete: Cascade)
  image   Image   @relation(fields: [imageId], references: [id], onDelete: Cascade)

  @@unique([galleryId, imageId])
  @@index([galleryId])
  @@map("gallery_images")
}

model RefreshToken {
  id        String   @id @default(cuid())
  userId    String
  token     String   @unique
  expiresAt DateTime
  isRevoked  Boolean  @default(false)
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([token])
  @@map("refresh_tokens")
}

model PasswordReset {
  id        String   @id @default(cuid())
  userId    String
  token     String   @unique
  expiresAt DateTime
  usedAt    DateTime?
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([token])
  @@map("password_resets")
}

model Notification {
  id        String   @id @default(cuid())
  userId    String
  type      String   // "email_verification", "password_reset", "image_ready", "info"
  title     String
  message   String
  isRead    Boolean  @default(false)
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([isRead])
  @@map("notifications")
}

enum Role {
  USER
  ADMIN
}

enum Plan {
  FREE
  PRO
  ENTERPRISE
}
```

### 3. Create `src/config/database.ts`

```typescript
import { PrismaClient } from '@prisma/client';
import { logger } from '../utils/logger.js';

export const prisma = new PrismaClient({
  log: [
    { emit: 'event', level: 'query' },
    { emit: 'event', level: 'error' },
    { emit: 'event', level: 'warn' },
  ],
});

prisma.$on('query', (e) => {
  logger.debug({ query: e.query, duration: e.duration, params: e.params }, 'Query');
});

prisma.$on('error', (e) => {
  logger.error({ message: e.message }, 'Prisma error');
});

prisma.$on('warn', (e) => {
  logger.warn({ message: e.message }, 'Prisma warning');
});

export async function disconnectDatabase(): Promise<void> {
  await prisma.$disconnect();
  logger.info('Database disconnected');
}
```

### 4. Create `prisma/seed.ts`

```typescript
import { PrismaClient, Role, Plan } from '@prisma/client';
import bcrypt from 'bcryptjs';

const prisma = new PrismaClient();

async function main() {
  console.log('Seeding database...');

  const passwordHash = await bcrypt.hash('Password123!', 12);

  const admin = await prisma.user.upsert({
    where: { email: 'admin@fastdepo.com' },
    update: {},
    create: {
      email: 'admin@fastdepo.com',
      username: 'admin',
      name: 'Admin User',
      passwordHash,
      role: Role.ADMIN,
      plan: Plan.ENTERPRISE,
      emailVerified: true,
    },
  });

  const user1 = await prisma.user.upsert({
    where: { email: 'alice@example.com' },
    update: {},
    create: {
      email: 'alice@example.com',
      username: 'alice',
      name: 'Alice Chen',
      passwordHash,
      role: Role.USER,
      plan: Plan.PRO,
      emailVerified: true,
    },
  });

  const user2 = await prisma.user.upsert({
    where: { email: 'bob@example.com' },
    update: {},
    create: {
      email: 'bob@example.com',
      username: 'bob',
      name: 'Bob Martinez',
      passwordHash,
      role: Role.USER,
      plan: Plan.FREE,
      emailVerified: false,
    },
  });

  const images = await Promise.all([
    prisma.image.create({
      data: {
        userId: admin.id,
        prompt: 'A serene mountain landscape at sunset with golden light',
        provider: 'dalle',
        url: 'https://placehold.co/1024x1024/png?text=Mountain+Sunset',
        width: 1024,
        height: 1024,
        isFavorite: true,
        isPublic: true,
      },
    }),
    prisma.image.create({
      data: {
        userId: user1.id,
        prompt: 'Cyberpunk city street at night with neon signs',
        negativePrompt: 'blurry, low quality',
        provider: 'flux',
        url: 'https://placehold.co/768x768/png?text=Cyberpunk+City',
        width: 768,
        height: 768,
        isFavorite: false,
        isPublic: true,
      },
    }),
    prisma.image.create({
      data: {
        userId: user1.id,
        prompt: 'Cute cat wearing a tiny astronaut helmet floating in space',
        provider: 'pollinations',
        url: 'https://placehold.co/512x512/png?text=Space+Cat',
        width: 512,
        height: 512,
        isFavorite: true,
        isPublic: false,
      },
    }),
    prisma.image.create({
      data: {
        userId: user2.id,
        prompt: 'Watercolor painting of a Japanese garden with cherry blossoms',
        provider: 'dalle',
        url: 'https://placehold.co/1024x1024/png?text=Japanese+Garden',
        width: 1024,
        height: 1024,
        isPublic: true,
      },
    }),
  ]);

  const gallery = await prisma.gallery.create({
    data: {
      userId: user1.id,
      title: 'My Best AI Art',
      description: 'A collection of my favorite AI-generated images',
      isPublic: true,
      shareToken: 'share-demo-token-123',
    },
  });

  await prisma.galleryImage.createMany({
    data: [
      { galleryId: gallery.id, imageId: images[1].id, position: 0 },
      { galleryId: gallery.id, imageId: images[2].id, position: 1 },
    ],
  });

  await prisma.notification.createMany({
    data: [
      { userId: user2.id, type: 'email_verification', title: 'Verify your email', message: 'Please verify your email address to unlock all features.' },
      { userId: user1.id, type: 'info', title: 'Welcome to FastDepo!', message: 'You have 50 image credits on your Pro plan.' },
    ],
  });

  console.log('Seeding complete.');
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

### 5. Install seed dependencies

```bash
npm install bcryptjs && npm install -D @types/bcryptjs
```

### 6. Add seed config to `package.json`

Add this to the top level of `package.json`:

```json
"prisma": {
  "seed": "tsx prisma/seed.ts"
}
```

### 7. Create migrations and seed

```bash
npx prisma migrate dev --name init
npm run db:seed
```

### 8. Verify database

```bash
npx prisma studio
```

Open Prisma Studio and confirm all tables exist and seed data is present.

## Verification

```bash
npx prisma migrate status
# Expected: Database schema is up to date!

npx prisma db execute --stdin <<< "SELECT count(*) FROM users;"
# Expected: count = 3

npm run db:seed
# Expected: Seeding complete. (no errors on re-run)

npm run typecheck
# Expected: no errors
```

## Rollback

```bash
npx prisma migrate reset --force
# Or to fully remove:
rm -rf prisma/migrations prisma/schema.prisma prisma/seed.ts src/config/database.ts
# Drop the database and recreate if needed
```

## ADR-002: Prisma ORM Over Raw SQL

Decision: Use Prisma as the ORM with PostgreSQL.
Reason: Prisma provides type-safe database access, automatic migration management, and an excellent developer experience with Prisma Studio. The schema-as-code approach keeps database documentation close to the source.
Consequences: Prisma adds a build step (`prisma generate`). Complex queries may need raw SQL escapes. The Prisma Client bundle adds ~5MB.
Alternatives Considered: TypeORM (heavier, decorators leak into domain), Drizzle (less mature, smaller ecosystem), raw SQL with `pg` (no type safety, hand-written migrations), MongoDB (different data model, less suitable for relational data).
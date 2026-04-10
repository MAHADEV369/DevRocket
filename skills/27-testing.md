# Skill 27: Testing Strategy

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 3-4 hours
Depends On: 03, 05

## Input Contract
- Skill 03 complete: auth module with routes, service, validation, types
- Skill 05 complete: image generation CRUD with provider pattern
- Express + TypeScript project with `src/` structure
- PostgreSQL database running and accessible
- All API endpoints documented and functional

## Output Contract
- Jest/Vitest configuration with test database setup
- Unit tests for auth service, image service, and gallery service
- Integration tests for all API endpoints using supertest
- E2E test outline for Playwright
- Test data factories for users, images, and galleries
- Coverage configuration with 80%+ targets for services/utils
- Test scripts in `package.json`

## Files to Create

| File | Description |
|------|-------------|
| `vitest.config.ts` | Vitest configuration with coverage |
| `src/test/setup.ts` | Global test setup (database, server) |
| `src/test/helpers.ts` | Test utilities (auth helpers, cleanup) |
| `src/test/factories.ts` | Test data factories |
| `src/test/app.ts` | Test Express app instance |
| `src/modules/auth/__tests__/auth.service.test.ts` | Auth service unit tests |
| `src/modules/auth/__tests__/auth.controller.test.ts` | Auth controller integration tests |
| `src/modules/images/__tests__/image.service.test.ts` | Image service unit tests |
| `src/modules/images/__tests__/image.controller.test.ts` | Image controller integration tests |
| `src/modules/gallery/__tests__/gallery.service.test.ts` | Gallery service unit tests |
| `src/modules/gallery/__tests__/gallery.controller.test.ts` | Gallery controller integration tests |
| `src/middleware/__tests__/auth.test.ts` | Auth middleware integration tests |
| `src/utils/__tests__/crypto.test.ts` | Crypto utility unit tests |
| `e2e/auth.spec.ts` | E2E auth test outline |
| `e2e/images.spec.ts` | E2E image generation test outline |

## Steps

### Step 1: Install Testing Dependencies

```bash
npm install -D vitest @vitest/coverage-v8 supertest @types/supertest
```

### Step 2: Configure Vitest

`vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config';
import path from 'path';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      include: ['src/**/*.ts'],
      exclude: [
        'src/**/*.d.ts',
        'src/**/types.ts',
        'src/**/__tests__/**',
        'src/test/**',
        'src/config/**',
      ],
      thresholds: {
        branches: 80,
        functions: 80,
        lines: 80,
        statements: 80,
      },
    },
    testTimeout: 30000,
    hookTimeout: 30000,
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

Add scripts to `package.json`:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "npx playwright test"
  }
}
```

### Step 3: Test Setup and Teardown

`src/test/setup.ts`:

```ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.TEST_DATABASE_URL || process.env.DATABASE_URL,
    },
  },
});

beforeAll(async () => {
  await prisma.$connect();
});

afterAll(async () => {
  await prisma.$disconnect();
});

export { prisma };
```

### Step 4: Test Helpers

`src/test/helpers.ts`:

```ts
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import app from './app';

const prisma = new PrismaClient();

interface AuthTokens {
  accessToken: string;
  refreshToken: string;
  user: { id: string; email: string };
}

export async function createTestUser(
  overrides: Record<string, any> = {}
): Promise<{ id: string; email: string; name: string } & AuthTokens> {
  const password = await bcrypt.hash('TestPass123!', 10);
  const user = await prisma.user.create({
    data: {
      name: overrides.name || 'Test User',
      email: overrides.email || `test-${Date.now()}@example.com`,
      password,
      role: overrides.role || 'user',
      emailVerified: overrides.emailVerified ?? true,
      ...overrides,
    },
  });

  const accessToken = jwt.sign(
    { id: user.id, email: user.email, role: user.role },
    process.env.JWT_SECRET!,
    { expiresIn: '15m' }
  );

  const refreshToken = jwt.sign(
    { id: user.id },
    process.env.JWT_REFRESH_SECRET!,
    { expiresIn: '7d' }
  );

  await prisma.refreshToken.create({
    data: {
      token: refreshToken,
      userId: user.id,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    },
  });

  return {
    ...user,
    accessToken,
    refreshToken,
  };
}

export async function cleanupDatabase(): Promise<void> {
  const tableOrder = [
    'notification',
    'passwordReset',
    'refreshToken',
    'galleryImage',
    'image',
    'gallery',
    'subscription',
    'payment',
    'user',
  ];

  for (const table of tableOrder) {
    await prisma.$executeRawUnsafe(`TRUNCATE TABLE "${table}" CASCADE;`);
  }
}

export async function closeTestConnections(): Promise<void> {
  await prisma.$disconnect();
}

export function authHeader(token: string): Record<string, string> {
  return { Authorization: `Bearer ${token}` };
}

export { prisma };
```

### Step 5: Test App Instance

`src/test/app.ts`:

```ts
import express from 'express';
import { prisma } from './helpers';
import { app } from '../app';

let server: any;

export function getTestApp() {
  return app;
}

export async function startTestServer(): Promise<string> {
  const port = 0; // Random available port
  server = app.listen(port);
  const address = server.address();
  return `http://127.0.0.1:${address.port}`;
}

export async function stopTestServer(): Promise<void> {
  if (server) {
    await new Promise((resolve) => server.close(resolve));
  }
}

export { app };
```

### Step 6: Test Data Factories

`src/test/factories.ts`:

```ts
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcryptjs';

const prisma = new PrismaClient();

let userCounter = 0;
let imageCounter = 0;
let galleryCounter = 0;

export const UserFactory = {
  build(overrides: Record<string, any> = {}) {
    userCounter++;
    return {
      name: `User ${userCounter}`,
      email: `user${userCounter}@test.com`,
      password: bcrypt.hashSync('Password123!', 10),
      role: 'user',
      emailVerified: true,
      ...overrides,
    };
  },

  async create(overrides: Record<string, any> = {}) {
    const data = this.build(overrides);
    return prisma.user.create({ data });
  },
};

export const ImageFactory = {
  build(overrides: Record<string, any> = {}) {
    imageCounter++;
    return {
      prompt: `Test prompt ${imageCounter}`,
      url: `https://storage.example.com/images/test-${imageCounter}.png`,
      thumbnailUrl: `https://storage.example.com/thumbs/test-${imageCounter}.png`,
      provider: 'dalle',
      width: 1024,
      height: 1024,
      isPublic: false,
      isFavorite: false,
      ...overrides,
    };
  },

  async create(overrides: Record<string, any> = {}) {
    const data = this.build(overrides);
    return prisma.image.create({ data });
  },
};

export const GalleryFactory = {
  build(overrides: Record<string, any> = {}) {
    galleryCounter++;
    return {
      name: `Gallery ${galleryCounter}`,
      description: `Test gallery ${galleryCounter}`,
      isPublic: false,
      ...overrides,
    };
  },

  async create(overrides: Record<string, any> = {}) {
    const data = this.build(overrides);
    return prisma.gallery.create({ data });
  },
};

export function resetCounters() {
  userCounter = 0;
  imageCounter = 0;
  galleryCounter = 0;
}
```

### Step 7: Auth Service Unit Tests

`src/modules/auth/__tests__/auth.service.test.ts`:

```ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import bcrypt from 'bcryptjs';
import { AuthService } from '../service';
import { prisma, cleanupDatabase, createTestUser } from '../../../test/helpers';

describe('AuthService', () => {
  beforeEach(async () => {
    await cleanupDatabase();
  });

  describe('register', () => {
    it('should create a new user with hashed password', async () => {
      const result = await AuthService.register({
        name: 'John Doe',
        email: 'john@example.com',
        password: 'Password123!',
      });

      expect(result.user).toBeDefined();
      expect(result.user.email).toBe('john@example.com');
      expect(result.user.name).toBe('John Doe');
      expect(result.accessToken).toBeDefined();
      expect(result.refreshToken).toBeDefined();

      const dbUser = await prisma.user.findUnique({
        where: { email: 'john@example.com' },
      });
      expect(dbUser).toBeDefined();
      expect(dbUser!.password).not.toBe('Password123!');
      expect(await bcrypt.compare('Password123!', dbUser!.password)).toBe(true);
    });

    it('should reject duplicate email', async () => {
      await AuthService.register({
        name: 'John Doe',
        email: 'john@example.com',
        password: 'Password123!',
      });

      await expect(
        AuthService.register({
          name: 'John Doe 2',
          email: 'john@example.com',
          password: 'Password456!',
        })
      ).rejects.toThrow(/already exists/i);
    });

    it('should reject weak passwords', async () => {
      await expect(
        AuthService.register({
          name: 'John Doe',
          email: 'john2@example.com',
          password: 'short',
        })
      ).rejects.toThrow();
    });
  });

  describe('login', () => {
    it('should return tokens for valid credentials', async () => {
      await createTestUser({
        email: 'login@example.com',
        password: await bcrypt.hash('Password123!', 10),
      });

      const result = await AuthService.login({
        email: 'login@example.com',
        password: 'Password123!',
      });

      expect(result.accessToken).toBeDefined();
      expect(result.refreshToken).toBeDefined();
      expect(result.user.email).toBe('login@example.com');
    });

    it('should reject invalid password', async () => {
      await createTestUser({
        email: 'login-fail@example.com',
        password: await bcrypt.hash('Password123!', 10),
      });

      await expect(
        AuthService.login({
          email: 'login-fail@example.com',
          password: 'WrongPassword!',
        })
      ).rejects.toThrow(/invalid credentials/i);
    });

    it('should reject non-existent user', async () => {
      await expect(
        AuthService.login({
          email: 'nonexistent@example.com',
          password: 'Password123!',
        })
      ).rejects.toThrow(/invalid credentials/i);
    });
  });

  describe('refreshToken', () => {
    it('should issue new tokens for valid refresh token', async () => {
      const testUser = await createTestUser();
      const result = await AuthService.refreshToken(testUser.refreshToken);

      expect(result.accessToken).toBeDefined();
      expect(result.refreshToken).toBeDefined();
      expect(result.refreshToken).not.toBe(testUser.refreshToken);
    });

    it('should reject expired refresh token', async () => {
      await expect(
        AuthService.refreshToken('invalid-refresh-token')
      ).rejects.toThrow();
    });
  });

  describe('logout', () => {
    it('should delete refresh token from database', async () => {
      const testUser = await createTestUser();

      await AuthService.logout(testUser.refreshToken);

      const token = await prisma.refreshToken.findUnique({
        where: { token: testUser.refreshToken },
      });
      expect(token).toBeNull();
    });
  });
});
```

### Step 8: Image Service Unit Tests

`src/modules/images/__tests__/image.service.test.ts`:

```ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { ImageService } from '../service';
import { prisma, cleanupDatabase, createTestUser } from '../../../test/helpers';
import { ImageFactory } from '../../../test/factories';

describe('ImageService', () => {
  let userId: string;

  beforeEach(async () => {
    await cleanupDatabase();
    const user = await createTestUser();
    userId = user.id;
  });

  describe('generate', () => {
    it('should call DALL-E provider and save image', async () => {
      const mockProvider = vi.fn().mockResolvedValue({
        url: 'https://storage.example.com/generated.png',
        provider: 'dalle',
      });

      // Assuming providers are injected or mocked
      const result = await ImageService.generate(userId, {
        prompt: 'A sunset over mountains',
        provider: 'dalle',
      });

      expect(result).toBeDefined();
      expect(result.prompt).toBe('A sunset over mountains');
      expect(result.userId).toBe(userId);
    });

    it('should reject empty prompt', async () => {
      await expect(
        ImageService.generate(userId, {
          prompt: '',
          provider: 'dalle',
        })
      ).rejects.toThrow();
    });

    it('should reject invalid provider', async () => {
      await expect(
        ImageService.generate(userId, {
          prompt: 'Test prompt',
          provider: 'invalid-provider',
        })
      ).rejects.toThrow();
    });
  });

  describe('list', () => {
    it('should return paginated images for user', async () => {
      await ImageFactory.create({ userId, prompt: 'Image 1' });
      await ImageFactory.create({ userId, prompt: 'Image 2' });
      await ImageFactory.create({ userId, prompt: 'Image 3' });

      const result = await ImageService.list(userId, { page: 1, limit: 2 });

      expect(result.images).toHaveLength(2);
      expect(result.total).toBe(3);
    });

    it('should not return other users images', async () => {
      const otherUser = await createTestUser({ email: 'other@test.com' });
      await ImageFactory.create({ userId: otherUser.id, prompt: 'Other image' });
      await ImageFactory.create({ userId, prompt: 'My image' });

      const result = await ImageService.list(userId, { page: 1, limit: 10 });

      expect(result.images).toHaveLength(1);
      expect(result.images[0].userId).toBe(userId);
    });
  });

  describe('delete', () => {
    it('should delete own image', async () => {
      const image = await ImageFactory.create({ userId });

      await ImageService.delete(userId, image.id);

      const deleted = await prisma.image.findUnique({
        where: { id: image.id },
      });
      expect(deleted).toBeNull();
    });

    it('should reject deleting another users image', async () => {
      const otherUser = await createTestUser({ email: 'other2@test.com' });
      const image = await ImageFactory.create({ userId: otherUser.id });

      await expect(
        ImageService.delete(userId, image.id)
      ).rejects.toThrow(/not authorized/i);
    });
  });

  describe('toggleFavorite', () => {
    it('should toggle favorite status', async () => {
      const image = await ImageFactory.create({ userId, isFavorite: false });

      const result = await ImageService.toggleFavorite(userId, image.id);
      expect(result.isFavorite).toBe(true);

      const result2 = await ImageService.toggleFavorite(userId, image.id);
      expect(result2.isFavorite).toBe(false);
    });
  });
});
```

### Step 9: API Integration Tests

`src/modules/auth/__tests__/auth.controller.test.ts`:

```ts
import { describe, it, expect, beforeEach } from 'vitest';
import request from 'supertest';
import { app } from '../../../test/app';
import { cleanupDatabase, createTestUser, authHeader } from '../../../test/helpers';

describe('Auth API', () => {
  beforeEach(async () => {
    await cleanupDatabase();
  });

  describe('POST /api/auth/register', () => {
    it('should register a new user and return tokens', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          name: 'John Doe',
          email: 'john@example.com',
          password: 'Password123!',
        });

      expect(response.status).toBe(201);
      expect(response.body).toHaveProperty('accessToken');
      expect(response.body).toHaveProperty('refreshToken');
      expect(response.body.user.email).toBe('john@example.com');
    });

    it('should reject missing email', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({ name: 'John', password: 'Password123!' });

      expect(response.status).toBe(400);
    });

    it('should reject duplicate email', async () => {
      await request(app).post('/api/auth/register').send({
        name: 'John Doe',
        email: 'john@example.com',
        password: 'Password123!',
      });

      const response = await request(app)
        .post('/api/auth/register')
        .send({
          name: 'John Doe 2',
          email: 'john@example.com',
          password: 'Password456!',
        });

      expect(response.status).toBe(409);
    });
  });

  describe('POST /api/auth/login', () => {
    it('should login with valid credentials', async () => {
      const testUser = await createTestUser();

      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: testUser.email,
          password: 'TestPass123!',
        });

      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('accessToken');
      expect(response.body).toHaveProperty('refreshToken');
    });

    it('should reject invalid credentials with 401', async () => {
      const testUser = await createTestUser();

      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: testUser.email,
          password: 'WrongPassword!',
        });

      expect(response.status).toBe(401);
    });
  });

  describe('GET /api/auth/me', () => {
    it('should return current user with valid token', async () => {
      const testUser = await createTestUser();

      const response = await request(app)
        .get('/api/auth/me')
        .set(authHeader(testUser.accessToken));

      expect(response.status).toBe(200);
      expect(response.body.user.email).toBe(testUser.email);
    });

    it('should reject request without token', async () => {
      const response = await request(app).get('/api/auth/me');

      expect(response.status).toBe(401);
    });
  });

  describe('POST /api/auth/logout', () => {
    it('should logout and invalidate refresh token', async () => {
      const testUser = await createTestUser();

      const response = await request(app)
        .post('/api/auth/logout')
        .set(authHeader(testUser.accessToken))
        .send({ refreshToken: testUser.refreshToken });

      expect(response.status).toBe(200);
    });
  });
});
```

### Step 10: Image Controller Integration Tests

`src/modules/images/__tests__/image.controller.test.ts`:

```ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import request from 'supertest';
import { app } from '../../../test/app';
import { cleanupDatabase, createTestUser, authHeader } from '../../../test/helpers';

vi.mock('../providers/dalle', () => ({
  DalleProvider: vi.fn().mockImplementation(() => ({
    generate: vi.fn().mockResolvedValue({
      url: 'https://storage.example.com/mock-image.png',
      provider: 'dalle',
    }),
  })),
}));

describe('Images API', () => {
  let testUser: any;

  beforeEach(async () => {
    await cleanupDatabase();
    testUser = await createTestUser();
  });

  describe('POST /api/images/generate', () => {
    it('should generate an image when authenticated', async () => {
      const response = await request(app)
        .post('/api/images/generate')
        .set(authHeader(testUser.accessToken))
        .send({ prompt: 'A beautiful sunset', provider: 'dalle' });

      expect(response.status).toBe(201);
      expect(response.body).toHaveProperty('id');
      expect(response.body.prompt).toBe('A beautiful sunset');
    });

    it('should reject unauthenticated requests', async () => {
      const response = await request(app)
        .post('/api/images/generate')
        .send({ prompt: 'A beautiful sunset' });

      expect(response.status).toBe(401);
    });

    it('should reject missing prompt', async () => {
      const response = await request(app)
        .post('/api/images/generate')
        .set(authHeader(testUser.accessToken))
        .send({});

      expect(response.status).toBe(400);
    });
  });

  describe('GET /api/images', () => {
    it('should return users images', async () => {
      const response = await request(app)
        .get('/api/images')
        .set(authHeader(testUser.accessToken));

      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('images');
      expect(response.body).toHaveProperty('total');
    });
  });

  describe('DELETE /api/images/:id', () => {
    it('should delete own image', async () => {
      const genResponse = await request(app)
        .post('/api/images/generate')
        .set(authHeader(testUser.accessToken))
        .send({ prompt: 'Delete me', provider: 'dalle' });

      const imageId = genResponse.body.id;

      const delResponse = await request(app)
        .delete(`/api/images/${imageId}`)
        .set(authHeader(testUser.accessToken));

      expect(delResponse.status).toBe(200);
    });
  });
});
```

### Step 11: Auth Middleware Tests

`src/middleware/__tests__/auth.test.ts`:

```ts
import { describe, it, expect, beforeEach } from 'vitest';
import request from 'supertest';
import { app } from '../../test/app';
import { cleanupDatabase, createTestUser, authHeader } from '../../test/helpers';

describe('Auth Middleware', () => {
  beforeEach(async () => {
    await cleanupDatabase();
  });

  it('should allow access with valid token', async () => {
    const testUser = await createTestUser();

    const response = await request(app)
      .get('/api/auth/me')
      .set(authHeader(testUser.accessToken));

    expect(response.status).toBe(200);
  });

  it('should reject access without token', async () => {
    const response = await request(app).get('/api/auth/me');

    expect(response.status).toBe(401);
  });

  it('should reject access with invalid token', async () => {
    const response = await request(app)
      .get('/api/auth/me')
      .set('Authorization', 'Bearer invalid-token');

    expect(response.status).toBe(401);
  });

  it('should reject access with expired token', async () => {
    const response = await request(app)
      .get('/api/auth/me')
      .set('Authorization', 'Bearer expired.jwt.token');

    expect(response.status).toBe(401);
  });

  it('should reject access with malformed authorization header', async () => {
    const response = await request(app)
      .get('/api/auth/me')
      .set('Authorization', 'NotBearer sometoken');

    expect(response.status).toBe(401);
  });
});
```

### Step 12: E2E Test Outlines (Playwright)

Install Playwright:

```bash
npm install -D @playwright/test
npx playwright install
```

`e2e/auth.spec.ts`:

```ts
import { test, expect } from '@playwright/test';

test.describe('Authentication Flow', () => {
  test('should register a new user', async ({ page }) => {
    await page.goto('http://localhost:5173/signup');

    await page.fill('[data-testid="name-input"]', 'E2E Test User');
    await page.fill('[data-testid="email-input"]', `e2e-${Date.now()}@example.com`);
    await page.fill('[data-testid="password-input"]', 'TestPass123!');
    await page.click('[data-testid="signup-button"]');

    await expect(page).toHaveURL(/.*dashboard/);
    await expect(page.locator('[data-testid="user-name"]')).toContainText('E2E Test User');
  });

  test('should login with existing credentials', async ({ page }) => {
    await page.goto('http://localhost:5173/login');

    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'TestPass123!');
    await page.click('[data-testid="login-button"]');

    await expect(page).toHaveURL(/.*dashboard/);
  });

  test('should logout and redirect to login', async ({ page }) => {
    // Login first
    await page.goto('http://localhost:5173/login');
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'TestPass123!');
    await page.click('[data-testid="login-button"]');

    // Click logout
    await page.click('[data-testid="settings-link"]');
    await page.click('[data-testid="logout-button"]');

    await expect(page).toHaveURL(/.*login/);
  });

  test('should show error on invalid credentials', async ({ page }) => {
    await page.goto('http://localhost:5173/login');

    await page.fill('[data-testid="email-input"]', 'wrong@example.com');
    await page.fill('[data-testid="password-input"]', 'wrongpassword');
    await page.click('[data-testid="login-button"]');

    await expect(page.locator('[data-testid="error-message"]')).toBeVisible();
  });
});
```

`e2e/images.spec.ts`:

```ts
import { test, expect } from '@playwright/test';

test.describe('Image Generation Flow', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:5173/login');
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'TestPass123!');
    await page.click('[data-testid="login-button"]');
    await expect(page).toHaveURL(/.*dashboard/);
  });

  test('should generate an image with valid prompt', async ({ page }) => {
    await page.fill('[data-testid="prompt-input"]', 'A beautiful mountain landscape');
    await page.click('[data-testid="generate-button"]');

    await expect(page.locator('[data-testid="generated-image"]')).toBeVisible({
      timeout: 60000,
    });
  });

  test('should switch between providers', async ({ page }) => {
    await page.click('[data-testid="provider-flux"]');
    await expect(page.locator('[data-testid="provider-flux"]')).toHaveClass(/active/);

    await page.click('[data-testid="provider-dalle"]');
    await expect(page.locator('[data-testid="provider-dalle"]')).toHaveClass(/active/);
  });

  test('should show error on empty prompt', async ({ page }) => {
    await page.click('[data-testid="generate-button"]');

    // HTML5 validation should prevent submission
    const isValid = await page.evaluate(() => {
      const input = document.querySelector('[data-testid="prompt-input"]') as HTMLInputElement;
      return input.validity.valueMissing;
    });
    expect(isValid).toBe(true);
  });
});
```

### Step 13: Run Tests and Coverage

```bash
# Run all unit and integration tests
npm test

# Run with coverage
npm run test:coverage

# Run specific test file
npx vitest run src/modules/auth/__tests__/auth.service.test.ts

# Run in watch mode
npm run test:watch

# Run E2E tests (requires both backend and frontend running)
npm run test:e2e
```

## Verification

```bash
# 1. Run the full test suite
npm test

# 2. Check coverage
npm run test:coverage

# Expected coverage thresholds:
# - src/modules/auth/service.ts: 80%+ lines
# - src/modules/images/service.ts: 80%+ lines
# - src/modules/gallery/service.ts: 80%+ lines
# - src/utils/crypto.ts: 80%+ lines
# - Overall: 80%+ lines

# 3. Verify test isolation
# Each test file should be able to run independently:
npx vitest run src/modules/auth/__tests__/auth.service.test.ts
npx vitest run src/modules/images/__tests__/image.controller.test.ts

# 4. Verify no test pollution
# Run the same test suite twice:
npm test && npm test

# 5. Check E2E test outline (won't run without UI)
npx playwright test --list
```

## Rollback

```bash
# Remove test files
rm -rf src/test/
rm -rf src/modules/auth/__tests__/
rm -rf src/modules/images/__tests__/
rm -rf src/modules/gallery/__tests__/
rm -rf src/middleware/__tests__/
rm -rf src/utils/__tests__/
rm -rf e2e/

# Remove test dependencies
npm uninstall vitest @vitest/coverage-v8 supertest @types/supertest @playwright/test

# Remove test scripts from package.json
# Remove "test", "test:watch", "test:coverage", "test:e2e" scripts

# Remove vitest.config.ts
rm vitest.config.ts
```

## ADR-027: Vitest over Jest for Testing

**Decision**: Use Vitest as the test runner with v8 coverage provider, supertest for API integration tests, and Playwright for E2E tests.

**Reason**: Vitest provides native ESM and TypeScript support without configuration overhead, has faster watch mode than Jest, and uses the same configuration as Vite (already in use for the frontend). The v8 coverage provider is faster than Istanbul for native ESM. Supertest works seamlessly with Express for HTTP-level integration tests without starting a real server. Playwright is the modern standard for cross-browser E2E testing.

**Consequences**:
- Vitest is still evolving; minor API changes between versions are possible (pin versions)
- v8 coverage doesn't support all edge cases of Istanbul but is faster
- supertest requires Express app export separation (`app` vs `server`)
- Playwright E2E tests need both frontend and backend running (use `webServer` config)
- Test database must be separate from development database to avoid data pollution

**Alternatives Considered**:
- Jest: More mature ecosystem but slower with TypeScript/ESM, requires ts-jest or babel transform
- Mocha + Chai: More configuration overhead, less modern DX
- Cypress: Good for E2E but heavier than Playwright, less CI-friendly
- Testing Library: Useful for React frontend tests but not needed for backend-only testing
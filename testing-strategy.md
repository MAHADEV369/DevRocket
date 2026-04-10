# Testing Strategy Guide

Complete testing strategy for the FastDepo application covering unit, integration, E2E, and load testing.

---

## 1. Test Pyramid

```
        ┌─────────┐
        │  E2E   │  ← Few, slow, expensive (Playwright)
        │ Tests  │
        ├─────────┤
        │Integration│ ← Moderate (supertest + DB)
        │  Tests  │
        ├─────────┤
        │  Unit   │  ← Many, fast, cheap (Vitest)
        │ Tests  │
        └─────────┘
```

- **Unit tests**: 60-70% of total tests, run in milliseconds, test isolated functions/services
- **Integration tests**: 20-25%, test API endpoints with real database, run in seconds
- **E2E tests**: 5-10%, test full user flows through the browser, run in minutes

---

## 2. Jest/Vitest Setup with TypeScript

Using Vitest for the backend (fast, native ESM, Vite-powered):

```bash
cd backend
npm install -D vitest @vitest/coverage-v8
npm install -D @types/supertest supertest
```

### `vitest.config.ts`

```typescript
import { defineConfig } from 'vitest/config';
import path from 'path';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      include: ['src/**/*.ts'],
      exclude: ['src/**/*.d.ts', 'src/types/**'],
      thresholds: {
        branches: 60,
        functions: 60,
        lines: 60,
        statements: 60,
      },
    },
    pool: 'forks',
    poolOptions: {
      forks: { maxForks: 4 },
    },
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

### `tests/setup.ts`

```typescript
import { beforeAll, afterAll, afterEach } from 'vitest';
import { PrismaClient } from '@prisma/client';
import { execSync } from 'child_process';

const prisma = new PrismaClient();

beforeAll(async () => {
  // Ensure we're pointing at the test database
  process.env.DATABASE_URL = process.env.TEST_DATABASE_URL || 'postgresql://localhost:5432/fastdepo_test';

  // Run migrations on test database
  execSync('npx prisma migrate deploy', {
    env: { ...process.env, DATABASE_URL: process.env.DATABASE_URL },
  });
});

afterEach(async () => {
  // Clean up data between tests using transactions
  const tablenames = await prisma.$queryRaw<
    Array<{ tablename: string }>
  >`SELECT tablename FROM pg_tables WHERE schemaname='public'`;

  for (const { tablename } of tablenames) {
    if (tablename !== '_prisma_migrations') {
      await prisma.$executeRawUnsafe(`TRUNCATE TABLE "${tablename}" CASCADE;`);
    }
  }
});

afterAll(async () => {
  await prisma.$disconnect();
});

export { prisma };
```

Add to `package.json`:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:integration": "vitest run --config vitest.integration.config.ts",
    "test:e2e": "playwright test"
  }
}
```

---

## 3. Test Database Setup

### Separate Test Database

In `.env.test`:

```
TEST_DATABASE_URL=postgresql://postgres:postgres@localhost:5432/fastdepo_test
JWT_SECRET=test-jwt-secret-do-not-use-in-production
JWT_REFRESH_SECRET=test-refresh-secret-do-not-use-in-production
```

### Docker Compose for Test Database

`docker-compose.test.yml`:

```yaml
version: '3.8'
services:
  test-db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: fastdepo_test
    ports:
      - "5433:5432"
    tmpfs:
      - /var/lib/postgresql/data
```

```bash
# Start test database
docker compose -f docker-compose.test.yml up -d

# Run tests against it
TEST_DATABASE_URL=postgresql://postgres:postgres@localhost:5433/fastdepo_test npm test
```

### Transaction Rollback Strategy

For tests that need full isolation without truncating between every test:

```typescript
// tests/helpers/database.ts
import { PrismaClient } from '@prisma/client';

export class TestDatabase {
  private prisma: PrismaClient;
  private transactionClient: any;

  constructor() {
    this.prisma = new PrismaClient();
  }

  async startTransaction() {
    this.transactionClient = await this.prisma.$transaction.begin();
    return this.transactionClient;
  }

  async rollback() {
    await this.transactionClient.$transaction.rollback();
  }

  async disconnect() {
    await this.prisma.$disconnect();
  }

  get client() {
    return this.transactionClient || this.prisma;
  }
}
```

---

## 4. Unit Tests for Services

### Auth Service Tests

`tests/unit/services/auth.service.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { AuthService } from '@/services/auth.service';
import { hash, compare } from 'bcryptjs';
import jwt from 'jsonwebtoken';

vi.mock('@/repositories/user.repository');
vi.mock('bcryptjs');
vi.mock('jsonwebtoken');

describe('AuthService', () => {
  let authService: AuthService;
  let mockUserRepo: any;

  beforeEach(() => {
    mockUserRepo = {
      findByEmail: vi.fn(),
      create: vi.fn(),
      findById: vi.fn(),
      updateRefreshToken: vi.fn(),
    };
    authService = new AuthService(mockUserRepo);
  });

  describe('signup', () => {
    it('signup_givenValidData_createsUserAndReturnsTokens', async () => {
      const signupData = { email: 'test@example.com', password: 'Password123!', name: 'Test User' };

      mockUserRepo.findByEmail.mockResolvedValue(null);
      vi.mocked(hash).mockResolvedValue('hashedpassword' as never);
      vi.mocked(jwt.sign).mockReturnValueOnce('access-token' as never);
      vi.mocked(jwt.sign).mockReturnValueOnce('refresh-token' as never);
      mockUserRepo.create.mockResolvedValue({
        id: 'user-id',
        email: signupData.email,
        name: signupData.name,
        createdAt: new Date(),
        updatedAt: new Date(),
      });

      const result = await authService.signup(signupData);

      expect(result).toEqual({
        user: expect.objectContaining({ email: signupData.email }),
        accessToken: 'access-token',
        refreshToken: 'refresh-token',
      });
      expect(mockUserRepo.create).toHaveBeenCalledWith(
        expect.objectContaining({ email: signupData.email })
      );
    });

    it('signup_givenExistingEmail_throwsConflictError', async () => {
      const signupData = { email: 'existing@example.com', password: 'Password123!', name: 'Test' };

      mockUserRepo.findByEmail.mockResolvedValue({ id: 'existing-user-id', email: signupData.email });

      await expect(authService.signup(signupData)).rejects.toThrow('Email already registered');
    });

    it('signup_givenWeakPassword_throwsValidationError', async () => {
      const signupData = { email: 'test@example.com', password: '123', name: 'Test' };

      await expect(authService.signup(signupData)).rejects.toThrow();
    });
  });

  describe('login', () => {
    it('login_givenValidCredentials_returnsTokens', async () => {
      const loginData = { email: 'test@example.com', password: 'Password123!' };

      mockUserRepo.findByEmail.mockResolvedValue({
        id: 'user-id',
        email: loginData.email,
        name: 'Test User',
        passwordHash: 'hashedpassword',
      });
      vi.mocked(compare).mockResolvedValue(true as never);
      vi.mocked(jwt.sign).mockReturnValueOnce('access-token' as never);
      vi.mocked(jwt.sign).mockReturnValueOnce('refresh-token' as never);

      const result = await authService.login(loginData);

      expect(result).toEqual({
        user: expect.objectContaining({ email: loginData.email }),
        accessToken: 'access-token',
        refreshToken: 'refresh-token',
      });
    });

    it('login_givenWrongPassword_throwsUnauthorizedError', async () => {
      const loginData = { email: 'test@example.com', password: 'WrongPassword!' };

      mockUserRepo.findByEmail.mockResolvedValue({
        id: 'user-id',
        email: loginData.email,
        passwordHash: 'hashedpassword',
      });
      vi.mocked(compare).mockResolvedValue(false as never);

      await expect(authService.login(loginData)).rejects.toThrow('Invalid credentials');
    });

    it('login_givenNonexistentEmail_throwsUnauthorizedError', async () => {
      const loginData = { email: 'noone@example.com', password: 'Password123!' };

      mockUserRepo.findByEmail.mockResolvedValue(null);

      await expect(authService.login(loginData)).rejects.toThrow('Invalid credentials');
    });
  });

  describe('refreshToken', () => {
    it('refreshToken_givenValidRefreshToken_returnsNewTokens', async () => {
      const oldRefreshToken = 'valid-refresh-token';
      const decoded = { userId: 'user-id', type: 'refresh' };

      vi.mocked(jwt.verify).mockReturnValue(decoded as never);
      mockUserRepo.findById.mockResolvedValue({
        id: 'user-id',
        email: 'test@example.com',
        refreshToken: oldRefreshToken,
      });
      vi.mocked(jwt.sign).mockReturnValueOnce('new-access-token' as never);
      vi.mocked(jwt.sign).mockReturnValueOnce('new-refresh-token' as never);

      const result = await authService.refreshToken(oldRefreshToken);

      expect(result.accessToken).toBe('new-access-token');
      expect(result.refreshToken).toBe('new-refresh-token');
      expect(mockUserRepo.updateRefreshToken).toHaveBeenCalledWith('user-id', 'new-refresh-token');
    });

    it('refreshToken_givenExpiredToken_throwsUnauthorizedError', async () => {
      vi.mocked(jwt.verify).mockImplementation(() => {
        throw new jwt.TokenExpiredError('jwt expired', new Date());
      });

      await expect(authService.refreshToken('expired-token')).rejects.toThrow();
    });

    it('refreshToken_givenTokenForDeletedUser_throwsUnauthorizedError', async () => {
      const decoded = { userId: 'deleted-user-id', type: 'refresh' };
      vi.mocked(jwt.verify).mockReturnValue(decoded as never);
      mockUserRepo.findById.mockResolvedValue(null);

      await expect(authService.refreshToken('valid-but-user-gone')).rejects.toThrow();
    });
  });
});
```

### Image Generation Service Tests

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { ImageGenerationService } from '@/services/image.service';

vi.mock('@/providers/image.provider');

describe('ImageGenerationService', () => {
  let service: ImageGenerationService;
  let mockProvider: any;
  let mockRepo: any;

  beforeEach(() => {
    mockProvider = {
      generate: vi.fn(),
    };
    mockRepo = {
      create: vi.fn(),
      findById: vi.fn(),
      update: vi.fn(),
    };
    service = new ImageGenerationService(mockProvider, mockRepo);
  });

  describe('generate', () => {
    it('generate_givenValidPrompt_callsProviderAndSavesResult', async () => {
      const prompt = 'A cat riding a dragon';
      const model = 'stable-diffusion-xl';

      mockProvider.generate.mockResolvedValue({
        imageUrl: 'https://cdn.fastdepo.com/images/test.png',
        status: 'completed',
      });

      mockRepo.create.mockResolvedValue({
        id: 'gen-id',
        prompt,
        model,
        imageUrl: 'https://cdn.fastdepo.com/images/test.png',
        status: 'completed',
      });

      const result = await service.generate({ prompt, model });

      expect(mockProvider.generate).toHaveBeenCalledWith({
        prompt,
        model,
      });
      expect(result.status).toBe('completed');
      expect(mockRepo.create).toHaveBeenCalled();
    });

    it('generate_givenProviderFailure_savesErrorStatus', async () => {
      mockProvider.generate.mockRejectedValue(new Error('Provider timeout'));

      mockRepo.create.mockResolvedValue({
        id: 'gen-id',
        status: 'failed',
        error: 'Provider timeout',
      });

      const result = await service.generate({ prompt: 'test', model: 'dall-e-3' });

      expect(result.status).toBe('failed');
    });

    it('generate_givenRateLimitedUser_throwsRateLimitError', async () => {
      mockRepo.countByUser.mockResolvedValue(10); // User has 10 generations today

      const RATE_LIMIT = 10;

      await expect(
        service.generate({ prompt: 'test', model: 'stable-diffusion-xl', userId: 'user-with-limit' })
      ).rejects.toThrow('Rate limit exceeded');
    });
  });
});
```

### Gallery Service Tests

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { GalleryService } from '@/services/gallery.service';

describe('GalleryService', () => {
  let service: GalleryService;
  let mockRepo: any;

  beforeEach(() => {
    mockRepo = {
      findByUserId: vi.fn(),
      findById: vi.fn(),
      delete: vi.fn(),
    };
    service = new GalleryService(mockRepo);
  });

  describe('listByUser', () => {
    it('listByUser_givenUserId_returnsPaginatedImages', async () => {
      const mockImages = Array.from({ length: 5 }, (_, i) => ({
        id: `img-${i}`,
        imageUrl: `https://cdn.fastdepo.com/images/${i}.png`,
        prompt: `Prompt ${i}`,
        createdAt: new Date(),
      }));

      mockRepo.findByUserId.mockResolvedValue({
        images: mockImages,
        total: 25,
        page: 1,
        limit: 5,
      });

      const result = await service.listByUser('user-id', { page: 1, limit: 5 });

      expect(result.images).toHaveLength(5);
      expect(result.total).toBe(25);
      expect(mockRepo.findByUserId).toHaveBeenCalledWith('user-id', { page: 1, limit: 5 });
    });

    it('listByUser_givenInvalidPage_defaultsToFirstPage', async () => {
      mockRepo.findByUserId.mockResolvedValue({ images: [], total: 0, page: 1, limit: 20 });

      await service.listByUser('user-id', { page: -1, limit: 20 });

      expect(mockRepo.findByUserId).toHaveBeenCalledWith('user-id', { page: 1, limit: 20 });
    });
  });

  describe('deleteImage', () => {
    it('deleteImage_givenOwnImage_deletesSuccessfully', async () => {
      mockRepo.findById.mockResolvedValue({
        id: 'img-1',
        userId: 'user-id',
      });
      mockRepo.delete.mockResolvedValue(true);

      const result = await service.deleteImage('img-1', 'user-id');

      expect(result).toBe(true);
      expect(mockRepo.delete).toHaveBeenCalledWith('img-1');
    });

    it('deleteImage_givenOtherUsersImage_throwsForbiddenError', async () => {
      mockRepo.findById.mockResolvedValue({
        id: 'img-1',
        userId: 'different-user-id',
      });

      await expect(service.deleteImage('img-1', 'user-id')).rejects.toThrow('Forbidden');
    });
  });
});
```

---

## 5. Integration Tests for API Endpoints

### Setup

```typescript
// tests/integration/helpers/app.ts
import express from 'express';
import { PrismaClient } from '@prisma/client';
import { app } from '@/app';
import supertest from 'supertest';

const prisma = new PrismaClient({
  datasourceUrl: process.env.TEST_DATABASE_URL,
});

export function createTestApp() {
  return supertest(app);
}

export { prisma };
```

### Auth Flow Integration Tests

`tests/integration/auth.test.ts`:

```typescript
import { describe, it, expect, beforeAll, afterAll, afterEach } from 'vitest';
import request from 'supertest';
import { app } from '@/app';
import { prisma } from '../helpers/app';

describe('Auth API', () => {
  afterAll(async () => {
    await prisma.$disconnect();
  });

  afterEach(async () => {
    await prisma.user.deleteMany();
  });

  describe('POST /api/auth/signup', () => {
    it('signup_givenValidData_returns201WithTokens', async () => {
      const response = await request(app)
        .post('/api/auth/signup')
        .send({
          email: 'newuser@example.com',
          password: 'SecurePassword123!',
          name: 'New User',
        });

      expect(response.status).toBe(201);
      expect(response.body).toHaveProperty('accessToken');
      expect(response.body).toHaveProperty('refreshToken');
      expect(response.body.user.email).toBe('newuser@example.com');
    });

    it('signup_givenDuplicateEmail_returns409', async () => {
      await request(app).post('/api/auth/signup').send({
        email: 'duplicate@example.com',
        password: 'SecurePassword123!',
        name: 'First User',
      });

      const response = await request(app)
        .post('/api/auth/signup')
        .send({
          email: 'duplicate@example.com',
          password: 'DifferentPassword456!',
          name: 'Second User',
        });

      expect(response.status).toBe(409);
      expect(response.body.message).toContain('already registered');
    });

    it('signup_givenMissingFields_returns400', async () => {
      const response = await request(app)
        .post('/api/auth/signup')
        .send({ email: 'incomplete@example.com' });

      expect(response.status).toBe(400);
      expect(response.body.errors).toBeDefined();
    });

    it('signup_givenShortPassword_returns400', async () => {
      const response = await request(app)
        .post('/api/auth/signup')
        .send({
          email: 'weakpass@example.com',
          password: '123',
          name: 'Test',
        });

      expect(response.status).toBe(400);
    });
  });

  describe('POST /api/auth/login', () => {
    it('login_givenValidCredentials_returns200WithTokens', async () => {
      // First, create a user
      await request(app).post('/api/auth/signup').send({
        email: 'logintest@example.com',
        password: 'SecurePassword123!',
        name: 'Login User',
      });

      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'logintest@example.com',
          password: 'SecurePassword123!',
        });

      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('accessToken');
      expect(response.body).toHaveProperty('refreshToken');
    });

    it('login_givenWrongPassword_returns401', async () => {
      await request(app).post('/api/auth/signup').send({
        email: 'wrongpass@example.com',
        password: 'SecurePassword123!',
        name: 'Test',
      });

      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'wrongpass@example.com',
          password: 'WrongPassword!',
        });

      expect(response.status).toBe(401);
    });

    it('login_givenNonexistentUser_returns401', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'noone@example.com',
          password: 'Whatever123!',
        });

      expect(response.status).toBe(401);
    });
  });

  describe('POST /api/auth/refresh', () => {
    it('refresh_givenValidRefreshToken_returnsNewTokens', async () => {
      const signupResponse = await request(app).post('/api/auth/signup').send({
        email: 'refresh@example.com',
        password: 'SecurePassword123!',
        name: 'Refresh User',
      });

      const response = await request(app)
        .post('/api/auth/refresh')
        .send({ refreshToken: signupResponse.body.refreshToken });

      expect(response.status).toBe(200);
      expect(response.body.accessToken).not.toBe(signupResponse.body.accessToken);
    });

    it('refresh_givenInvalidRefreshToken_returns401', async () => {
      const response = await request(app)
        .post('/api/auth/refresh')
        .send({ refreshToken: 'invalid-token' });

      expect(response.status).toBe(401);
    });
  });
});
```

### Gallery CRUD Integration Tests

```typescript
import { describe, it, expect, afterEach } from 'vitest';
import request from 'supertest';
import { app } from '@/app';
import { prisma } from '../helpers/app';

async function createAuthenticatedUser() {
  const response = await request(app).post('/api/auth/signup').send({
    email: `test-${Date.now()}@example.com`,
    password: 'SecurePassword123!',
    name: 'Test User',
  });
  return response.body.accessToken;
}

describe('Gallery API', () => {
  afterEach(async () => {
    await prisma.generatedImage.deleteMany();
    await prisma.user.deleteMany();
  });

  describe('GET /api/gallery', () => {
    it('list_givenAuthenticatedUser_returnsEmptyGallery', async () => {
      const token = await createAuthenticatedUser();

      const response = await request(app)
        .get('/api/gallery')
        .set('Authorization', `Bearer ${token}`);

      expect(response.status).toBe(200);
      expect(response.body.images).toEqual([]);
    });

    it('list_givenNoAuth_returns401', async () => {
      const response = await request(app).get('/api/gallery');
      expect(response.status).toBe(401);
    });

    it('list_givenPagination_returnsCorrectPage', async () => {
      const token = await createAuthenticatedUser();

      // Create 25 images (would need a helper or factory)
      // ...

      const response = await request(app)
        .get('/api/gallery?page=2&limit=10')
        .set('Authorization', `Bearer ${token}`);

      expect(response.status).toBe(200);
      expect(response.body.page).toBe(2);
      expect(response.body.limit).toBe(10);
    });
  });

  describe('DELETE /api/gallery/:id', () => {
    it('delete_givenOwnImage_returns204', async () => {
      const token = await createAuthenticatedUser();

      // Create an image first
      const createResponse = await request(app)
        .post('/api/images/generate')
        .set('Authorization', `Bearer ${token}`)
        .send({ prompt: 'test', model: 'stable-diffusion-xl' });

      const imageId = createResponse.body.id;

      const response = await request(app)
        .delete(`/api/gallery/${imageId}`)
        .set('Authorization', `Bearer ${token}`);

      expect(response.status).toBe(204);
    });

    it('delete_givenOtherUsersImage_returns403', async () => {
      const token1 = await createAuthenticatedUser();
      const token2 = await createAuthenticatedUser();

      // User 1 creates an image
      const createResponse = await request(app)
        .post('/api/images/generate')
        .set('Authorization', `Bearer ${token1}`)
        .send({ prompt: 'test', model: 'stable-diffusion-xl' });

      const imageId = createResponse.body.id;

      // User 2 tries to delete it
      const response = await request(app)
        .delete(`/api/gallery/${imageId}`)
        .set('Authorization', `Bearer ${token2}`);

      expect(response.status).toBe(403);
    });
  });
});
```

---

## 6. Test Data Factories using Fishery

```bash
npm install -D fishery
```

### `tests/factories/user.factory.ts`

```typescript
import { Factory } from 'fishery';
import type { User } from '@prisma/client';

export const userFactory = Factory.define<User>(({ sequence }) => ({
  id: `user-${sequence}`,
  email: `user-${sequence}@example.com`,
  name: `Test User ${sequence}`,
  passwordHash: '$2a$10$abcdefghijklmnopqrstuvwxABCDEFGHIJ', // hash of "Password123!"
  createdAt: new Date('2026-01-01'),
  updatedAt: new Date('2026-01-01'),
}));
```

### `tests/factories/image.factory.ts`

```typescript
import { Factory } from 'fishery';
import type { GeneratedImage } from '@prisma/client';
import { userFactory } from './user.factory';

export const imageFactory = Factory.define<GeneratedImage>(({ sequence, associations }) => ({
  id: `image-${sequence}`,
  prompt: `Test prompt ${sequence}`,
  model: 'stable-diffusion-xl',
  imageUrl: `https://cdn.fastdepo.com/images/test-${sequence}.png`,
  status: 'completed',
  userId: associations.userId || userFactory.build().id,
  width: 1024,
  height: 1024,
  createdAt: new Date('2026-01-01'),
  updatedAt: new Date('2026-01-01'),
}));
```

### `tests/factories/index.ts`

```typescript
export { userFactory } from './user.factory';
export { imageFactory } from './image.factory';
```

### Using Factories in Tests

```typescript
import { userFactory, imageFactory } from '../factories';

it('createUser_withFactory Works', async () => {
  const user = userFactory.build({ email: 'specific@example.com' });
  const image = imageFactory.build({ userId: user.id });

  expect(user.email).toBe('specific@example.com');
  expect(image.userId).toBe(user.id);
});
```

---

## 7. E2E Tests with Playwright

### Installation

```bash
npm install -D @playwright/test
npx playwright install
```

### `playwright.config.ts`

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30000,
  expect: { timeout: 10000 },
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [['html', { open: 'never' }]],
  use: {
    baseURL: process.env.E2E_BASE_URL || 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { browserName: 'chromium' } },
    { name: 'mobile', use: { browserName: 'chromium', viewport: { width: 390, height: 844 } } },
  ],
  webServer: {
    command: 'npm run dev',
    port: 5173,
    reuseExistingServer: !process.env.CI,
  },
});
```

### Signup Flow

`e2e/signup.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';

test.describe('Signup Flow', () => {
  test('signup_givenValidData_redirectsToHome', async ({ page }) => {
    await page.goto('/signup');

    await page.fill('[data-testid="name-input"]', 'E2E Test User');
    await page.fill('[data-testid="email-input"]', `e2e-${Date.now()}@example.com`);
    await page.fill('[data-testid="password-input"]', 'SecurePassword123!');
    await page.click('[data-testid="submit-button"]');

    await expect(page).toHaveURL('/');
    await expect(page.locator('[data-testid="user-menu"]')).toBeVisible();
  });

  test('signup_givenExistingEmail_showsError', async ({ page }) => {
    // First signup
    await page.goto('/signup');
    await page.fill('[data-testid="name-input"]', 'First User');
    await page.fill('[data-testid="email-input"]', 'duplicate-e2e@example.com');
    await page.fill('[data-testid="password-input"]', 'SecurePassword123!');
    await page.click('[data-testid="submit-button"]');
    await expect(page).toHaveURL('/');

    // Logout
    await page.click('[data-testid="user-menu"]');
    await page.click('[data-testid="logout-button"]');

    // Try same email
    await page.goto('/signup');
    await page.fill('[data-testid="name-input"]', 'Second User');
    await page.fill('[data-testid="email-input"]', 'duplicate-e2e@example.com');
    await page.fill('[data-testid="password-input"]', 'DifferentPassword456!');
    await page.click('[data-testid="submit-button"]');

    await expect(page.locator('[data-testid="error-message"]')).toHaveText(/already registered/i);
  });

  test('signup_givenWeakPassword_showsValidationError', async ({ page }) => {
    await page.goto('/signup');
    await page.fill('[data-testid="name-input"]', 'Test');
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', '123');
    await page.click('[data-testid="submit-button"]');

    await expect(page.locator('[data-testid="password-error"]')).toBeVisible();
  });
});
```

### Login Flow

`e2e/login.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';

test.describe('Login Flow', () => {
  test('login_givenValidCredentials_redirectsToHome', async ({ page }) => {
    // Create user via API
    await page.request.post('/api/auth/signup', {
      data: {
        email: 'login-e2e@example.com',
        password: 'SecurePassword123!',
        name: 'Login Test',
      },
    });

    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'login-e2e@example.com');
    await page.fill('[data-testid="password-input"]', 'SecurePassword123!');
    await page.click('[data-testid="submit-button"]');

    await expect(page).toHaveURL('/');
  });

  test('login_givenWrongPassword_showsError', async ({ page }) => {
    await page.request.post('/api/auth/signup', {
      data: {
        email: 'wrongpass-e2e@example.com',
        password: 'SecurePassword123!',
        name: 'Wrong Pass Test',
      },
    });

    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'wrongpass-e2e@example.com');
    await page.fill('[data-testid="password-input"]', 'WrongPassword!');
    await page.click('[data-testid="submit-button"]');

    await expect(page.locator('[data-testid="error-message"]')).toHaveText(/invalid/i);
  });
});
```

### Generation Flow

`e2e/generation.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';

test.describe('Image Generation Flow', () => {
  let authCookie: string;

  test.beforeEach(async ({ page }) => {
    // Authenticate
    const response = await page.request.post('/api/auth/signup', {
      data: {
        email: `gen-e2e-${Date.now()}@example.com`,
        password: 'SecurePassword123!',
        name: 'Gen Test',
      },
    });
    const { accessToken } = await response.json();

    // Set auth state
    await page.goto('/login');
    await page.evaluate((token) => {
      sessionStorage.setItem('fastdepo_access_token', token);
    }, accessToken);
    await page.goto('/');
  });

  test('generate_givenValidPrompt_showsResultImage', async ({ page }) => {
    await page.fill('[data-testid="prompt-input"]', 'A cat riding a dragon, digital art');
    await page.click('[data-testid="generate-button"]');

    // Wait for loading state
    await expect(page.locator('[data-testid="generating-spinner"]')).toBeVisible();

    // Wait for result (may take up to 60 seconds)
    await expect(page.locator('[data-testid="generated-image"]')).toBeVisible({ timeout: 60000 });
  });

  test('generate_givenEmptyPrompt_showsValidationError', async ({ page }) => {
    await page.click('[data-testid="generate-button"]');

    await expect(page.locator('[data-testid="prompt-error"]')).toHaveText(/required/i);
  });
});
```

### Gallery Flow

`e2e/gallery.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';

test.describe('Gallery Flow', () => {
  test('gallery_givenNoImages_showsEmptyState', async ({ page }) => {
    // Authenticate and navigate to gallery
    const response = await page.request.post('/api/auth/signup', {
      data: {
        email: `gallery-empty-${Date.now()}@example.com`,
        password: 'SecurePassword123!',
        name: 'Gallery Test',
      },
    });
    const { accessToken } = await response.json();

    await page.goto('/login');
    await page.evaluate((token) => {
      sessionStorage.setItem('fastdepo_access_token', token);
    }, accessToken);
    await page.goto('/gallery');

    await expect(page.locator('[data-testid="empty-gallery-message"]')).toBeVisible();
  });

  test('gallery_givenImages_showsImageGrid', async ({ page }) => {
    // Setup with existing images would go here
    // ...

    await expect(page.locator('[data-testid="gallery-grid"]')).toBeVisible();
    await expect(page.locator('[data-testid="image-card"]').first()).toBeVisible();
  });

  test('deleteImage_givenOwnImage_removesFromGallery', async ({ page }) => {
    // Create image, then delete it
    // ...

    await page.hover('[data-testid="image-card"]:first-child');
    await page.click('[data-testid="delete-button"]');

    // Confirm deletion
    await page.click('[data-testid="confirm-delete"]');

    await expect(page.locator('[data-testid="toast-success"]')).toHaveText(/deleted/i);
  });
});
```

---

## 8. k6 Load Testing Scripts

### Installation

```bash
brew install k6
```

### `load-tests/auth-load.js`

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '30s', target: 20 },   // Ramp up to 20 users
    { duration: '60s', target: 20 },   // Stay at 20 users
    { duration: '30s', target: 100 },   // Ramp up to 100 users
    { duration: '60s', target: 100 },   // Stay at 100 users
    { duration: '30s', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],   // 95% of requests must complete below 500ms
    errors: ['rate<0.05'],                // Error rate must be below 5%
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000/api';

export default function () {
  // Test signup
  const uniqueEmail = `load-test-${Date.now()}-${__VU}-${__ITER}@example.com`;
  const signupRes = http.post(`${BASE_URL}/auth/signup`, JSON.stringify({
    email: uniqueEmail,
    password: 'LoadTestPassword123!',
    name: `Load Test User ${__VU}`,
  }), {
    headers: { 'Content-Type': 'application/json' },
  });

  check(signupRes, {
    'signup status is 201': (r) => r.status === 201,
    'signup has accessToken': (r) => JSON.parse(r.body).accessToken !== undefined,
  }) || errorRate.add(1);

  const { accessToken } = JSON.parse(signupRes.body);

  // Test authenticated endpoint
  const meRes = http.get(`${BASE_URL}/auth/me`, {
    headers: { Authorization: `Bearer ${accessToken}` },
  });

  check(meRes, {
    'me status is 200': (r) => r.status === 200,
  }) || errorRate.add(1);

  sleep(1);
}
```

### `load-tests/generation-load.js`

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '1m', target: 10 },   // Ramp up
    { duration: '5m', target: 10 },    // Sustained load
    { duration: '1m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'],  // 95% under 2s (generation is slow)
    errors: ['rate<0.10'],               // Error rate under 10%
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000/api';

// Pre-create users for load testing
const users = __ENV.TEST_USERS ? JSON.parse(__ENV.TEST_USERS) : [];

export default function () {
  const user = users[__VU % users.length];
  if (!user) {
    console.error('No test user available');
    return;
  }

  const res = http.post(`${BASE_URL}/images/generate`, JSON.stringify({
    prompt: `A load test image ${__VU}-${__ITER}`,
    model: 'stable-diffusion-xl',
  }), {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${user.accessToken}`,
    },
    timeout: '120s',  // Image generation can be slow
  });

  check(res, {
    'generation status is 200 or 202': (r) => r.status === 200 || r.status === 202,
  }) || errorRate.add(1);

  sleep(2);
}
```

### Running Load Tests

```bash
# Run auth load test
k6 run load-tests/auth-load.js

# Run with custom base URL
k6 run -e BASE_URL=https://staging-api.fastdepo.com/api load-tests/auth-load.js

# Run generation load test (needs pre-created users)
k6 run -e BASE_URL=http://localhost:3000/api -e TEST_USERS='[{"accessToken":"token1"}]' load-tests/generation-load.js

# Generate HTML report
k6 run --out json=results.json load-tests/auth-load.js
k6 run --summary-export=summary.json load-tests/auth-load.js
```

---

## 9. Test Naming Convention

Follow the pattern: `methodName_givenCondition_expectedResult`

```typescript
describe('AuthService.signup', () => {
  it('signup_givenValidData_createsUserAndReturnsTokens', async () => { /* ... */ });
  it('signup_givenExistingEmail_throwsConflictError', async () => { /* ... */ });
  it('signup_givenWeakPassword_throwsValidationError', async () => { /* ... */ });
});

describe('AuthService.login', () => {
  it('login_givenValidCredentials_returnsTokens', async () => { /* ... */ });
  it('login_givenWrongPassword_throwsUnauthorizedError', async () => { /* ... */ });
  it('login_givenNonexistentEmail_throwsUnauthorizedError', async () => { /* ... */ });
});

describe('GalleryService.listByUser', () => {
  it('listByUser_givenValidUserId_returnsPaginatedImages', async () => { /* ... */ });
  it('listByUser_givenInvalidPage_defaultsToFirstPage', async () => { /* ... */ });
  it('listByUser_givenNoImages_returnsEmptyList', async () => { /* ... */ });
});
```

---

## 10. Running Tests in CI

### GitHub Actions Workflow

`.github/workflows/ci.yml`:

```yaml
name: Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npm run test:coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/lcov.info

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
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
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/fastdepo_test

      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/fastdepo_test
          JWT_SECRET: test-secret
          JWT_REFRESH_SECRET: test-refresh-secret

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npx playwright install --with-deps

      - run: npm run test:e2e
        env:
          E2E_BASE_URL: http://localhost:5173

      - name: Upload test results
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: e2e-results
          path: playwright-report/
```

---

## 11. Coverage Targets

| Layer | Lines | Branches | Functions |
|-------|-------|----------|-----------|
| Services/utils | 80%+ | 80%+ | 80%+ |
| Routes/controllers | 70%+ | 60%+ | 70%+ |
| Overall | 60%+ | 50%+ | 60%+ |

Configuration in `vitest.config.ts`:

```typescript
coverage: {
  thresholds: {
    './src/services': {
      branches: 80,
      functions: 80,
      lines: 80,
    },
    './src/routes': {
      branches: 60,
      functions: 70,
      lines: 70,
    },
    perFile: true,
    branches: 50,
    functions: 60,
    lines: 60,
    statements: 60,
  },
},
```

---

## 12. Mocking Strategies

### When to Mock vs Real DB

**Mock dependencies** (other services, external APIs, file system):
- External API calls (image generation providers, payment gateways)
- Email services
- File system operations
- Time-dependent logic

**Use real database** for:
- Repository/data access layer tests (use test DB)
- Service methods that perform complex queries
- Integration tests that test the full query pipeline

### Mock Repository Pattern

```typescript
// tests/mocks/user.repository.mock.ts
import { vi } from 'vitest';

export function createMockUserRepository() {
  return {
    findById: vi.fn(),
    findByEmail: vi.fn(),
    create: vi.fn(),
    update: vi.fn(),
    delete: vi.fn(),
    countByUser: vi.fn(),
  };
}
```

### Mock External Provider

```typescript
// tests/mocks/image.provider.mock.ts
import { vi } from 'vitest';

export function createMockImageProvider() {
  return {
    generate: vi.fn().mockResolvedValue({
      imageUrl: 'https://cdn.fastdepo.com/test/test-image.png',
      status: 'completed',
    }),
    getStatus: vi.fn().mockResolvedValue({
      status: 'completed',
      imageUrl: 'https://cdn.fastdepo.com/test/test-image.png',
    }),
  };
}
```

---

## 13. Debugging Failing Tests

### Run a Single Test

```bash
npx vitest run tests/unit/services/auth.service.test.ts
npx vitest run -t "signup_givenValidData_createsUserAndReturnsTokens"
```

### Verbose Output

```bash
npx vitest run --reporter=verbose
```

### Debug with Node Inspector

```bash
npx vitest run --inspect
# Then open chrome://inspect in Chrome
```

### Common Test Failures

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Tests pass individually but fail in batch | State leakage between tests | Ensure `afterEach` cleans up DB or reset mocks |
| `PrismaClientKnownRequestError: P2002` | Unique constraint violation in test DB | Truncate tables in `afterEach` |
| `TypeError: Cannot read property of undefined` | Mock not returning expected shape | Check mock return value matches service expectation |
| Timeout after 5000ms | Async operation not completing | Check for missing `await` or infinite loops |
| Tests fail on CI but pass locally | Environment difference | Check `DATABASE_URL`, env vars, timezone |
| `ECONNREFUSED` in integration tests | Test DB not running | Start Docker container or verify DB connection |

### Database Cleanup Helper

```typescript
// tests/helpers/cleanup.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

export async function cleanupDatabase() {
  const models = Reflect.ownKeys(prisma).filter(
    (key) => typeof prisma[key as keyof PrismaClient] === 'object' && key !== '_prisma'
  );

  for (const model of models) {
    try {
      await (prisma as any)[model].deleteMany();
    } catch (e) {
      // Model may not exist or may have foreign key constraints
    }
  }
}
```

---

## 14. TDD Workflow

### Red-Green-Refactor Cycle

1. **Red**: Write a failing test for the feature/behavior
2. **Green**: Write the minimum code to make the test pass
3. **Refactor**: Clean up the code while keeping tests green
4. **Repeat**

### Example TDD Session

```bash
# 1. RED: Write the test first
# tests/unit/services/rate-limit.service.test.ts

import { describe, it, expect } from 'vitest';
import { RateLimitService } from '@/services/rate-limit.service';

describe('RateLimitService', () => {
  it('checkLimit_givenUnderLimit_returnsAllowed', async () => {
    const service = new RateLimitService({ maxRequests: 5, windowMs: 60000 });
    const result = await service.checkLimit('user-1');
    expect(result.allowed).toBe(true);
    expect(result.remaining).toBe(4);
  });
});

# 2. Run test (should fail)
npx vitest run tests/unit/services/rate-limit.service.test.ts
# ❌ Cannot find module '@/services/rate-limit.service'

# 3. GREEN: Implement the minimum code
# src/services/rate-limit.service.ts

export class RateLimitService {
  private maxRequests: number;
  private windowMs: number;
  private requests: Map<string, number[]> = new Map();

  constructor(options: { maxRequests: number; windowMs: number }) {
    this.maxRequests = options.maxRequests;
    this.windowMs = options.windowMs;
  }

  async checkLimit(userId: string): Promise<{ allowed: boolean; remaining: number }> {
    const now = Date.now();
    const userRequests = this.requests.get(userId) || [];
    const windowStart = now - this.windowMs;
    const recentRequests = userRequests.filter((t) => t > windowStart);

    if (recentRequests.length >= this.maxRequests) {
      return { allowed: false, remaining: 0 };
    }

    recentRequests.push(now);
    this.requests.set(userId, recentRequests);

    return { allowed: true, remaining: this.maxRequests - recentRequests.length };
  }
}

# 4. Run test (should pass)
npx vitest run tests/unit/services/rate-limit.service.test.ts
# ✓ checkLimit_givenUnderLimit_returnsAllowed

# 5. REFACTOR: Add more tests, then refactor
```

### Watch Mode for TDD

```bash
# Run tests on file changes
npm run test:watch

# Run specific test file
npx vitest watch tests/unit/services/rate-limit.service.test.ts
```
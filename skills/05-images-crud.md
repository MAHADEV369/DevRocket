# Skill 05: Image Generation CRUD

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 4-5 hours
Depends On: 01, 03, 04

## Input Contract
- Skills 01, 03, 04 complete
- Auth middleware (`authenticate`, `requireAdmin`, `requirePlan`) available
- Prisma client with `Image` model available
- Middleware pipeline in `src/app.ts` wired up
- `src/utils/errors.ts` with AppError hierarchy

## Output Contract
- `src/modules/images/` with routes, controller, service, validation, types
- `src/modules/images/providers/` with base provider, DALL-E, Flux, and Pollinations implementations
- Endpoints: POST `/api/images/generate`, GET `/api/images`, GET `/api/images/:id`, DELETE `/api/images/:id`, PATCH `/api/images/:id/favorite`
- Provider pattern allows adding new AI image providers without modifying service code

## Files to Create

| File | Description |
|------|-------------|
| `src/modules/images/types.ts` | Image-related TypeScript types |
| `src/modules/images/validation.ts` | Zod schemas for image endpoints |
| `src/modules/images/providers/base.ts` | Abstract ImageProvider class |
| `src/modules/images/providers/dalle.ts` | DALL-E provider implementation |
| `src/modules/images/providers/flux.ts` | Flux provider implementation |
| `src/modules/images/providers/pollinations.ts` | Pollinations provider implementation |
| `src/modules/images/providers/index.ts` | Provider registry and factory |
| `src/modules/images/service.ts` | Image business logic |
| `src/modules/images/controller.ts` | Express route handlers |
| `src/modules/images/routes.ts` | Express router for image endpoints |

## Steps

### 1. Create `src/modules/images/types.ts`

```typescript
export type ImageProvider = 'dalle' | 'flux' | 'pollinations';

export interface GenerateImageInput {
  prompt: string;
  negativePrompt?: string;
  provider?: ImageProvider;
  width?: number;
  height?: number;
  isPublic?: boolean;
}

export interface GenerateProviderResult {
  url: string;
  providerId?: string;
  width?: number;
  height?: number;
  metadata?: Record<string, unknown>;
}

export interface ListImagesQuery {
  page?: number;
  limit?: number;
  provider?: ImageProvider;
  favorite?: boolean;
  sort?: 'createdAt' | 'updatedAt';
  order?: 'asc' | 'desc';
}

export interface UpdateFavoriteInput {
  isFavorite: boolean;
}

export interface ImageResponse {
  id: string;
  userId: string;
  prompt: string;
  negativePrompt: string | null;
  provider: string;
  providerId: string | null;
  url: string;
  thumbnailUrl: string | null;
  width: number | null;
  height: number | null;
  isFavorite: boolean;
  isPublic: boolean;
  metadata: Record<string, unknown> | null;
  createdAt: string;
  updatedAt: string;
}
```

### 2. Create `src/modules/images/validation.ts`

```typescript
import { z } from 'zod';

export const generateImageSchema = z.object({
  prompt: z.string().min(1, 'Prompt is required').max(4000, 'Prompt must be under 4000 characters'),
  negativePrompt: z.string().max(1000).optional(),
  provider: z.enum(['dalle', 'flux', 'pollinations']).default('pollinations'),
  width: z.number().int().min(256).max(2048).optional(),
  height: z.number().int().min(256).max(2048).optional(),
  isPublic: z.boolean().default(false),
});

export const listImagesSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  provider: z.enum(['dalle', 'flux', 'pollinations']).optional(),
  favorite: z.coerce.boolean().optional(),
  sort: z.enum(['createdAt', 'updatedAt']).default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
});

export const imageIdParamSchema = z.object({
  id: z.string().cuid('Invalid image ID'),
});

export const updateFavoriteSchema = z.object({
  isFavorite: z.boolean(),
});

export type GenerateImageInput = z.infer<typeof generateImageSchema>;
export type ListImagesQuery = z.infer<typeof listImagesSchema>;
export type ImageIdParam = z.infer<typeof imageIdParamSchema>;
export type UpdateFavoriteInput = z.infer<typeof updateFavoriteSchema>;
```

### 3. Create `src/modules/images/providers/base.ts`

```typescript
import type { GenerateProviderResult } from '../types.js';

export interface ProviderConfig {
  apiKey?: string;
  baseUrl?: string;
  defaultModel?: string;
}

export abstract class ImageProvider {
  abstract readonly name: string;
  abstract readonly maxResolution: number;

  abstract generate(
    prompt: string,
    options: {
      negativePrompt?: string;
      width?: number;
      height?: number;
      model?: string;
    }
  ): Promise<GenerateProviderResult>;

  abstract isAvailable(): boolean;
}
```

### 4. Create `src/modules/images/providers/dalle.ts`

```typescript
import { ImageProvider, type ProviderConfig } from './base.js';
import type { GenerateProviderResult } from '../types.js';
import { config } from '../../../config/index.js';
import { logger } from '../../../utils/logger.js';
import { AppError } from '../../../utils/errors.js';

interface DallEImageResponse {
  data: Array<{
    url: string;
    revised_prompt?: string;
    b64_json?: string;
  }>;
}

export class DallEProvider extends ImageProvider {
  readonly name = 'dalle';
  readonly maxResolution = 2048;
  private apiKey: string;
  private baseUrl: string;

  constructor(config?: ProviderConfig) {
    super();
    this.apiKey = config?.apiKey || '';
    this.baseUrl = config?.baseUrl || 'https://api.openai.com/v1';
  }

  isAvailable(): boolean {
    return !!this.apiKey;
  }

  async generate(
    prompt: string,
    options: {
      negativePrompt?: string;
      width?: number;
      height?: number;
      model?: string;
    }
  ): Promise<GenerateProviderResult> {
    if (!this.isAvailable()) {
      throw new AppError(503, 'DALL-E provider is not configured');
    }

    const size = this.resolveSize(options.width, options.height);
    const model = options.model || 'dall-e-3';

    let requestBody: Record<string, unknown>;
    if (model === 'dall-e-3') {
      requestBody = {
        model,
        prompt: options.negativePrompt
          ? `${prompt}. Avoid: ${options.negativePrompt}`
          : prompt,
        n: 1,
        size,
        quality: 'standard',
        response_format: 'url',
      };
    } else {
      requestBody = {
        model: model || 'dall-e-2',
        prompt,
        n: 1,
        size,
        response_format: 'url',
      };
    }

    logger.info({ model, size, promptLength: prompt.length }, 'DALL-E generation request');

    const response = await fetch(`${this.baseUrl}/images/generations`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${this.apiKey}`,
      },
      body: JSON.stringify(requestBody),
    });

    if (!response.ok) {
      const errorBody = await response.text();
      logger.error({ status: response.status, body: errorBody }, 'DALL-E API error');
      throw new AppError(502, `DALL-E API error: ${response.status}`);
    }

    const data = (await response.json()) as DallEImageResponse;
    const image = data.data[0];

    return {
      url: image.url,
      providerId: undefined,
      width: size === '1024x1024' ? 1024 : size === '1024x1792' ? 1024 : 1792,
      height: size === '1024x1024' ? 1024 : size === '1024x1792' ? 1792 : 1024,
      metadata: {
        revisedPrompt: image.revised_prompt,
        model,
      },
    };
  }

  private resolveSize(width?: number, height?: number): string {
    if (width && height) {
      const w = Math.min(width, 2048);
      const h = Math.min(height, 2048);
      if (w === 1024 && h === 1024) return '1024x1024';
      if (w === 1792 && h === 1024) return '1792x1024';
      if (w === 1024 && h === 1792) return '1024x1792';
    }
    return '1024x1024';
  }
}
```

### 5. Create `src/modules/images/providers/flux.ts`

```typescript
import { ImageProvider, type ProviderConfig } from './base.js';
import type { GenerateProviderResult } from '../types.js';
import { AppError } from '../../../utils/errors.js';
import { logger } from '../../../utils/logger.js';

interface FluxImageResponse {
  image: string;
  seed?: number;
  timing?: number;
}

export class FluxProvider extends ImageProvider {
  readonly name = 'flux';
  readonly maxResolution = 2048;
  private apiKey: string;
  private baseUrl: string;

  constructor(config?: ProviderConfig) {
    super();
    this.apiKey = config?.apiKey || '';
    this.baseUrl = config?.baseUrl || 'https://api.flux.ai/v1';
  }

  isAvailable(): boolean {
    return !!this.apiKey;
  }

  async generate(
    prompt: string,
    options: {
      negativePrompt?: string;
      width?: number;
      height?: number;
      model?: string;
    }
  ): Promise<GenerateProviderResult> {
    if (!this.isAvailable()) {
      throw new AppError(503, 'Flux provider is not configured');
    }

    const width = options.width || 1024;
    const height = options.height || 1024;
    const model = options.model || 'flux-pro';

    const requestBody = {
      prompt,
      negative_prompt: options.negativePrompt,
      width: Math.min(width, this.maxResolution),
      height: Math.min(height, this.maxResolution),
      model,
      steps: 30,
    };

    logger.info({ model, width, height }, 'Flux generation request');

    const response = await fetch(`${this.baseUrl}/images/generate`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey,
      },
      body: JSON.stringify(requestBody),
    });

    if (!response.ok) {
      const errorBody = await response.text();
      logger.error({ status: response.status, body: errorBody }, 'Flux API error');
      throw new AppError(502, `Flux API error: ${response.status}`);
    }

    const data = (await response.json()) as FluxImageResponse;

    return {
      url: data.image,
      width,
      height,
      metadata: {
        seed: data.seed,
        model,
      },
    };
  }
}
```

### 6. Create `src/modules/images/providers/pollinations.ts`

```typescript
import { ImageProvider } from './base.js';
import type { GenerateProviderResult } from '../types.js';
import { logger } from '../../../utils/logger.js';

export class PollinationsProvider extends ImageProvider {
  readonly name = 'pollinations';
  readonly maxResolution = 2048;

  isAvailable(): boolean {
    return true;
  }

  async generate(
    prompt: string,
    options: {
      negativePrompt?: string;
      width?: number;
      height?: number;
      model?: string;
    }
  ): Promise<GenerateProviderResult> {
    const width = Math.min(options.width || 1024, this.maxResolution);
    const height = Math.min(options.height || 1024, this.maxResolution);
    const model = options.model || 'flux';
    const seed = Math.floor(Math.random() * 999999999);

    const encodedPrompt = encodeURIComponent(prompt);
    const url = `https://image.pollinations.ai/prompt/${encodedPrompt}?width=${width}&height=${height}&model=${model}&seed=${seed}&nologo=true`;

    logger.info({ model, width, height, seed }, 'Pollinations generation request');

    const response = await fetch(url, { method: 'HEAD' });
    if (!response.ok) {
      logger.error({ status: response.status }, 'Pollinations validation error');
    }

    return {
      url,
      width,
      height,
      metadata: {
        seed,
        model,
        endpoint: 'pollinations',
      },
    };
  }
}
```

### 7. Create `src/modules/images/providers/index.ts`

```typescript
import { DallEProvider } from './dalle.js';
import { FluxProvider } from './flux.js';
import { PollinationsProvider } from './pollinations.js';
import type { ImageProvider } from './base.js';
import type { ImageProvider as ProviderName } from '../types.js';

const providers: Map<string, ImageProvider> = new Map();

export function initializeProviders(): void {
  const pollinations = new PollinationsProvider();
  providers.set('pollinations', pollinations);

  const dalle = new DallEProvider();
  if (dalle.isAvailable()) {
    providers.set('dalle', dalle);
  }

  const flux = new FluxProvider();
  if (flux.isAvailable()) {
    providers.set('flux', flux);
  }

  if (!providers.has('pollinations')) {
    throw new Error('At least the Pollinations provider must be available');
  }
}

export function getProvider(name: ProviderName): ImageProvider {
  const provider = providers.get(name);
  if (!provider) {
    const available = Array.from(providers.keys()).join(', ');
    throw new Error(`Provider "${name}" not available. Available: ${available}`);
  }
  return provider;
}

export function getAvailableProviders(): string[] {
  return Array.from(providers.keys());
}

initializeProviders();
```

### 8. Create `src/modules/images/service.ts`

```typescript
import { prisma } from '../../config/database.js';
import { getProvider } from './providers/index.js';
import { AppError, NotFoundError, ForbiddenError } from '../../utils/errors.js';
import type { GenerateImageInput, ListImagesQuery } from './types.js';

export async function generateImage(userId: string, input: GenerateImageInput) {
  const providerName = input.provider || 'pollinations';
  const provider = getProvider(providerName);

  const result = await provider.generate(input.prompt, {
    negativePrompt: input.negativePrompt,
    width: input.width,
    height: input.height,
  });

  const image = await prisma.image.create({
    data: {
      userId,
      prompt: input.prompt,
      negativePrompt: input.negativePrompt,
      provider: providerName,
      providerId: result.providerId,
      url: result.url,
      thumbnailUrl: result.url,
      width: result.width || input.width,
      height: result.height || input.height,
      isPublic: input.isPublic ?? false,
      metadata: result.metadata ?? undefined,
    },
  });

  return image;
}

export async function listImages(userId: string, query: ListImagesQuery) {
  const page = query.page || 1;
  const limit = query.limit || 20;
  const skip = (page - 1) * limit;

  const where: Record<string, unknown> = { userId };
  if (query.provider) {
    where.provider = query.provider;
  }
  if (query.favorite !== undefined) {
    where.isFavorite = query.favorite;
  }

  const [images, total] = await Promise.all([
    prisma.image.findMany({
      where,
      skip,
      take: limit,
      orderBy: { [query.sort || 'createdAt']: query.order || 'desc' },
    }),
    prisma.image.count({ where }),
  ]);

  return {
    data: images,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  };
}

export async function getImage(userId: string, imageId: string) {
  const image = await prisma.image.findUnique({ where: { id: imageId } });
  if (!image) {
    throw new NotFoundError('Image');
  }
  if (image.userId !== userId && !image.isPublic) {
    throw new ForbiddenError('You do not have access to this image');
  }
  return image;
}

export async function getPublicImage(imageId: string) {
  const image = await prisma.image.findUnique({ where: { id: imageId } });
  if (!image) {
    throw new NotFoundError('Image');
  }
  if (!image.isPublic) {
    throw new ForbiddenError('This image is not public');
  }
  return image;
}

export async function deleteImage(userId: string, imageId: string) {
  const image = await prisma.image.findUnique({ where: { id: imageId } });
  if (!image) {
    throw new NotFoundError('Image');
  }
  if (image.userId !== userId) {
    throw new ForbiddenError('You can only delete your own images');
  }
  await prisma.image.delete({ where: { id: imageId } });
  return { id: imageId, deleted: true };
}

export async function toggleFavorite(userId: string, imageId: string, isFavorite: boolean) {
  const image = await prisma.image.findUnique({ where: { id: imageId } });
  if (!image) {
    throw new NotFoundError('Image');
  }
  if (image.userId !== userId) {
    throw new ForbiddenError('You can only favorite your own images');
  }

  return prisma.image.update({
    where: { id: imageId },
    data: { isFavorite },
  });
}
```

### 9. Create `src/modules/images/controller.ts`

```typescript
import type { Request, Response, NextFunction } from 'express';
import type { AuthenticatedRequest } from '../auth/types.js';
import * as imageService from './service.js';

export async function generateImage(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const image = await imageService.generateImage(req.user.userId, req.body);
    res.status(201).json(image);
  } catch (error) {
    next(error);
  }
}

export async function listImages(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const result = await imageService.listImages(req.user.userId, req.query as any);
    res.json(result);
  } catch (error) {
    next(error);
  }
}

export async function getImage(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const image = await imageService.getImage(req.user.userId, req.params.id);
    res.json(image);
  } catch (error) {
    next(error);
  }
}

export async function deleteImage(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const result = await imageService.deleteImage(req.user.userId, req.params.id);
    res.json(result);
  } catch (error) {
    next(error);
  }
}

export async function toggleFavorite(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const image = await imageService.toggleFavorite(req.user.userId, req.params.id, req.body.isFavorite);
    res.json(image);
  } catch (error) {
    next(error);
  }
}
```

### 10. Create `src/modules/images/routes.ts`

```typescript
import { Router } from 'express';
import * as controller from './controller.js';
import { authenticate } from '../../middleware/auth.js';
import { validate } from '../../middleware/validate.js';
import { imageGenerationLimiter } from '../../middleware/rateLimiter.js';
import { generateImageSchema, listImagesSchema, imageIdParamSchema, updateFavoriteSchema } from './validation.js';

const router = Router();

router.post(
  '/generate',
  authenticate,
  imageGenerationLimiter,
  validate(generateImageSchema),
  controller.generateImage
);

router.get(
  '/',
  authenticate,
  validate({ query: listImagesSchema }),
  controller.listImages
);

router.get(
  '/:id',
  authenticate,
  validate({ params: imageIdParamSchema }),
  controller.getImage
);

router.delete(
  '/:id',
  authenticate,
  validate({ params: imageIdParamSchema }),
  controller.deleteImage
);

router.patch(
  '/:id/favorite',
  authenticate,
  validate({ params: imageIdParamSchema, body: updateFavoriteSchema }),
  controller.toggleFavorite
);

export default router;
```

### 11. Register image routes in `src/app.ts`

Add import and route:

```typescript
import imageRoutes from './modules/images/routes.js';
// ...
app.use('/api/images', imageRoutes);
```

### 12. Verify

```bash
npm run typecheck && npm run dev
```

## Verification

```bash
npm run typecheck
# Expected: no errors

# Start server and test
npm run dev &
sleep 2

# Generate image (requires auth token from Skill 03)
TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"Password123!"}' | jq -r '.accessToken')

# List images
curl -s http://localhost:3000/api/images \
  -H "Authorization: Bearer $TOKEN" | jq .

# Generate an image
curl -s -X POST http://localhost:3000/api/images/generate \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"prompt":"a sunset over mountains","provider":"pollinations"}' | jq .

# Toggle favorite
curl -s -X PATCH http://localhost:3000/api/images/<IMAGE_ID>/favorite \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"isFavorite":true}' | jq .
```

## Rollback

```bash
rm -rf src/modules/images
# Remove image routes import and registration from src/app.ts
```

## ADR-005: Provider Pattern for Image Generation

Decision: Use a Provider pattern with a base abstract class and concrete implementations for each AI image service.
Reason: Each provider (DALL-E, Flux, Pollinations) has different APIs, auth methods, and response shapes. The provider pattern isolates this complexity behind a common interface. Adding a new provider just requires implementing the abstract class and registering it.
Consequences: Each provider must be tested independently. Error handling varied provider APIs adds complexity. Pollinations being free (no key) makes it the default.
Alternatives Considered: Single service with switch/case (harder to test, grows unwieldy), webhook-based async pattern (overkill for MVP), direct API calls in controller (no separation of concerns).
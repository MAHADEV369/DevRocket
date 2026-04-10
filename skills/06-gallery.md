# Skill 06: Gallery & Sharing

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 2-3 hours
Depends On: 02, 03, 04

## Input Contract
- Skills 02, 03, 04 complete
- Prisma models `Gallery`, `GalleryImage` available
- Auth middleware (`authenticate`) available
- Validation middleware (`validate`) available
- Error classes available in `src/utils/errors.ts`

## Output Contract
- `src/modules/gallery/` with routes, controller, service, validation, types
- Endpoints:
  - POST `/api/galleries` - Create gallery
  - GET `/api/galleries` - List user's galleries
  - GET `/api/galleries/:id` - Get gallery by ID (owner or public)
  - PUT `/api/galleries/:id` - Update gallery
  - DELETE `/api/galleries/:id` - Delete gallery
  - POST `/api/galleries/:id/images` - Add images to gallery
  - DELETE `/api/galleries/:id/images/:imageId` - Remove image from gallery
  - PATCH `/api/galleries/:id/images/reorder` - Reorder images in gallery
  - POST `/api/galleries/:id/share` - Generate/regenerate share token
  - DELETE `/api/galleries/:id/share` - Revoke share access
  - GET `/api/galleries/shared/:shareToken` - View shared gallery (public, no auth)

## Files to Create

| File | Description |
|------|-------------|
| `src/modules/gallery/types.ts` | Gallery-related TypeScript types |
| `src/modules/gallery/validation.ts` | Zod schemas for gallery endpoints |
| `src/modules/gallery/service.ts` | Gallery business logic |
| `src/modules/gallery/controller.ts` | Express route handlers |
| `src/modules/gallery/routes.ts` | Express router with all gallery endpoints |

## Steps

### 1. Create `src/modules/gallery/types.ts`

```typescript
export interface CreateGalleryInput {
  title: string;
  description?: string;
  coverUrl?: string;
  isPublic?: boolean;
}

export interface UpdateGalleryInput {
  title?: string;
  description?: string;
  coverUrl?: string;
  isPublic?: boolean;
}

export interface AddImagesInput {
  imageIds: string[];
}

export interface ReorderImagesInput {
  orders: Array<{ imageId: string; position: number }>;
}

export interface ListGalleriesQuery {
  page?: number;
  limit?: number;
  sort?: 'createdAt' | 'updatedAt' | 'title';
  order?: 'asc' | 'desc';
}

export interface GalleryResponse {
  id: string;
  userId: string;
  title: string;
  description: string | null;
  coverUrl: string | null;
  isPublic: boolean;
  shareToken: string | null;
  images: Array<{
    id: string;
    imageId: string;
    position: number;
    image: {
      id: string;
      url: string;
      thumbnailUrl: string | null;
      prompt: string;
      provider: string;
      width: number | null;
      height: number | null;
    };
  }>;
  imageCount: number;
  createdAt: string;
  updatedAt: string;
}
```

### 2. Create `src/modules/gallery/validation.ts`

```typescript
import { z } from 'zod';

export const createGallerySchema = z.object({
  title: z.string().min(1, 'Title is required').max(200, 'Title must be under 200 characters'),
  description: z.string().max(1000, 'Description must be under 1000 characters').optional(),
  coverUrl: z.string().url('Invalid URL').optional(),
  isPublic: z.boolean().default(false),
});

export const updateGallerySchema = z.object({
  title: z.string().min(1).max(200).optional(),
  description: z.string().max(1000).nullable().optional(),
  coverUrl: z.string().url().nullable().optional(),
  isPublic: z.boolean().optional(),
});

export const addImagesSchema = z.object({
  imageIds: z.array(z.string().cuid('Invalid image ID')).min(1, 'At least one image ID is required').max(50, 'Cannot add more than 50 images at once'),
});

export const removeImageParamSchema = z.object({
  id: z.string().cuid('Invalid gallery ID'),
  imageId: z.string().cuid('Invalid image ID'),
});

export const reorderImagesSchema = z.object({
  orders: z.array(
    z.object({
      imageId: z.string().cuid('Invalid image ID'),
      position: z.number().int().min(0),
    })
  ).min(1, 'At least one order entry is required'),
});

export const listGalleriesSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  sort: z.enum(['createdAt', 'updatedAt', 'title']).default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
});

export const galleryIdParamSchema = z.object({
  id: z.string().cuid('Invalid gallery ID'),
});

export const shareTokenParamSchema = z.object({
  shareToken: z.string().min(1, 'Share token is required'),
});

export type CreateGalleryInput = z.infer<typeof createGallerySchema>;
export type UpdateGalleryInput = z.infer<typeof updateGallerySchema>;
export type AddImagesInput = z.infer<typeof addImagesSchema>;
export type ReorderImagesInput = z.infer<typeof reorderImagesSchema>;
export type ListGalleriesQuery = z.infer<typeof listGalleriesSchema>;
```

### 3. Create `src/modules/gallery/service.ts`

```typescript
import { prisma } from '../../config/database.js';
import { generateRandomToken } from '../../utils/crypto.js';
import { AppError, NotFoundError, ForbiddenError, ValidationError } from '../../utils/errors.js';
import type {
  CreateGalleryInput,
  UpdateGalleryInput,
  AddImagesInput,
  ReorderImagesInput,
  ListGalleriesQuery,
} from './types.js';

export async function createGallery(userId: string, input: CreateGalleryInput) {
  const gallery = await prisma.gallery.create({
    data: {
      userId,
      title: input.title,
      description: input.description,
      coverUrl: input.coverUrl,
      isPublic: input.isPublic ?? false,
    },
    include: {
      galleryImages: {
        include: { image: true },
        orderBy: { position: 'asc' },
      },
      _count: { select: { galleryImages: true } },
    },
  });

  return formatGallery(gallery, gallery._count.galleryImages);
}

export async function listGalleries(userId: string, query: ListGalleriesQuery) {
  const page = query.page || 1;
  const limit = query.limit || 20;
  const skip = (page - 1) * limit;

  const [galleries, total] = await Promise.all([
    prisma.gallery.findMany({
      where: { userId },
      skip,
      take: limit,
      orderBy: { [query.sort || 'createdAt']: query.order || 'desc' },
      include: {
        _count: { select: { galleryImages: true } },
      },
    }),
    prisma.gallery.count({ where: { userId } }),
  ]);

  return {
    data: galleries.map((g) => ({
      id: g.id,
      userId: g.userId,
      title: g.title,
      description: g.description,
      coverUrl: g.coverUrl,
      isPublic: g.isPublic,
      shareToken: g.shareToken,
      imageCount: g._count.galleryImages,
      createdAt: g.createdAt,
      updatedAt: g.updatedAt,
    })),
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  };
}

export async function getGallery(userId: string, galleryId: string) {
  const gallery = await prisma.gallery.findUnique({
    where: { id: galleryId },
    include: {
      galleryImages: {
        include: { image: true },
        orderBy: { position: 'asc' },
      },
      _count: { select: { galleryImages: true } },
    },
  });

  if (!gallery) {
    throw new NotFoundError('Gallery');
  }

  if (!gallery.isPublic && gallery.userId !== userId) {
    throw new ForbiddenError('You do not have access to this gallery');
  }

  return formatGallery(gallery, gallery._count.galleryImages);
}

export async function updateGallery(userId: string, galleryId: string, input: UpdateGalleryInput) {
  const gallery = await prisma.gallery.findUnique({ where: { id: galleryId } });
  if (!gallery) throw new NotFoundError('Gallery');
  if (gallery.userId !== userId) throw new ForbiddenError('You can only edit your own galleries');

  const updated = await prisma.gallery.update({
    where: { id: galleryId },
    data: {
      title: input.title,
      description: input.description === null ? null : input.description,
      coverUrl: input.coverUrl === null ? null : input.coverUrl,
      isPublic: input.isPublic,
    },
    include: {
      galleryImages: {
        include: { image: true },
        orderBy: { position: 'asc' },
      },
      _count: { select: { galleryImages: true } },
    },
  });

  return formatGallery(updated, updated._count.galleryImages);
}

export async function deleteGallery(userId: string, galleryId: string) {
  const gallery = await prisma.gallery.findUnique({ where: { id: galleryId } });
  if (!gallery) throw new NotFoundError('Gallery');
  if (gallery.userId !== userId) throw new ForbiddenError('You can only delete your own galleries');

  await prisma.galleryImage.deleteMany({ where: { galleryId } });
  await prisma.gallery.delete({ where: { id: galleryId } });

  return { id: galleryId, deleted: true };
}

export async function addImages(userId: string, galleryId: string, input: AddImagesInput) {
  const gallery = await prisma.gallery.findUnique({
    where: { id: galleryId },
    include: { _count: { select: { galleryImages: true } } },
  });
  if (!gallery) throw new NotFoundError('Gallery');
  if (gallery.userId !== userId) throw new ForbiddenError('You can only add images to your own galleries');

  const existingImages = await prisma.image.findMany({
    where: { id: { in: input.imageIds }, userId },
    select: { id: true },
  });

  const validIds = new Set(existingImages.map((img) => img.id));
  const invalidIds = input.imageIds.filter((id) => !validIds.has(id));
  if (invalidIds.length > 0) {
    throw new ValidationError(`Images not found or not owned by you: ${invalidIds.join(', ')}`);
  }

  const existingGalleryImages = await prisma.galleryImage.findMany({
    where: { galleryId, imageId: { in: input.imageIds } },
    select: { imageId: true },
  });
  const alreadyAdded = new Set(existingGalleryImages.map((gi) => gi.imageId));
  const newImageIds = input.imageIds.filter((id) => !alreadyAdded.has(id));

  if (newImageIds.length === 0) {
    throw new ValidationError('All images are already in this gallery');
  }

  const maxPosition = gallery._count.galleryImages;
  await prisma.galleryImage.createMany({
    data: newImageIds.map((imageId, index) => ({
      galleryId,
      imageId,
      position: maxPosition + index,
    })),
  });

  return getGallery(userId, galleryId);
}

export async function removeImage(userId: string, galleryId: string, imageId: string) {
  const gallery = await prisma.gallery.findUnique({ where: { id: galleryId } });
  if (!gallery) throw new NotFoundError('Gallery');
  if (gallery.userId !== userId) throw new ForbiddenError('You can only remove images from your own galleries');

  await prisma.galleryImage.deleteMany({
    where: { galleryId, imageId },
  });

  return { removed: true };
}

export async function reorderImages(userId: string, galleryId: string, input: ReorderImagesInput) {
  const gallery = await prisma.gallery.findUnique({ where: { id: galleryId } });
  if (!gallery) throw new NotFoundError('Gallery');
  if (gallery.userId !== userId) throw new ForbiddenError('You can only reorder your own galleries');

  await prisma.$transaction(
    input.orders.map((item) =>
      prisma.galleryImage.updateMany({
        where: { galleryId, imageId: item.imageId },
        data: { position: item.position },
      })
    )
  );

  return getGallery(userId, galleryId);
}

export async function generateShareToken(userId: string, galleryId: string) {
  const gallery = await prisma.gallery.findUnique({ where: { id: galleryId } });
  if (!gallery) throw new NotFoundError('Gallery');
  if (gallery.userId !== userId) throw new ForbiddenError('You can only share your own galleries');

  const shareToken = generateRandomToken();
  const updated = await prisma.gallery.update({
    where: { id: galleryId },
    data: { shareToken, isPublic: true },
  });

  return { shareToken: updated.shareToken, isPublic: updated.isPublic };
}

export async function revokeShareToken(userId: string, galleryId: string) {
  const gallery = await prisma.gallery.findUnique({ where: { id: galleryId } });
  if (!gallery) throw new NotFoundError('Gallery');
  if (gallery.userId !== userId) throw new ForbiddenError('You can only revoke sharing on your own galleries');

  await prisma.gallery.update({
    where: { id: galleryId },
    data: { shareToken: null, isPublic: false },
  });

  return { revoked: true };
}

export async function getSharedGallery(shareToken: string) {
  const gallery = await prisma.gallery.findUnique({
    where: { shareToken },
    include: {
      galleryImages: {
        include: {
          image: {
            select: {
              id: true,
              url: true,
              thumbnailUrl: true,
              prompt: true,
              provider: true,
              width: true,
              height: true,
              isPublic: true,
            },
          },
        },
        orderBy: { position: 'asc' },
      },
      _count: { select: { galleryImages: true } },
    },
  });

  if (!gallery) {
    throw new NotFoundError('Shared gallery');
  }

  const publicImages = gallery.galleryImages.filter((gi) => gi.image.isPublic);

  return {
    id: gallery.id,
    title: gallery.title,
    description: gallery.description,
    coverUrl: gallery.coverUrl,
    imageCount: gallery._count.galleryImages,
    images: publicImages.map((gi) => ({
      id: gi.image.id,
      url: gi.image.url,
      thumbnailUrl: gi.image.thumbnailUrl,
      prompt: gi.image.prompt,
      provider: gi.image.provider,
      width: gi.image.width,
      height: gi.image.height,
      position: gi.position,
    })),
    createdAt: gallery.createdAt,
    updatedAt: gallery.updatedAt,
  };
}

function formatGallery(gallery: any, imageCount: number) {
  const { _count, galleryImages, ...galleryData } = gallery;
  return {
    ...galleryData,
    imageCount,
    images: (galleryImages || []).map((gi: any) => ({
      id: gi.id,
      imageId: gi.imageId,
      position: gi.position,
      image: {
        id: gi.image.id,
        url: gi.image.url,
        thumbnailUrl: gi.image.thumbnailUrl,
        prompt: gi.image.prompt,
        provider: gi.image.provider,
        width: gi.image.width,
        height: gi.image.height,
      },
    })),
  };
}
```

### 4. Create `src/modules/gallery/controller.ts`

```typescript
import type { Response, NextFunction } from 'express';
import type { AuthenticatedRequest } from '../auth/types.js';
import * as galleryService from './service.js';

export async function createGallery(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const gallery = await galleryService.createGallery(req.user.userId, req.body);
    res.status(201).json(gallery);
  } catch (error) {
    next(error);
  }
}

export async function listGalleries(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const result = await galleryService.listGalleries(req.user.userId, req.query as any);
    res.json(result);
  } catch (error) {
    next(error);
  }
}

export async function getGallery(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const gallery = await galleryService.getGallery(req.user.userId, req.params.id);
    res.json(gallery);
  } catch (error) {
    next(error);
  }
}

export async function updateGallery(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const gallery = await galleryService.updateGallery(req.user.userId, req.params.id, req.body);
    res.json(gallery);
  } catch (error) {
    next(error);
  }
}

export async function deleteGallery(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    await galleryService.deleteGallery(req.user.userId, req.params.id);
    res.json({ deleted: true });
  } catch (error) {
    next(error);
  }
}

export async function addImages(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const gallery = await galleryService.addImages(req.user.userId, req.params.id, req.body);
    res.json(gallery);
  } catch (error) {
    next(error);
  }
}

export async function removeImage(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    await galleryService.removeImage(req.user.userId, req.params.id, req.params.imageId);
    res.json({ removed: true });
  } catch (error) {
    next(error);
  }
}

export async function reorderImages(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const gallery = await galleryService.reorderImages(req.user.userId, req.params.id, req.body);
    res.json(gallery);
  } catch (error) {
    next(error);
  }
}

export async function generateShareToken(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const result = await galleryService.generateShareToken(req.user.userId, req.params.id);
    res.json(result);
  } catch (error) {
    next(error);
  }
}

export async function revokeShareToken(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    await galleryService.revokeShareToken(req.user.userId, req.params.id);
    res.json({ revoked: true });
  } catch (error) {
    next(error);
  }
}

export async function getSharedGallery(req: any, res: Response, next: NextFunction) {
  try {
    const gallery = await galleryService.getSharedGallery(req.params.shareToken);
    res.json(gallery);
  } catch (error) {
    next(error);
  }
}
```

### 5. Create `src/modules/gallery/routes.ts`

```typescript
import { Router } from 'express';
import * as controller from './controller.js';
import { authenticate } from '../../middleware/auth.js';
import { validate } from '../../middleware/validate.js';
import {
  createGallerySchema,
  updateGallerySchema,
  addImagesSchema,
  removeImageParamSchema,
  reorderImagesSchema,
  listGalleriesSchema,
  galleryIdParamSchema,
  shareTokenParamSchema,
} from './validation.js';

const router = Router();

router.post(
  '/',
  authenticate,
  validate(createGallerySchema),
  controller.createGallery
);

router.get(
  '/',
  authenticate,
  validate({ query: listGalleriesSchema }),
  controller.listGalleries
);

router.get(
  '/shared/:shareToken',
  validate({ params: shareTokenParamSchema }),
  controller.getSharedGallery
);

router.get(
  '/:id',
  authenticate,
  validate({ params: galleryIdParamSchema }),
  controller.getGallery
);

router.put(
  '/:id',
  authenticate,
  validate({ params: galleryIdParamSchema, body: updateGallerySchema }),
  controller.updateGallery
);

router.delete(
  '/:id',
  authenticate,
  validate({ params: galleryIdParamSchema }),
  controller.deleteGallery
);

router.post(
  '/:id/images',
  authenticate,
  validate({ params: galleryIdParamSchema, body: addImagesSchema }),
  controller.addImages
);

router.delete(
  '/:id/images/:imageId',
  authenticate,
  validate(removeImageParamSchema),
  controller.removeImage
);

router.patch(
  '/:id/images/reorder',
  authenticate,
  validate({ params: galleryIdParamSchema, body: reorderImagesSchema }),
  controller.reorderImages
);

router.post(
  '/:id/share',
  authenticate,
  validate({ params: galleryIdParamSchema }),
  controller.generateShareToken
);

router.delete(
  '/:id/share',
  authenticate,
  validate({ params: galleryIdParamSchema }),
  controller.revokeShareToken
);

export default router;
```

### 6. Register gallery routes in `src/app.ts`

Add import and route:

```typescript
import galleryRoutes from './modules/gallery/routes.js';
// ...
app.use('/api/galleries', galleryRoutes);
```

## Verification

```bash
npm run typecheck
# Expected: no errors

npm run dev &
sleep 2

# Get auth token
TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"Password123!"}' | jq -r '.accessToken')

# Create gallery
curl -s -X POST http://localhost:3000/api/galleries \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"My Gallery","description":"Test gallery","isPublic":true}' | jq .

# List galleries
curl -s http://localhost:3000/api/galleries \
  -H "Authorization: Bearer $TOKEN" | jq .

# Share a gallery
curl -s -X POST http://localhost:3000/api/galleries/<GALLERY_ID>/share \
  -H "Authorization: Bearer $TOKEN" | jq .

# View shared gallery (no auth)
curl -s http://localhost:3000/api/galleries/shared/<SHARE_TOKEN> | jq .
```

## Rollback

```bash
rm -rf src/modules/gallery
# Remove gallery routes import and registration from src/app.ts
```

## ADR-006: Share Tokens for Gallery Access

Decision: Use random cryptographic tokens (32 bytes hex) for gallery sharing, stored in the `Gallery.shareToken` column.
Reason: Share tokens are simple, stateless, and don't require authentication. They're easy to share via URL. Using `generateRandomToken()` from `crypto.randomBytes(32)` ensures tokens are not guessable.
Consequences: Anyone with the token can view the gallery (even non-users). Revoking changes/invalidates the token. Tokens in URLs may leak via browser history or referrer headers.
Alternatives Considered: JWT-based sharing (overkill, too large for URLs), password-protected sharing (more complex, not requested), access codes (shorter but less secure).
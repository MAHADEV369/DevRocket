# Skill 07: File Upload & Storage

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 2-3 hours
Depends On: 01, 02

## Input Contract
- Skill 01 scaffold complete
- Skill 02 database schema with `Image` model (has `url`, `thumbnailUrl` fields)
- S3-compatible storage endpoint available (AWS S3, Cloudflare R2, or MinIO)
- Storage credentials configured in `.env`

## Output Contract
- `src/services/storage.service.ts` - StorageService class with upload, download, delete, and signed URL methods
- `src/modules/storage/routes.ts` - Express router with upload endpoint
- `src/modules/storage/controller.ts` - Upload controller
- `src/modules/storage/validation.ts` - Upload validation schemas
- Multer integration for multipart file handling
- Supports S3-compatible storage (AWS S3, Cloudflare R2, MinIO)

## Files to Create

| File | Description |
|------|-------------|
| `src/services/storage.service.ts` | S3-compatible storage service |
| `src/modules/storage/validation.ts` | Upload validation schemas |
| `src/modules/storage/controller.ts` | Upload controller |
| `src/modules/storage/routes.ts` | Express router for upload endpoints |

## Steps

### 1. Install dependencies

```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner multer sharp uuid
npm install -D @types/multer @types/sharp
```

### 2. Update `.env.example` with storage config

Add these if not already present:

```env
# Storage
S3_ENDPOINT=https://s3.amazonaws.com
S3_REGION=us-east-1
S3_BUCKET=fastdepo-uploads
S3_ACCESS_KEY=your-access-key
S3_SECRET_KEY=your-secret-key
S3_PUBLIC_URL=https://fastdepo-uploads.s3.amazonaws.com
```

### 3. Update `src/config/index.ts` to include public URL

Add to the `storage` config object:

```typescript
storage: {
  endpoint: process.env.S3_ENDPOINT || '',
  region: process.env.S3_REGION || 'us-east-1',
  bucket: process.env.S3_BUCKET || '',
  accessKey: process.env.S3_ACCESS_KEY || '',
  secretKey: process.env.S3_SECRET_KEY || '',
  publicUrl: process.env.S3_PUBLIC_URL || '',
},
```

### 4. Create `src/services/storage.service.ts`

```typescript
import {
  S3Client,
  PutObjectCommand,
  GetObjectCommand,
  DeleteObjectCommand,
  CopyObjectCommand,
  HeadObjectCommand,
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import sharp from 'sharp';
import { v4 as uuidv4 } from 'uuid';
import { config } from '../config/index.js';
import { logger } from '../utils/logger.js';
import { AppError } from '../utils/errors.js';
import { createReadStream } from 'fs';

interface StorageClientOptions {
  endpoint?: string;
  region?: string;
  credentials?: { accessKeyId: string; secretAccessKey: string };
  forcePathStyle?: boolean;
}

export class StorageService {
  private client: S3Client;
  private bucket: string;
  private publicUrl: string;

  constructor(options?: StorageClientOptions) {
    const endpoint = options?.endpoint || config.storage.endpoint;
    const region = options?.region || config.storage.region;
    const accessKey = options?.credentials?.accessKeyId || config.storage.accessKey;
    const secretKey = options?.credentials?.secretAccessKey || config.storage.secretKey;
    this.bucket = config.storage.bucket;
    this.publicUrl = config.storage.publicUrl;

    this.client = new S3Client({
      endpoint,
      region,
      credentials: {
        accessKeyId: accessKey,
        secretAccessKey: secretKey,
      },
      forcePathStyle: options?.forcePathStyle ?? !!endpoint,
    });

    logger.info({ bucket: this.bucket, region }, 'Storage service initialized');
  }

  async uploadFile(params: {
    file: Buffer | NodeJS.ReadableStream;
    key: string;
    contentType: string;
    metadata?: Record<string, string>;
  }): Promise<string> {
    const { file, key, contentType, metadata } = params;

    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: file,
      ContentType: contentType,
      Metadata: metadata,
    });

    try {
      await this.client.send(command);
      const url = this.buildPublicUrl(key);
      logger.info({ key, contentType }, 'File uploaded');
      return url;
    } catch (error) {
      logger.error({ error, key }, 'Failed to upload file');
      throw new AppError(500, `Failed to upload file: ${(error as Error).message}`);
    }
  }

  async uploadImageWithThumbnail(params: {
    file: Buffer;
    key: string;
    thumbnailSize?: number;
    contentType?: string;
    metadata?: Record<string, string>;
  }): Promise<{ url: string; thumbnailUrl: string; width: number; height: number }> {
    const { file, key, thumbnailSize = 300, metadata } = params;

    try {
      const image = sharp(file);
      const metadata_img = await image.metadata();
      const width = metadata_img.width || 0;
      const height = metadata_img.height || 0;
      const contentType = params.contentType || `image/${metadata_img.format || 'png'}`;

      const fullKey = key;
      const thumbnailKey = `thumbnails/${key}`;

      await this.uploadFile({
        file,
        key: fullKey,
        contentType,
        metadata,
      });

      const thumbnailBuffer = await sharp(file)
        .resize(thumbnailSize, thumbnailSize, { fit: 'inside', withoutEnlargement: true })
        .webp({ quality: 80 })
        .toBuffer();

      await this.uploadFile({
        file: thumbnailBuffer,
        key: thumbnailKey,
        contentType: 'image/webp',
        metadata,
      });

      return {
        url: this.buildPublicUrl(fullKey),
        thumbnailUrl: this.buildPublicUrl(thumbnailKey),
        width,
        height,
      };
    } catch (error) {
      logger.error({ error, key }, 'Failed to upload image with thumbnail');
      throw new AppError(500, `Failed to process image: ${(error as Error).message}`);
    }
  }

  async downloadFile(key: string): Promise<Buffer> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    try {
      const response = await this.client.send(command);
      const body = response.Body;
      if (!body) {
        throw new AppError(404, 'File not found');
      }
      const buffer = await body.transformToByteArray();
      return Buffer.from(buffer);
    } catch (error) {
      if ((error as any).name === 'NoSuchKey') {
        throw new AppError(404, 'File not found');
      }
      logger.error({ error, key }, 'Failed to download file');
      throw new AppError(500, `Failed to download file: ${(error as Error).message}`);
    }
  }

  async deleteFile(key: string): Promise<void> {
    const command = new DeleteObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    try {
      await this.client.send(command);
      logger.info({ key }, 'File deleted');

      const thumbnailKey = `thumbnails/${key}`;
      try {
        await this.client.send(
          new DeleteObjectCommand({ Bucket: this.bucket, Key: thumbnailKey })
        );
      } catch {
        // Thumbnail may not exist, ignore
      }
    } catch (error) {
      logger.error({ error, key }, 'Failed to delete file');
      throw new AppError(500, `Failed to delete file: ${(error as Error).message}`);
    }
  }

  async getSignedUrl(key: string, expiresIn: number = 3600): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    try {
      return await getSignedUrl(this.client, command, { expiresIn });
    } catch (error) {
      logger.error({ error, key }, 'Failed to generate signed URL');
      throw new AppError(500, `Failed to generate signed URL: ${(error as Error).message}`);
    }
  }

  async getUploadSignedUrl(params: {
    key: string;
    contentType: string;
    expiresIn?: number;
  }): Promise<string> {
    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: params.key,
      ContentType: params.contentType,
    });

    try {
      return await getSignedUrl(this.client, command, { expiresIn: params.expiresIn || 3600 });
    } catch (error) {
      logger.error({ error, key: params.key }, 'Failed to generate upload signed URL');
      throw new AppError(500, `Failed to generate upload signed URL: ${(error as Error).message}`);
    }
  }

  async fileExists(key: string): Promise<boolean> {
    const command = new HeadObjectCommand({
      Bucket: this.bucket,
      Key: key,
    });

    try {
      await this.client.send(command);
      return true;
    } catch {
      return false;
    }
  }

  generateKey(prefix: string, filename: string): string {
    const ext = filename.split('.').pop() || 'png';
    const uniqueId = uuidv4();
    const datePath = new Date().toISOString().slice(0, 10).replace(/-/g, '/');
    return `${prefix}/${datePath}/${uniqueId}.${ext}`;
  }

  private buildPublicUrl(key: string): string {
    if (this.publicUrl) {
      return `${this.publicUrl}/${key}`;
    }
    return `https://${this.bucket}.s3.${config.storage.region}.amazonaws.com/${key}`;
  }
}

export const storageService = new StorageService();
```

### 5. Create `src/modules/storage/validation.ts`

```typescript
import { z } from 'zod';

export const uploadImageSchema = z.object({
  purpose: z.enum(['avatar', 'gallery', 'image"]).default('image'),
});

export const presignedUrlSchema = z.object({
  contentType: z.string().regex(/^image\/(jpeg|png|gif|webp|svg\+xml)$/, 'Invalid content type'),
  purpose: z.enum(['avatar', 'gallery', 'image']).default('image'),
  extension: z.string().regex(/^(jpg|jpeg|png|gif|webp|svg)$/, 'Invalid extension').default('png'),
});

export type UploadImageInput = z.infer<typeof uploadImageSchema>;
export type PresignedUrlInput = z.infer<typeof presignedUrlSchema>;

export const ALLOWED_MIME_TYPES = [
  'image/jpeg',
  'image/png',
  'image/gif',
  'image/webp',
  'image/svg+xml',
];

export const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB
```

### 6. Create `src/modules/storage/controller.ts`

```typescript
import type { Response, NextFunction } from 'express';
import type { AuthenticatedRequest } from '../auth/types.js';
import { storageService } from '../../services/storage.service.js';
import { AppError } from '../../utils/errors.js';
import { ALLOWED_MIME_TYPES, MAX_FILE_SIZE } from './validation.js';

export async function uploadImage(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    if (!req.file) {
      throw new AppError(400, 'No file provided');
    }

    if (!ALLOWED_MIME_TYPES.includes(req.file.mimetype)) {
      throw new AppError(400, `Invalid file type. Allowed: ${ALLOWED_MIME_TYPES.join(', ')}`);
    }

    if (req.file.size > MAX_FILE_SIZE) {
      throw new AppError(400, `File too large. Maximum size: ${MAX_FILE_SIZE / 1024 / 1024}MB`);
    }

    const purpose = (req.body.purpose as string) || 'image';
    const key = storageService.generateKey(purpose, req.file.originalname);
    const userId = req.user.userId;

    const result = await storageService.uploadImageWithThumbnail({
      file: req.file.buffer,
      key,
      contentType: req.file.mimetype,
      metadata: { userId },
    });

    res.status(201).json({
      url: result.url,
      thumbnailUrl: result.thumbnailUrl,
      width: result.width,
      height: result.height,
      key,
    });
  } catch (error) {
    next(error);
  }
}

export async function getPresignedUploadUrl(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const { contentType, purpose } = req.body;
    const key = storageService.generateKey(purpose, `upload.${req.body.extension || 'png'}`);

    const uploadUrl = await storageService.getUploadSignedUrl({
      key,
      contentType,
      expiresIn: 3600,
    });

    res.json({
      uploadUrl,
      key,
      publicUrl: storageService.buildPublicUrl ? undefined : undefined,
    });
  } catch (error) {
    next(error);
  }
}

export async function getPresignedDownloadUrl(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const { key } = req.params;
    const downloadUrl = await storageService.getSignedUrl(key, 3600);

    res.json({ downloadUrl, key });
  } catch (error) {
    next(error);
  }
}

export async function deleteFile(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  try {
    const { key } = req.params;
    await storageService.deleteFile(key);

    res.json({ key, deleted: true });
  } catch (error) {
    next(error);
  }
}
```

### 7. Create `src/modules/storage/routes.ts`

```typescript
import { Router } from 'express';
import multer from 'multer';
import * as controller from './controller.js';
import { authenticate } from '../../middleware/auth.js';
import { validate } from '../../middleware/validate.js';
import { presignedUrlSchema } from './validation.js';

const upload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: 10 * 1024 * 1024,
  },
  fileFilter: (_req, file, cb) => {
    const allowed = ['image/jpeg', 'image/png', 'image/gif', 'image/webp', 'image/svg+xml'];
    if (allowed.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error(`Invalid file type: ${file.mimetype}`));
    }
  },
});

const router = Router();

router.post(
  '/upload',
  authenticate,
  upload.single('file'),
  controller.uploadImage
);

router.post(
  '/presigned-upload',
  authenticate,
  validate(presignedUrlSchema),
  controller.getPresignedUploadUrl
);

router.get(
  '/presigned-download/:key',
  authenticate,
  controller.getPresignedDownloadUrl
);

router.delete(
  '/:key',
  authenticate,
  controller.deleteFile
);

export default router;
```

### 8. Register storage routes in `src/app.ts`

Add import and route:

```typescript
import storageRoutes from './modules/storage/routes.js';
// ...
app.use('/api/storage', storageRoutes);
```

Also add `multer` error handling to the error handler in `src/middleware/errorHandler.ts`:

```typescript
import multer from 'multer';

// Add this case in the errorHandler before the final fallback:
if (err instanceof multer.MulterError) {
  const message = err.code === 'LIMIT_FILE_SIZE'
    ? 'File too large. Maximum size: 10MB'
    : `Upload error: ${err.message}`;
  return res.status(400).json({
    status: 'error',
    message,
    requestId: (req as any).id || 'unknown',
  });
}
```

### 9. Verify

```bash
npm run typecheck && npm run dev
```

## Verification

```bash
npm run typecheck
# Expected: no errors

# Test upload (requires S3-compatible storage configured)
TOKEN=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com","password":"Password123!"}' | jq -r '.accessToken')

# Upload an image
curl -X POST http://localhost:3000/api/storage/upload \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@test-image.png" \
  -F "purpose=gallery"
# Expected: 201 with url, thumbnailUrl, width, height, key

# Get presigned upload URL
curl -X POST http://localhost:3000/api/storage/presigned-upload \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"contentType":"image/png","purpose":"image"}'
# Expected: 200 with uploadUrl, key

# Get presigned download URL
curl http://localhost:3000/api/storage/presigned-download/<KEY> \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 with downloadUrl
```

## Rollback

```bash
rm src/services/storage.service.ts
rm -rf src/modules/storage
# Remove storage routes import and registration from src/app.ts
# Remove multer error handling from errorHandler.ts
# Uninstall deps: npm uninstall @aws-sdk/client-s3 @aws-sdk/s3-request-presigner multer sharp
```

## ADR-007: S3-Compatible Abstracted Storage

Decision: Use AWS S3 SDK (`@aws-sdk/client-s3`) with an abstraction layer, supporting any S3-compatible provider (AWS S3, Cloudflare R2, MinIO).
Reason: S3 is the de facto standard for object storage. Using the AWS SDK gives us type-safe access to all S3 operations while remaining provider-agnostic. Cloudflare R2 offers zero egress fees which is cost-effective for an image-heavy app. The `StorageService` class abstracts away provider-specific details.
Consequences: Requires S3-compatible infrastructure. `sharp` adds a native dependency for image resizing (requires build tools on deployment). Presigned URLs enable direct browser-to-S3 uploads for larger files. The `forcePathStyle` flag toggles between AWS S3 (virtual-hosted style) and self-hosted (path style).
Alternatives Considered: Local filesystem storage (doesn't scale, no CDN), Firebase Storage (vendor lock-in), Azure Blob Storage (different SDK, less common for this use case), direct Cloudflare R2 SDK (less portable).
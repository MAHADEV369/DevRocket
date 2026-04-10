# Skill 04: Middleware Layer

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 2-3 hours
Depends On: 01, 03

## Input Contract
- Skill 01 scaffold complete
- Skill 03 auth module complete (`src/utils/errors.ts`, `src/middleware/validate.ts` exist)
- `src/config/index.ts` and `src/utils/logger.ts` available

## Output Contract
- `src/middleware/rateLimiter.ts` - Configurable rate limiting per route
- `src/middleware/validate.ts` - Updated with full Zod validation (body, params, query)
- `src/middleware/errorHandler.ts` - Global error handler with AppError support
- `src/middleware/requestId.ts` - UUID per request attached to `req.id`
- `src/middleware/requestLogger.ts` - Logs method, url, status, duration
- `src/middleware/cors.ts` - Configurable CORS middleware
- `src/utils/errors.ts` - Updated with full AppError hierarchy
- All middleware wired into `src/app.ts`

## Files to Create

| File | Description |
|------|-------------|
| `src/middleware/rateLimiter.ts` | Rate limiting with configurable windows |
| `src/middleware/validate.ts` | Zod validation for body, params, query |
| `src/middleware/errorHandler.ts` | Global error handler |
| `src/middleware/requestId.ts` | UUID request ID middleware |
| `src/middleware/requestLogger.ts` | Request/response logging |
| `src/middleware/cors.ts` | CORS configuration |
| `src/utils/errors.ts` | AppError hierarchy (overwrite) |

## Steps

### 1. Install rate-limit dependency

```bash
npm install express-rate-limit
```

### 2. Update `src/utils/errors.ts`

Overwrite with the full error hierarchy:

```typescript
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly isOperational: boolean;
  public readonly details?: unknown;

  constructor(statusCode: number, message: string, isOperational = true, details?: unknown) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    this.details = details;
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(404, `${resource} not found`);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(403, message);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(401, message);
  }
}

export class ValidationError extends AppError {
  constructor(message: string, details?: unknown) {
    super(400, message, true, details);
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(409, message);
  }
}

export class TooManyRequestsError extends AppError {
  constructor(message = 'Too many requests, please try again later') {
    super(429, message);
  }
}

export class InternalError extends AppError {
  constructor(message = 'Internal server error') {
    super(500, message, false);
  }
}
```

### 3. Create `src/middleware/rateLimiter.ts`

```typescript
import rateLimit from 'express-rate-limit';
import type { Request, Response, NextFunction } from 'express';
import { TooManyRequestsError } from '../utils/errors.js';

interface RateLimitOptions {
  windowMs?: number;
  max?: number;
  message?: string;
  keyGenerator?: (req: Request) => string;
}

function createRateLimiter(options: RateLimitOptions = {}) {
  const {
    windowMs = 15 * 60 * 1000,
    max = 100,
    message = 'Too many requests, please try again later',
    keyGenerator,
  } = options;

  return rateLimit({
    windowMs,
    max,
    handler: (_req: Request, _res: Response, next: NextFunction) => {
      next(new TooManyRequestsError(message));
    },
    standardHeaders: true,
    legacyHeaders: false,
    keyGenerator: keyGenerator || ((req: Request) => {
      return req.ip || 'unknown';
    }),
  });
}

export const globalLimiter = createRateLimiter({
  windowMs: 15 * 60 * 1000,
  max: 100,
});

export const authLimiter = createRateLimiter({
  windowMs: 15 * 60 * 1000,
  max: 10,
  message: 'Too many auth attempts, please try again after 15 minutes',
});

export const imageGenerationLimiter = createRateLimiter({
  windowMs: 60 * 1000,
  max: 5,
  message: 'Too many image generation requests, please try again after 1 minute',
});

export const passwordResetLimiter = createRateLimiter({
  windowMs: 60 * 60 * 1000,
  max: 3,
  message: 'Too many password reset requests, please try again after 1 hour',
});
```

### 4. Update `src/middleware/validate.ts`

Replace existing with full validation support:

```typescript
import type { NextFunction, Request, Response } from 'express';
import type { ZodSchema, ZodObject, ZodTypeDef } from 'zod';
import { ValidationError } from '../utils/errors.js';

interface ValidationSchemas {
  body?: ZodSchema;
  params?: ZodSchema;
  query?: ZodSchema;
}

export function validate(schemas: ValidationSchemas | ZodSchema) {
  return (req: Request, _res: Response, next: NextFunction) => {
    try {
      if (schemas instanceof Object && 'safeParse' in schemas) {
        const result = (schemas as ZodSchema).safeParse(req.body);
        if (!result.success) {
          const errors = result.error.errors.map(
            (e) => `${e.path.join('.')}: ${e.message}`
          );
          return next(new ValidationError(errors.join(', ')));
        }
        req.body = result.data;
        return next();
      }

      const schemasObj = schemas as ValidationSchemas;
      const errors: string[] = [];

      if (schemasObj.body) {
        const result = schemasObj.body.safeParse(req.body);
        if (!result.success) {
          errors.push(...result.error.errors.map((e) => `body.${e.path.join('.')}: ${e.message}`));
        } else {
          req.body = result.data;
        }
      }

      if (schemasObj.params) {
        const result = schemasObj.params.safeParse(req.params);
        if (!result.success) {
          errors.push(...result.error.errors.map((e) => `params.${e.path.join('.')}: ${e.message}`));
        } else {
          req.params = result.data as Record<string, string>;
        }
      }

      if (schemasObj.query) {
        const result = schemasObj.query.safeParse(req.query);
        if (!result.success) {
          errors.push(...result.error.errors.map((e) => `query.${e.path.join('.')}: ${e.message}`));
        } else {
          req.query = result.data as Record<string, unknown>;
        }
      }

      if (errors.length > 0) {
        return next(new ValidationError(errors.join('; ')));
      }

      next();
    } catch (error) {
      next(error);
    }
  };
}
```

### 5. Create `src/middleware/errorHandler.ts`

```typescript
import type { NextFunction, Request, Response } from 'express';
import { logger } from '../utils/logger.js';
import { AppError } from '../utils/errors.js';

export function errorHandler(err: Error, req: Request, res: Response, _next: NextFunction) {
  const requestId = (req as Request & { id?: string }).id || 'unknown';

  if (err instanceof AppError) {
    const response: Record<string, unknown> = {
      status: 'error',
      message: err.message,
      requestId,
    };

    if (err.details) {
      response.details = err.details;
    }

    if (!err.isOperational) {
      logger.error({ err, requestId }, 'Unexpected operational error');
    }

    return res.status(err.statusCode).json(response);
  }

  logger.error({ err, requestId }, 'Unhandled error');

  res.status(500).json({
    status: 'error',
    message: 'Internal server error',
    requestId,
  });
}
```

### 6. Create `src/middleware/requestId.ts`

```typescript
import type { NextFunction, Request, Response } from 'express';
import { v4 as uuidv4 } from 'uuid';

declare global {
  namespace Express {
    interface Request {
      id?: string;
    }
  }
}

export function requestId(req: Request, res: Response, next: NextFunction) {
  req.id = req.headers['x-request-id'] as string || uuidv4();
  res.setHeader('X-Request-Id', req.id);
  next();
}
```

### 7. Create `src/middleware/requestLogger.ts`

```typescript
import type { NextFunction, Request, Response } from 'express';
import { logger } from '../utils/logger.js';

export function requestLogger(req: Request, res: Response, next: NextFunction) {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    const level = res.statusCode >= 500 ? 'error' : res.statusCode >= 400 ? 'warn' : 'info';

    logger.log(level, {
      method: req.method,
      url: req.originalUrl,
      status: res.statusCode,
      duration: `${duration}ms`,
      requestId: req.id,
      ip: req.ip,
      userAgent: req.headers['user-agent'],
    });
  });

  next();
}
```

### 8. Create `src/middleware/cors.ts`

```typescript
import cors from 'cors';
import { config } from '../config/index.js';

const allowedOrigins = config.corsOrigin.split(',').map((o) => o.trim());

export const corsMiddleware = cors({
  origin(origin, callback) {
    if (config.nodeEnv === 'development' || !origin) {
      return callback(null, true);
    }

    if (allowedOrigins.includes(origin) || allowedOrigins.includes('*')) {
      return callback(null, true);
    }

    callback(new Error(`CORS origin ${origin} not allowed`));
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-Id'],
  exposedHeaders: ['X-Request-Id'],
  maxAge: 86400,
});
```

### 9. Update `src/app.ts` to wire all middleware

Replace `src/app.ts` with:

```typescript
import express from 'express';
import helmet from 'helmet';
import compression from 'compression';
import cookieParser from 'cookie-parser';
import { config } from './config/index.js';
import { requestId } from './middleware/requestId.js';
import { requestLogger } from './middleware/requestLogger.js';
import { corsMiddleware } from './middleware/cors.js';
import { globalLimiter } from './middleware/rateLimiter.js';
import { errorHandler } from './middleware/errorHandler.js';
import authRoutes from './modules/auth/routes.js';

export function createApp() {
  const app = express();

  app.use(requestId);
  app.use(requestLogger);
  app.use(helmet());
  app.use(corsMiddleware);
  app.use(compression());
  app.use(cookieParser());
  app.use(express.json({ limit: '10mb' }));
  app.use(express.urlencoded({ extended: true }));
  app.use(globalLimiter);

  app.get('/health', (_req, res) => {
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
  });

  app.use('/api/auth', authRoutes);

  // TODO: Register image and gallery routes in later skills

  app.use((_req, res) => {
    res.status(404).json({ status: 'error', message: 'Not found' });
  });

  app.use(errorHandler);

  return app;
}
```

### 10. Verify compilation and startup

```bash
npm run typecheck && npm run dev
```

## Verification

```bash
npm run typecheck
# Expected: no TypeScript errors

npm run dev &
sleep 2

curl http://localhost:3000/health
# Expected: {"status":"ok","timestamp":"..."} with X-Request-Id header

curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":""}'
# Expected: 400 validation error with request ID

# Rate limiting test - send 100+ requests rapidly
for i in $(seq 1 110); do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/health; done
# Expected: 429 Too Many Requests after limit exceeded
```

## Rollback

```bash
rm src/middleware/rateLimiter.ts src/middleware/validate.ts src/middleware/errorHandler.ts src/middleware/requestId.ts src/middleware/requestLogger.ts src/middleware/cors.ts
# Restore src/utils/errors.ts to simpler version
# Restore src/app.ts to original version from Skill 01
```

## ADR-004: Middleware Pipeline Architecture

Decision: Use a layered middleware pipeline: requestId -> requestLogger -> helmet -> cors -> compression -> body parsing -> rate limiting -> routes -> 404 -> errorHandler.
Reason: EarlyRequestId ensures all logs have a correlation ID. Helmet/CORS before business logic. Rate limiting before heavy route processing. Centralized error handler catches everything.
Consequences: Order matters - changing middleware order can break behavior. Rate limiting applies globally unless explicitly scoped. All errors follow the same response format.
Alternatives Considered: Per-route middleware composition (harder to maintain), separate gateway for rate limiting (extra infrastructure), no centralized error handling (inconsistent responses).
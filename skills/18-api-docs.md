# Skill 18: API Documentation with Swagger/OpenAPI

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 1-2 hours
Depends On: 01, 05

## Input Contract
- Express application with route files in `src/routes/`
- Authentication middleware using JWT Bearer tokens
- Existing route handlers for: auth (signup, login), images (list, upload, generate), health
- TypeScript project with `tsconfig.json`

## Output Contract
- Swagger UI accessible at `/api/docs`
- OpenAPI 3.0 specification auto-generated from JSDoc comments
- Bearer authentication security definition
- All endpoints documented with request/response schemas
- Interactive API explorer available in the browser

## Files to Create
- `src/config/swagger.ts` — Swagger configuration and specification builder
- Updated route files with JSDoc annotations (auth, images, health)

## Steps

### Step 1: Install Dependencies

```bash
npm install swagger-ui-express swagger-jsdoc
npm install -D @types/swagger-ui-express
```

### Step 2: Create `src/config/swagger.ts`

```typescript
import swaggerJsdoc from 'swagger-jsdoc';
import { SwaggerUiOptions } from 'swagger-ui-express';

const options: swaggerJsdoc.Options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'Image Processing API',
      version: '1.0.0',
      description: 'REST API for image upload, processing, and AI-powered generation. Provides user authentication, image management, and background processing capabilities.',
      contact: {
        name: 'API Support',
        email: 'support@example.com',
      },
      license: {
        name: 'MIT',
        url: 'https://opensource.org/licenses/MIT',
      },
    },
    servers: [
      {
        url: 'http://localhost:3000',
        description: 'Local development server',
      },
      {
        url: 'https://api.example.com',
        description: 'Production server',
      },
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
          description: 'Enter your JWT token in the format: Bearer <token>',
        },
      },
      schemas: {
        User: {
          type: 'object',
          required: ['id', 'email', 'createdAt'],
          properties: {
            id: {
              type: 'string',
              format: 'uuid',
              description: 'Unique user identifier',
            },
            email: {
              type: 'string',
              format: 'email',
              description: 'User email address',
            },
            createdAt: {
              type: 'string',
              format: 'date-time',
              description: 'Account creation timestamp',
            },
          },
        },
        AuthSignupRequest: {
          type: 'object',
          required: ['email', 'password'],
          properties: {
            email: {
              type: 'string',
              format: 'email',
              example: 'user@example.com',
            },
            password: {
              type: 'string',
              minLength: 8,
              description: 'Password must be at least 8 characters',
              example: 'StrongP@ss1',
            },
          },
        },
        AuthLoginRequest: {
          type: 'object',
          required: ['email', 'password'],
          properties: {
            email: {
              type: 'string',
              format: 'email',
              example: 'user@example.com',
            },
            password: {
              type: 'string',
              example: 'StrongP@ss1',
            },
          },
        },
        AuthResponse: {
          type: 'object',
          properties: {
            token: {
              type: 'string',
              description: 'JWT access token',
            },
            user: {
              $ref: '#/components/schemas/User',
            },
          },
        },
        Error: {
          type: 'object',
          required: ['message', 'statusCode'],
          properties: {
            message: {
              type: 'string',
              description: 'Human-readable error message',
            },
            statusCode: {
              type: 'integer',
              description: 'HTTP status code',
            },
            errors: {
              type: 'array',
              items: {
                type: 'object',
                properties: {
                  field: { type: 'string' },
                  message: { type: 'string' },
                },
              },
              description: 'Validation errors detail',
            },
          },
        },
        Image: {
          type: 'object',
          required: ['id', 'userId', 'originalName', 'status', 'createdAt'],
          properties: {
            id: {
              type: 'string',
              format: 'uuid',
              description: 'Unique image identifier',
            },
            userId: {
              type: 'string',
              format: 'uuid',
              description: 'Owner user ID',
            },
            originalName: {
              type: 'string',
              description: 'Original file name',
            },
            url: {
              type: 'string',
              format: 'uri',
              description: 'URL to access the image',
            },
            status: {
              type: 'string',
              enum: ['pending', 'processing', 'completed', 'failed'],
              description: 'Processing status of the image',
            },
            width: {
              type: 'integer',
              nullable: true,
              description: 'Image width in pixels',
            },
            height: {
              type: 'integer',
              nullable: true,
              description: 'Image height in pixels',
            },
            size: {
              type: 'integer',
              description: 'File size in bytes',
            },
            createdAt: {
              type: 'string',
              format: 'date-time',
            },
            updatedAt: {
              type: 'string',
              format: 'date-time',
            },
          },
        },
        ImageGenerateRequest: {
          type: 'object',
          required: ['prompt'],
          properties: {
            prompt: {
              type: 'string',
              description: 'Text description for AI image generation',
              example: 'A serene mountain landscape at sunset',
            },
            width: {
              type: 'integer',
              default: 512,
              description: 'Desired image width in pixels',
              example: 512,
            },
            height: {
              type: 'integer',
              default: 512,
              description: 'Desired image height in pixels',
              example: 512,
            },
          },
        },
        PaginatedImages: {
          type: 'object',
          properties: {
            data: {
              type: 'array',
              items: {
                $ref: '#/components/schemas/Image',
              },
            },
            pagination: {
              type: 'object',
              properties: {
                page: { type: 'integer', example: 1 },
                limit: { type: 'integer', example: 20 },
                total: { type: 'integer', example: 100 },
                totalPages: { type: 'integer', example: 5 },
              },
            },
          },
        },
        HealthResponse: {
          type: 'object',
          properties: {
            status: {
              type: 'string',
              enum: ['ok', 'degraded', 'down'],
            },
            timestamp: {
              type: 'string',
              format: 'date-time',
            },
            uptime: {
              type: 'number',
              description: 'Process uptime in seconds',
            },
          },
        },
      },
    },
    tags: [
      {
        name: 'Auth',
        description: 'Authentication endpoints — signup, login, token refresh',
      },
      {
        name: 'Images',
        description: 'Image management — upload, list, retrieve, generate, delete',
      },
      {
        name: 'Health',
        description: 'Service health and readiness checks',
      },
    ],
  },
  apis: [
    './src/routes/*.ts',
    './src/routes/**/*.ts',
  ],
};

const swaggerSpec = swaggerJsdoc(options);

const swaggerUiOptions: SwaggerUiOptions = {
  customCss: '.swagger-ui .topbar { display: none }',
  customCssUrl: undefined,
  customJs: undefined,
  customFavIcon: undefined,
  swaggerOptions: {
    persistAuthorization: true,
    displayRequestDuration: true,
    filter: true,
    tryItOutEnabled: true,
  },
};

export { swaggerSpec, swaggerUiOptions };
```

### Step 3: Add Swagger JSDoc Annotations to Auth Routes

Open `src/routes/auth.ts` and add JSDoc comments above each route handler. Use the following pattern for every endpoint:

```typescript
/**
 * @swagger
 * /api/v1/auth/signup:
 *   post:
 *     summary: Register a new user
 *     description: Creates a new user account and returns a JWT token.
 *     tags: [Auth]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/AuthSignupRequest'
 *     responses:
 *       201:
 *         description: User created successfully
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/AuthResponse'
 *       400:
 *         description: Validation error or duplicate email
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 *       500:
 *         description: Internal server error
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */
router.post('/signup', validate(signupSchema), signupHandler);

/**
 * @swagger
 * /api/v1/auth/login:
 *   post:
 *     summary: Authenticate a user
 *     description: Validates credentials and returns a JWT token.
 *     tags: [Auth]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/AuthLoginRequest'
 *     responses:
 *       200:
 *         description: Authentication successful
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/AuthResponse'
 *       401:
 *         description: Invalid credentials
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 *       500:
 *         description: Internal server error
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */
router.post('/login', validate(loginSchema), loginHandler);
```

### Step 4: Add Swagger JSDoc Annotations to Image Routes

Open `src/routes/images.ts` and add annotations:

```typescript
/**
 * @swagger
 * /api/v1/images:
 *   get:
 *     summary: List user images
 *     description: Returns a paginated list of images owned by the authenticated user.
 *     tags: [Images]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *           minimum: 1
 *         description: Page number for pagination
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *           default: 20
 *           minimum: 1
 *           maximum: 100
 *         description: Number of items per page
 *       - in: query
 *         name: status
 *         schema:
 *           type: string
 *           enum: [pending, processing, completed, failed]
 *         description: Filter by processing status
 *     responses:
 *       200:
 *         description: Paginated list of images
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/PaginatedImages'
 *       401:
 *         description: Unauthorized — missing or invalid token
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 *       500:
 *         description: Internal server error
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */
router.get('/', authMiddleware, listImagesHandler);

/**
 * @swagger
 * /api/v1/images/upload:
 *   post:
 *     summary: Upload an image
 *     description: Uploads an image file for the authenticated user. Supports JPEG, PNG, GIF, and WebP up to 20MB.
 *     tags: [Images]
 *     security:
 *       - bearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         multipart/form-data:
 *           schema:
 *             type: object
 *             required:
 *               - file
 *             properties:
 *               file:
 *                 type: string
 *                 format: binary
 *                 description: Image file (JPEG, PNG, GIF, WebP)
 *     responses:
 *       201:
 *         description: Image uploaded successfully
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Image'
 *       400:
 *         description: Invalid file or missing file
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 *       401:
 *         description: Unauthorized
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 *       413:
 *         description: File exceeds 20MB size limit
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */
router.post('/upload', authMiddleware, uploadMiddleware, uploadHandler);

/**
 * @swagger
 * /api/v1/images/generate:
 *   post:
 *     summary: Generate an AI image
 *     description: Generates an image from a text prompt using AI.
 *     tags: [Images]
 *     security:
 *       - bearerAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/ImageGenerateRequest'
 *     responses:
 *       202:
 *         description: Image generation started — returns the image record with pending status
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Image'
 *       400:
 *         description: Invalid prompt or parameters
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 *       401:
 *         description: Unauthorized
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */
router.post('/generate', authMiddleware, generateHandler);

/**
 * @swagger
 * /api/v1/images/{id}:
 *   get:
 *     summary: Get image by ID
 *     description: Returns a single image owned by the authenticated user.
 *     tags: [Images]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: string
 *           format: uuid
 *         description: Image ID
 *     responses:
 *       200:
 *         description: Image details
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Image'
 *       401:
 *         description: Unauthorized
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 *       404:
 *         description: Image not found
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */
router.get('/:id', authMiddleware, getImageHandler);

/**
 * @swagger
 * /api/v1/images/{id}:
 *   delete:
 *     summary: Delete an image
 *     description: Deletes an image owned by the authenticated user.
 *     tags: [Images]
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: string
 *           format: uuid
 *         description: Image ID
 *     responses:
 *       204:
 *         description: Image deleted successfully
 *       401:
 *         description: Unauthorized
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 *       404:
 *         description: Image not found
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Error'
 */
router.delete('/:id', authMiddleware, deleteImageHandler);
```

### Step 5: Add Swagger JSDoc Annotations to Health Route

Open `src/routes/health.ts` and add:

```typescript
/**
 * @swagger
 * /api/v1/health:
 *   get:
 *     summary: Health check
 *     description: Returns the health status of the API server.
 *     tags: [Health]
 *     responses:
 *       200:
 *         description: API is healthy
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/HealthResponse'
 */
router.get('/health', healthHandler);
```

### Step 6: Mount Swagger UI in `src/server.ts`

```typescript
import express from 'express';
import swaggerUi from 'swagger-ui-express';
import { swaggerSpec, swaggerUiOptions } from './config/swagger';

// ... existing imports ...

const app = express();

// Swagger UI — must be mounted before auth middleware
app.use('/api/docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec, swaggerUiOptions));

// Alternative: also serve the raw spec as JSON
app.get('/api/docs.json', (_req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.send(swaggerSpec);
});

// ... rest of middleware and routes ...
```

### Step 7: Configure TypeScript to Support JSDoc

Ensure `tsconfig.json` does not strip JSDoc. The existing config should work, but verify:

```json
{
  "compilerOptions": {
    "removeComments": false
  }
}
```

If `removeComments` is `true`, change it to `false` — otherwise `swagger-jsdoc` cannot read the JSDoc annotations from compiled files. If `removeComments` is needed for production builds, ensure `swagger-jsdoc` reads from source files (the `apis` glob points to `./src/routes/*.ts`).

## Verification

1. **Install and build:**
   ```bash
   npm install && npm run build
   ```

2. **Start the server:**
   ```bash
   npm start
   ```

3. **Access Swagger UI:**
   Open browser to `http://localhost:3000/api/docs`
   Expected: Interactive Swagger UI loads with all endpoints organized by tags (Auth, Images, Health).

4. **Verify raw spec:**
   ```bash
   curl http://localhost:3000/api/docs.json | python3 -m json.tool
   ```
   Expected: Valid OpenAPI 3.0 JSON with `paths`, `components`, `securitySchemes`, and `tags`.

5. **Test "Authorize" button:**
   - Click the "Authorize" button in Swagger UI
   - Enter a JWT token: `Bearer <your-token>`
   - Expected: Token is saved and sent with subsequent requests.

6. **Test an endpoint via Swagger UI:**
   - Expand the `POST /api/v1/auth/signup` endpoint
   - Click "Try it out", fill in email and password
   - Click "Execute"
   - Expected: 201 response with token, or 400/409 for validation/duplicate errors.

7. **Lint check:**
   ```bash
   npm run lint
   ```
   Expected: No linting errors in new files.

## Rollback

1. Uninstall swagger packages:
   ```bash
   npm uninstall swagger-ui-express swagger-jsdoc @types/swagger-ui-express
   ```

2. Remove swagger configuration:
   ```bash
   rm src/config/swagger.ts
   ```

3. Remove Swagger UI mount from `src/server.ts`:
   ```typescript
   // Remove these lines:
   // import swaggerUi from 'swagger-ui-express';
   // import { swaggerSpec, swaggerUiOptions } from './config/swagger';
   // app.use('/api/docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec, swaggerUiOptions));
   // app.get('/api/docs.json', ...);
   ```

4. Remove JSDoc annotations from route files (the `/** @swagger ... */` blocks above each handler).

5. Verify server starts cleanly after rollback:
   ```bash
   npm run build && npm start
   ```

## ADR-018: swagger-jsdoc with JSDoc Annotations

**Decision:** Use `swagger-jsdoc` to generate OpenAPI specifications from JSDoc comments embedded in route files, served via `swagger-ui-express`.

**Reason:** Keeping API documentation co-located with the code it describes increases the likelihood that developers will keep docs updated when changing endpoints. JSDoc annotations are visible during code review, unlike separate YAML files that are easy to overlook. `swagger-jsdoc` generates the spec at runtime from source files, so no extra build step is needed.

**Consequences:**
- Route files become more verbose with JSDoc blocks
- Spec is generated at runtime — minimal performance impact (spec is computed once and cached)
- Changes to routes require updating the JSDoc — this is a feature, not a bug
- `removeComments: true` in tsconfig would break spec generation — must be `false` or read from `.ts` source

**Alternatives Considered:**
- **Separate openapi.yaml file:** Clean route files but docs drift from code easily
- **tsoa decorators:** Type-safe but requires decorator support and changes the route definition pattern
- **NestJS Swagger:** Only applicable if using NestJS framework
- **Postman documentation:** External tool, not kept in sync with code changes
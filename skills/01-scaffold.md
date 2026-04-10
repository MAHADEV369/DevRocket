# Skill 01: Project Scaffold

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 1-2 hours
Depends On: None

## Input Contract
- Node.js >=18 installed
- npm >=9 installed
- Empty project directory exists

## Output Contract
- Express + TypeScript project with full folder structure
- `package.json` with all dependencies and scripts
- `tsconfig.json` configured for strict TypeScript
- `.env.example` with all required environment variables
- `.gitignore` for Node.js / TypeScript / Prisma projects
- `src/index.ts` entry point and `src/app.ts` Express app skeleton
- Project compiles and starts without errors

## Files to Create

| File | Description |
|------|-------------|
| `package.json` | Project manifest with deps and scripts |
| `tsconfig.json` | Strict TypeScript configuration |
| `.env.example` | Template for required environment variables |
| `.gitignore` | ignore patterns for Node/TS/Prisma |
| `src/index.ts` | Entry point: loads env, starts server |
| `src/app.ts` | Express app factory with middleware pipeline |
| `src/config/index.ts` | Typed config object from env vars |
| `src/utils/logger.ts` | Pino logger instance |

## Steps

### 1. Initialize npm project

```bash
npm init -y
```

### 2. Install production dependencies

```bash
npm install express cors helmet morgan compression cookie-parser dotenv pino pino-pretty uuid
```

### 3. Install dev dependencies

```bash
npm install -D typescript @types/express @types/cors @types/morgan @types/compression @types/cookie-parser @types/uuid ts-node tsx nodemon prisma @prisma/client
```

### 4. Create folder structure

```bash
mkdir -p src/{config,modules/{auth,images,gallery},middleware,services,utils,types,routes}
mkdir -p prisma
```

### 5. Create `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 6. Create `package.json` scripts section

Replace the default `"scripts"` block:

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "lint": "eslint src/",
    "typecheck": "tsc --noEmit",
    "db:migrate": "prisma migrate dev",
    "db:push": "prisma db push",
    "db:seed": "tsx prisma/seed.ts",
    "db:studio": "prisma studio",
    "db:generate": "prisma generate"
  }
}
```

### 7. Create `.env.example`

```env
# Server
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/fastdepo

# Auth
JWT_SECRET=change-me-to-a-64-char-random-string
JWT_EXPIRES_IN=15m
JWT_REFRESH_SECRET=change-me-to-another-64-char-random-string
JWT_REFRESH_EXPIRES_IN=7d

# Email
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
EMAIL_FROM=noreply@fastdepo.com

# Storage
S3_ENDPOINT=
S3_REGION=us-east-1
S3_BUCKET=fastdepo-uploads
S3_ACCESS_KEY=
S3_SECRET_KEY=

# CORS
CORS_ORIGIN=http://localhost:3001
```

### 8. Create `.gitignore`

```gitignore
node_modules/
dist/
.env
*.log
.DS_Store
prisma/migrations/
!prisma/migrations/.gitkeep
coverage/
```

### 9. Create `src/config/index.ts`

```typescript
import dotenv from 'dotenv';
dotenv.config();

export const config = {
  port: parseInt(process.env.PORT || '3000', 10),
  nodeEnv: process.env.NODE_ENV || 'development',
  databaseUrl: process.env.DATABASE_URL || '',
  jwt: {
    secret: process.env.JWT_SECRET || '',
    expiresIn: process.env.JWT_EXPIRES_IN || '15m',
    refreshSecret: process.env.JWT_REFRESH_SECRET || '',
    refreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN || '7d',
  },
  email: {
    host: process.env.SMTP_HOST || '',
    port: parseInt(process.env.SMTP_PORT || '587', 10),
    user: process.env.SMTP_USER || '',
    pass: process.env.SMTP_PASS || '',
    from: process.env.EMAIL_FROM || '',
  },
  storage: {
    endpoint: process.env.S3_ENDPOINT || '',
    region: process.env.S3_REGION || 'us-east-1',
    bucket: process.env.S3_BUCKET || '',
    accessKey: process.env.S3_ACCESS_KEY || '',
    secretKey: process.env.S3_SECRET_KEY || '',
  },
  corsOrigin: process.env.CORS_ORIGIN || '*',
} as const;

function validateConfig() {
  const required: (keyof typeof config)[] = [];
  if (config.nodeEnv === 'production') {
    required.push('databaseUrl');
  }
  for (const key of required) {
    if (!config[key]) {
      throw new Error(`Missing required config: ${key}`);
    }
  }
}

validateConfig();
```

### 10. Create `src/utils/logger.ts`

```typescript
import pino from 'pino';
import { config } from '../config/index.js';

const transport =
  config.nodeEnv === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined;

export const logger = pino({
  level: config.nodeEnv === 'production' ? 'info' : 'debug',
  transport: transport ? { transports: [transport] } : undefined,
});
```

### 11. Create `src/app.ts`

```typescript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import compression from 'compression';
import cookieParser from 'cookie-parser';
import { config } from './config/index.js';
import { logger } from './utils/logger.js';

export function createApp() {
  const app = express();

  app.use(helmet());
  app.use(cors({ origin: config.corsOrigin, credentials: true }));
  app.use(compression());
  app.use(cookieParser());
  app.use(express.json({ limit: '10mb' }));
  app.use(express.urlencoded({ extended: true }));

  if (config.nodeEnv === 'development') {
    app.use(morgan('dev'));
  }

  app.get('/health', (_req, res) => {
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
  });

  // TODO: Register module routes here (auth, images, gallery)

  // 404 handler
  app.use((_req, res) => {
    res.status(404).json({ error: 'Not found' });
  });

  return app;
}
```

### 12. Create `src/index.ts`

```typescript
import { createApp } from './app.js';
import { config } from './config/index.js';
import { logger } from './utils/logger.js';

const app = createApp();

const server = app.listen(config.port, () => {
  logger.info(`Server running on port ${config.port} in ${config.nodeEnv} mode`);
});

function gracefulShutdown(signal: string) {
  logger.info(`${signal} received, shutting down gracefully`);
  server.close(() => {
    logger.info('Server closed');
    process.exit(0);
  });

  setTimeout(() => {
    logger.error('Forced shutdown after timeout');
    process.exit(1);
  }, 10000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

### 13. Install and verify

```bash
npm install && npm run typecheck && npm run dev
```

Confirm the server starts and `GET /health` returns `{ "status": "ok" }`.

## Verification

```bash
npm run typecheck
# Expected: no errors

npm run dev &
# Wait 2 seconds, then:
curl http://localhost:3000/health
# Expected: {"status":"ok","timestamp":"..."}
```

## Rollback

```bash
rm -rf src/ prisma/ dist/
rm package.json package-lock.json tsconfig.json .env.example .gitignore
# Ifcommitted: git rm -r . && git commit -m "Revert scaffold"
```

## ADR-001: Express + TypeScript as Base Framework

Decision: Use Express.js with TypeScript and ESM modules (`"module": "NodeNext"`).
Reason: Express is the most widely-known Node.js framework; team familiarity reduces onboarding time. TypeScript with strict mode catches bugs early. ESM is the future of Node.js modules.
Consequences: Requires `.js` extensions in local imports. Slightly more verbose import syntax. Older libraries may need `esModuleInterop`.
Alternatives Considered: Fastify (faster but less ecosystem), NestJS (too opinionated), Hono (too new for production), CommonJS (deprecated path).
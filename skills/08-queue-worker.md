# Skill 08: Background Jobs with BullMQ + Redis

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 2-3 hours
Depends On: 01, 02

## Input Contract
- Skill 01 (project scaffold) completed with Express + TypeScript setup
- Skill 02 (Prisma + PostgreSQL) completed with database client configured
- Redis server running locally or connection string available
- `@prisma/client` and express dependencies installed
- `.env` file exists with database URL configured

## Output Contract
- `src/services/queue.service.ts` with BullMQ queue definitions for image generation and email sending
- `src/worker.ts` as dedicated worker process entry point
- Job retry logic with exponential backoff
- Concurrency limits per queue
- Docker Compose entry for Redis
- Graceful shutdown handling
- All configs validated via zod in `src/config/env.ts`

## Files to Create
- `src/services/queue.service.ts` - BullMQ queue instances, job types, and helper functions
- `src/worker.ts` - Worker process entry point with processor registrations
- `src/processors/image-generation.processor.ts` - Image generation job processor
- `src/processors/email.processor.ts` - Email sending job processor
- `src/config/redis.ts` - Redis connection configuration
- `docker-compose.yml` - Updated with Redis service (or created if missing)

## Steps

### 1. Install BullMQ and Redis dependencies

```bash
npm install bullmq ioredis
npm install -D @types/ioredis
```

### 2. Add Redis configuration to environment schema

In `src/config/env.ts`, add Redis-related environment variables to the zod schema:

```typescript
REDIS_URL: z.string().default('redis://localhost:6379'),
REDIS_HOST: z.string().default('localhost'),
REDIS_PORT: z.coerce.number().default(6379),
REDIS_PASSWORD: z.string().optional(),
REDIS_DB: z.coerce.number().default(0),
```

### 3. Create Redis connection config

Create `src/config/redis.ts`:

```typescript
import { env } from './env';
import Redis from 'ioredis';

export const redisConnection = new Redis({
  host: env.REDIS_HOST,
  port: env.REDIS_PORT,
  password: env.REDIS_PASSWORD,
  db: env.REDIS_DB,
  maxRetriesPerRequest: null,
  enableReadyCheck: false,
});

redisConnection.on('error', (err) => {
  console.error('Redis connection error:', err);
});

redisConnection.on('connect', () => {
  console.log('Connected to Redis');
});
```

### 4. Create the queue service

Create `src/services/queue.service.ts`:

```typescript
import { Queue, QueueScheduler } from 'bullmq';
import { redisConnection } from '../config/redis';

export const QUEUE_NAMES = {
  IMAGE_GENERATION: 'image-generation',
  EMAIL: 'email',
} as const;

export type QueueName = (typeof QUEUE_NAMES)[keyof typeof QUEUE_NAMES];

export interface ImageGenerationJobData {
  userId: string;
  prompt: string;
  style?: string;
  width?: number;
  height?: number;
  generationId: string;
}

export interface EmailJobData {
  to: string;
  subject: string;
  template: string;
  context: Record<string, unknown>;
}

const defaultQueueOptions = {
  connection: redisConnection,
  defaultJobOptions: {
    removeOnComplete: { count: 1000 },
    removeOnFail: { count: 5000 },
    attempts: 3,
    backoff: {
      type: 'exponential' as const,
      delay: 1000,
    },
  },
};

export const imageGenerationQueue = new Queue<ImageGenerationJobData>(
  QUEUE_NAMES.IMAGE_GENERATION,
  {
    connection: redisConnection,
    defaultJobOptions: {
      ...defaultQueueOptions.defaultJobOptions,
      attempts: 5,
      backoff: { type: 'exponential' as const, delay: 2000 },
    },
  }
);

export const emailQueue = new Queue<EmailJobData>(QUEUE_NAMES.EMAIL, {
  connection: redisConnection,
  defaultJobOptions: {
    ...defaultQueueOptions.defaultJobOptions,
    attempts: 3,
    backoff: { type: 'exponential' as const, delay: 1000 },
  },
});

export async function addImageGenerationJob(data: ImageGenerationJobData) {
  return imageGenerationQueue.add('generate', data, {
    attempts: 5,
    backoff: { type: 'exponential', delay: 2000 },
  });
}

export async function addEmailJob(data: EmailJobData) {
  return emailQueue.add('send', data);
}

export async function getQueueStatus(name: QueueName) {
  const queue = name === QUEUE_NAMES.IMAGE_GENERATION
    ? imageGenerationQueue
    : emailQueue;

  const [waiting, active, completed, failed, delayed] = await Promise.all([
    queue.getWaitingCount(),
    queue.getActiveCount(),
    queue.getCompletedCount(),
    queue.getFailedCount(),
    queue.getDelayedCount(),
  ]);

  return { waiting, active, completed, failed, delayed };
}

export async function closeQueues() {
  await Promise.all([
    imageGenerationQueue.close(),
    emailQueue.close(),
  ]);
}
```

### 5. Create the image generation processor

Create `src/processors/image-generation.processor.ts`:

```typescript
import { Worker, Job } from 'bullmq';
import { redisConnection } from '../config/redis';
import { prisma } from '../config/database';
import { ImageGenerationJobData } from '../services/queue.service';

export function createImageGenerationWorker() {
  const worker = new Worker<ImageGenerationJobData>(
    'image-generation',
    async (job: Job<ImageGenerationJobData>) => {
      const { userId, prompt, style, width, height, generationId } = job.data;

      await job.updateProgress(10);

      const generation = await prisma.generation.update({
        where: { id: generationId },
        data: { status: 'PROCESSING' },
      });

      await job.updateProgress(30);

      // Placeholder: call actual image generation API here
      // const imageUrl = await callImageGenerationAPI({ prompt, style, width, height });

      await job.updateProgress(70);

      // Simulate image URL for now
      const imageUrl = `https://storage.example.com/generations/${generationId}.png`;

      await prisma.generation.update({
        where: { id: generationId },
        data: {
          status: 'COMPLETED',
          resultUrl: imageUrl,
        },
      });

      await job.updateProgress(100);

      return { generationId, imageUrl };
    },
    {
      connection: redisConnection,
      concurrency: 3,
      limiter: {
        max: 5,
        duration: 1000,
      },
    }
  );

  worker.on('completed', (job) => {
    console.log(`Image generation job ${job.id} completed`);
  });

  worker.on('failed', (job, err) => {
    console.error(`Image generation job ${job?.id} failed:`, err.message);
  });

  return worker;
}
```

### 6. Create the email processor

Create `src/processors/email.processor.ts`:

```typescript
import { Worker, Job } from 'bullmq';
import { redisConnection } from '../config/redis';
import { EmailJobData } from '../services/queue.service';

export function createEmailWorker() {
  const worker = new Worker<EmailJobData>(
    'email',
    async (job: Job<EmailJobData>) => {
      const { to, subject, template, context } = job.data;

      // Will be replaced by actual email service in skill 10
      console.log(`Sending email to ${to}: ${subject} using template ${template}`);

      // Placeholder: call email service
      // await emailService.send(to, subject, template, context);

      return { sent: true, to };
    },
    {
      connection: redisConnection,
      concurrency: 5,
      limiter: {
        max: 10,
        duration: 1000,
      },
    }
  );

  worker.on('completed', (job) => {
    console.log(`Email job ${job.id} completed`);
  });

  worker.on('failed', (job, err) => {
    console.error(`Email job ${job?.id} failed:`, err.message);
  });

  return worker;
}
```

### 7. Create the worker entry point

Create `src/worker.ts`:

```typescript
import { createImageGenerationWorker } from './processors/image-generation.processor';
import { createEmailWorker } from './processors/email.processor';
import { closeQueues } from './services/queue.service';

console.log('Starting BullMQ worker process...');

const imageWorker = createImageGenerationWorker();
const emailWorker = createEmailWorker();

const workers = [imageWorker, emailWorker];

async function gracefulShutdown(signal: string) {
  console.log(`Received ${signal}. Shutting down workers...`);

  await Promise.all(workers.map((w) => w.close()));
  await closeQueues();

  console.log('Workers shut down successfully');
  process.exit(0);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

process.on('uncaughtException', (err) => {
  console.error('Uncaught exception in worker:', err);
  gracefulShutdown('uncaughtException');
});

process.on('unhandledRejection', (reason) => {
  console.error('Unhandled rejection in worker:', reason);
  gracefulShutdown('unhandledRejection');
});
```

### 8. Add worker start script to package.json

Add to the `scripts` section of `package.json`:

```json
{
  "scripts": {
    "worker": "tsx src/worker.ts",
    "dev:worker": "tsx watch src/worker.ts",
    "dev:all": "concurrently \"npm run dev\" \"npm run dev:worker\""
  }
}
```

### 9. Update docker-compose.yml with Redis

Add Redis service to `docker-compose.yml`:

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  redis_data:
```

### 10. Update .env.example

Add these entries to `.env.example`:

```
REDIS_URL=redis://localhost:6379
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0
```

## Verification

1. Start Redis: `docker compose up -d redis`
2. Compile the project: `npx tsc --noEmit` (no type errors)
3. Start the worker process: `npm run dev:worker`
4. In a separate terminal, run a test script that adds a job:

```bash
npx tsx -e "
const { addEmailJob, addImageGenerationJob } = require('./src/services/queue.service');
addEmailJob({ to: 'test@test.com', subject: 'Test', template: 'test', context: {} }).then(console.log);
"
```

5. Verify in worker logs that the job was picked up and processed.
6. Check Redis connection: `docker compose exec redis redis-cli ping` should return `PONG`.

Expected output:
- Worker logs show `Starting BullMQ worker process...`
- Job processing logs appear for both queues
- `PONG` response from Redis ping

## Rollback

1. Remove installed packages:
   ```bash
   npm uninstall bullmq ioredis @types/ioredis
   ```
2. Delete created files:
   ```bash
   rm src/services/queue.service.ts src/worker.ts src/processors/image-generation.processor.ts src/processors/email.processor.ts src/config/redis.ts
   ```
3. Remove Redis from `docker-compose.yml` and `.env.example`
4. Remove worker scripts from `package.json`
5. Remove Redis env vars from `src/config/env.ts`

## ADR-008: BullMQ for Background Job Processing

**Decision**: Use BullMQ with Redis as the queue backend for background job processing.

**Reason**: BullMQ provides a robust, Redis-backed queue with built-in support for job retries with exponential backoff, concurrency limits, rate limiting, priority queues, and delayed jobs. It has first-class TypeScript support and an active community. It fits naturally into our stack alongside Prisma and Express.

**Consequences**:
- Adds Redis as a new infrastructure dependency (must monitor Redis health and memory)
- Worker process must be deployed and scaled separately from the API server
- Job state is durable in Redis, surviving API server restarts
- Requires operational knowledge of Redis monitoring and persistence configuration
- BullMQ dashboard (available via `@bull-board/api`) can be added later for monitoring

**Alternatives Considered**:
- **Bull**: Predecessor to BullMQ; less performant, older API, not recommended for new projects
- **Agenda**: MongoDB-based; would add a second database dependency just for queues
- **pg-boss**: PostgreSQL-based; avoids Redis but lacks the rich feature set and community support of BullMQ
- **AWS SQS / Cloud Tasks**: Vendor lock-in, higher latency, over-engineered for current scale
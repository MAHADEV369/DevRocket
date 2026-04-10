# Skill 09: WebSocket Real-Time Updates with Socket.IO

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 2-3 hours
Depends On: 01, 03

## Input Contract
- Skill 01 (project scaffold) completed with Express + TypeScript setup
- Skill 03 (authentication) completed with JWT-based auth and middleware
- Express HTTP server running and accessible
- `jsonwebtoken` package installed for JWT verification
- `.env` file exists with `JWT_SECRET` configured

## Output Contract
- `src/services/websocket.service.ts` with Socket.IO server setup and event definitions
- JWT authentication middleware for WebSocket handshake
- User-specific rooms for targeted messaging
- Typed event interfaces for `generation_complete` and `notification` events
- WebSocket initialization integrated into `src/index.ts`
- CORS configuration matching the Express server

## Files to Create
- `src/services/websocket.service.ts` - Socket.IO server, auth middleware, event handlers, and room management
- `src/types/events.ts` - TypeScript interfaces for WebSocket events and payloads
- `src/middleware/ws-auth.ts` - JWT authentication middleware for Socket.IO handshake

## Steps

### 1. Install Socket.IO dependencies

```bash
npm install socket.io
npm install -D @types/socket.io
```

### 2. Create WebSocket event types

Create `src/types/events.ts`:

```typescript
export interface ServerToClientEvents {
  generation_complete: (payload: GenerationCompletePayload) => void;
  notification: (payload: NotificationPayload) => void;
  credit_update: (payload: CreditUpdatePayload) => void;
  connection_error: (payload: { message: string }) => void;
}

export interface ClientToServerEvents {
  subscribe_generation: (generationId: string) => void;
  unsubscribe_generation: (generationId: string) => void;
  ping: () => void;
}

export interface InterServerEvents {
  broadcast_generation_complete: (payload: GenerationCompletePayload) => void;
  broadcast_notification: (payload: NotificationPayload) => void;
}

export interface SocketData {
  userId: string;
  email: string;
}

export interface GenerationCompletePayload {
  generationId: string;
  status: 'COMPLETED' | 'FAILED';
  resultUrl?: string;
  error?: string;
  completedAt: string;
}

export interface NotificationPayload {
  id: string;
  type: string;
  title: string;
  message: string;
  read: boolean;
  createdAt: string;
}

export interface CreditUpdatePayload {
  credits: number;
  creditsUsed: number;
  creditsRemaining: number;
}
```

### 3. Create WebSocket authentication middleware

Create `src/middleware/ws-auth.ts`:

```typescript
import { ExtendedError } from 'socket.io';
import { verify } from 'jsonwebtoken';
import { env } from '../config/env';
import { SocketData } from '../types/events';

export function wsAuthMiddleware(socket: any, next: (err?: ExtendedError) => void) {
  const token = socket.handshake.auth?.token
    || socket.handshake.headers?.authorization?.replace('Bearer ', '')
    || socket.handshake.query?.token;

  if (!token) {
    return next(new Error('Authentication required: no token provided'));
  }

  if (typeof token !== 'string') {
    return next(new Error('Authentication required: invalid token format'));
  }

  try {
    const decoded = verify(token, env.JWT_SECRET) as {
      userId: string;
      email: string;
    };

    socket.data = {
      userId: decoded.userId,
      email: decoded.email,
    } satisfies SocketData;

    next();
  } catch (err: any) {
    if (err.name === 'TokenExpiredError') {
      return next(new Error('Authentication failed: token expired'));
    }
    return next(new Error('Authentication failed: invalid token'));
  }
}
```

### 4. Create the WebSocket service

Create `src/services/websocket.service.ts`:

```typescript
import {
  Server,
  Namespace,
} from 'socket.io';
import {
  Server as HttpServer,
} from 'http';
import { wsAuthMiddleware } from '../middleware/ws-auth';
import {
  ServerToClientEvents,
  ClientToServerEvents,
  InterServerEvents,
  SocketData,
  GenerationCompletePayload,
  NotificationPayload,
  CreditUpdatePayload,
} from '../types/events';

export type AppSocket = {
  id: string;
  data: SocketData;
};

type TypedServer = Server<
  ClientToServerEvents,
  ServerToClientEvents,
  InterServerEvents,
  SocketData
>;

let io: TypedServer | null = null;

export function initializeWebSocket(httpServer: HttpServer): TypedServer {
  io = new Server<
    ClientToServerEvents,
    ServerToClientEvents,
    InterServerEvents,
    SocketData
  >(httpServer, {
    cors: {
      origin: process.env.CORS_ORIGIN?.split(',') || ['http://localhost:3000'],
      methods: ['GET', 'POST'],
      credentials: true,
    },
    transports: ['websocket', 'polling'],
    pingInterval: 25000,
    pingTimeout: 20000,
  });

  io.use(wsAuthMiddleware as any);

  io.on('connection', (socket) => {
    const { userId } = socket.data;

    console.log(`Socket connected: ${socket.id} (user: ${userId})`);

    socket.join(`user:${userId}`);

    socket.on('subscribe_generation', (generationId: string) => {
      socket.join(`generation:${generationId}`);
    });

    socket.on('unsubscribe_generation', (generationId: string) => {
      socket.leave(`generation:${generationId}`);
    });

    socket.on('ping', () => {
      socket.emit('connection_error', { message: 'pong' });
    });

    socket.on('disconnect', (reason) => {
      console.log(`Socket disconnected: ${socket.id} (user: ${userId}), reason: ${reason}`);
    });

    socket.on('error', (err) => {
      console.error(`Socket error for user ${userId}:`, err.message);
    });
  });

  console.log('WebSocket server initialized');
  return io;
}

export function getIO(): TypedServer {
  if (!io) {
    throw new Error('WebSocket server not initialized. Call initializeWebSocket first.');
  }
  return io;
}

export function emitGenerationComplete(payload: GenerationCompletePayload & { userId: string }) {
  const server = getIO();
  server.to(`user:${payload.userId}`).emit('generation_complete', {
    generationId: payload.generationId,
    status: payload.status,
    resultUrl: payload.resultUrl,
    error: payload.error,
    completedAt: payload.completedAt,
  });

  server.to(`generation:${payload.generationId}`).emit('generation_complete', {
    generationId: payload.generationId,
    status: payload.status,
    resultUrl: payload.resultUrl,
    error: payload.error,
    completedAt: payload.completedAt,
  });
}

export function emitNotification(userId: string, payload: NotificationPayload) {
  const server = getIO();
  server.to(`user:${userId}`).emit('notification', payload);
}

export function emitCreditUpdate(userId: string, payload: CreditUpdatePayload) {
  const server = getIO();
  server.to(`user:${userId}`).emit('credit_update', payload);
}

export function getConnectedUsersCount(): number {
  const server = getIO();
  return server.sockets.sockets.size;
}

export async function closeWebSocket(): Promise<void> {
  if (io) {
    io.disconnectSockets();
    io.close();
    io = null;
  }
}
```

### 5. Integrate WebSocket into the HTTP server

In `src/index.ts`, import and initialize the WebSocket server alongside Express:

```typescript
import { createServer } from 'http';
import { initializeWebSocket, closeWebSocket } from './services/websocket.service';

// ... existing Express setup ...

const httpServer = createServer(app);
initializeWebSocket(httpServer);

const PORT = env.PORT || 3000;

httpServer.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

process.on('SIGTERM', async () => {
  console.log('Shutting down...');
  closeWebSocket();
  httpServer.close();
  process.exit(0);
});

process.on('SIGINT', async () => {
  console.log('Shutting down...');
  closeWebSocket();
  httpServer.close();
  process.exit(0);
});
```

### 6. Update .env.example

Add the following entry to `.env.example`:

```
CORS_ORIGIN=http://localhost:3000,http://localhost:5173
```

### 7. Add CORS origin to env.ts schema

In `src/config/env.ts`, add:

```typescript
CORS_ORIGIN: z.string().default('http://localhost:3000'),
```

## Verification

1. Compile the project: `npx tsc --noEmit` (no type errors)
2. Start the server: `npm run dev`
3. Connect via a Socket.IO client test script:

```typescript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000', {
  auth: { token: 'your-valid-jwt-token' },
  transports: ['websocket'],
});

socket.on('connect', () => {
  console.log('Connected:', socket.id);
  socket.emit('ping');
});

socket.on('connection_error', (data) => {
  console.log('Event:', data);
});

socket.on('connect_error', (err) => {
  console.error('Connection error:', err.message);
});
```

4. Verify that connecting without a token logs an authentication error.
5. Verify that connecting with a valid JWT succeeds and joins a `user:{userId}` room.
6. Test `subscribe_generation` and `unsubscribe_generation` by emitting those events and verifying room membership.

Expected output:
- Server logs `WebSocket server initialized` on startup
- Authenticated client connects successfully
- Unauthenticated client receives `Authentication required` error
- Server logs socket connect/disconnect events with user IDs

## Rollback

1. Remove installed packages:
   ```bash
   npm uninstall socket.io @types/socket.io
   ```
2. Delete created files:
   ```bash
   rm src/services/websocket.service.ts src/types/events.ts src/middleware/ws-auth.ts
   ```
3. Revert changes in `src/index.ts` to remove WebSocket initialization
4. Remove `CORS_ORIGIN` from `src/config/env.ts` and `.env.example`
5. Remove the `httpServer` creation, reverting to `app.listen()` if that's how it was before

## ADR-009: Socket.IO for Real-Time Communication

**Decision**: Use Socket.IO for WebSocket-based real-time communication between server and clients.

**Reason**: Socket.IO provides automatic fallback to long-polling when WebSocket is unavailable, built-in room support for broadcasting to user-specific groups, and a well-typed event system. It handles reconnection logic, heartbeat/ping, and multiplexing out of the box. The middleware system allows JWT verification at handshake time, ensuring only authenticated users connect.

**Consequences**:
- Client must use the Socket.IO client library (not a raw WebSocket client)
- Additional ~30KB client-side bundle size for the Socket.IO client
- Memory usage scales with concurrent connections; need to monitor socket count and set limits
- Socket.IO protocol version must match between client and server
- Requires sticky sessions if running multiple API servers behind a load balancer (or use the Redis adapter from `@socket.io/redis-adapter`)

**Alternatives Considered**:
- **ws**: Lower-level, lighter weight, but requires implementing rooms, reconnection, and fallback manually
- **Fastify + socket.io**: Would require migrating from Express; too much disruption for this feature alone
- **Server-Sent Events (SSE)**: One-directional (server-to-client only), no room support, no binary transport; would not support future client-to-server real-time needs
- **Pusher/Ably**: Third-party managed services; adds vendor dependency and cost
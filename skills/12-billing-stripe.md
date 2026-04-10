# Skill 12: Stripe Billing & Subscriptions

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 3-4 hours
Depends On: 01, 03

## Input Contract
- Skill 01 (project scaffold) completed with Express + TypeScript setup
- Skill 03 (authentication) completed with JWT auth middleware and user model
- Stripe account created with API keys available
- Prisma schema has a `User` model with `id` and `email` fields
- Express app has body parsing middleware configured

## Output Contract
- `src/modules/billing/` module with checkout session, webhook handler, subscription status, cancel, and customer portal endpoints
- Stripe webhook handler with raw body verification for signature validation
- Prisma models for `Subscription` and `Plan` tracking subscription state
- Customer portal integration for self-service billing management
- All Stripe config validated via zod in `src/config/env.ts`

## Files to Create
- `src/modules/billing/billing.routes.ts` - Express router with billing endpoints
- `src/modules/billing/billing.controller.ts` - Request handlers for billing operations
- `src/modules/billing/billing.service.ts` - Business logic for Stripe API interactions
- `src/modules/billing/billing.types.ts` - TypeScript types for billing module
- `src/modules/billing/webhook.controller.ts` - Stripe webhook handler with signature verification
- `src/modules/billing/index.ts` - Module barrel export

## Steps

### 1. Install Stripe SDK

```bash
npm install stripe
```

### 2. Add Stripe configuration to environment schema

In `src/config/env.ts`, add the following to the zod schema:

```typescript
STRIPE_SECRET_KEY: z.string().startsWith('sk_').default('sk_test_placeholder'),
STRIPE_PUBLISHABLE_KEY: z.string().startsWith('pk_').default('pk_test_placeholder'),
STRIPE_WEBHOOK_SECRET: z.string().startsWith('whsec_').default('whsec_placeholder'),
STRIPE_PRO_PRICE_ID: z.string().default('price_pro_placeholder'),
STRIPE_BUSINESS_PRICE_ID: z.string().default('price_business_placeholder'),
APP_URL: z.string().url().default('http://localhost:3000'),
```

### 3. Add Subscription and Plan models to Prisma schema

Add the following to `prisma/schema.prisma`:

```prisma
enum SubscriptionStatus {
  ACTIVE
  PAST_DUE
  CANCELED
  INCOMPLETE
  TRIALING
  PAUSED
  UNPAID
}

model Subscription {
  id                     String             @id @default(cuid())
  userId                 String
  stripeCustomerId       String             @unique
  stripeSubscriptionId   String?            @unique
  stripePriceId          String
  status                 SubscriptionStatus @default(INCOMPLETE)
  currentPeriodStart    DateTime?
  currentPeriodEnd      DateTime?
  cancelAtPeriodEnd     Boolean            @default(false)
  createdAt             DateTime           @default(now())
  updatedAt             DateTime           @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([stripeCustomerId])
  @@map("subscriptions")
}
```

Also add the relation to the existing `User` model:

```prisma
model User {
  // ... existing fields ...
  subscription Subscription?
}
```

Then run the migration:

```bash
npx prisma migrate dev --name add-subscriptions
npx prisma generate
```

### 4. Create billing types

Create `src/modules/billing/billing.types.ts`:

```typescript
export interface CreateCheckoutSessionInput {
  priceId: string;
  userId: string;
  userEmail: string;
  successUrl?: string;
  cancelUrl?: string;
}

export interface CreatePortalSessionInput {
  customerId: string;
  returnUrl?: string;
}

export interface SubscriptionResponse {
  id: string;
  status: SubscriptionStatus;
  currentPeriodStart: string | null;
  currentPeriodEnd: string | null;
  cancelAtPeriodEnd: boolean;
  stripePriceId: string;
}

export enum SubscriptionStatus {
  ACTIVE = 'ACTIVE',
  PAST_DUE = 'PAST_DUE',
  CANCELED = 'CANCELED',
  INCOMPLETE = 'INCOMPLETE',
  TRIALING = 'TRIALING',
  PAUSED = 'PAUSED',
  UNPAID = 'UNPAID',
}

export type StripeEvent =
  | 'checkout.session.completed'
  | 'customer.subscription.created'
  | 'customer.subscription.updated'
  | 'customer.subscription.deleted'
  | 'invoice.payment_failed'
  | 'invoice.paid';
```

### 5. Create the billing service

Create `src/modules/billing/billing.service.ts`:

```typescript
import Stripe from 'stripe';
import { env } from '../../config/env';
import { prisma } from '../../config/database';
import { SubscriptionStatus } from './billing.types';

const stripe = new Stripe(env.STRIPE_SECRET_KEY, {
  apiVersion: '2024-12-18.acacia',
  typescript: true,
});

export function getStripeClient(): Stripe {
  return stripe;
}

export async function createCheckoutSession(input: {
  priceId: string;
  userId: string;
  userEmail: string;
  successUrl?: string;
  cancelUrl?: string;
}) {
  const { priceId, userId, userEmail, successUrl, cancelUrl } = input;

  let subscription = await prisma.subscription.findUnique({
    where: { userId },
  });

  let customerId: string;

  if (subscription?.stripeCustomerId) {
    customerId = subscription.stripeCustomerId;
  } else {
    const customer = await stripe.customers.create({
      email: userEmail,
      metadata: { userId },
    });
    customerId = customer.id;

    if (subscription) {
      await prisma.subscription.update({
        where: { id: subscription.id },
        data: { stripeCustomerId: customerId },
      });
    }
  }

  const session = await stripe.checkout.sessions.create({
    customer: customerId,
    mode: 'subscription',
    payment_method_types: ['card'],
    line_items: [
      {
        price: priceId,
        quantity: 1,
      },
    ],
    success_url: successUrl || `${env.APP_URL}/billing?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: cancelUrl || `${env.APP_URL}/billing?canceled=true`,
    metadata: { userId },
    subscription_data: {
      metadata: { userId },
    },
  });

  return { url: session.url, sessionId: session.id };
}

export async function createCustomerPortalSession(input: {
  userId: string;
  returnUrl?: string;
}) {
  const { userId, returnUrl } = input;

  const subscription = await prisma.subscription.findUnique({
    where: { userId },
  });

  if (!subscription?.stripeCustomerId) {
    throw new Error('No Stripe customer found for this user');
  }

  const session = await stripe.billingPortal.sessions.create({
    customer: subscription.stripeCustomerId,
    return_url: returnUrl || `${env.APP_URL}/billing`,
  });

  return { url: session.url };
}

export async function getSubscriptionStatus(userId: string) {
  const subscription = await prisma.subscription.findUnique({
    where: { userId },
  });

  if (!subscription) {
    return { status: null, hasSubscription: false };
  }

  return {
    id: subscription.id,
    status: subscription.status,
    currentPeriodStart: subscription.currentPeriodStart?.toISOString() || null,
    currentPeriodEnd: subscription.currentPeriodEnd?.toISOString() || null,
    cancelAtPeriodEnd: subscription.cancelAtPeriodEnd,
    stripePriceId: subscription.stripePriceId,
    hasSubscription: true,
  };
}

export async function cancelSubscription(userId: string) {
  const subscription = await prisma.subscription.findUnique({
    where: { userId },
  });

  if (!subscription?.stripeSubscriptionId) {
    throw new Error('No active subscription found');
  }

  const canceledSub = await stripe.subscriptions.update(
    subscription.stripeSubscriptionId,
    {
      cancel_at_period_end: true,
    }
  );

  await prisma.subscription.update({
    where: { id: subscription.id },
    data: { cancelAtPeriodEnd: true },
  });

  return {
    status: canceledSub.status,
    cancelAtPeriodEnd: canceledSub.cancel_at_period_end,
    currentPeriodEnd: new Date(canceledSub.current_period_end * 1000).toISOString(),
  };
}

export async function reactivateSubscription(userId: string) {
  const subscription = await prisma.subscription.findUnique({
    where: { userId },
  });

  if (!subscription?.stripeSubscriptionId) {
    throw new Error('No active subscription found');
  }

  const reactivatedSub = await stripe.subscriptions.update(
    subscription.stripeSubscriptionId,
    {
      cancel_at_period_end: false,
    }
  );

  await prisma.subscription.update({
    where: { id: subscription.id },
    data: { cancelAtPeriodEnd: false },
  });

  return {
    status: reactivatedSub.status,
    cancelAtPeriodEnd: reactivatedSub.cancel_at_period_end,
  };
}

function mapStripeStatus(stripeStatus: string): SubscriptionStatus {
  const statusMap: Record<string, SubscriptionStatus> = {
    active: SubscriptionStatus.ACTIVE,
    past_due: SubscriptionStatus.PAST_DUE,
    canceled: SubscriptionStatus.CANCELED,
    incomplete: SubscriptionStatus.INCOMPLETE,
    trialing: SubscriptionStatus.TRIALING,
    paused: SubscriptionStatus.PAUSED,
    unpaid: SubscriptionStatus.UNPAID,
  };
  return statusMap[stripeStatus] || SubscriptionStatus.INCOMPLETE;
}

export async function handleWebhookEvent(event: Stripe.Event) {
  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object as Stripe.Checkout.Session;
      const userId = session.metadata?.userId;

      if (!userId) break;

      const subscription = await stripe.subscriptions.retrieve(
        session.subscription as string
      );

      await prisma.subscription.upsert({
        where: { userId },
        create: {
          userId,
          stripeCustomerId: session.customer as string,
          stripeSubscriptionId: subscription.id,
          stripePriceId: subscription.items.data[0]?.price.id || '',
          status: mapStripeStatus(subscription.status),
          currentPeriodStart: new Date(subscription.current_period_start * 1000),
          currentPeriodEnd: new Date(subscription.current_period_end * 1000),
        },
        update: {
          stripeCustomerId: session.customer as string,
          stripeSubscriptionId: subscription.id,
          stripePriceId: subscription.items.data[0]?.price.id || '',
          status: mapStripeStatus(subscription.status),
          currentPeriodStart: new Date(subscription.current_period_start * 1000),
          currentPeriodEnd: new Date(subscription.current_period_end * 1000),
        },
      });
      break;
    }

    case 'customer.subscription.updated': {
      const subscription = event.data.object as Stripe.Subscription;
      const userId = subscription.metadata?.userId;

      if (!userId) break;

      await prisma.subscription.update({
        where: { stripeSubscriptionId: subscription.id },
        data: {
          status: mapStripeStatus(subscription.status),
          stripePriceId: subscription.items.data[0]?.price.id || '',
          currentPeriodStart: new Date(subscription.current_period_start * 1000),
          currentPeriodEnd: new Date(subscription.current_period_end * 1000),
          cancelAtPeriodEnd: subscription.cancel_at_period_end,
        },
      });
      break;
    }

    case 'customer.subscription.deleted': {
      const subscription = event.data.object as Stripe.Subscription;

      await prisma.subscription.update({
        where: { stripeSubscriptionId: subscription.id },
        data: {
          status: SubscriptionStatus.CANCELED,
          stripeSubscriptionId: null,
          cancelAtPeriodEnd: false,
        },
      });
      break;
    }

    case 'invoice.payment_failed': {
      const invoice = event.data.object as Stripe.Invoice;
      const subscriptionId = invoice.subscription as string;

      if (subscriptionId) {
        await prisma.subscription.update({
          where: { stripeSubscriptionId: subscriptionId },
          data: { status: SubscriptionStatus.PAST_DUE },
        });
      }
      break;
    }

    default:
      console.log(`Unhandled Stripe event type: ${event.type}`);
  }
}
```

### 6. Create the webhook controller

Create `src/modules/billing/webhook.controller.ts`:

```typescript
import { Request, Response } from 'express';
import { getStripeClient } from './billing.service';
import { handleWebhookEvent } from './billing.service';
import { env } from '../../config/env';

export async function stripeWebhookHandler(req: Request, res: Response) {
  const sig = req.headers['stripe-signature'] as string;

  if (!sig) {
    res.status(400).json({ error: 'Missing stripe-signature header' });
    return;
  }

  let event: any;

  try {
    event = getStripeClient().webhooks.constructEvent(
      req.body,
      sig,
      env.STRIPE_WEBHOOK_SECRET
    );
  } catch (err: any) {
    console.error('Webhook signature verification failed:', err.message);
    res.status(400).json({ error: `Webhook Error: ${err.message}` });
    return;
  }

  try {
    await handleWebhookEvent(event);
  } catch (err: any) {
    console.error('Webhook handler error:', err.message);
    res.status(500).json({ error: 'Webhook handler failed' });
    return;
  }

  res.json({ received: true });
}
```

### 7. Create the billing controller

Create `src/modules/billing/billing.controller.ts`:

```typescript
import { Request, Response, NextFunction } from 'express';
import * as billingService from './billing.service';

export async function createCheckoutSession(req: Request, res: Response, next: NextFunction) {
  try {
    const userId = req.user!.id;
    const userEmail = req.user!.email;
    const { priceId } = req.body;

    if (!priceId) {
      res.status(400).json({ error: 'priceId is required' });
      return;
    }

    const session = await billingService.createCheckoutSession({
      priceId,
      userId,
      userEmail,
    });

    res.json(session);
  } catch (error) {
    next(error);
  }
}

export async function createPortalSession(req: Request, res: Response, next: NextFunction) {
  try {
    const userId = req.user!.id;

    const session = await billingService.createCustomerPortalSession({ userId });

    res.json(session);
  } catch (error) {
    next(error);
  }
}

export async function getSubscription(req: Request, res: Response, next: NextFunction) {
  try {
    const userId = req.user!.id;

    const subscription = await billingService.getSubscriptionStatus(userId);

    res.json(subscription);
  } catch (error) {
    next(error);
  }
}

export async function cancelSubscription(req: Request, res: Response, next: NextFunction) {
  try {
    const userId = req.user!.id;

    const result = await billingService.cancelSubscription(userId);

    res.json(result);
  } catch (error) {
    next(error);
  }
}

export async function reactivateSubscription(req: Request, res: Response, next: NextFunction) {
  try {
    const userId = req.user!.id;

    const result = await billingService.reactivateSubscription(userId);

    res.json(result);
  } catch (error) {
    next(error);
  }
}
```

### 8. Create the billing routes

Create `src/modules/billing/billing.routes.ts`:

```typescript
import { Router } from 'express';
import { authenticate } from '../../middleware/auth';
import {
  createCheckoutSession,
  createPortalSession,
  getSubscription,
  cancelSubscription,
  reactivateSubscription,
} from './billing.controller';
import { stripeWebhookHandler } from './webhook.controller';

const router = Router();

// Webhook endpoint must use raw body - no body parsing middleware
router.post('/webhook', stripeWebhookHandler);

// Authenticated endpoints
router.post('/checkout', authenticate, createCheckoutSession);
router.post('/portal', authenticate, createPortalSession);
router.get('/subscription', authenticate, getSubscription);
router.post('/cancel', authenticate, cancelSubscription);
router.post('/reactivate', authenticate, reactivateSubscription);

export default router;
```

### 9. Create module barrel export

Create `src/modules/billing/index.ts`:

```typescript
export { default as billingRoutes } from './billing.routes';
export * from './billing.service';
export * from './billing.types';
```

### 10. Register billing routes and configure raw body for webhooks

In `src/index.ts`, register the billing routes and ensure raw body is available for Stripe webhook signature verification:

```typescript
import { billingRoutes } from './modules/billing';

// Stripe webhook needs the raw body, so we configure it on the main app
// The webhook route handler will use req.body directly (must be raw)
// Other routes continue to use JSON parsing as normal
app.use('/api/billing', billingRoutes);
```

**Important**: The Stripe webhook endpoint requires the raw request body for signature verification. Ensure that the Express body parsing middleware is configured such that the webhook route receives the raw body. This typically means adding a specific raw body parser for the webhook path:

```typescript
// Before other body parsing middleware
app.use('/api/billing/webhook', express.raw({ type: 'application/json' }));

// General JSON parsing for all other routes
app.use(express.json());
```

### 11. Update .env.example

Add the following entries to `.env.example`:

```
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_PUBLISHABLE_KEY=pk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_PRO_PRICE_ID=price_pro_xxx
STRIPE_BUSINESS_PRICE_ID=price_business_xxx
APP_URL=http://localhost:3000
```

## Verification

1. Run the Prisma migration:
   ```bash
   npx prisma migrate dev --name add-subscriptions
   npx prisma generate
   ```
2. Compile the project: `npx tsc --noEmit` (no type errors)
3. Start the server: `npm run dev`
4. Test the subscription status endpoint:
   ```bash
   curl -H "Authorization: Bearer <token>" http://localhost:3000/api/billing/subscription
   ```
   Expected: `{ "status": null, "hasSubscription": false }`
5. Install the Stripe CLI and test webhooks locally:
   ```bash
   stripe listen --forward-to http://localhost:3000/api/billing/webhook
   ```
6. Trigger a test event:
   ```bash
   stripe trigger checkout.session.completed
   ```
7. Verify the webhook handler processes the event and creates/updates the subscription in the database.

## Rollback

1. Remove installed packages:
   ```bash
   npm uninstall stripe
   ```
2. Revert Prisma migration:
   ```bash
   npx prisma migrate rollback
   ```
3. Remove the `Subscription` model and `SubscriptionStatus` enum from `prisma/schema.prisma`
4. Remove the `subscription` relation from the `User` model
5. Delete created files:
   ```bash
   rm -rf src/modules/billing/
   ```
6. Remove billing routes and raw body middleware from `src/index.ts`
7. Remove Stripe environment variables from `src/config/env.ts` and `.env.example`
8. Run `npx prisma generate` to update the client

## ADR-012: Stripe for Billing and Subscriptions

**Decision**: Use Stripe Checkout for payment collection and Stripe Billing for subscription management, with webhook-driven state synchronization to the local database.

**Reason**: Stripe Checkout provides PCI-compliant payment collection without handling card data on our servers. The customer portal offers self-service subscription management out of the box. Webhooks ensure our database stays in sync with Stripe's state, handling edge cases like payment failures and expired cards. Using `upsert` on webhook events provides idempotency for duplicate webhook deliveries.

**Consequences**:
- Webhook endpoint must receive raw (unparsed) body for signature verification; requires special Express middleware configuration
- Stripe API version is pinned; changes require manual review
- All subscription state mutations should go through Stripe first, then be reflected locally via webhooks; never mutate local state independently
- The `SubscriptionStatus` enum must be kept in sync with Stripe's subscription statuses
- Requires setting up Stripe products and prices in the Stripe Dashboard before testing checkout
- Customer portal configuration must be set up in Stripe Dashboard (return URL, allowed features)

**Alternatives Considered**:
- **Lemon Squeezy**: Simpler API and handles tax compliance, but less flexible for custom billing workflows and smaller ecosystem
- **Paddle**:Handles tax compliance and Merchant of Record, but higher fees and less control over checkout UX
- **Custom billing with Stripe Elements**: More control but significantly more development effort for PCI compliance and checkout flows
- **Recurly**: Good for complex billing scenarios but less developer-friendly API and higher minimum costs
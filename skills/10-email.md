# Skill 10: Email Service with Resend

Version: 1.0.0
Last Updated: 2025-01-15
Estimated Time: 1-2 hours
Depends On: 01, 03

## Input Contract
- Skill 01 (project scaffold) completed with Express + TypeScript setup
- Skill 03 (authentication) completed with user model and JWT setup
- Resend account created and API key available
- `@prisma/client` available for user lookups
- Skill 08 (queue worker) recommended but not required; uses fire-and-forget pattern as fallback

## Output Contract
- `src/services/email.service.ts` with Resend SDK integration and typed send functions
- HTML email templates for: email verification, password reset, welcome, and notification emails
- Template rendering with inline styles for maximum email client compatibility
- Fire-and-forget pattern for non-blocking email dispatch
- All email config validated via zod in `src/config/env.ts`

## Files to Create
- `src/services/email.service.ts` - Core email service with Resend client and typed send methods
- `src/templates/email/verification.ts` - Email verification template
- `src/templates/email/password-reset.ts` - Password reset template
- `src/templates/email/welcome.ts` - Welcome email template
- `src/templates/email/notification.ts` - Generic notification email template
- `src/templates/email/layouts/base.ts` - Base HTML layout with shared styles

## Steps

### 1. Install Resend SDK

```bash
npm install resend
```

### 2. Add email configuration to environment schema

In `src/config/env.ts`, add the following to the zod schema:

```typescript
RESEND_API_KEY: z.string().startsWith('re_').default('re_placeholder'),
EMAIL_FROM: z.string().email().default('noreply@example.com'),
EMAIL_FROM_NAME: z.string().default('FastDepo'),
APP_URL: z.string().url().default('http://localhost:3000'),
```

### 3. Create base email layout

Create `src/templates/email/layouts/base.ts`:

```typescript
export interface BaseEmailProps {
  title: string;
  previewText: string;
  content: string;
  actionUrl?: string;
  actionLabel?: string;
}

export function renderBaseEmail(props: BaseEmailProps): string {
  const { title, previewText, content, actionUrl, actionLabel } = props;

  const actionButton = actionUrl && actionLabel
    ? `
      <table border="0" cellpadding="0" cellspacing="0" role="presentation" style="border-radius:6px;background-color:#4F46E5;color:#ffffff;margin:20px auto;">
        <tr>
          <td style="padding:12px 24px;">
            <a href="${actionUrl}" target="_blank" style="color:#ffffff;text-decoration:none;font-weight:600;font-size:16px;">${actionLabel}</a>
          </td>
        </tr>
      </table>`
    : '';

  return `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>${title}</title>
  <!--[if mso]>
  <noscript>
    <xml>
      <o:OfficeDocumentSettings>
        <o:PixelsPerInch>96</o:PixelsPerInch>
      </o:OfficeDocumentSettings>
    </xml>
  </noscript>
  <![endif]-->
</head>
<body style="margin:0;padding:0;background-color:#f4f4f5;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,Helvetica,Arial,sans-serif;">
  <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background-color:#f4f4f5;padding:40px 0;">
    <tr>
      <td align="center">
        <table role="presentation" width="600" cellpadding="0" cellspacing="0" style="background-color:#ffffff;border-radius:8px;overflow:hidden;box-shadow:0 1px 3px rgba(0,0,0,0.1);">
          <tr>
            <td style="background-color:#4F46E5;padding:30px 40px;text-align:center;">
              <h1 style="margin:0;color:#ffffff;font-size:24px;font-weight:700;">FastDepo</h1>
            </td>
          </tr>
          <tr>
            <td style="padding:40px;">
              <div style="color:#1f2937;font-size:16px;line-height:1.6;">
                ${content}
              </div>
              ${actionButton}
            </td>
          </tr>
          <tr>
            <td style="padding:24px 40px;background-color:#f9fafb;border-top:1px solid #e5e7eb;">
              <p style="margin:0;color:#6b7280;font-size:13px;text-align:center;">
                &copy; 2025 FastDepo. All rights reserved.<br>
                If you did not request this email, you can safely ignore it.
              </p>
            </td>
          </tr>
        </table>
      </td>
    </tr>
  </table>
</body>
</html>`;
}
```

### 4. Create email verification template

Create `src/templates/email/verification.ts`:

```typescript
import { renderBaseEmail } from '../layouts/base';

export interface VerificationEmailProps {
  username: string;
  verificationUrl: string;
}

export function renderVerificationEmail(props: VerificationEmailProps): string {
  const content = `
    <p style="margin:0 0 16px;">Hi <strong>${props.username}</strong>,</p>
    <p style="margin:0 0 16px;">Thank you for signing up! Please verify your email address to activate your account.</p>
    <p style="margin:0 0 16px;">Click the button below to confirm your email:</p>
  `;

  return renderBaseEmail({
    title: 'Verify Your Email',
    previewText: `Hi ${props.username}, please verify your email address.`,
    content,
    actionUrl: props.verificationUrl,
    actionLabel: 'Verify Email',
  });
}
```

### 5. Create password reset template

Create `src/templates/email/password-reset.ts`:

```typescript
import { renderBaseEmail } from '../layouts/base';

export interface PasswordResetEmailProps {
  username: string;
  resetUrl: string;
  expiryHours: number;
}

export function renderPasswordResetEmail(props: PasswordResetEmailProps): string {
  const content = `
    <p style="margin:0 0 16px;">Hi <strong>${props.username}</strong>,</p>
    <p style="margin:0 0 16px;">We received a request to reset your password. Click the button below to choose a new one:</p>
    <p style="margin:0 0 16px;color:#6b7280;font-size:14px;">This link will expire in <strong>${props.expiryHours} hours</strong>. If you did not request a password reset, you can safely ignore this email.</p>
  `;

  return renderBaseEmail({
    title: 'Reset Your Password',
    previewText: `Hi ${props.username}, reset your password.`,
    content,
    actionUrl: props.resetUrl,
    actionLabel: 'Reset Password',
  });
}
```

### 6. Create welcome template

Create `src/templates/email/welcome.ts`:

```typescript
import { renderBaseEmail } from '../layouts/base';

export interface WelcomeEmailProps {
  username: string;
  dashboardUrl: string;
}

export function renderWelcomeEmail(props: WelcomeEmailProps): string {
  const content = `
    <p style="margin:0 0 16px;">Welcome aboard, <strong>${props.username}</strong>!</p>
    <p style="margin:0 0 16px;">Your account is all set up and ready to go. Here is what you can do next:</p>
    <ul style="margin:0 0 16px;padding-left:20px;">
      <li style="margin-bottom:8px;">Set up your profile and preferences</li>
      <li style="margin-bottom:8px;">Explore the dashboard and try your first generation</li>
      <li style="margin-bottom:8px;">Check your available credits</li>
    </ul>
    <p style="margin:0 0 16px;">Head to your dashboard to get started:</p>
  `;

  return renderBaseEmail({
    title: 'Welcome to FastDepo',
    previewText: `Welcome ${props.username}, your FastDepo account is ready!`,
    content,
    actionUrl: props.dashboardUrl,
    actionLabel: 'Go to Dashboard',
  });
}
```

### 7. Create notification template

Create `src/templates/email/notification.ts`:

```typescript
import { renderBaseEmail } from '../layouts/base';

export interface NotificationEmailProps {
  username: string;
  title: string;
  message: string;
  actionUrl?: string;
  actionLabel?: string;
}

export function renderNotificationEmail(props: NotificationEmailProps): string {
  const content = `
    <p style="margin:0 0 16px;">Hi <strong>${props.username}</strong>,</p>
    <h2 style="margin:0 0 16px;color:#1f2937;font-size:20px;">${props.title}</h2>
    <p style="margin:0 0 16px;">${props.message}</p>
  `;

  return renderBaseEmail({
    title: props.title,
    previewText: `${props.title}: ${props.message}`,
    content,
    actionUrl: props.actionUrl,
    actionLabel: props.actionLabel,
  });
}
```

### 8. Create the email service

Create `src/services/email.service.ts`:

```typescript
import { Resend } from 'resend';
import { env } from '../config/env';
import { renderVerificationEmail } from '../templates/email/verification';
import { renderPasswordResetEmail } from '../templates/email/password-reset';
import { renderWelcomeEmail } from '../templates/email/welcome';
import { renderNotificationEmail } from '../templates/email/notification';

const resend = new Resend(env.RESEND_API_KEY);

const FROM_ADDRESS = `${env.EMAIL_FROM_NAME} <${env.EMAIL_FROM}>`;

type EmailResult = Promise<{ success: boolean; id?: string; error?: string }>;

async function sendEmail(to: string, subject: string, html: string): EmailResult {
  try {
    const { data, error } = await resend.emails.send({
      from: FROM_ADDRESS,
      to,
      subject,
      html,
    });

    if (error) {
      console.error('Resend API error:', error.message);
      return { success: false, error: error.message };
    }

    return { success: true, id: data?.id };
  } catch (err: any) {
    console.error('Failed to send email:', err.message);
    return { success: false, error: err.message };
  }
}

export async function sendVerificationEmail(
  to: string,
  username: string,
  verificationToken: string
): EmailResult {
  const verificationUrl = `${env.APP_URL}/auth/verify?token=${verificationToken}`;
  const html = renderVerificationEmail({ username, verificationUrl });
  return sendEmail(to, 'Verify Your Email - FastDepo', html);
}

export async function sendPasswordResetEmail(
  to: string,
  username: string,
  resetToken: string,
  expiryHours: number = 1
): EmailResult {
  const resetUrl = `${env.APP_URL}/auth/reset-password?token=${resetToken}`;
  const html = renderPasswordResetEmail({ username, resetUrl, expiryHours });
  return sendEmail(to, 'Reset Your Password - FastDepo', html);
}

export async function sendWelcomeEmail(
  to: string,
  username: string
): EmailResult {
  const dashboardUrl = `${env.APP_URL}/dashboard`;
  const html = renderWelcomeEmail({ username, dashboardUrl });
  return sendEmail(to, 'Welcome to FastDepo!', html);
}

export async function sendNotificationEmail(
  to: string,
  username: string,
  title: string,
  message: string,
  actionUrl?: string,
  actionLabel?: string
): EmailResult {
  const html = renderNotificationEmail({
    username,
    title,
    message,
    actionUrl,
    actionLabel,
  });
  return sendEmail(to, `${title} - FastDepo`, html);
}

export function fireAndForget(promise: Promise<any>): void {
  promise.catch((err) => {
    console.error('Fire-and-forget email error:', err.message);
  });
}
```

### 9. Update .env.example

Add the following entries to `.env.example`:

```
RESEND_API_KEY=re_your_api_key_here
EMAIL_FROM=noreply@example.com
EMAIL_FROM_NAME=FastDepo
APP_URL=http://localhost:3000
```

## Verification

1. Compile the project: `npx tsc --noEmit` (no type errors)
2. Set a valid Resend API key in `.env` (use a test key from the Resend dashboard)
3. Run a quick test:

```bash
npx tsx -e "
import { sendNotificationEmail } from './src/services/email.service';
const result = await sendNotificationEmail(
  'your-test-email@example.com',
  'Test User',
  'Test Notification',
  'This is a test email from FastDepo.'
);
console.log(result);
"
```

4. Verify the email arrives in the test inbox with proper HTML formatting, inline styles, and the FastDepo branding.
5. Check that the `fireAndForget` wrapper catches and logs errors without throwing.

Expected output:
- `{ success: true, id: 're_...' }` result from the send
- Test email received with proper layout and branding
- No unhandled promise rejections from the fire-and-forget pattern

## Rollback

1. Remove installed packages:
   ```bash
   npm uninstall resend
   ```
2. Delete created files:
   ```bash
   rm -rf src/services/email.service.ts src/templates/email/
   ```
3. Remove email-related environment variables from `src/config/env.ts` and `.env.example`

## ADR-010: Resend for Transactional Email

**Decision**: Use Resend as the email delivery provider with HTML templates rendered server-side and delivered via the Resend SDK.

**Reason**: Resend offers a clean developer experience with a simple SDK, reliable delivery, and built-in DKIM/SPF handling on custom domains. Server-side template rendering with inline styles ensures maximum compatibility across email clients (Gmail, Outlook, Apple Mail) without relying on external template engines or CSS-inlining services.

**Consequences**:
- Email templates are maintained in TypeScript code, requiring deployment for template changes
- Each template uses inline styles via the base layout, ensuring consistent rendering across clients
- Sending is done via fire-and-forget to avoid blocking request handlers; failures are logged but do not surface to users
- Rate limits on Resend's free tier (100 emails/day) may require monitoring
- For high-volume transactional email, consider integrating with the queue system from Skill 08

**Alternatives Considered**:
- **SendGrid**: Mature and widely used, but the SDK and API are more verbose; requires more setup for domain authentication
- **AWS SES**: Lowest cost at scale, but complex setup (DKIM, DMARC, IP warmup) and no SDK-level templating
- **Nodemailer + SMTP**: Self-hosted; requires managing deliverability reputation, no built-in analytics
- **MJML for templates**: Adds a build step and dependency; hand-crafted inline styles are simpler for our 4 templates
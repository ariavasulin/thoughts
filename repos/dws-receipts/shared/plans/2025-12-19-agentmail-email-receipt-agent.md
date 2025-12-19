# AgentMail Email Receipt Agent Implementation Plan

## Overview

Implement an email-based receipt submission system using AgentMail that allows carpenters to forward receipt images via email instead of using the mobile app. The agent receives emails, identifies the user, processes attachments through the existing OCR pipeline, creates receipt records, and sends confirmation emails.

## Current State Analysis

### Existing Infrastructure
- **Receipt creation**: `dws-app/src/app/api/receipts/route.ts` - handles receipt CRUD with Supabase
- **Image upload**: `dws-app/src/app/api/receipts/upload/route.ts` - compresses images with Sharp, stores in Supabase
- **OCR extraction**: `dws-app/src/app/api/receipts/ocr/route.ts` - uses BAML to extract date/amount/category
- **User profiles**: `user_profiles` table with `user_id`, `role`, `full_name`, `preferred_name`, `employee_id_internal`
- **Categories**: 5 default categories (Parking, Gas, Meals & Entertainment, Office Supplies, Other)
- **Deployment**: Vercel (public HTTPS endpoints available for webhooks)

### What's Missing
- No email field in `user_profiles` for sender identification
- No way to track receipt submission source (app vs email)
- No AgentMail integration or webhook endpoint
- No email sending capability

### Key Discoveries
- Receipt creation expects: `receipt_date`, `amount`, `category_id`, `notes`, `tempFilePath` (`dws-app/src/app/api/receipts/route.ts:21-24`)
- Image compression uses Sharp to resize to 1200x1200 max, 80% JPEG quality (`dws-app/src/app/api/receipts/upload/route.ts:62-70`)
- OCR uses BAML's `ExtractReceiptFromImage` function (`dws-app/src/app/api/receipts/ocr/route.ts:56`)
- Duplicate detection checks same date/amount/user (`dws-app/src/app/api/receipts/ocr/route.ts:77-92`)
- Allowed file types: JPEG, PNG, PDF (`dws-app/src/app/api/receipts/upload/route.ts:36`)

## Desired End State

After implementation:
1. Carpenters can forward emails with receipt image attachments to `receipts@[domain].agentmail.to`
2. System identifies the sender by matching email to `user_profiles.email`
3. Each valid attachment becomes a separate receipt record with `submission_source: 'email'`
4. Sender receives a confirmation email listing processed receipts and any errors
5. Admins can filter receipts by submission source in the dashboard

### Verification
- Send test email with 2 receipt images → 2 receipt records created
- Send email from unknown address → receive "unknown user" reply
- Send email with no attachments → receive "no attachments" reply
- View receipts in admin dashboard with "email" source indicator

## What We're NOT Doing

- **Conversational AI**: No back-and-forth email threads; simple acknowledge/error responses only
- **AI category classification**: Using "Other" as default, not training a classifier
- **Email notifications for admins**: No push notifications; admins use existing dashboard
- **SMS-based submission**: Staying focused on email via AgentMail
- **Inline image extraction**: Only processing attached files, not embedded images
- **PDF multi-page support**: Treating PDFs as single images (first page if multi-page)

## Implementation Approach

1. **Schema first**: Add `email` to user_profiles and `submission_source` to receipts
2. **AgentMail setup**: Create inbox, configure webhook
3. **Core processing**: Build webhook endpoint that reuses existing OCR/upload logic
4. **Reply emails**: Send confirmation/error responses
5. **Admin UI enhancement**: Show submission source in dashboard (optional, lower priority)

---

## Phase 1: Database Schema Changes

### Overview
Add email mapping to user profiles and track receipt submission source.

### Changes Required

#### 1. Migration: Add email to user_profiles
**File**: New migration via Supabase

```sql
-- Add email column to user_profiles for email-to-user mapping
ALTER TABLE user_profiles ADD COLUMN email TEXT;

-- Create unique index (each email maps to one user)
CREATE UNIQUE INDEX idx_user_profiles_email ON user_profiles(email) WHERE email IS NOT NULL;

-- Add comment for documentation
COMMENT ON COLUMN user_profiles.email IS 'Email address for AgentMail receipt submission identification';
```

#### 2. Migration: Add submission_source to receipts
**File**: New migration via Supabase

```sql
-- Track how receipt was submitted
ALTER TABLE receipts ADD COLUMN submission_source TEXT DEFAULT 'app';

-- Add check constraint for valid values
ALTER TABLE receipts ADD CONSTRAINT receipts_submission_source_check
  CHECK (submission_source IN ('app', 'email'));

-- Add comment for documentation
COMMENT ON COLUMN receipts.submission_source IS 'How receipt was submitted: app (mobile) or email (AgentMail)';
```

#### 3. Update TypeScript types
**File**: `dws-app/src/lib/types.ts`

```typescript
// Update UserProfile interface
export interface UserProfile {
  user_id: string;
  role: 'employee' | 'admin';
  full_name?: string;
  preferred_name?: string;
  employee_id_internal?: string;
  email?: string; // NEW: for AgentMail identification
  created_at?: string;
  updated_at?: string;
  deleted_at?: string | null;
}

// Update Receipt interface
export interface Receipt {
  id: string;
  user_id: string;
  // ... existing fields ...
  submission_source?: 'app' | 'email'; // NEW
}
```

### Success Criteria

#### Automated Verification:
- [ ] Migration applies cleanly: `npx supabase db push` (or via dashboard)
- [ ] TypeScript types compile: `cd dws-app && npm run build`
- [ ] No linting errors: `cd dws-app && npm run lint`

#### Manual Verification:
- [ ] Can add email address to user profile via Supabase dashboard
- [ ] Email column appears in user_profiles table
- [ ] submission_source column appears in receipts table with 'app' default

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the migrations applied correctly before proceeding.

---

## Phase 2: AgentMail Integration Setup

### Overview
Set up AgentMail account, create inbox, and configure environment variables.

### Changes Required

#### 1. Install AgentMail SDK
**File**: `dws-app/package.json`

```bash
cd dws-app && npm install agentmail
```

#### 2. Add environment variables
**File**: `dws-app/.env.local` (and Vercel environment settings)

```env
# AgentMail Configuration
AGENTMAIL_API_KEY=your_api_key_here
AGENTMAIL_INBOX_ID=your_inbox_id_here
AGENTMAIL_WEBHOOK_SECRET=your_webhook_secret_here

# Default category for email submissions (UUID of "Other" category)
DEFAULT_CATEGORY_ID=uuid_of_other_category
```

#### 3. Create AgentMail client utility
**File**: `dws-app/src/lib/agentmail.ts` (NEW)

```typescript
import { AgentMailClient } from 'agentmail';

let client: AgentMailClient | null = null;

export function getAgentMailClient(): AgentMailClient {
  if (!client) {
    const apiKey = process.env.AGENTMAIL_API_KEY;
    if (!apiKey) {
      throw new Error('AGENTMAIL_API_KEY environment variable is not set');
    }
    client = new AgentMailClient({ apiKey });
  }
  return client;
}

export function getInboxId(): string {
  const inboxId = process.env.AGENTMAIL_INBOX_ID;
  if (!inboxId) {
    throw new Error('AGENTMAIL_INBOX_ID environment variable is not set');
  }
  return inboxId;
}
```

#### 4. Manual setup steps (not code)
1. Sign up at https://console.agentmail.to
2. Create an inbox (e.g., `receipts@yourcompany.agentmail.to`)
3. Note the inbox ID
4. Create a webhook pointing to `https://your-app.vercel.app/api/agent/email-webhook`
5. Generate a webhook secret for signature verification
6. Look up the UUID of the "Other" category from your database

### Success Criteria

#### Automated Verification:
- [ ] Package installs: `cd dws-app && npm install`
- [ ] Build succeeds: `cd dws-app && npm run build`
- [ ] AgentMail types available in IDE

#### Manual Verification:
- [ ] AgentMail console shows inbox created
- [ ] Webhook configured in AgentMail dashboard
- [ ] Environment variables set in Vercel project settings
- [ ] Test email to inbox appears in AgentMail console

**Implementation Note**: Pause here to confirm AgentMail account is set up and webhook is configured.

---

## Phase 3: Email Webhook Endpoint

### Overview
Create the webhook endpoint that receives emails from AgentMail, processes attachments, and creates receipts.

### Changes Required

#### 1. Create webhook route
**File**: `dws-app/src/app/api/agent/email-webhook/route.ts` (NEW)

```typescript
import { NextResponse, type NextRequest } from 'next/server';
import { createSupabaseAdminClient } from '@/lib/supabaseServerClient';
import { getAgentMailClient, getInboxId } from '@/lib/agentmail';
import { processEmailReceipt } from '@/lib/email-receipt-processor';

// AgentMail webhook payload types
interface AgentMailAttachment {
  attachment_id: string;
  filename: string;
  content_type: string;
  size: number;
  inline: boolean;
}

interface AgentMailMessage {
  from_: string[];
  to: string[];
  subject: string;
  message_id: string;
  inbox_id: string;
  text: string;
  html: string;
  attachments: AgentMailAttachment[];
  labels: string[];
  timestamp: string;
}

interface AgentMailWebhookPayload {
  event_type: string;
  event_id: string;
  message: AgentMailMessage;
}

export async function POST(request: NextRequest) {
  console.log('Email webhook: Received request');

  try {
    const payload: AgentMailWebhookPayload = await request.json();
    console.log('Email webhook: Event type:', payload.event_type);

    // Ignore sent messages to avoid processing our own replies
    if (payload.message.labels.includes('sent')) {
      console.log('Email webhook: Ignoring sent message');
      return NextResponse.json({ success: true, skipped: true });
    }

    // Only process received messages
    if (payload.event_type !== 'message.received') {
      console.log('Email webhook: Ignoring non-receive event');
      return NextResponse.json({ success: true, skipped: true });
    }

    const message = payload.message;
    const senderEmail = message.from_[0]; // First sender address

    if (!senderEmail) {
      console.error('Email webhook: No sender email found');
      return NextResponse.json({ error: 'No sender email' }, { status: 400 });
    }

    console.log('Email webhook: Processing email from:', senderEmail);
    console.log('Email webhook: Subject:', message.subject);
    console.log('Email webhook: Attachments:', message.attachments.length);

    // Process the email and get results
    const result = await processEmailReceipt({
      senderEmail,
      subject: message.subject,
      messageId: message.message_id,
      inboxId: message.inbox_id,
      attachments: message.attachments,
    });

    // Send reply email with results
    await sendReplyEmail(message.inbox_id, message.message_id, senderEmail, result);

    return NextResponse.json({ success: true, result });

  } catch (error) {
    console.error('Email webhook: Unhandled error:', error);
    const errorMessage = error instanceof Error ? error.message : 'Unknown error';
    return NextResponse.json({ error: errorMessage }, { status: 500 });
  }
}

async function sendReplyEmail(
  inboxId: string,
  messageId: string,
  senderEmail: string,
  result: ProcessingResult
) {
  const client = getAgentMailClient();

  let subject: string;
  let body: string;

  if (result.error) {
    subject = 'Receipt Submission Failed';
    body = `Hello,\n\n${result.error}\n\nPlease try again or use the mobile app.\n\nDWS Receipts`;
  } else if (result.receipts.length === 0) {
    subject = 'No Receipts Processed';
    body = `Hello,\n\nNo valid receipt images were found in your email. Please attach JPEG, PNG, or PDF files.\n\nDWS Receipts`;
  } else {
    subject = `${result.receipts.length} Receipt(s) Submitted`;
    const receiptList = result.receipts.map((r, i) => {
      const status = r.success ? '✓' : '⚠';
      const details = r.success
        ? `$${r.amount?.toFixed(2) || '?.??'} on ${r.date || 'unknown date'}`
        : r.error;
      return `${status} Receipt ${i + 1}: ${details}`;
    }).join('\n');

    body = `Hello,\n\nProcessed ${result.receipts.length} receipt(s):\n\n${receiptList}\n\nView your receipts in the DWS Receipts app.\n\nDWS Receipts`;
  }

  try {
    await client.inboxes.messages.reply(inboxId, messageId, {
      text: body,
    });
    console.log('Email webhook: Reply sent to', senderEmail);
  } catch (error) {
    console.error('Email webhook: Failed to send reply:', error);
  }
}

interface ProcessingResult {
  error?: string;
  receipts: {
    success: boolean;
    receiptId?: string;
    date?: string;
    amount?: number;
    error?: string;
  }[];
}
```

#### 2. Create email receipt processor
**File**: `dws-app/src/lib/email-receipt-processor.ts` (NEW)

```typescript
import { createSupabaseAdminClient } from '@/lib/supabaseServerClient';
import { getAgentMailClient, getInboxId } from '@/lib/agentmail';
import { v4 as uuidv4 } from 'uuid';
import sharp from 'sharp';
import { b } from '@baml/index';
import { Image } from '@boundaryml/baml';

interface AttachmentInfo {
  attachment_id: string;
  filename: string;
  content_type: string;
  size: number;
}

interface ProcessEmailParams {
  senderEmail: string;
  subject: string;
  messageId: string;
  inboxId: string;
  attachments: AttachmentInfo[];
}

interface ReceiptResult {
  success: boolean;
  receiptId?: string;
  date?: string;
  amount?: number;
  error?: string;
}

interface ProcessingResult {
  error?: string;
  receipts: ReceiptResult[];
}

const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'application/pdf', 'image/jpg'];
const MAX_SIZE = 10 * 1024 * 1024; // 10MB

export async function processEmailReceipt(params: ProcessEmailParams): Promise<ProcessingResult> {
  const { senderEmail, subject, messageId, inboxId, attachments } = params;
  const supabase = createSupabaseAdminClient();

  // 1. Look up user by email
  const { data: userProfile, error: userError } = await supabase
    .from('user_profiles')
    .select('user_id, full_name, email')
    .eq('email', senderEmail.toLowerCase())
    .single();

  if (userError || !userProfile) {
    console.log('Email processor: Unknown sender:', senderEmail);
    return {
      error: `We couldn't find an account associated with ${senderEmail}. Please contact your administrator to add your email address to your profile, or use the mobile app.`,
      receipts: [],
    };
  }

  const userId = userProfile.user_id;
  console.log('Email processor: Matched user:', userId, userProfile.full_name);

  // 2. Filter valid attachments
  const validAttachments = attachments.filter(att => {
    const isValidType = ALLOWED_TYPES.includes(att.content_type.toLowerCase());
    const isValidSize = att.size <= MAX_SIZE;
    const isNotInline = !att.filename.startsWith('inline'); // Skip inline images
    return isValidType && isValidSize && isNotInline;
  });

  if (validAttachments.length === 0) {
    return {
      error: 'No valid receipt images found. Please attach JPEG, PNG, or PDF files (max 10MB each).',
      receipts: [],
    };
  }

  console.log('Email processor: Processing', validAttachments.length, 'attachments');

  // 3. Get default category (Other)
  const defaultCategoryId = process.env.DEFAULT_CATEGORY_ID;
  if (!defaultCategoryId) {
    console.error('Email processor: DEFAULT_CATEGORY_ID not configured');
  }

  // 4. Process each attachment
  const results: ReceiptResult[] = [];
  const client = getAgentMailClient();

  for (const attachment of validAttachments) {
    try {
      const result = await processAttachment({
        attachment,
        userId,
        messageId,
        inboxId,
        defaultCategoryId,
        subject,
        supabase,
        client,
      });
      results.push(result);
    } catch (error) {
      console.error('Email processor: Error processing attachment:', attachment.filename, error);
      results.push({
        success: false,
        error: `Failed to process ${attachment.filename}`,
      });
    }
  }

  return { receipts: results };
}

async function processAttachment(params: {
  attachment: AttachmentInfo;
  userId: string;
  messageId: string;
  inboxId: string;
  defaultCategoryId?: string;
  subject: string;
  supabase: ReturnType<typeof createSupabaseAdminClient>;
  client: ReturnType<typeof getAgentMailClient>;
}): Promise<ReceiptResult> {
  const { attachment, userId, messageId, inboxId, defaultCategoryId, subject, supabase, client } = params;

  console.log('Email processor: Processing attachment:', attachment.filename);

  // 1. Download attachment from AgentMail
  const attachmentData = await client.inboxes.messages.attachments.get(
    inboxId,
    messageId,
    attachment.attachment_id
  );

  // Convert to buffer (attachmentData is raw file)
  let fileBuffer: Buffer;
  if (attachmentData instanceof ArrayBuffer) {
    fileBuffer = Buffer.from(attachmentData);
  } else if (Buffer.isBuffer(attachmentData)) {
    fileBuffer = attachmentData;
  } else {
    // Handle as readable stream or other format
    const chunks: Uint8Array[] = [];
    for await (const chunk of attachmentData as AsyncIterable<Uint8Array>) {
      chunks.push(chunk);
    }
    fileBuffer = Buffer.concat(chunks);
  }

  // 2. Compress image if not PDF
  let processedBuffer = fileBuffer;
  let contentType = attachment.content_type;

  if (attachment.content_type.startsWith('image/') && attachment.content_type !== 'application/pdf') {
    try {
      processedBuffer = await sharp(fileBuffer)
        .resize({
          width: 1200,
          height: 1200,
          fit: 'inside',
          withoutEnlargement: true,
        })
        .jpeg({ quality: 80, progressive: true })
        .toBuffer();
      contentType = 'image/jpeg';
      console.log('Email processor: Compressed image from', fileBuffer.length, 'to', processedBuffer.length);
    } catch (err) {
      console.warn('Email processor: Sharp compression failed, using original');
      processedBuffer = fileBuffer;
    }
  }

  // 3. Upload to Supabase Storage
  const fileExtension = contentType === 'application/pdf' ? 'pdf' : 'jpg';
  const randomString = uuidv4().slice(0, 8);
  const tempFilePath = `${userId}/temp_email_${randomString}_${Date.now()}.${fileExtension}`;

  const { error: uploadError } = await supabase.storage
    .from('receipt-images')
    .upload(tempFilePath, processedBuffer, {
      contentType,
      upsert: false,
    });

  if (uploadError) {
    console.error('Email processor: Upload failed:', uploadError);
    return { success: false, error: 'Failed to upload image' };
  }

  console.log('Email processor: Uploaded to', tempFilePath);

  // 4. Run OCR extraction
  const base64 = processedBuffer.toString('base64');
  const image = Image.fromBase64(contentType, base64);

  let extractedDate: string | null = null;
  let extractedAmount: number | null = null;
  let categoryId = defaultCategoryId || null;

  try {
    const extracted = await b.ExtractReceiptFromImage(image);
    console.log('Email processor: OCR result:', extracted);

    extractedDate = extracted.date || null;
    extractedAmount = extracted.amount ?? null;

    // Try to match extracted category
    if (extracted.category) {
      const { data: categoryMatch } = await supabase
        .from('categories')
        .select('id')
        .ilike('name', extracted.category)
        .single();

      if (categoryMatch) {
        categoryId = categoryMatch.id;
      }
    }
  } catch (ocrError) {
    console.error('Email processor: OCR failed:', ocrError);
    // Continue with null values - receipt will be flagged for review
  }

  // 5. Create receipt record
  const { data: newReceipt, error: insertError } = await supabase
    .from('receipts')
    .insert({
      user_id: userId,
      receipt_date: extractedDate,
      amount: extractedAmount,
      category_id: categoryId,
      description: `Email: ${subject}`.slice(0, 500), // Use email subject as description
      status: 'Pending',
      image_url: tempFilePath,
      submission_source: 'email',
    })
    .select('id')
    .single();

  if (insertError || !newReceipt) {
    console.error('Email processor: Insert failed:', insertError);
    // Clean up uploaded file
    await supabase.storage.from('receipt-images').remove([tempFilePath]);
    return { success: false, error: 'Failed to create receipt record' };
  }

  // 6. Move file to final location
  const finalPath = `${userId}/${newReceipt.id}.${fileExtension}`;
  const { error: moveError } = await supabase.storage
    .from('receipt-images')
    .move(tempFilePath, finalPath);

  if (moveError) {
    console.error('Email processor: Move failed:', moveError);
    // Receipt exists but image path is wrong - update with temp path
  } else {
    // Update receipt with final path
    await supabase
      .from('receipts')
      .update({ image_url: finalPath })
      .eq('id', newReceipt.id);
  }

  console.log('Email processor: Created receipt:', newReceipt.id);

  return {
    success: true,
    receiptId: newReceipt.id,
    date: extractedDate || undefined,
    amount: extractedAmount || undefined,
  };
}
```

#### 3. Create Supabase admin client (if not exists)
**File**: `dws-app/src/lib/supabaseServerClient.ts` (ADD to existing file)

```typescript
// Add this function if it doesn't exist
export function createSupabaseAdminClient() {
  const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!;
  const supabaseServiceKey = process.env.SUPABASE_SERVICE_ROLE_KEY!;

  if (!supabaseServiceKey) {
    throw new Error('SUPABASE_SERVICE_ROLE_KEY is required for admin operations');
  }

  return createClient(supabaseUrl, supabaseServiceKey, {
    auth: {
      autoRefreshToken: false,
      persistSession: false,
    },
  });
}
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles: `cd dws-app && npm run build`
- [ ] No linting errors: `cd dws-app && npm run lint`
- [ ] Endpoint exists: `curl -X POST http://localhost:3000/api/agent/email-webhook` returns error (not 404)

#### Manual Verification:
- [ ] Send test email to AgentMail inbox → webhook receives request (check Vercel logs)
- [ ] Email from known user → receipt created in database
- [ ] Email from unknown user → appropriate error logged
- [ ] Multiple attachments → multiple receipts created

**Implementation Note**: Pause here to test the full email-to-receipt flow before adding UI enhancements.

---

## Phase 4: Admin Dashboard Enhancement (Optional)

### Overview
Add submission source indicator to admin receipt views.

### Changes Required

#### 1. Update receipt display
**File**: `dws-app/src/components/receipt-dashboard.tsx`

Add a badge or indicator showing "Email" vs "App" for each receipt's submission source.

```typescript
// In the receipt row/card component, add:
{receipt.submission_source === 'email' && (
  <Badge variant="outline" className="ml-2">
    <Mail className="h-3 w-3 mr-1" />
    Email
  </Badge>
)}
```

#### 2. Add filter option (optional)
Allow admins to filter by submission source in the dashboard.

### Success Criteria

#### Manual Verification:
- [ ] Email-submitted receipts show "Email" badge
- [ ] App-submitted receipts show no badge (or "App" badge)
- [ ] Filter works if implemented

---

## Testing Strategy

### Unit Tests
- Mock AgentMail client and test `processEmailReceipt` function
- Test user lookup with known/unknown emails
- Test attachment filtering (valid types, size limits)
- Test OCR failure handling (partial data)

### Integration Tests
- Send real email to test inbox
- Verify receipt appears in database with correct fields
- Verify confirmation email is received
- Test multiple attachments in single email

### Manual Testing Steps
1. Add your email address to your user profile in Supabase
2. Send email with single receipt image → verify receipt created
3. Send email with 3 receipt images → verify 3 receipts created
4. Send email from unknown address → verify error reply
5. Send email with no attachments → verify "no attachments" reply
6. Send email with invalid file type (e.g., .txt) → verify it's skipped
7. Check admin dashboard shows email-submitted receipts

## Performance Considerations

- **Webhook timeout**: AgentMail expects quick responses. Processing happens synchronously but should complete within 30 seconds. For very large batches, consider queueing.
- **Image compression**: Sharp processing adds latency but reduces storage costs and OCR processing time.
- **Concurrent attachments**: Process attachments sequentially to avoid overwhelming BAML/OCR service.

## Migration Notes

- **Existing receipts**: Will have `submission_source: 'app'` by default (via DEFAULT constraint)
- **User emails**: Admins need to manually add email addresses to user profiles before users can submit via email
- **Rollback**: Remove webhook from AgentMail dashboard; email endpoint becomes a no-op

## Security Considerations

1. **Webhook verification**: Consider adding signature verification for AgentMail webhooks
2. **Rate limiting**: Add rate limiting to webhook endpoint to prevent abuse
3. **Email validation**: Normalize email addresses (lowercase) for consistent matching
4. **File type validation**: Double-check content-type matches actual file content

## References

- Original research: `thoughts/shared/2025-12-17-agentmail-email-receipt-agent.md`
- Receipt upload pattern: `dws-app/src/app/api/receipts/upload/route.ts`
- OCR extraction: `dws-app/src/app/api/receipts/ocr/route.ts`
- Receipt creation: `dws-app/src/app/api/receipts/route.ts`
- AgentMail docs: https://docs.agentmail.to
- AgentMail SDK: https://www.npmjs.com/package/agentmail

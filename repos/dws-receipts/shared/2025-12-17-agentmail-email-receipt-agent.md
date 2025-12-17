---
date: 2025-12-17T00:00:00-08:00
researcher: claude
git_commit: c063e6e036b3b95555aecdb978e43dfb90c511f3
branch: test-branch
repository: DWS-Receipts
topic: "AgentMail Integration for Email-Based Receipt Processing"
tags: [research, agentmail, email-agent, receipts, automation]
status: complete
last_updated: 2025-12-17
last_updated_by: claude
---

# Research: AgentMail Integration for Email-Based Receipt Processing

**Date**: 2025-12-17
**Researcher**: claude
**Git Commit**: c063e6e036b3b95555aecdb978e43dfb90c511f3
**Branch**: test-branch
**Repository**: DWS-Receipts

## Research Question

How can we set up an AI agent using AgentMail to receive forwarded emails with receipt attachments and automatically upload them to the DWS-Receipts system, effectively automating the admin's role?

## Summary

This research explores integrating AgentMail as an email-receiving AI agent that can process receipt submissions from carpenters who prefer emailing receipts rather than using the mobile app. The integration would create a dedicated email inbox that receives forwarded emails, extracts receipt attachments, processes them through the existing OCR pipeline, and creates receipt records in the database.

## AgentMail Platform Overview

### What is AgentMail?
AgentMail is "The First Email Provider For AI Agents" - an API-first email platform designed specifically for autonomous systems. Key capabilities:

- **Send & Receive Emails**: Full bidirectional email communication
- **Email Threading**: Organized conversation management
- **Unlimited Inboxes**: Scale with usage-based pricing
- **Smart Parsing**: Automatic extraction of email content, HTML bodies, and attachments
- **Attachment Parsing**: AI-native feature for document handling
- **API Key Authentication**: Simple auth without OAuth complexity

### Technical Details

**SDK Installation**:
```bash
npm i -s agentmail
```

**Client Initialization**:
```typescript
import { AgentMailClient } from "agentmail";
const client = new AgentMailClient({ apiKey: "YOUR_API_KEY" });
```

**Key SDK Methods**:
- `client.inboxes.create()` - Create new inbox
- `client.inboxes.list()` - List existing inboxes
- Pagination support with async iteration
- TypeScript types under `AgentMail` namespace
- Automatic retries with exponential backoff

**Runtime Support**: Node.js 18+, Vercel, Cloudflare Workers, Deno, Bun, React Native

**Pricing**: Usage-based (free to sign up)

**Documentation**: https://docs.agentmail.to
**Console**: https://console.agentmail.to

## Current Receipt System Architecture

### Receipt Creation Flow (Mobile App)

1. **File Upload** (`/api/receipts/upload`)
   - User selects image via mobile app
   - Image compressed with Sharp (max 1200x1200px, 80% JPEG quality)
   - Uploaded to Supabase Storage bucket `receipt-images`
   - Returns temporary file path

2. **OCR Processing** (`/api/receipts/ocr`)
   - Downloads image from Supabase Storage
   - Calls Google Vision API for text detection
   - Parses date using regex patterns (supports multiple formats)
   - Extracts amount from total lines
   - Returns extracted `{ date, amount }`

3. **User Confirmation** (Mobile UI)
   - Shows extracted date/amount (editable)
   - User selects category from dropdown
   - User adds optional notes/description

4. **Receipt Creation** (`/api/receipts` POST)
   - Validates required fields
   - Checks for duplicates (same date/amount/user)
   - Creates receipt record with status "Pending"
   - Moves file from temp path to final path

### Database Schema

**receipts table**:
- `id`: uuid (primary key)
- `user_id`: uuid (foreign key to auth.users)
- `receipt_date`: date
- `amount`: numeric
- `category_id`: uuid (foreign key to categories)
- `description`: text
- `status`: 'Pending' | 'Approved' | 'Rejected' | 'Reimbursed'
- `image_url`: text (Supabase Storage path)
- `created_at`, `updated_at`: timestamps

**user_profiles table**:
- `user_id`: uuid (primary key)
- `role`: 'employee' | 'admin'
- `full_name`: text ("LastName, FirstName" format)
- `preferred_name`: text
- `employee_id_internal`: text

### Admin Dashboard Workflow

Located at `/dashboard` (admin only):
- Fetches receipts via `/api/admin/receipts`
- Filters by status, date range
- Search by employee name or description
- View receipt images
- Approve/Reject individual receipts
- Bulk update Approved → Reimbursed
- Export CSV for payroll

## Proposed Email Agent Integration

### High-Level Architecture

```
┌─────────────────────┐     ┌──────────────────────┐
│  Carpenter Email    │────▶│  AgentMail Inbox     │
│  (forwarded receipt)│     │  receipts@domain.com │
└─────────────────────┘     └──────────┬───────────┘
                                       │
                                       ▼
                            ┌──────────────────────┐
                            │  Email Agent         │
                            │  (webhook/polling)   │
                            └──────────┬───────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              ▼                        ▼                        ▼
   ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
   │ Identify User    │    │ Extract          │    │ Download         │
   │ (email→user map) │    │ Attachment       │    │ Attachment       │
   └────────┬─────────┘    └────────┬─────────┘    └────────┬─────────┘
            │                       │                        │
            └───────────────────────┼────────────────────────┘
                                    ▼
                         ┌──────────────────────┐
                         │ Process via OCR      │
                         │ (existing pipeline)  │
                         └──────────┬───────────┘
                                    ▼
                         ┌──────────────────────┐
                         │ Create Receipt       │
                         │ (default category)   │
                         └──────────┬───────────┘
                                    ▼
                         ┌──────────────────────┐
                         │ Send Confirmation    │
                         │ Email Response       │
                         └──────────────────────┘
```

### Integration Components

#### 1. AgentMail Inbox Setup
- Create dedicated inbox (e.g., `receipts@yourcompany.agentmail.to` or custom domain)
- Configure inbox settings for attachment handling

#### 2. Email Processing Options

**Option A: Webhook-Based (Recommended)**
- AgentMail sends POST request to `/api/agent/email-webhook` on new email
- Real-time processing
- Requires publicly accessible endpoint

**Option B: Polling-Based**
- Cron job or scheduled function polls AgentMail for new emails
- Works behind firewalls
- Slight delay in processing

#### 3. User Identification Challenge

**Current System**: Users identified by phone number (Supabase Auth phone login)

**Challenge**: Emails don't contain phone numbers - need to map sender email to user

**Possible Solutions**:

1. **Add email to user_profiles table**
   - New column: `email` in user_profiles
   - Admin maintains email-to-user mapping
   - Agent looks up user by sender email

2. **Email Registration Flow**
   - Users send initial email with their phone number
   - System creates mapping automatically
   - Future emails recognized by address

3. **Phone Number in Email Body**
   - Instruct users to include phone in email
   - Agent parses phone from email body
   - Match against auth.users.phone

4. **Reply-to Verification**
   - Agent replies asking for verification
   - User confirms identity via link or code

#### 4. Category Assignment

**Options**:
1. **Default Category**: Assign all email receipts to default category (e.g., "Other")
2. **AI Classification**: Use LLM to infer category from receipt text/vendor
3. **User Specified**: User includes category in email subject/body
4. **Admin Review**: Leave category blank, admin assigns during review

#### 5. New API Endpoint: `/api/agent/process-email`

```typescript
// Pseudo-code for email processing endpoint
POST /api/agent/process-email
{
  "from": "carpenter@email.com",
  "subject": "Receipt from Home Depot",
  "body": "Please process this receipt",
  "attachments": [
    {
      "filename": "receipt.jpg",
      "contentType": "image/jpeg",
      "content": "<base64 or URL>"
    }
  ]
}

Response:
{
  "success": true,
  "receiptId": "uuid",
  "extractedData": {
    "date": "2024-12-15",
    "amount": 45.99
  },
  "message": "Receipt processed successfully"
}
```

#### 6. Confirmation Emails

Agent should reply to confirm:
- **Success**: "Your receipt from {date} for ${amount} has been submitted successfully."
- **Failure - No User**: "We couldn't find your account. Please include your phone number."
- **Failure - No Attachment**: "No receipt image found. Please attach a photo of your receipt."
- **Failure - OCR Failed**: "We couldn't read the receipt. It has been submitted for manual review."

### Technical Implementation Considerations

#### Database Changes Required

```sql
-- Add email column to user_profiles
ALTER TABLE user_profiles ADD COLUMN email TEXT;
CREATE INDEX idx_user_profiles_email ON user_profiles(email);

-- Optional: Track email-submitted receipts
ALTER TABLE receipts ADD COLUMN submission_source TEXT DEFAULT 'app';
-- Values: 'app', 'email'
```

#### Environment Variables

```env
AGENTMAIL_API_KEY=your_api_key
AGENTMAIL_INBOX_ID=your_inbox_id
AGENTMAIL_WEBHOOK_SECRET=webhook_verification_secret
DEFAULT_CATEGORY_ID=uuid_of_default_category
```

#### Key Code Reuse Opportunities

| Existing Code | Reusable For |
|--------------|--------------|
| `/api/receipts/upload/route.ts` (lines 58-84) | Image compression with Sharp |
| `/api/receipts/ocr/route.ts` (lines 33-168) | Date/amount extraction logic |
| `/api/receipts/check-duplicate/route.ts` | Duplicate detection |
| `/api/receipts/route.ts` (lines 32-54) | Receipt database insertion |
| `supabaseServerClient.ts` | Supabase client setup |

#### Error Handling Scenarios

| Scenario | Agent Response |
|----------|----------------|
| Unknown sender email | Request phone number or registration |
| No attachment | Request receipt image |
| Invalid file type | Request supported format (JPEG, PNG, PDF) |
| OCR extraction failed | Submit anyway with notes, flag for review |
| Duplicate receipt | Inform user, don't create duplicate |
| System error | Apologize, suggest retry or use app |

### Security Considerations

1. **Webhook Verification**: Validate AgentMail webhook signatures
2. **Email Spoofing**: Consider SPF/DKIM verification of sender
3. **File Type Validation**: Only accept expected MIME types
4. **Rate Limiting**: Prevent abuse via excessive email submissions
5. **Attachment Size Limits**: Match existing 10MB limit

### Estimated Complexity

| Component | Complexity | Notes |
|-----------|------------|-------|
| AgentMail setup | Low | Simple SDK, good docs |
| Webhook endpoint | Medium | New API route, validation |
| User identification | Medium-High | Requires schema change, mapping logic |
| OCR integration | Low | Reuse existing code |
| Receipt creation | Low | Reuse existing code |
| Email responses | Low | AgentMail SDK handles sending |
| Error handling | Medium | Many edge cases |

## Alternative Approaches

### 1. SMS-Based Submission
Instead of email, use Twilio/similar for SMS-based receipt submission. Advantage: Already have phone→user mapping.

### 2. WhatsApp Business API
Similar to SMS but with richer media support. Users send receipt photos via WhatsApp.

### 3. Email Forwarding to App
Rather than full automation, email triggers a magic link that opens the app with pre-filled data.

## Open Questions

1. **User Mapping Strategy**: Which approach to identify users from email? Add email column seems cleanest.

2. **Category Handling**: Default category vs AI classification vs manual admin review?

3. **Webhook vs Polling**: Is the app hosted on a platform with public endpoints (Vercel) or behind a firewall?

4. **Volume Expectations**: How many email submissions expected? Affects architecture decisions.

5. **Reply Requirements**: Should agent engage in back-and-forth conversation or just acknowledge?

6. **Partial Data Handling**: If OCR gets date but not amount (or vice versa), submit anyway or reject?

7. **Admin Notification**: Should admin be notified of email-submitted receipts differently?

## Next Steps

1. **Sign up for AgentMail** at https://console.agentmail.to
2. **Create test inbox** and experiment with SDK
3. **Decide on user identification strategy** (recommend: add email to user_profiles)
4. **Implement webhook endpoint** `/api/agent/email-webhook`
5. **Create email processing service** that reuses existing OCR/receipt code
6. **Build email response templates** for success/failure scenarios
7. **Test end-to-end** with real email submissions

## Code References

- `dws-app/src/app/api/receipts/upload/route.ts` - Image upload and compression
- `dws-app/src/app/api/receipts/ocr/route.ts` - OCR extraction logic
- `dws-app/src/app/api/receipts/route.ts` - Receipt CRUD operations
- `dws-app/src/app/api/receipts/check-duplicate/route.ts` - Duplicate detection
- `dws-app/src/app/api/admin/receipts/route.ts` - Admin receipt fetching
- `dws-app/src/components/receipt-dashboard.tsx` - Admin dashboard UI
- `dws-app/src/lib/supabaseServerClient.ts` - Supabase server client setup
- `dws-app/src/lib/types.ts` - Receipt type definitions

## External Resources

- AgentMail Homepage: https://agentmail.to
- AgentMail Documentation: https://docs.agentmail.to
- AgentMail Console: https://console.agentmail.to
- AgentMail Node SDK: https://github.com/agentmail-to/agentmail-node
- npm package: `agentmail`

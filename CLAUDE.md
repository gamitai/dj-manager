# Master PRD, System Description & Development Plan: DJ Booking & Client Autopilot

> Working notes: track anything skipped or deferred in `gaps.md`. Update it whenever a requirement is skipped, simplified, or found to be ambiguous.

## 1. Product Description & Vision

DJ Autopilot is a two-sided client management, contract, and payment automation platform built specifically for self-employed event and wedding DJs.

- **The DJ Experience (Mobile App):** A fast, mobile-first workspace (React Native / Expo or Mobile-First PWA) designed for DJs on the move: between soundchecks, venue walkthroughs, and client meetings. It centralizes inquiry scheduling, 60-second offer creation, e-signature tracking, Google Calendar/Drive syncing, and deposit/post-event balance tracking.
- **The Client Experience (Zero-App WhatsApp Flow):** Clients never download an app or create an account. Every touchpoint (office arrival directions, opening a mobile offer and signing it "DocuSign-style", receiving the final signed PDF, meeting/payment reminders, and review links 10 days after the event) happens natively inside WhatsApp, with email as an optional backup copy.

## 2. End-to-End User Journey & Functional Requirements

### Module 1: Onboarding & DJ Workspace Settings (Mobile App)

- **Google Workspace OAuth Connection:** DJ signs in with Google and grants scoped permissions for Google Calendar (`calendar.events`, `calendar.readonly`) and Google Drive (`drive.file`).
- **Office & Meeting Setup:** DJ configures reusable office arrival instructions:
  - Street address + Google Maps / Waze navigation link.
  - Parking notes, building/floor code, and arrival buffer instructions.
- **Offer & Contract Template Setup:** DJ sets default contract clauses (cancellation policy, overtime hourly rate, sound/lighting setup requirements, payment terms) and default package pricing.
- **Post-Event Review Links:** DJ configures one or more review destinations (Google Business Profile review link, Mit4Mit / wedding directory link, or Instagram handle) to be sent automatically 10 days after each gig.

### Module 2: Meeting Scheduling & Office Directions (Calendar + WhatsApp)

- **Meeting Creation:** DJ picks a date/time in the mobile app (checking real-time free/busy slots against Google Calendar) and enters the client's name, WhatsApp phone number, optional email, and tentative event date.
- **Automated Calendar & WhatsApp Dispatch:**
  1. Creates the meeting event on the DJ's Google Calendar.
  2. Immediately sends a WhatsApp confirmation to the client containing the meeting date/time and the saved office arrival instructions.
- **Automated Pre-Meeting Reminder:** Sends an automated WhatsApp reminder (e.g., 24 hours and/or 2 hours before the meeting) with the office address and pin link.

### Module 3: Offer Builder & Mobile "DocuSign" E-Signature (App + WhatsApp)

- **60-Second Mobile Offer Builder:** After (or without) a meeting, the DJ creates an offer inside the mobile app:
  - Event details: Event type (Wedding, Party, Corporate, etc.), Event Date, Start/End hours, Venue name.
  - Financial breakdown: Package base price, add-ons (ceremony sound, extra hours, lighting), Total Fee, Down Payment / Deposit Amount, and Remaining Balance (auto-calculated).
- **WhatsApp Signing Link Dispatch:** System generates a unique, cryptographically signed, mobile-optimized web link (`https://app.domain.com/sign/[token]`) and sends it to the client via WhatsApp.
- **Mobile Signing Experience (Client Side):**
  - Clean, mobile-responsive contract view displaying all event terms, pricing, deposit rules, and legal clauses.
  - Client enters full name/ID (optional) and draws their signature on a touch canvas, then taps "Approve & Sign".
- **Automated Post-Signature Chain (Triggered Immediately on Sign):**
  1. **PDF Generation:** Renders a tamper-evident signed PDF embedding the client's signature image, timestamp, IP address, and document hash.
  2. **Client Delivery:** Sends the signed PDF directly to the client on WhatsApp (and Email if an email address was provided).
  3. **Google Drive Archival:** Creates a dedicated folder in the DJ's Google Drive (e.g., `/DJ Bookings/YYYY-MM-DD - Client Name - Venue/`) and uploads the signed PDF.
  4. **Google Calendar Gig Creation:** Creates the official event block on the DJ's Google Calendar with venue details, event hours, client phone number, and a link to the Drive contract.
  5. **DJ Push Notification:** Sends a push notification to the DJ's mobile app: "Contract signed by [Client Name] for [Event Date]!"

### Module 4: Two-Stage Payment Tracking (Deposit & Post-Event Balance)

- **Explicit Two-Stage Ledger:** Every booking tracks two independent financial milestones:
  1. Down Payment / Deposit (`deposit_status`): `unpaid` | `paid` (tracks expected amount, due date, and date received).
  2. Post-Event Remaining Balance (`balance_status`): `pending` | `paid` | `overdue` (tracks remaining amount due on/after the event date).
- **Mobile App Payment Dashboard:**
  - One-tap "Mark Deposit Paid" and "Mark Balance Paid" buttons on each booking card.
  - Quick filter tabs: `All Gigs`, `Awaiting Signature`, `Awaiting Deposit`, `Upcoming`, `Balance Due (Post-Event)`.
- **Automated Payment Nudges on WhatsApp:**
  - If `contract_status = 'signed'` and `deposit_status = 'unpaid'` after X days (configurable, e.g., 3 days), send a polite WhatsApp deposit reminder.
  - If `event_date` has passed (`NOW() > event_date + 24h`) and `balance_status != 'paid'`, flag the gig as `Balance Due` in the DJ's app and optionally trigger a polite post-event balance reminder to the client.

### Module 5: Automated WhatsApp Lifecycle Nudges & 10-Day Review Loop

- **Unsigned Contract Nudge:** If an offer is sent (`contract_status = 'sent'`) and unsigned after 48 hours, send a friendly WhatsApp check-in with the signing link.
- **Pre-Event Check-In (Optional Toggle):** 7 days before the event, ping the client on WhatsApp to confirm final timeline/music details.
- **10-Day Post-Event Review Request:** Exactly 10 days after `event_date` (for completed events), automatically send a warm WhatsApp message thanking the client and sharing the DJ's configured review links (e.g., Google Reviews, wedding directory).

## 3. System Guardrails & Technical Constraints

1. **WhatsApp Business Policy & Template Compliance:**
   - All business-initiated outbound messages sent outside a 24-hour customer-reply window (pre-meeting reminders, 48h signing nudges, deposit follow-ups, 10-day post-event review links) must use pre-approved WhatsApp Cloud API Message Templates with dynamic variables (`{{client_name}}`, `{{event_date}}`, `{{signing_url}}`, `{{review_url}}`).
2. **Idempotency & Anti-Spam Guard:**
   - Every automated WhatsApp message must be recorded in an `automation_logs` table with a `UNIQUE(booking_id, trigger_type)` constraint before or upon dispatch so cron retries never double-message a client.
   - DJ can toggle automation rules ON/OFF globally or per individual client (e.g., mute payment reminders for a friend/VIP client).
3. **E-Signature Integrity:**
   - Signing tokens must expire after 30 days or immediately upon signing (preventing re-signing/tampering).
   - Store raw signature vector/PNG, signer IP, user-agent, and UTC timestamp in the database alongside the generated PDF.
4. **Google API Resilience:**
   - Store encrypted Google OAuth `refresh_token` securely on the backend. Automatically refresh expired access tokens before calling Google Calendar or Google Drive APIs.
   - If a Google Drive upload or Calendar sync fails transiently, queue an automatic retry without blocking the client's signing confirmation screen.

## 4. Database Schema (PostgreSQL / Supabase)

```sql
-- 1. DJ Profiles & Configuration
CREATE TABLE djs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  full_name TEXT NOT NULL,
  business_name TEXT NOT NULL,
  phone_whatsapp TEXT,
  office_address TEXT,
  office_map_url TEXT,
  office_directions_note TEXT,
  default_contract_terms TEXT,
  review_links JSONB DEFAULT '[]', -- [{"platform": "Google", "url": "https://..."}]
  automation_settings JSONB DEFAULT '{
    "meeting_reminder_hours": 24,
    "unsigned_nudge_hours": 48,
    "deposit_nudge_days": 3,
    "post_event_balance_reminder_days": 2,
    "post_event_review_days": 10
  }',
  google_calendar_id TEXT DEFAULT 'primary',
  google_drive_root_folder_id TEXT,
  google_refresh_token_encrypted TEXT,
  expo_push_token TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2. Clients
CREATE TABLE clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  dj_id UUID REFERENCES djs(id) ON DELETE CASCADE,
  full_name TEXT NOT NULL,
  phone_whatsapp TEXT NOT NULL, -- E.164 format e.g. +972501234567
  email TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Bookings, Offers, Contracts & Payments
CREATE TABLE bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  dj_id UUID REFERENCES djs(id) ON DELETE CASCADE,
  client_id UUID REFERENCES clients(id) ON DELETE CASCADE,

  -- Event Info
  event_title TEXT NOT NULL,
  event_type TEXT, -- 'wedding' | 'party' | 'corporate' | 'other'
  event_date TIMESTAMPTZ NOT NULL,
  event_end_time TIMESTAMPTZ,
  venue_name TEXT,

  -- Discovery Meeting
  meeting_at TIMESTAMPTZ,
  meeting_calendar_event_id TEXT,

  -- Offer & E-Signature
  offer_items JSONB NOT NULL DEFAULT '[]', -- [{"label": "DJ Set (5 hrs)", "price": 5000}]
  contract_terms_snapshot TEXT,
  signing_token UUID DEFAULT gen_random_uuid() UNIQUE,
  contract_status TEXT NOT NULL DEFAULT 'draft', -- 'draft' | 'sent' | 'signed' | 'cancelled'
  contract_sent_at TIMESTAMPTZ,
  signed_at TIMESTAMPTZ,
  signature_audit JSONB, -- {"signer_name": "...", "ip": "...", "signature_image_url": "..."}

  -- Post-Sign External Sync IDs
  signed_pdf_drive_file_id TEXT,
  signed_pdf_drive_url TEXT,
  gig_calendar_event_id TEXT,

  -- Two-Stage Financial Tracking
  total_amount NUMERIC(10, 2) NOT NULL DEFAULT 0,
  deposit_amount NUMERIC(10, 2) NOT NULL DEFAULT 0,
  deposit_status TEXT NOT NULL DEFAULT 'unpaid', -- 'unpaid' | 'paid'
  deposit_paid_at TIMESTAMPTZ,

  balance_amount NUMERIC(10, 2) NOT NULL DEFAULT 0,
  balance_status TEXT NOT NULL DEFAULT 'pending', -- 'pending' | 'paid' | 'overdue'
  balance_paid_at TIMESTAMPTZ,

  -- Per-Booking Automation Override
  automations_paused BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 4. Idempotent Automation & WhatsApp Delivery Ledger
CREATE TABLE automation_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID REFERENCES bookings(id) ON DELETE CASCADE,
  trigger_type TEXT NOT NULL,
  -- 'meeting_confirmation' | 'meeting_reminder' | 'offer_sent' | 'unsigned_nudge' |
  -- 'signed_copy_delivery' | 'deposit_nudge' | 'balance_followup' | 'review_10d'
  channel TEXT NOT NULL, -- 'whatsapp' | 'email'
  provider_message_id TEXT,
  status TEXT DEFAULT 'sent', -- 'sent' | 'delivered' | 'read' | 'failed'
  sent_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(booking_id, trigger_type, channel)
);
```

## 5. Recommended Tech Stack

- **DJ Mobile App:** React Native (Expo) with TypeScript (iOS & Android, push notifications, fast local caching).
- **Client Signing Web Page:** Next.js (App Router) on Vercel (`/sign/[token]`): lightweight, instant mobile page opened directly in WhatsApp's in-app browser.
- **Backend & Database:** Supabase (PostgreSQL + Auth + Storage for temporary signature assets + Edge Functions / REST API).
- **PDF Generation:** `@react-pdf/renderer` or `pdf-lib` on the server to stamp the signature, date, and audit metadata into a clean PDF.
- **External Integrations:**
  - WhatsApp Business Cloud API (Meta), or Twilio WhatsApp API, for template messages, PDF delivery, and delivery/read receipt webhooks.
  - Google Calendar API & Google Drive API (`googleapis` Node SDK).
  - Resend / Postmark for optional email backup of signed contracts.
- **Scheduled Automation Engine:** GitHub Actions Cron or Supabase `pg_cron` running hourly to evaluate meeting reminders, unsigned nudges, deposit reminders, post-event balance checks, and the 10-day review trigger.

## 6. Phased Development Plan

### Phase 1: Foundation, Database & Google Workspace Sync (Week 1)

1. **Database & Backend Setup:** Initialize Supabase project, execute the SQL schema in Section 4, configure row-level security scoped to `dj_id`.
2. **Google OAuth & Token Vault:** Build Google Sign-In requesting `calendar.events`, `calendar.readonly`, and `drive.file`. Store encrypted refresh tokens and auto-create a root `DJ Bookings` folder in Google Drive.
3. **DJ Mobile App Shell (Expo):** Core navigation tabs (`Dashboard / Today`, `Bookings`, `New Offer`, `Settings`) and a Settings screen where the DJ saves office address, arrival instructions, default contract terms, and review links.

### Phase 2: Meeting Scheduling & WhatsApp Office Directions (Week 2)

1. **WhatsApp Cloud API Integration:** Set up Meta WhatsApp Business account, register approved message templates (meeting confirmation + directions, meeting reminder, offer signing link, signed PDF delivery, payment reminder, 10-day review request), and wire up webhook status listeners (`sent`, `delivered`, `read`).
2. **Meeting Scheduler Flow:** Mobile screen to schedule a client meeting:
   - Fetch busy slots from Google Calendar so the DJ avoids double-booking.
   - On save, create the Google Calendar meeting event and immediately trigger the WhatsApp confirmation + office arrival instructions.

### Phase 3: Mobile Offer Builder & "DocuSign" E-Signature Pipeline (Weeks 3-4)

1. **Offer Builder UI (Mobile App):** Fast form to select a client, set event date/venue, add package line items, calculate total/deposit/balance, and tap "Send via WhatsApp".
2. **Client Mobile Signing Web App (`/sign/[token]`):**
   - Clean, mobile-friendly contract summary + full terms.
   - HTML5 touch-canvas signature pad + "Sign & Confirm" button.
3. **Post-Signature Orchestration Worker:** When the client submits their signature, execute the pipeline atomically:
   - Stamp and generate the final signed PDF.
   - Upload PDF to the client's folder in the DJ's Google Drive.
   - Create the gig event on the DJ's Google Calendar (venue, hours, Drive PDF link).
   - Send the signed PDF to the client on WhatsApp (and Email if provided) + push notification to the DJ's phone.

### Phase 4: Deposit & Post-Event Balance Tracker (Week 5)

1. **Payment Ledger UI (Mobile App):** Visual pills on every booking (`Deposit: Unpaid / Paid`, `Balance: Pending / Paid / Overdue`).
2. **One-Tap Payment Updates:** "Mark Deposit Paid" and "Mark Balance Paid" actions with timestamp logging (`deposit_paid_at`, `balance_paid_at`) and automatic balance recalculation.
3. **Financial Overview Widget:** Summary card on the DJ's home screen: Total Collected This Month, Pending Deposits, Post-Event Balances Due.

### Phase 5: Automated WhatsApp Nudges & 10-Day Review Loop (Week 6)

1. **Scheduled Automation Worker (Hourly Cron):**
   - **Pre-Meeting Reminder:** Meetings within the next `meeting_reminder_hours` → WhatsApp reminder + office directions.
   - **Unsigned Contract Nudge:** `contract_status = 'sent'` older than `unsigned_nudge_hours` → WhatsApp nudge with signing link.
   - **Deposit Follow-Up:** `contract_status = 'signed'` AND `deposit_status = 'unpaid'` older than `deposit_nudge_days` → WhatsApp deposit reminder.
   - **Post-Event Balance Check:** `NOW() > event_date + post_event_balance_reminder_days` AND `balance_status != 'paid'` → mark `overdue` in DJ app and send polite WhatsApp balance follow-up.
   - **10-Day Post-Event Review Request:** `NOW() >= event_date + INTERVAL '10 days'` → WhatsApp thank-you message with the DJ's review links.
2. **Per-Booking Mute Toggle:** "Automations Active / Paused" switch in each booking's detail screen so the DJ stays in full control.

### Phase 6: QA, Edge-Case Testing & Beta Launch (Week 7)

1. **End-to-End Sandbox Testing:** Full lifecycle (book meeting → receive directions → create offer → sign on mobile → verify Drive PDF + Calendar gig → mark deposit paid → simulate +10 days → verify WhatsApp review message).
2. **Failure & Retry Verification:** Test expired Google tokens, invalid WhatsApp phone numbers, and duplicate cron runs to confirm zero duplicate messages.
3. **TestFlight / Internal App Track Rollout:** Deploy backend/web signing portal to production and distribute the mobile app to initial beta DJs.

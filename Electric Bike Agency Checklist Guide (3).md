# Electric Bike Agency Platform: Feature Guide & Implementation Checklist

## 1. System Architecture Overview

The system is built as a modern, decoupled architecture separating the client-facing presentation layer, the workflow automation engine, data persistence, and transactional communication services.

```
                      +------------------------------------------+
                      |         Decoupled Frontend               |
                      |   (Next.js / Vite React + Tailwind)      |
                      +--------------------+---------------------+
                                           |
                    REST / GraphQL APIs    | Auth & Session Tokens
                                           v
+-----------------------+     +------------+-------------+     +------------------------+
|   PostgreSQL / DB     |<--->|     Core API / Auth      |<--->|   Admin Dashboard      |
|  (Fleet, Users,       |     |   (Node.js / Supabase)   |     | (Role-based operations)|
|   Bookings, Reviews)  |     +------------+-------------+     +------------------------+
+-----------------------+                  |
                                           | Webhook Events
                                           v
                              +------------+-------------+
                              |      N8N Automation      |
                              |     (Workflow Engine)    |
                              +------------+-------------+
                                           |
                                           | Cloud API (HTTPS)
                                           v
                              +------------+-------------+
                              |    Meta Official API     |
                              |  (Transactional OTP &    |
                              |   Booking Confirmations) |
                              +--------------------------+
```

---

## 2. Decoupled Frontend Catalog

The public catalog delivers high-speed browsing, dynamic filtering, responsive previews, and conversion-optimized booking flows.

### Core Features
- **Hero & Featured Showcase**: Flagship e-bike models with range, top speed, motor wattage, and quick-reserve CTAs.
- **Model Catalog & Filtering**:
  - Filter by category (Commuter, Cargo, Mountain/All-Terrain, High-Speed Sport).
  - Filter by battery capacity, detachable battery options, charging time, and motor rating.
  - Price slider, EMI estimate calculator, and availability status (In-Stock, Pre-Order, Test Ride Only).
- **Interactive Product Detail Pages (PDP)**:
  - 360-degree interactive image viewer and spec breakdown (range per charge, dual disc brakes, regenerative braking).
  - Downloadable brochures and maintenance schedule sheets.
  - Live availability check across agency locations/showrooms.
- **Dual Booking Flows**:
  - **Showroom Test Ride Booking**: Date/time slot picker, driver license verification, agency location selection.
  - **Direct Purchase / Token Booking**: Variant & color selection, accessory bundle add-ons, advance payment deposit, delivery preference.

### Frontend Technical Checklist
- [ ] Implement responsive UI using React / Next.js with server-side caching for catalog assets.
- [ ] Connect state management (Zustand or React Query) for instant filter and search responses.
- [ ] Set up client-side form validation (Zod + React Hook Form) for phone numbers and booking inputs.
- [ ] Integrate SEO meta tags, OpenGraph images, and JSON-LD schema for local business and product listings.
- [ ] Add offline/service-worker fallback for smooth browsing on low-bandwidth mobile connections.

---

## 3. User Authentication & 'My Bookings' Portal

A lightweight, frictionless phone-number-driven authentication model ensuring buyers and test riders can view and manage their reservations without password fatigue.

### Core Features
- **Passwordless OTP Login**:
  - Secure phone number verification using Meta WhatsApp Official API (with SMS fallback).
  - Secure session storage using HTTP-only JWTs.
- **Customer Dashboard ('My Bookings')**:
  - **Active Bookings Card**: Confirmed test rides, token reservations, vehicle delivery dates, and assigned vehicle VIN/chassis number if allocated.
  - **Booking Lifecycle Tracker**: Step-by-step progress bar (Reserved -> Verification Complete -> Showroom Ready / In Transit -> Delivered).
  - **Self-Service Actions**: Reschedule appointment (up to 4 hours prior), cancel reservation, or change pickup branch.
  - **Digital Documentation**: Download booking confirmation receipts, test ride terms, and payment receipts.
  - **Booking History & Feedback**: Archived past rides/purchases with a direct prompt to leave a verified review.

### Customer Portal Checklist
- [ ] Implement rate-limiting on OTP requests (maximum 3 requests per 10 minutes per IP/Phone).
- [ ] Enforce session expiration and refresh token rotation.
- [ ] Build responsive booking summary cards with clear status badges (Pending, Confirmed, Completed, Cancelled).
- [ ] Provide one-click calendar sync (Google Calendar / Apple Calendar `.ics`) for scheduled test rides.
- [ ] Ensure strict authorization checks ensuring users can only fetch their own booking IDs.

---

## 4. Customer Reviews & Reputation Engine

Social proof and local buyer trust are critical for high-ticket electric vehicle sales. The review engine enforces verified authenticity.

### Core Features
- **Verified Buyer Reviews Only**: Review submission links are unlocked exclusively after a booking status transitions to `Completed` (either completed test ride or verified purchase).
- **Multi-Factor Rating Criteria**:
  - Vehicle performance (Range accuracy, acceleration, comfort).
  - Showroom & staff experience (Consultation quality, delivery timeliness).
  - After-sales support and maintenance service rating.
- **Media Uploads**: Option for customers to upload photos of their e-bike delivery or test drive.
- **Automated Moderation Workflow**:
  - Profanity and spam filtering at intake.
  - Review queue sent to Admin Dashboard for approval before public display.
  - Verified Customer badge displayed on approved reviews.

### Reviews System Checklist
- [ ] Set up database foreign keys linking each review to a verified `booking_id` and `user_id`.
- [ ] Implement image compression and secure S3/Cloud storage upload for review photos.
- [ ] Trigger an automated WhatsApp review request via N8N 24 hours post-delivery or post-test-ride.
- [ ] Add aggregate star rating breakdown component on model PDPs.

---

## 5. N8N Backend Automation Engine

N8N acts as the central orchestration engine connecting the web store, internal database, admin notifications, and Meta WhatsApp API.

```
[ Frontend Booking Submission ] 
              │
              ▼
   [ N8N Webhook Receiver ]
              │
              ├──► [ Validate Payload & Check Duplicate Bookings ]
              │
              ├──► [ Write Record to PostgreSQL Database ]
              │
              ├──► [ Call Meta Cloud API: Send WhatsApp Confirmation ]
              │
              ├──► [ Push Alert to Admin Telegram / Slack / Email ]
              │
              └──► [ Schedule 24h Pre-Appointment Reminder in Queue ]
```

### Key Automated Workflows
1. **Instant Booking Ingestion**:
   - Webhook receives booking request from decoupled frontend.
   - Generates unique reference code (e.g., `EB-2026-XXXX`).
   - Inserts record into database with status `CONFIRMED` or `PENDING_DEPOSIT`.
2. **Meta WhatsApp Dispatcher**:
   - Compiles template payload parameters.
   - Authenticates against Meta Graph API and triggers official template message.
   - Handles delivery status callbacks (`sent`, `delivered`, `read`) via webhook.
3. **Automated Reminders & Scheduling**:
   - Test ride reminder dispatched 24 hours and 2 hours before the appointment window.
   - Follow-up automation for missed test rides inviting easy rescheduling.
4. **CRM & Lead Sync**:
   - Automatically synchronizes buyer interest data into Google Sheets or CRM database for agency sales reps.
5. **Error Handling & Dead-Letter Queue**:
   - Automatic retry mechanism (exponential backoff) for third-party network failures.
   - Real-time error alerts routed to the agency developer/admin.

### N8N Technical Checklist
- [ ] Host N8N on a persistent Docker container with SSL/TLS encryption.
- [ ] Protect all webhook entry points with shared HMAC secret tokens or Bearer auth.
- [ ] Implement idempotency checks to prevent duplicate booking confirmations if webhooks retry.
- [ ] Store environment credentials (Meta Access Tokens, Database URI, Encryption Keys) in secure N8N credential managers.
- [ ] Set up workflow execution log pruning to maintain lean database storage.

---

## 6. Meta WhatsApp Official API Integration

Strictly reserved for mission-critical transactional messaging. Avoids third-party gateway bans and complies with Meta Business Platform policies.

### Strict Scope of Use
- **Authentication**: High-priority one-time passcodes (OTP) for customer authentication.
- **Booking Confirmations**: Immediate confirmation upon token payment or test-ride reservation.
- **Schedule Updates**: Automated notices if slot is rescheduled or agency location updates.
- **Service & Delivery Alerts**: Milestone updates (Vehicle Dispatched, Ready for Showroom Pickup).

> **Policy Guardrail**: No promotional broadcasts or unsolicited marketing messages are sent through this channel. All communications utilize Meta pre-approved utility/authentication templates.

### Pre-Approved Message Templates

| Template Name | Category | Purpose | Variable Parameters |
| :--- | :--- | :--- | :--- |
| `ebike_auth_otp` | AUTHENTICATION | Login / Verification OTP | `{{1}}` (Code), `{{2}}` (Validity mins) |
| `test_ride_confirmed` | UTILITY | Test ride schedule details | `{{1}}` (Customer Name), `{{2}}` (Model), `{{3}}` (Date/Time), `{{4}}` (Showroom Location), `{{5}}` (Booking Ref) |
| `booking_token_received` | UTILITY | Token payment confirmation | `{{1}}` (Customer Name), `{{2}}` (Model & Color), `{{3}}` (Token Amount), `{{4}}` (Booking Ref) |
| `test_ride_reminder` | UTILITY | 24h & 2h appointment alert | `{{1}}` (Customer Name), `{{2}}` (Time), `{{3}}` (Agency Address), `{{4}}` (Contact Phone) |

### Meta API Integration Checklist
- [ ] Complete Meta Business Manager verification and register official WhatsApp Business Account (WABA).
- [ ] Submit message templates for approval under `AUTHENTICATION` and `UTILITY` categories.
- [ ] Configure System User with permanent access token and minimum required scopes (`whatsapp_business_messaging`).
- [ ] Implement webhook receiver for status updates (`sent`, `delivered`, `read`, `failed`).
- [ ] Transition development testing off unofficial gateways (e.g., Evolution API / OpenWA) directly to Meta Cloud API sandbox before staging deployment.

---

## 7. Admin Dashboard & Fleet Operations

An intuitive management control panel enabling agency staff to track operations, update vehicle status, and oversee leads.

### Core Modules
- **Real-Time Booking Pipeline**:
  - Filter by date range, vehicle model, agency branch, and booking status.
  - Status transition controls: `Pending` -> `Confirmed` -> `Completed` / `No-Show` / `Cancelled`.
  - Instant manual rescheduling and customer notes log.
- **Fleet & Inventory Manager**:
  - Track test-ride demo vehicles vs. ready-for-sale inventory.
  - Mark demo bikes as `Available`, `Out on Test Ride`, or `Under Maintenance`.
  - Battery health tracking and odometer logging for test vehicles.
- **Customer Review Moderation**:
  - Moderation queue showing star rating, buyer identifier, and review text.
  - Approve, reject, or request edit actions with one click.
- **Activity Tracker & Audit Logs**:
  - Real-time audit trail of admin actions (status changes, price updates, review approvals).
  - Webhook delivery logs and communication status history.
- **Operational Metrics**:
  - Total test rides completed, conversion percentage to purchase, top-performing bike models, and customer satisfaction rating.

### Admin Dashboard Checklist
- [ ] Set up Role-Based Access Control (RBAC): Super Admin (Agency Owner), Sales Consultant, Fleet Technician.
- [ ] Implement search and CSV/Excel export for bookings and customer lists.
- [ ] Add instant desktop notification sound/alert when a new booking arrives.
- [ ] Enforce two-factor authentication (2FA) for all administrative logins.

---

## 8. Deployment, Infrastructure & Maintenance Retainer

### Production Infrastructure Stack
- **Frontend Hosting**: Vercel / Cloudflare Pages (Edge-cached CDN, SSL).
- **Backend & Database**: Managed PostgreSQL (Supabase / AWS RDS / DigitalOcean Managed DB).
- **Automation Engine**: N8N self-hosted on lightweight VPS (Docker + Traefik reverse proxy) or N8N Cloud.
- **Object Storage**: AWS S3 or Cloudflare R2 for vehicle images, spec sheets, and customer review uploads.

### Ongoing Maintenance Retainer Model
To guarantee uninterrupted uptime, secure transactional delivery, and system integrity, the agency should be maintained under a monthly support retainer.

| Tier / Model | Recommended Pricing | Deliverables & Scope |
| :--- | :--- | :--- |
| **Standard Retainer** | ₹2,500 – ₹4,000 / month | Server uptime monitoring, security patches, DB daily backups, WhatsApp API token maintenance, up to 3 minor UI/text updates. |
| **Comprehensive Retainer** | ₹5,000 – ₹7,500 / month | All standard items + N8N workflow optimizations, monthly performance & conversion analytics report, bug fixes within 4-hour SLA, catalog seasonal promotions. |
| **Annual Retainer Option** | 10% – 15% of initial build | Annual contract paid quarterly; includes complete system health audits and priority developer access. |

### Retainer Scope Checklist
- [ ] Daily automated database snapshots with off-site backup storage.
- [ ] Monitoring for Meta WhatsApp API quota, delivery rates, and template health.
- [ ] Periodic package and security vulnerability updates across frontend and backend dependencies.
- [ ] Incident response SLA: Critical outages (< 2 hours), minor bug fixes (< 24 hours).
- [ ] Clear contract terms specifying that code and database management remain exclusively under authorized developer control to protect client data integrity.

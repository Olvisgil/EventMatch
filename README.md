# EventMatch — B2B Event Booking & Networking Platform
### Integration Guide · v1.0 · OG Technologies EU

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Technology Stack](#technology-stack)
4. [Feature Map](#feature-map)
5. [Infrastructure Setup](#infrastructure-setup)
   - [pretix (Ticketing)](#1-pretix-ticketing-backend)
   - [Cal.com (Scheduling)](#2-calcom-scheduling-backend)
   - [EventMatch Frontend](#3-eventmatch-frontend)
6. [API Reference](#api-reference)
   - [pretix REST API v1](#pretix-rest-api-v1)
   - [Cal.com API v2](#calcom-api-v2)
7. [End-to-End Integration Flow](#end-to-end-integration-flow)
8. [Attendee Matchmaking Engine](#attendee-matchmaking-engine)
9. [Webhook Event Catalogue](#webhook-event-catalogue)
10. [Environment Variables](#environment-variables)
11. [Data Model](#data-model)
12. [Security Considerations](#security-considerations)
13. [Troubleshooting](#troubleshooting)

---

## Overview

**EventMatch** is a self-hosted, API-first B2B event booking and intelligent networking platform designed for enterprise-grade professional events — such as the *Web3 & Fintech Summit Vienna 2026* and the *INATBA Blockchain Education Forum*.

It combines three layers:

| Layer | Role | Technology |
|---|---|---|
| Ticketing | Order management, payments, check-in | pretix (self-hosted) |
| Scheduling | 1:1 meeting booking & slot management | Cal.com (self-hosted) |
| Intelligence | Attendee matchmaking & profile analysis | Custom PostgreSQL + TF-IDF engine |

The frontend (`Frontend_Eventmatch-platform-VBW.html`) is a standalone, dependency-free HTML/JS/CSS file that serves as a fully interactive prototype and integration reference. It includes a live **API Console** explorer for both pretix and Cal.com endpoints.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   EventMatch Frontend                   │
│         (Single-file HTML/JS — no build step)           │
│                                                         │
│  Dashboard · Events · Booking · Matching · Slots        │
│  Meeting Modal · API Console · Setup Guide              │
└────────────┬──────────────────────┬────────────────────-┘
             │                      │
             ▼                      ▼
┌────────────────────┐   ┌──────────────────────────┐
│  pretix REST v1    │   │     Cal.com API v2        │
│  (Ticketing)       │   │     (Scheduling)          │
│                    │   │                           │
│  Organizers        │   │  Event Types              │
│  Events            │   │  Schedules                │
│  Items / Quotas    │   │  Slots                    │
│  Questions         │   │  Bookings                 │
│  Orders            │   │  Users                    │
│  Check-in          │   │  Webhooks                 │
│  Webhooks          │   │                           │
└────────┬───────────┘   └───────────┬──────────────┘
         │                           │
         └──────────┬────────────────┘
                    ▼
      ┌─────────────────────────┐
      │   EventMatch Backend    │
      │  (Webhook Handler /     │
      │   Matchmaking Engine)   │
      │                         │
      │  PostgreSQL DB          │
      │  TF-IDF Similarity      │
      │  Cal.com User Provision │
      └─────────────────────────┘
```

---

## Technology Stack

| Component | Technology | Notes |
|---|---|---|
| Frontend | Vanilla HTML5 / CSS3 / ES5 JavaScript | No framework, no build step |
| Ticketing | [pretix](https://pretix.eu) `standalone:stable` | Docker-deployed |
| Scheduling | [Cal.com](https://cal.com) (self-hosted) | Next.js 14 + PostgreSQL |
| Database | PostgreSQL 14+ | Used by both pretix & Cal.com |
| Cache / Queue | Redis 6+ | Required by pretix |
| Reverse Proxy | Nginx | SSL termination |
| Payments | Stripe (via pretix plugin) | SEPA, cards, iDEAL supported |
| Styling | CSS custom properties (design tokens) | Dark mode via `prefers-color-scheme` |
| Fonts | System UI stack + SF Mono / Fira Code | No external font dependencies |

---

## Feature Map

### 1. Dashboard
- KPI stats: Registrations, Meetings Booked, Match Rate, Revenue
- Upcoming event list with status badges (Live · Draft)
- Quick-action buttons: **Book** → Booking view, **API** → API Console

### 2. My Profile
Four sub-tabs:
- **Personal Info** — Name, title, company, bio, photo
- **Professional** — Sector, expertise tags, spoken languages
- **Matching Preferences** — Looking for (Investors / Partners / Clients / Advisors), availability toggle
- **Privacy** — Visibility controls per field

### 3. Events
List of managed events with organiser slug, date range, capacity, currency, and live/draft status.

### 4. Book Tickets
- Pre-filled attendee form (First name, Last name, Email, Organisation)
- Ticket type selector: **Standard Attendee** (€299) / **VIP / Speaker Pass** (€699)
- Booking confirmation code (e.g. `WEB3-A48X-2026`) with deep-link to Matching view

### 5. Attendee Matching (AI-Powered)
- 2×2 match card grid with match percentage (e.g. 94%, 88%, 82%, 79%)
- Match tags (DLT, ISO/TC 307, Payments, Standards, CBDC, etc.)
- Filter by tag or score threshold (90%+, Booked)
- One-click meeting request via modal

### 6. Meeting Slots
- Day-by-day slot grid (Apr 14–16, 2026)
- Slot states: **Available** / **Busy** (dashed) / **Booked** (green)
- Cal.com sync button
- Live meeting list with pending / confirmed status

### 7. Meeting Request Modal
- Attendee avatar + company info header
- Available time slot selector (30-min blocks)
- Optional message field
- Cal.com booking confirmation sent to both parties on submit

### 8. API Console
Interactive explorer for:
- **pretix REST API v1**: Organizers, Events, Items, Quotas, Questions, Orders, Check-in, Webhooks
- **Cal.com API v2**: Event Types, Schedules, Slots, Bookings, Webhooks, Users
- Colour-coded HTTP methods (GET · POST · PATCH · DELETE)
- Syntax-highlighted request headers, body, and 200 OK response

### 9. Setup Guide
Step-by-step instructions across three tabs:
- **pretix Setup** (7 steps)
- **Cal.com Setup** (6 steps)
- **Integration** (4 steps)

---

## Infrastructure Setup

### 1. pretix (Ticketing Backend)

#### Step 1 — Deploy via Docker

```bash
docker pull pretix/standalone:stable

docker run -d \
  --name pretix \
  -p 8000:80 \
  -e PRETIX_SITE_URL=https://tickets.yourdomain.com \
  -e PRETIX_DB_TYPE=postgresql \
  -e PRETIX_DB_HOST=your-pg-host \
  -e PRETIX_DB_NAME=pretix \
  -e PRETIX_DB_USER=pretix \
  -e PRETIX_DB_PASS=yourpassword \
  -e PRETIX_REDIS_LOCATION=redis://your-redis-host:6379/0 \
  pretix/standalone:stable
```

Requirements: PostgreSQL 14+, Redis 6+, Nginx as reverse proxy with SSL.

#### Step 2 — Generate API Token

1. Log in to the pretix Admin panel.
2. Navigate to **Your Organiser → API Keys → Create**.
3. Grant the following scopes:

| Scope | Purpose |
|---|---|
| `events:read` `events:write` | Create and manage events |
| `orders:read` `orders:write` | Access attendee registrations |
| `checkin:write` | Gate check-in integration |
| `items:write` | Create ticket types |
| `quotas:write` | Set capacity limits |
| `questions:write` | Add profile/matchmaking questions |

Use the token as: `Authorization: Token YOUR_PRETIX_TOKEN`

#### Step 3 — Create Organiser & Event

```http
POST /api/v1/organizers/
Authorization: Token YOUR_PRETIX_TOKEN
Content-Type: application/json

{
  "name": "OG Technologies EU",
  "slug": "ogtec",
  "public": true
}
```

```http
POST /api/v1/organizers/ogtec/events/
Content-Type: application/json

{
  "name": { "en": "Web3 & Fintech Summit Vienna 2026" },
  "slug": "web3-vienna-26",
  "date_from": "2026-04-14T09:00:00Z",
  "date_to": "2026-04-16T18:00:00Z",
  "currency": "EUR",
  "timezone": "Europe/Vienna",
  "is_public": true,
  "plugins": ["pretix.plugins.stripe"]
}
```

#### Step 4 — Add Ticket Items & Quotas

```http
POST /api/v1/organizers/ogtec/events/web3-vienna-26/items/
{
  "name": { "en": "Standard Attendee" },
  "default_price": "299.00",
  "active": true,
  "admission": true
}
```

```http
POST /api/v1/organizers/ogtec/events/web3-vienna-26/quotas/
{
  "name": "Standard capacity",
  "size": 500,
  "items": [12],
  "close_when_sold_out": true
}
```

#### Step 5 — Add Profile Questions (for Matchmaking)

Question types: `S` (string), `T` (text), `C` (choice), `M` (multi-choice)

```http
POST /api/v1/organizers/ogtec/events/web3-vienna-26/questions/
{
  "question": { "en": "Company" },
  "type": "S",
  "required": true,
  "items": [12]
}
```

Recommended matchmaking fields: **Company**, **Title**, **Industry**, **Interests**, **Looking For**.

#### Step 6 — Enable Stripe Payments

Set the following environment variables or pretix admin settings:

```env
PRETIX_STRIPE_PUBLISHABLE_KEY=pk_live_...
PRETIX_STRIPE_SECRET_KEY=sk_live_...
```

Supported methods: Credit/debit cards, SEPA Direct Debit, iDEAL.

#### Step 7 — Configure Webhooks

In the pretix Admin panel: **Webhooks → Add**

```json
{
  "target_url": "https://yourdomain.com/webhook/pretix",
  "enabled": true,
  "all_events": false,
  "limit_events": ["web3-vienna-26"],
  "action_types": [
    "pretix.event.order.paid",
    "pretix.event.order.refunded",
    "pretix.event.checkin"
  ]
}
```

---

### 2. Cal.com (Scheduling Backend)

#### Step 1 — Self-host Cal.com

```bash
git clone https://github.com/calcom/cal.com.git
cd cal.com
cp .env.example .env
# Edit .env with your values, then:
docker compose up -d
```

Required `.env` values:

```env
DATABASE_URL=postgresql://user:pass@host:5432/calcom
NEXTAUTH_SECRET=your-random-secret
NEXTAUTH_URL=https://cal.yourdomain.com
```

#### Step 2 — Create API Key

1. Go to **Settings → Developer → API Keys → Add**.
2. Key format: `cal_live_xxxxxxxx`
3. Pass on all requests as: `Authorization: Bearer cal_live_xxxxxxxx`

#### Step 3 — Create Meeting Event Type

```http
POST /v2/event-types
Authorization: Bearer cal_live_xxxxxxxx
Content-Type: application/json

{
  "title": "30-min B2B Meeting",
  "slug": "b2b-meeting",
  "length": 30,
  "schedulingType": "ROUND_ROBIN"
}
```

#### Step 4 — Set Availability Schedules

```http
POST /v2/schedules
{
  "name": "Web3 Summit Apr 14",
  "timeZone": "Europe/Vienna",
  "isDefault": true,
  "availability": [
    { "days": [1], "startTime": "09:00", "endTime": "12:00" },
    { "days": [1], "startTime": "13:00", "endTime": "17:00" }
  ]
}
```

> Block busy periods that conflict with event sessions (keynotes, panels) by excluding those windows from the schedule.

#### Step 5 — Create Bookings via API

```http
POST /v2/bookings
{
  "eventTypeId": 10,
  "start": "2026-04-14T10:00:00Z",
  "attendee": {
    "name": "Olvis Gil Ríos",
    "email": "olvis@ogtechnologies.eu",
    "timeZone": "Europe/Vienna"
  },
  "metadata": {
    "pretixOrderCode": "WEB3-A48X",
    "matchScore": "94"
  }
}
```

Include `pretixOrderCode` and `matchScore` in `metadata` for full traceability.

#### Step 6 — Subscribe to Cal.com Webhooks

```http
POST /v2/webhooks
{
  "subscriberUrl": "https://yourdomain.com/webhook/calcom",
  "triggers": [
    "BOOKING_CREATED",
    "BOOKING_CANCELLED",
    "BOOKING_RESCHEDULED"
  ],
  "active": true
}
```

---

### 3. EventMatch Frontend

The frontend is a **single self-contained HTML file** — no build pipeline, no npm, no framework.

#### Deployment Options

| Option | Command |
|---|---|
| Local preview | Open `Frontend_Eventmatch-platform-VBW.html` in any modern browser |
| Static host (nginx) | `cp Frontend_Eventmatch-platform-VBW.html /var/www/html/index.html` |
| Docker static server | `docker run -v $(pwd):/usr/share/nginx/html:ro -p 80:80 nginx:alpine` |
| GitHub Pages / Netlify | Drop the file in the repo root or `public/` folder |

> The file is 952 lines, fully functional in offline/demo mode. API Console calls and matchmaking data are mocked in JavaScript — no live backend is required to run the demo.

---

## API Reference

### pretix REST API v1

**Base URL:** `https://tickets.yourdomain.com`  
**Auth header:** `Authorization: Token YOUR_PRETIX_TOKEN`

| Resource | Method | Endpoint |
|---|---|---|
| List organizers | GET | `/api/v1/organizers/` |
| Create organizer | POST | `/api/v1/organizers/` |
| List events | GET | `/api/v1/organizers/{org}/events/` |
| Create event | POST | `/api/v1/organizers/{org}/events/` |
| Get event | GET | `/api/v1/organizers/{org}/events/{event}/` |
| Update event | PATCH | `/api/v1/organizers/{org}/events/{event}/` |
| Delete event | DELETE | `/api/v1/organizers/{org}/events/{event}/` |
| List items | GET | `/api/v1/organizers/{org}/events/{event}/items/` |
| Create item | POST | `/api/v1/organizers/{org}/events/{event}/items/` |
| List quotas | GET | `/api/v1/organizers/{org}/events/{event}/quotas/` |
| Create quota | POST | `/api/v1/organizers/{org}/events/{event}/quotas/` |
| Get availability | GET | `/api/v1/organizers/{org}/events/{event}/quotas/{id}/availability/` |
| List questions | GET | `/api/v1/organizers/{org}/events/{event}/questions/` |
| Create question | POST | `/api/v1/organizers/{org}/events/{event}/questions/` |
| List orders | GET | `/api/v1/organizers/{org}/events/{event}/orders/` |
| Create order | POST | `/api/v1/organizers/{org}/events/{event}/orders/` |
| Get order | GET | `/api/v1/organizers/{org}/events/{event}/orders/{code}/` |
| Refund order | POST | `/api/v1/organizers/{org}/events/{event}/orders/{code}/refund/` |
| Resend link | POST | `/api/v1/organizers/{org}/events/{event}/orders/{code}/resend_link/` |
| List check-in lists | GET | `/api/v1/organizers/{org}/events/{event}/checkinlists/` |
| Redeem (check in) | POST | `/api/v1/organizers/{org}/events/{event}/checkinlists/{list}/positions/{secret}/redeem/` |
| List positions | GET | `/api/v1/organizers/{org}/events/{event}/checkinlists/{list}/positions/` |
| List webhooks | GET | `/api/v1/organizers/{org}/webhooks/` |
| Create webhook | POST | `/api/v1/organizers/{org}/webhooks/` |

---

### Cal.com API v2

**Base URL:** `https://cal.yourdomain.com`  
**Auth header:** `Authorization: Bearer cal_live_xxxxxxxx`

| Resource | Method | Endpoint |
|---|---|---|
| List event types | GET | `/v2/event-types` |
| Create event type | POST | `/v2/event-types` |
| Update event type | PATCH | `/v2/event-types/{id}` |
| Delete event type | DELETE | `/v2/event-types/{id}` |
| Create schedule | POST | `/v2/schedules` |
| List schedules | GET | `/v2/schedules` |
| Get schedule | GET | `/v2/schedules/{id}` |
| Get available slots | GET | `/v2/slots/available?eventTypeId={id}&startTime=...&endTime=...` |
| Reserve slot | POST | `/v2/slots/reserve` |
| Create booking | POST | `/v2/bookings` |
| List bookings | GET | `/v2/bookings?attendeeEmail={email}&status=upcoming` |
| Get booking | GET | `/v2/bookings/{uid}` |
| Cancel booking | DELETE | `/v2/bookings/{uid}` |
| Reschedule booking | PATCH | `/v2/bookings/{uid}` |
| Create webhook | POST | `/v2/webhooks` |
| List webhooks | GET | `/v2/webhooks` |
| Create user | POST | `/v2/users` |
| Get user | GET | `/v2/users/{id}` |

---

## End-to-End Integration Flow

### Step 1 — Attendee Registers (pretix)

```
Attendee → EventMatch UI → POST /orders/ → pretix
  └─ stripe payment processed
  └─ pretix fires: pretix.event.order.paid webhook
```

### Step 2 — Webhook Handler: Provision Cal.com Account

```python
# Pseudo-code — webhook handler
@app.post("/webhook/pretix")
def handle_pretix_webhook(payload):
    if payload["action"] == "pretix.event.order.paid":
        order = payload["order"]
        attendee_email = order["email"]
        answers = extract_profile_answers(order)

        # Provision Cal.com scheduling account
        calcom_user = post("/v2/users", {
            "email": attendee_email,
            "name": answers["name"],
            "timeZone": "Europe/Vienna"
        })

        # Insert into EventMatch matchmaking DB
        db.insert("attendees", {
            "email": attendee_email,
            "company": answers["company"],
            "interests": answers["interests"],
            "industry": answers["industry"],
            "looking_for": answers["looking_for"],
            "calcom_user_id": calcom_user["id"],
            "pretix_order_code": order["code"]
        })

        # Recompute top-10 matches for this attendee
        recompute_matches(attendee_email)
```

### Step 3 — Matchmaking Score Computation

After each new attendee is inserted, the engine runs TF-IDF cosine similarity across `interests`, `industry`, and `looking_for` fields to produce a ranked list of top-10 matches per attendee. Scores are stored in a `matches` table and surfaced by the frontend's Matching view.

### Step 4 — Meeting Request Flow

```
Attendee A clicks "Request meeting" with Attendee B
  └─ EventMatch calls GET /v2/slots?eventTypeId=10&...
  └─ UI shows available 30-min slots
  └─ Attendee A selects slot → EventMatch calls POST /v2/bookings
      {
        eventTypeId: 10,
        start: "2026-04-14T10:00:00Z",
        attendee: { name: B.name, email: B.email, timeZone: "Europe/Vienna" },
        metadata: { matchScore: "94", pretixOrderCode: "WEB3-A48X" }
      }
  └─ Cal.com sends confirmation emails to both A and B
  └─ EventMatch updates slot status → "Booked"
  └─ Match card on UI updates to "Meeting booked" state
```

### Step 5 — Check-in Gate Activation

```
Attendee arrives at venue → pretix check-in scanner
  └─ POST /checkinlists/{id}/positions/{secret}/redeem/
  └─ pretix fires: pretix.event.checkin webhook
  └─ EventMatch webhook handler:
      - Activates attendee's Cal.com account
      - Slots become bookable by other checked-in attendees only
      - Updates attendee status to "on-site"
```

---

## Attendee Matchmaking Engine

### Data Inputs

The matchmaking engine reads from pretix order answers (questions configured in Step 5 of pretix setup):

| Field | Question Type | Used In Scoring |
|---|---|---|
| `company` | String (S) | Display only |
| `title` | String (S) | Display only |
| `industry` | Choice (C) | TF-IDF vector |
| `interests` | Multi-choice (M) | TF-IDF vector |
| `looking_for` | Multi-choice (M) | TF-IDF vector |

### Algorithm (TF-IDF Cosine Similarity)

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

def compute_matches(attendees):
    corpus = [
        f"{a['interests']} {a['industry']} {a['looking_for']}"
        for a in attendees
    ]
    vectorizer = TfidfVectorizer()
    tfidf_matrix = vectorizer.fit_transform(corpus)
    similarity = cosine_similarity(tfidf_matrix)
    return similarity  # NxN matrix, score 0.0–1.0 → multiply by 100 for %
```

Scores are displayed as percentages (e.g. 94%, 88%, 82%, 79%) and cached in PostgreSQL. Recomputed on each new registration.

---

## Webhook Event Catalogue

### pretix Events

| Event | Trigger | EventMatch Action |
|---|---|---|
| `pretix.event.order.paid` | Payment confirmed | Provision Cal.com user, insert into matchmaking DB |
| `pretix.event.order.refunded` | Refund processed | Deactivate Cal.com user, remove from matching pool |
| `pretix.event.checkin` | On-site check-in scan | Activate slot booking for checked-in attendees only |

### Cal.com Events

| Trigger | EventMatch Action |
|---|---|
| `BOOKING_CREATED` | Update slot status → Booked, add to meeting list UI |
| `BOOKING_CANCELLED` | Re-open slot, notify both parties, update UI |
| `BOOKING_RESCHEDULED` | Update slot time in EventMatch DB and UI |

---

## Environment Variables

### pretix

```env
PRETIX_SITE_URL=https://tickets.yourdomain.com
PRETIX_DB_TYPE=postgresql
PRETIX_DB_HOST=your-pg-host
PRETIX_DB_NAME=pretix
PRETIX_DB_USER=pretix
PRETIX_DB_PASS=your-db-password
PRETIX_REDIS_LOCATION=redis://your-redis-host:6379/0
PRETIX_STRIPE_PUBLISHABLE_KEY=pk_live_...
PRETIX_STRIPE_SECRET_KEY=sk_live_...
PRETIX_API_TOKEN=your-pretix-token          # For outbound calls from your backend
```

### Cal.com

```env
DATABASE_URL=postgresql://user:pass@host:5432/calcom
NEXTAUTH_SECRET=your-random-secret-32chars
NEXTAUTH_URL=https://cal.yourdomain.com
CALCOM_API_KEY=cal_live_xxxxxxxx             # For outbound calls from your backend
```

### EventMatch Backend

```env
EVENTMATCH_DB_URL=postgresql://user:pass@host:5432/eventmatch
PRETIX_WEBHOOK_SECRET=your-pretix-webhook-secret
CALCOM_WEBHOOK_SECRET=your-calcom-webhook-secret
PRETIX_BASE_URL=https://tickets.yourdomain.com
CALCOM_BASE_URL=https://cal.yourdomain.com
PRETIX_API_TOKEN=your-pretix-token
CALCOM_API_KEY=cal_live_xxxxxxxx
PRETIX_ORGANIZER_SLUG=ogtec
PRETIX_EVENT_SLUG=web3-vienna-26
CALCOM_EVENT_TYPE_ID=10
```

---

## Data Model

### `attendees` table (EventMatch PostgreSQL)

```sql
CREATE TABLE attendees (
  id                SERIAL PRIMARY KEY,
  email             TEXT UNIQUE NOT NULL,
  name              TEXT,
  company           TEXT,
  title             TEXT,
  industry          TEXT,
  interests         TEXT,           -- comma-separated tags
  looking_for       TEXT,           -- comma-separated tags
  pretix_order_code TEXT,
  calcom_user_id    INTEGER,
  checked_in        BOOLEAN DEFAULT FALSE,
  slots_active      BOOLEAN DEFAULT FALSE,
  created_at        TIMESTAMPTZ DEFAULT NOW()
);
```

### `matches` table

```sql
CREATE TABLE matches (
  attendee_a_id  INTEGER REFERENCES attendees(id),
  attendee_b_id  INTEGER REFERENCES attendees(id),
  score          NUMERIC(5,2),      -- 0.00–100.00
  updated_at     TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (attendee_a_id, attendee_b_id)
);
```

### `meetings` table

```sql
CREATE TABLE meetings (
  id              SERIAL PRIMARY KEY,
  attendee_a_id   INTEGER REFERENCES attendees(id),
  attendee_b_id   INTEGER REFERENCES attendees(id),
  calcom_uid      TEXT UNIQUE,
  start_time      TIMESTAMPTZ,
  duration_min    INTEGER DEFAULT 30,
  status          TEXT DEFAULT 'pending',  -- pending | confirmed | cancelled | rescheduled
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Security Considerations

1. **Verify webhook signatures** — Both pretix and Cal.com support HMAC-SHA256 signatures on webhook payloads. Always verify before processing.
2. **Rotate API tokens** — Store `PRETIX_API_TOKEN` and `CALCOM_API_KEY` in a secrets manager (e.g. HashiCorp Vault, AWS Secrets Manager). Never commit to source control.
3. **Rate limiting** — Apply Nginx rate limiting on `/webhook/*` endpoints to prevent replay attacks.
4. **HTTPS everywhere** — All endpoints (pretix, Cal.com, EventMatch) must use TLS. Use Let's Encrypt / Certbot for free SSL.
5. **Attendee data privacy** — Store only what is needed for matchmaking. Honour GDPR deletion requests by purging the `attendees`, `matches`, and `meetings` tables, and deleting the corresponding Cal.com user via `DELETE /v2/users/{id}`.
6. **Slot visibility** — Gate slot availability behind check-in status. Only checked-in attendees' slots should be bookable by other checked-in attendees (see Step 5 of Integration Flow).

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| pretix returns 401 | Invalid or missing API token | Check `Authorization: Token ...` header |
| Cal.com slots show empty | Schedule not created for event date | POST to `/v2/schedules` with correct `days` and time window |
| Webhook not firing | Webhook target URL unreachable | Ensure EventMatch backend is publicly accessible; check Nginx config |
| Match scores all zero | Matchmaking fields not set as required | Mark `interests`, `industry`, `looking_for` questions as `required: true` in pretix |
| Booking fails with 409 | Slot already reserved by another attendee | Check `/v2/slots/reserve` before booking; handle 409 gracefully in UI |
| Check-in not activating slots | Webhook handler not receiving `pretix.event.checkin` | Confirm `checkin` action type is listed in pretix webhook config |
| Dark mode not applying | Browser color scheme override | Check `prefers-color-scheme: dark` is not blocked by browser settings |
| Frontend API Console shows stale data | APIS object hardcoded in JS | Update `APIS` object in the `<script>` block at line ~794 |

---

## Resources

- pretix API Docs: https://docs.pretix.eu/en/latest/api/
- Cal.com API Docs: https://cal.com/docs/api-reference/v2/introduction
- pretix Docker Hub: https://hub.docker.com/r/pretix/standalone
- Cal.com GitHub: https://github.com/calcom/cal.com
- Stripe pretix Plugin: https://pretix.eu/about/en/plugins/stripe/
- scikit-learn TF-IDF: https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.TfidfVectorizer.html

---

*Built with ❤️ by OG Technologies EU GmbH · Vienna, Austria*  
*Contact: olvis@ogtechnologies.co · ogtechnologies.co*

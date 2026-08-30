# Design: host-cleaner-bridge

## Problem

Hosts with one or more Airbnb listings manually watch for guest checkouts and
text the cleaner on WhatsApp with the property and time. This is manual,
error-prone (missed or late messages), and doesn't scale past a couple of
properties.

## MVP scope (locked in)

- **Calendar source:** Airbnb only. Booking.com is a fast-follow once the
  core loop works.
- **Notification channel:** SMS via Twilio. See "Why not WhatsApp on day
  one" below.
- **Stack:** Next.js (TypeScript, App Router) + PostgreSQL + Prisma.

## Key constraint: how Airbnb calendar access actually works

There is no OAuth-based, self-serve API for reading a single host's Airbnb
calendar. The only realistic option without becoming a vetted Airbnb
channel-manager partner (a heavy application process aimed at PMS companies,
not viable for an MVP) is the **iCal export** every host already has:

Airbnb dashboard → Calendar → Availability settings → Sync calendars →
**Export calendar** → gives a private `.ics` URL.

Implications this design has to account for:

1. **Pull-only, no webhooks.** We must poll the `.ics` URL on an interval
   (starting at hourly — polling much more aggressively risks the URL being
   throttled or flagged).
2. **No exact checkout time in the feed.** Airbnb's calendar events are
   date-based ("blocked" from date A to date B), not timestamped to the
   actual checkout hour. The real checkout time is a per-listing house rule
   Airbnb enforces on their side, not data available on our side. → **The
   owner sets a default checkout time per property in our app**, and that's
   what drives message timing, not anything parsed from the feed.
3. **No guest identity in the feed**, by design (Airbnb strips guest name and
   contact info from the export for privacy). We don't need it — we only
   need the date range — but it means we can never show "guest info" in the
   unified calendar, only occupied/vacant blocks per property.
4. **Same booking can show adjacent blocked dates** (host-configured
   turnover buffers) that aren't actually part of the stay. Sync logic needs
   to treat the iCal `DTEND` as the checkout date and not assume every
   contiguous blocked range is one booking.
5. **The URL itself is a bearer credential** — anyone with it can read the
   full booking calendar. Store it encrypted at rest, never log it in full.

## Why not WhatsApp on day one

WhatsApp has no API for a personal/Business *app* account — nothing external
can send through it. The only sanctioned path is the official WhatsApp Cloud
API, which requires: a phone number dedicated to the API (can't stay active
in the regular app simultaneously), Meta Business verification (hours to
days), and pre-approved message templates for anything the host initiates
outside a live conversation window. None of that blocks an MVP built on SMS,
and the notification layer will be built channel-agnostic so WhatsApp is a
provider addition later, not a rewrite. (Unofficial WhatsApp automation
libraries exist but violate WhatsApp's ToS and risk the number being banned
— not something to build a real product on.)

## Architecture overview

```
┌─────────────┐      ┌──────────────────┐      ┌───────────────┐
│  Next.js app │◄────►│   PostgreSQL      │      │  Twilio (SMS) │
│  (web UI +   │      │   (Prisma)        │      └───────▲───────┘
│  API routes) │      └──────────────────┘              │
└──────┬───────┘               ▲                        │
       │                       │                        │
       │              ┌────────┴─────────┐      ┌───────┴───────┐
       │              │  Scheduled poller │─────►│ Notification  │
       └─────────────►│  (per-property     │      │ job (renders  │
                       │  iCal fetch)       │      │ template,     │
                       └────────────────────┘      │ sends SMS)    │
                                                    └───────────────┘
```

MVP-simplest path: no separate worker process. A scheduled job (e.g. Vercel
Cron hitting an internal endpoint, or `node-cron` in a single long-running
process) polls all active calendar connections, upserts bookings, and
enqueues notifications for checkouts that just passed and haven't been
messaged yet. Move to a real queue (BullMQ + Redis) only if reliability or
scale demands it.

## Data model

- **User** (host) — id, email, password hash, timezone
- **Property** — id, user_id, name, address, default_checkin_time,
  default_checkout_time, timezone
- **CalendarConnection** — id, property_id, platform (`airbnb`, later
  `booking_com`), ical_url (encrypted), last_synced_at, last_sync_status
- **Booking** — id, property_id, calendar_connection_id, external_uid (iCal
  UID, used to dedupe on re-sync), start_date, end_date, status
  (`upcoming` / `active` / `completed` / `cancelled`)
- **Cleaner** — id, user_id, name, phone_number, notification_channel
  (`sms` for now)
- **PropertyCleanerAssignment** — property_id, cleaner_id (many-to-many —
  a property can have a backup cleaner; a cleaner can serve several
  properties)
- **MessageRule** — id, property_id (nullable = account-wide default),
  lead_time_minutes (0 = at checkout time, negative = before, positive =
  after), template_text
- **MessageLog** — id, booking_id, cleaner_id, channel, status
  (`sent`/`failed`/`delivered`), provider_message_id, sent_at, error_message

## Message rules engine

- Template variables: `{{property_name}}`, `{{checkout_date}}`,
  `{{checkout_time}}`, `{{next_checkin_date}}`.
- `next_checkin_date` matters more than it sounds: if the next guest checks
  in the **same day**, the message should flag urgency — same-day turnovers
  are the case hosts most need automated reliably.
- Timing is configurable per property (default: send exactly at the
  property's configured checkout time).

## Web UI outline

- **Dashboard** — unified calendar, all properties overlaid, color-coded;
  upcoming-checkouts list for the next 7 days.
- **Properties** — CRUD; paste-in iCal URL with a short guided walkthrough
  (since it's copy-paste, not OAuth); set checkin/checkout times and
  timezone.
- **Cleaners** — CRUD; phone number; assign to one or more properties.
- **Message settings** — template editor with live preview, timing rule,
  a "send test message to myself" button.
- **Message log** — history with delivery status, so a host can confirm a
  message actually went out.
- **Manual send** — one-off "notify now" button per property, as a safety
  net if the automated sync is delayed or misses a booking.

## Stack detail

- Next.js (App Router, TypeScript)
- PostgreSQL + Prisma
- Auth.js (email/password to start)
- Twilio SDK for SMS
- Deploy: Vercel for the app; Vercel Cron (or a small always-on poller if
  Vercel's cron granularity is too coarse) for calendar polling
- Vitest for tests

## Open questions to resolve before building

1. Who pays for SMS costs — bundled into a subscription, or metered
   per-message to the host?
2. Do cleaners need an account at all, or is SMS-only (no login) enough for
   MVP? Leaning toward **no login for cleaners** — lowest possible friction.
3. Should message delivery failures (bad phone number, carrier block)
   trigger a fallback notification to the host, so a missed cleaning is
   never silent?

## Roadmap after MVP

- Booking.com iCal integration; double-booking detection across platforms
  on the unified calendar
- WhatsApp Cloud API as a second channel, once business-verified
- Two-way cleaner replies ("Done" / "Running late") via Twilio inbound
  webhook, feeding a live cleaning-status board
- Escalation to a backup cleaner if no reply within N minutes
- Multi-owner / property-manager team accounts
- No-login cleaner status page (magic link) to mark a job done, attach
  photos
- Timezone-correct scheduling per property (important once a host has
  properties across regions)

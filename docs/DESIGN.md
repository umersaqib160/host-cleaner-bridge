# Design: host-cleaner-bridge

## Problem

Hosts with one or more Airbnb listings manually watch for guest checkouts and
text the cleaner on WhatsApp with the property and time. This is manual,
error-prone (missed or late messages), and doesn't scale past a couple of
properties.

## MVP scope (locked in)

- **Calendar source:** Airbnb only. Booking.com is a fast-follow once the
  core loop works.
- **Notification channel:** Telegram-first with SMS as the onboarding and
  fallback channel (both via Twilio + the Telegram Bot API). See "Notification
  channel strategy" below.
- **Monetization:** platform pays for messaging costs; tiered subscription by
  property count with a capped free tier. See "Pricing" below.
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

## Notification channel strategy: Telegram-first, SMS fallback

Since the platform (not the host) pays for messaging, the channel choice is
also a cost-control decision, not just a UX one.

**Telegram** (via the free Bot API) is the target steady-state channel:
no business verification, no template approval, unlimited free messages once
a chat exists. The one constraint: **a bot cannot cold-message a user** — the
cleaner must open a chat with the bot once (tap an invite link, hit "Start")
before the bot can ever message them. That's a one-time opt-in, not a
recurring window like WhatsApp's — after it happens, messaging that cleaner
is free forever (or until they block the bot).

That constraint is solved with **SMS as the onboarding step, not just a
backup**:

1. Host adds a cleaner (name + phone number only — no cleaner login, ever).
2. App sends one SMS containing a unique Telegram deep link
   (`t.me/<bot_username>?start=<token>`).
3. Cleaner taps it once → `Cleaner.telegram_chat_id` gets set → all future
   checkout notifications go via Telegram, at zero marginal cost.
4. If a cleaner never completes step 3 (no Telegram, ignored the link),
   every checkout notification for them falls back to SMS indefinitely —
   reliability never depends on adoption, only cost does.

This means real-world SMS spend is roughly: one message per new cleaner
(onboarding) + ongoing messages only for cleaners who never adopt Telegram.
It also means Telegram-adoption rate is a metric worth tracking, since it
directly drives messaging cost down over time.

### Why not WhatsApp

WhatsApp has no API for a personal/Business *app* account — nothing external
can send through it. The only sanctioned path is the official WhatsApp Cloud
API, which requires: a phone number dedicated to the API (can't stay active
in the regular app simultaneously), Meta Business verification (hours to
days), and pre-approved message templates for anything the host initiates
outside a live conversation window. Telegram gets the same "free, rich
messaging" benefit without any of that setup cost, so it's the better choice
for now. (Unofficial WhatsApp automation libraries exist but violate
WhatsApp's ToS and risk the number being banned — not something to build a
real product on.) WhatsApp stays on the roadmap as an optional channel once
there's demand to justify the Business verification effort.

## Controlling messaging cost

The platform pays for every message, so cost control is a core design
requirement, not an afterthought:

- **Idempotency**: `MessageLog` has a unique constraint on
  (`booking_id`, `trigger_type`) — a duplicate poll run or a retried job can
  never send the same checkout notification twice.
- **Bounded retries**: a failed send retries a fixed number of times (e.g. 3)
  with backoff, then stops and raises a host alert — never an unbounded
  retry loop.
- **Per-plan message quotas**: every subscription tier includes a hard
  monthly message cap (see Pricing). This is the real backstop against a
  bug or abusive account producing a runaway bill — the ceiling is
  structural, not just monitored after the fact.
- **Twilio-side spend limits**: an account-level balance/spend alert on
  Twilio itself, independent of application logic, as a last line of
  defense.
- **Throttled test sends**: the "send test message to myself" button gets
  its own small daily cap, separate from real notification quota.

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
- **Cleaner** — id, user_id, name, phone_number, telegram_chat_id (nullable
  until opted in), telegram_invite_token, opted_in_at
- **PropertyCleanerAssignment** — property_id, cleaner_id (many-to-many —
  a property can have a backup cleaner; a cleaner can serve several
  properties)
- **MessageRule** — id, property_id (nullable = account-wide default),
  lead_time_minutes (0 = at checkout time, negative = before, positive =
  after), template_text
- **MessageLog** — id, booking_id, cleaner_id, channel (`telegram`/`sms`),
  trigger_type, status (`sent`/`failed`/`delivered`), provider_message_id,
  sent_at, error_message — unique on (booking_id, trigger_type) for
  idempotency
- **Plan** — id, name, max_properties, monthly_message_quota, price_cents
- **Subscription** — id, user_id, plan_id, status, messages_used_this_period

## Message rules engine

- Template variables: `{{property_name}}`, `{{checkout_date}}`,
  `{{checkout_time}}`, `{{next_checkin_date}}`.
- `next_checkin_date` matters more than it sounds: if the next guest checks
  in the **same day**, the message should flag urgency — same-day turnovers
  are the case hosts most need automated reliably.
- Timing is configurable per property (default: send exactly at the
  property's configured checkout time).
- Channel resolution per send: use Telegram if `telegram_chat_id` is set,
  otherwise SMS.
- **Failed delivery**: any `MessageLog` row that lands in `failed` status
  (after retries are exhausted) immediately emails the host and sets a
  persistent "needs attention" state on the dashboard until acknowledged —
  a missed cleaning should never be silent on the host's end.

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

## Pricing

The platform absorbs messaging cost, so the pricing model is also the
primary defense against a runaway bill — see "Controlling messaging cost"
above. **Recommendation: tiered subscription by property count, with a
capped free tier**, not a flat fee and not per-cleaner pricing.

Reasoning:

- Value scales with property count, not cleaner count. A host with 8
  properties gets far more coordination value than one with 1, so price
  should scale with them. A flat fee is either too expensive for the
  single-property host (the largest acquisition pool) or leaves money on
  the table for power users.
- This matches the mental model hosts already have from adjacent tools
  (Hospitable, Smoobu, Guesty all price per-listing) — no buyer education
  needed.
- Cleaners are deliberately **not** a pricing axis — charging per-cleaner
  would push hosts to skip adding a backup cleaner to save money, which
  works against the product's own reliability promise.

Starting ladder (numbers are a first draft, not final):

| Tier | Properties | Included messages/mo | Price |
|---|---|---|---|
| Free | 1 | 20 | $0 |
| Starter | up to 3 | 150 | ~$9–15/mo |
| Growth | up to 10 | 600 | ~$29–39/mo |
| Pro | unlimited | custom | contact / per-property overage rate |

Each tier's message quota is both a pricing lever and the structural cost
cap described above. Overage handling for the MVP: soft-block with an
upgrade prompt rather than metered overage billing — simpler to reason
about on both sides while the product is new. As Telegram adoption among
cleaners grows, actual cost-per-account trends down over time even as
included quotas stay the same, which improves margin without needing to
touch pricing.

## Roadmap after MVP

- Booking.com iCal integration; double-booking detection across platforms
  on the unified calendar
- WhatsApp Cloud API as an optional third channel, once there's demand to
  justify Business verification
- Two-way cleaner replies ("Done" / "Running late") via Telegram/SMS inbound
  webhook, feeding a live cleaning-status board
- Escalation to a backup cleaner if no reply within N minutes
- Multi-owner / property-manager team accounts
- No-login cleaner status page (magic link) to mark a job done, attach
  photos
- Timezone-correct scheduling per property (important once a host has
  properties across regions)
- Metered overage billing, if the flat soft-block approach proves too
  blunt once there's real usage data

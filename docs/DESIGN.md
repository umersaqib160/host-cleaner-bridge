# Design: host-cleaner-bridge

## Problem

Hosts with one or more Airbnb listings manually watch for guest checkouts and
text the cleaner on WhatsApp with the property and time. This is manual,
error-prone (missed or late messages), and doesn't scale past a couple of
properties.

## MVP scope (locked in)

- **Calendar source:** Airbnb only. Booking.com is a fast-follow once the
  core loop works.
- **Launch market:** Mexico. This drives several decisions below — channel
  choice, pricing level, and currency.
- **Notification channel:** SMS (Twilio) is the primary channel. Telegram is
  supported as an optional secondary channel per cleaner. See "Notification
  channel strategy" below.
- **Monetization:** platform pays for messaging costs; tiered subscription by
  property count with a capped free tier, priced for the Mexican market. See
  "Pricing" below.
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

## Notification channel strategy: SMS primary, Telegram optional

**SMS is the primary and default channel.** It's the only channel that works
for every cleaner with zero adoption effort, no app install, and no opt-in
step — which matters most in the launch market, where cleaners can't be
assumed to use any particular messaging app. Every cleaner is reachable by
SMS from the moment the host adds their phone number.

**Telegram is supported as an optional per-cleaner upgrade.** Via the free
Bot API it costs nothing per message, so any cleaner who does opt in reduces
platform cost permanently. The constraint: **a Telegram bot cannot
cold-message a user** — the cleaner must open a chat with the bot once (tap
an invite link, press "Start") before the bot can message them at all. After
that one-time opt-in, messaging is free indefinitely.

How the opt-in is offered without hurting reliability:

1. Host adds a cleaner (name + phone number only — no cleaner login, ever).
2. Notifications start going out over SMS immediately. Nothing is gated on
   the cleaner doing anything.
3. The host can optionally trigger a Telegram invite for a given cleaner,
   which sends a one-time SMS with a deep link
   (`t.me/<bot_username>?start=<token>`). Framed as opt-in, never as a
   requirement.
4. If the cleaner taps it, `Cleaner.telegram_chat_id` is set and that
   cleaner's future notifications switch to Telegram automatically.
5. If they never do, nothing changes — SMS continues indefinitely.

Deliberately **not** doing: auto-blasting every new cleaner with a Telegram
invite. In a market where nobody uses Telegram, an unsolicited "install this
app" SMS to a cleaner is friction the host has to explain, spends money on a
message that mostly won't convert, and risks making the product feel pushy
on the host's behalf. Keep it host-initiated and per-cleaner.

Telegram adoption rate is worth tracking as a cost metric, but it should not
be treated as a product goal in the launch market — see below.

### On WhatsApp (the likely real answer for Mexico)

Mexico is one of the most WhatsApp-saturated markets in the world; it's the
default messaging app and the reason hosts coordinate with cleaners there by
WhatsApp today. Telegram penetration is low. So while Telegram is cheap to
support, its realistic adoption ceiling in this market is low, and pushing
cleaners toward it means fighting the market rather than following it.

The channel with a genuine path to near-universal adoption in Mexico is
**WhatsApp via the official Cloud API**. Its per-message utility rate is
substantially cheaper than SMS to Mexican carriers, so it is the strongest
candidate for driving messaging cost down at scale in this market — but it
carries real setup cost: a phone number dedicated to the API (it can't stay
active in the regular WhatsApp app), Meta Business verification, and
pre-approved message templates for host-initiated messages. That setup is
worth doing once there's enough volume to justify it, not before first
launch.

(Unofficial WhatsApp automation libraries driving a personal account violate
WhatsApp's ToS and risk the number being banned — not viable for a product
cleaners depend on.)

**Design implication:** the notification layer is a pluggable provider
interface with a per-cleaner channel preference, so SMS / Telegram /
WhatsApp are configuration, not architecture. Which channel to invest in
stays a business decision made on real usage data, and adding WhatsApp later
is a provider implementation, not a rewrite.

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

**Pre-launch task:** confirm current Twilio per-segment SMS pricing to
Mexican carriers and Mexico's A2P registration / sender-ID requirements
(rules differ from the US, and long-code or short-code registration may be
required). Since the platform absorbs this cost and SMS is the primary
channel, this number directly sets the margin on every tier and should be
verified before the quota figures below are finalized. Note also that
messages exceeding the GSM-7 single-segment limit (160 chars, or 70 if any
non-GSM character such as an emoji is present) bill as multiple segments —
so the default message template should be kept short and accent-safe, and
the template editor should show a live segment counter.

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

- **User** (host) — id, email, phone_number (for failure alerts), password
  hash, timezone, locale
- **Property** — id, user_id, name, address, default_checkin_time,
  default_checkout_time, timezone
- **CalendarConnection** — id, property_id, platform (`airbnb`, later
  `booking_com`), ical_url (encrypted), last_synced_at, last_sync_status
- **Booking** — id, property_id, calendar_connection_id, external_uid (iCal
  UID, used to dedupe on re-sync), start_date, end_date, status
  (`upcoming` / `active` / `completed` / `cancelled`)
- **Cleaner** — id, user_id, name, phone_number, preferred_channel
  (defaults to `sms`), telegram_chat_id (nullable — set only if opted in),
  telegram_invite_token, telegram_invited_at, telegram_opted_in_at
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
- Channel resolution per send: SMS by default; Telegram only if that cleaner
  has opted in (`telegram_chat_id` set).
- **Failed delivery → alert the host on two channels.** When a `MessageLog`
  row lands in `failed` status (after retries are exhausted), the host is
  alerted by **both email and SMS**, sent independently. The two are
  deliberately redundant: if one fails to deliver, the other still gets
  through, and a missed cleaning is never silent. A persistent "needs
  attention" state is also set on the dashboard until the host acknowledges
  it.
  - These host alerts are capped and deduplicated (e.g. at most one alert
    SMS per host per N minutes, collapsing multiple failures into one
    message) so a systemic outage can't turn into an SMS storm on the
    platform's bill.
  - Host alert messages don't count against the host's plan quota — the host
    shouldn't burn quota on the system telling them something broke.

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

### Localization

The launch market is Mexico, so **Spanish is a first-class requirement, not
a later i18n pass**:

- Default message templates ship in Spanish (with English available).
- The host-facing web UI should be built i18n-ready from the first commit
  (string catalog, no hardcoded copy) even if English ships first — retrofitting
  i18n later is far more expensive than starting with it.
- Message templates should stay accent-safe where practical: accented
  characters (á, é, ñ) push an SMS out of the GSM-7 alphabet into UCS-2,
  cutting the per-segment limit from 160 to 70 characters and multiplying
  cost per message. The template editor's segment counter should make this
  visible to the host as they type.
- Dates/times formatted per locale, and phone numbers stored in E.164 with
  Mexico (+52) as the default country.

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

Priced for the **Mexican market**, deliberately at the lower end — these are
not US/EU price points. Charge in **MXN**, not USD: local-currency pricing
converts better and avoids card foreign-transaction friction.

Starting ladder (first draft, to be finalized after the Twilio Mexico SMS
cost check above):

| Tier | Properties | Included messages/mo | Price (MXN/mo) | ≈ USD |
|---|---|---|---|---|
| Free | 1 | 15 | $0 | $0 |
| Starter | up to 3 | 120 | ~$149 | ~$8 |
| Growth | up to 10 | 500 | ~$399 | ~$22 |
| Pro | unlimited | custom | contact | — |

Notes on these numbers:

- Message quotas are sized generously relative to real usage — a property
  turns over maybe 8–15 times a month, one notification each — so a normal
  host never notices the cap. The cap exists to bound a runaway bug or an
  abusive account, not to nickel-and-dime real hosts.
- The **gap between the quota and typical real usage is the safety margin**;
  the gap between subscription price and expected SMS cost per account is
  the margin. Both need re-checking once actual Twilio Mexico rates are
  confirmed, since SMS is the primary (and paid) channel here — if
  per-message cost turns out high, the lever to pull is tightening quotas,
  not raising prices out of the market.
- Free tier stays genuinely useful (one property, real automation) — it's
  the acquisition hook, and a single-property host who upgrades to a second
  property is the most natural conversion path.

Overage handling for the MVP: soft-block with an upgrade prompt rather than
metered overage billing — simpler to reason about on both sides while the
product is new, and it hard-caps platform cost by construction.

## Roadmap after MVP

- **WhatsApp Cloud API as a channel** — the highest-leverage cost item for
  the Mexican market specifically (cheaper per message than SMS, and the app
  cleaners actually use). Worth doing once volume justifies the Meta
  Business verification and template approval effort.
- Booking.com iCal integration; double-booking detection across platforms
  on the unified calendar
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

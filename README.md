# host-cleaner-bridge

Automates the "guest checked out, go clean unit X" message that Airbnb hosts
currently send cleaners by hand over WhatsApp. Connects to a host's Airbnb
calendar, watches for checkouts, and texts the assigned cleaner automatically.

Status: **planning** — see [docs/DESIGN.md](docs/DESIGN.md) for the MVP scope,
architecture, and data model before any code is written.

## MVP decisions (locked in)

- **Platform:** Airbnb only (iCal export). Booking.com is a fast-follow.
- **Channel:** Telegram-first (free, unlimited once opted in), with SMS via
  Twilio as the one-time onboarding step and fallback for cleaners who never
  adopt Telegram. See design doc for why not WhatsApp.
- **Monetization:** platform-paid messaging; tiered subscription by property
  count with a capped free tier and per-plan message quotas (also the cost
  safety net).
- **Stack:** Next.js (TypeScript) + PostgreSQL + Prisma + a scheduled worker
  for calendar polling.

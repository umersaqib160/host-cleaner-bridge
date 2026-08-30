# host-cleaner-bridge

Automates the "guest checked out, go clean unit X" message that Airbnb hosts
currently send cleaners by hand over WhatsApp. Connects to a host's Airbnb
calendar, watches for checkouts, and texts the assigned cleaner automatically.

Status: **planning** — see [docs/DESIGN.md](docs/DESIGN.md) for the MVP scope,
architecture, and data model before any code is written.

## MVP decisions (locked in)

- **Platform:** Airbnb only (iCal export). Booking.com is a fast-follow.
- **Channel:** SMS via Twilio. WhatsApp Cloud API is a fast-follow (see
  design doc for why WhatsApp can't launch on day one).
- **Stack:** Next.js (TypeScript) + PostgreSQL + Prisma + a scheduled worker
  for calendar polling.

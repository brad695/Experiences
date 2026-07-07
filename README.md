# Grey's Restaurant Booking System

Two drop-in pages backed by the same Supabase project as the classes/events site,
so everything shares one calendar and tables can never be double-booked.

## Files

- **book.html** — customer page: reserve a table, book paid experiences, cancel by code.
- **admin.html** — host dashboard: day timeline, month calendar, floor plan (live view +
  drag-and-drop layout editor), reservation search, booking settings. Sign-in via
  Supabase magic link; access limited to approved emails in `app_users`.

Both are static files — host them anywhere (same place as the existing site is easiest).

## How double-booking prevention works

Every seating (regular reservation, experience party, or event table-block) writes a row to
`rb_reservation_tables` with a time range. A Postgres **exclusion constraint** rejects any
two active rows for the same table with overlapping time ranges — enforcement lives in the
database, so no code path (this site, the classes site, manual SQL) can double-book.

## Database objects (all in the existing Supabase project)

| Object | Purpose |
|---|---|
| `rb_tables` | Physical tables: seats, zone, shape, floor x/y (edited in admin Floor Plan) |
| `rb_table_combos` | Push-together combinations (T1+T2 → 5, T3/T4/T5 pairs → 8, T3+T4+T5 → 12, T9+T10 → 8) |
| `rb_reservations` | All bookings: `standard`, `experience`, `event_block`, `private` |
| `rb_reservation_tables` | Table↔time assignments with the no-overlap constraint |
| `rb_settings` | Hours, slot interval, durations, lead time, closures (edited in admin Settings) |

## Automatic table assignment

Venue layout mirrors the events/tickets page: Bar:4 · T1:2 · T2:2 · T3:4 · T4:4 · T5:4 ·
T9:4 · T10:4, with combos T1+T2 (5 seats), T3/T4/T5 (any order), T9+T10.

- **Dinner reservations** get the best single table (least leftover seats); parties too big
  for one table get a combo — all member tables are held together.
- **Experiences** follow the tickets-page roster rules: singles seat at the Bar first,
  parties share tables' spare seats, and big groups span a combo. Physical tables are
  claimed for the event as needed, so dinner reservations can never collide with them.
- Combos are data (`rb_table_combos`) — change a combined capacity with one update.

RPC functions (called by the pages): `rb_get_availability`, `rb_create_reservation`,
`rb_cancel_reservation`, `rb_list_experiences`, `rb_book_experience`, `rb_block_event`,
`rb_is_admin`.

## Ties to the existing site

- **Experiences are paid reservations** (Grazing Table, Fondue Dinner, Raclette Dinner,
  Wine & Cheese Tasting) — defined in `rb_experience_types`, edited in dashboard Settings,
  booked on book.html like a normal reservation with a per-person price.
- **Events are separate**: classes/mahjong etc. sell on the tickets site; a database
  trigger on `registrations` automatically seats every event booking onto the table
  calendar (kind `event`), and refunds free the seats. No sales channel can bypass it.
- Classes/events hosted in the dining room: open the day in the admin, hit
  **"Block all tables"** on the event — existing reservations are kept, free tables are
  blocked for the event's time window.
- **Experiences are prepaid via Stripe Checkout.** Booking holds the tables and
  redirects to Stripe (30-minute session). Edge functions on Supabase:
  `rb-stripe-checkout` (creates the session; also `status`/`cancel` actions used on
  return) and `rb-stripe-webhook` (marks `payment_status = paid` on
  `checkout.session.completed`, cancels the reservation and frees tables on
  `checkout.session.expired`). The webhook trusts nothing it receives — it re-fetches
  each event from the Stripe API, so no signing secret is needed. Secrets: only
  `STRIPE_SECRET_KEY` in Supabase Edge Function secrets. The Stripe session id is
  stored on the reservation as `external_id = stripe:<id>`. A webhook endpoint must be
  registered in the Stripe dashboard for `checkout.session.completed/expired` (and the
  `async_payment_*` pair) pointing at
  `https://vdvtrevhqalmjwhjhjug.supabase.co/functions/v1/rb-stripe-webhook`.

## Local preview

`npx http-server -p 8321 .` then open `/book.html` and `/admin.html`
(or use the configured "booking-site" launch config).

# v1 Plan — Deposit-taking booking links for solo providers

Scope: one shareable link where a client picks a service and slot, pays a deposit,
and gets reminders. Two flows only: provider onboarding, public booking.

## Decisions locked

| Question | Decision |
|---|---|
| Slot step | Start at availability window start, step by `duration + buffer` |
| Stripe | Connect Express, **direct charges + `application_fee_amount`** |
| Cancellation | Tokenized cancel link; auto-refund outside window, keep deposit inside |
| Provider auth | Supabase magic link, SMTP pointed at Resend |

## Stack additions

Beyond the existing scaffold (Next 16 / React 19 / TS / Tailwind v4):

- `@supabase/supabase-js` + `@supabase/ssr` — DB and auth
- `stripe` — server SDK only, no Stripe.js needed (Checkout is a redirect)
- `resend` + `react-email` — transactional email
- `date-fns` + `@date-fns/tz` — timezone math
- `vitest` — unit tests for slot generation
- `zod` — parse form and webhook input

## Next.js 16 notes that affect the code

Confirmed against `node_modules/next/dist/docs`:

- `params`, `searchParams`, `cookies()`, `headers()` are **async-only**; sync access
  was removed in 16. Use the generated `PageProps<'/[slug]'>` / `RouteContext` helpers.
- `middleware.ts` is now `proxy.ts`, Node runtime only, exported function named `proxy`.
- `revalidateTag` requires a `cacheLife` profile as a second arg. Prefer `updateTag`
  in Server Actions for read-your-writes on the dashboard.
- Turbopack is the default for `dev` and `build`; no flags needed.

## Data model

Given tables, plus the following concrete typing. All timestamps `timestamptz`, UTC.

```
providers          + slug unique, timezone IANA text, cancellation_window_hours int
                     default 24, buffer_minutes int default 0
services           + sort_order int, is_active bool
availability_rules + weekday 0-6, start_time/end_time are `time` (wall clock in
                     provider timezone — NOT timestamps)
blackouts          + starts_at/ends_at timestamptz UTC
bookings           + status: pending|confirmed|cancelled|expired|completed
                     deposit_status: unpaid|paid|refunded|failed
                     + cancel_token uuid, hold_expires_at, stripe_checkout_session_id
```

Two additions I need and will call out rather than sneak in:

1. `bookings.cancel_token uuid` — required for the cancel link.
2. `bookings.hold_expires_at timestamptz` — required to stop two clients paying for
   the same slot (see below).

### Double-booking guard

A DB-level guard, not application logic:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings ADD CONSTRAINT no_overlap EXCLUDE USING gist (
  provider_id WITH =,
  tstzrange(starts_at, starts_at + (duration_minutes || ' minutes')::interval) WITH &&
) WHERE (status IN ('pending','confirmed'));
```

Flow: create booking as `pending` with `hold_expires_at = now() + 30 min` → redirect to
Checkout → `checkout.session.completed` flips it to `confirmed`, `checkout.session.expired`
flips it to `expired` (releasing the slot). A daily sweep catches anything the webhook missed.

The constraint enforces literal overlap only. Buffer is enforced in slot generation, not
in the constraint, so that changing `buffer_minutes` never invalidates existing rows.

### RLS

All DB access is server-side with the service role key. RLS on, no public policies —
the anon key is never used for table reads. Simplest correct posture for v1.

## Slot generation — the hard part

Pure function, zero I/O, in `src/lib/slots.ts`:

```ts
generateSlots(input: {
  date: string              // YYYY-MM-DD in provider tz
  timezone: string          // IANA
  durationMinutes: number
  bufferMinutes: number
  rules: { weekday: number; startTime: string; endTime: string }[]
  bookings: { startsAt: Date; durationMinutes: number }[]
  blackouts: { startsAt: Date; endsAt: Date }[]
  now: Date                 // injected, never Date.now()
}): Date[]                  // UTC instants
```

Algorithm: resolve each weekday rule to a UTC interval for that local date → step by
`duration + buffer` from the window start → keep a candidate if `[start, start+duration)`
fits the window, doesn't overlap any booking padded by buffer on both sides, doesn't
intersect a blackout, and starts after `now`.

Unit tests cover: exact window fit, trailing partial slot dropped, buffer applied on both
sides of an existing booking, blackout mid-window splitting the day, DST spring-forward
(a 9-5 day that is 7 real hours), DST fall-back (9 hours), booking that spans midnight UTC
but not local, past slots filtered on today, multiple rules on one weekday, empty rules.

Everything stored UTC, rendered via `Intl.DateTimeFormat` in the provider timezone.

## Build order

1. **Schema + migrations** — SQL in `supabase/migrations/`, typed client, seed script.
2. **Slot generation + tests** — pure function first, green Vitest suite, no UI.
3. **Public booking `/[slug]`** — header, service list, slot picker, client form,
   pending booking, Checkout redirect, webhook, `/[slug]/confirmed`. Stripe test mode.
4. **Cancel flow** — `/cancel/[token]`, window check, refund with
   `refund_application_fee: true` so the platform fee returns with the deposit.
5. **Provider onboarding** — magic link, Connect Express onboarding + return/refresh
   URLs, services CRUD, weekly hours, settings, "here's your link".
6. **Dashboard** — upcoming bookings list, settings page. No charts.
7. **Emails** — confirmation to both parties on webhook; 24h reminder via daily cron.

Small commits, `tsc --noEmit` and `eslint` green before each.

## Reminders on $0

Vercel Hobby crons fire **once per day**. So reminders are a single daily job at a fixed
hour that emails everyone with an appointment 24–48h out, rather than an exact T-24h send.
Fine for v1; it's the one place the free tier visibly shapes the product. Same job sweeps
expired pending holds.

## Env vars

```
NEXT_PUBLIC_SUPABASE_URL / NEXT_PUBLIC_SUPABASE_ANON_KEY / SUPABASE_SERVICE_ROLE_KEY
STRIPE_SECRET_KEY / STRIPE_WEBHOOK_SECRET / STRIPE_CONNECT_CLIENT_ID
PLATFORM_FEE_BPS          # basis points, e.g. 200 = 2%
RESEND_API_KEY / EMAIL_FROM
NEXT_PUBLIC_APP_URL
CRON_SECRET
```

## Where I'm guessing, not certain

- **Platform fee size.** You chose direct charges + application fee but not an amount.
  I've made it `PLATFORM_FEE_BPS` and will default it to `0` so nothing is silently
  skimmed. Tell me the number and it's a one-line change.
- **Who absorbs the fee.** With direct charges the provider is merchant of record and
  pays Stripe's processing fee; your application fee comes out of their side too. If you
  meant the *client* to cover it, that's a different calculation and I should know now.
- **Supabase free tier pauses** a project after ~7 days of inactivity. Harmless while
  you're building, fatal for a live booking link — worth knowing before you hand the URL
  to a real barber.
- **Express onboarding is not instant.** Some providers get `charges_enabled: false`
  pending verification. I'll gate the public link on `charges_enabled` and show a
  "finish Stripe setup" state, since a live link that can't take money is worse than no link.
- **No SMS** means the reminder is email-only, and this audience's clients are texters.
  Your call and I'm not relitigating it — just flagging that it's the likeliest v1 gap.

## Non-goals (not building)

Staff/multi-user, recurring appointments, in-app messaging, native app, calendar sync,
custom branding, analytics, provider subscription billing, intake forms, packages.

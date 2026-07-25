# v1 Plan — Deposit-taking booking links for solo providers

Scope: one shareable link where a client picks a service and slot, pays a deposit,
and gets reminders. Two flows only: provider onboarding, public booking.

## Stack

| Layer | Choice |
|---|---|
| App | Next.js 16 App Router + TypeScript + Tailwind v4 |
| Auth | **Better Auth** (magic link plugin) |
| DB | **Postgres 17 in Docker** + Drizzle ORM |
| Payments | **Stripe Connect Express**, direct charges + application fee |
| Email | Resend |
| Deploy | **Docker Compose on your VPS**, Caddy in front for TLS |

## Decisions locked

| Question | Decision |
|---|---|
| Slot step | Start at availability window start, step by `duration + buffer` |
| Cancellation | Tokenized cancel link; auto-refund outside window, keep deposit inside |
| Provider auth | Magic link, email sent through Resend |
| Payments | Stripe Connect Express, direct charges + `application_fee_amount` |

> Polar was evaluated and rejected: its acceptable use policy excludes human services, and
> it has no third-party merchant onboarding. Reasoning kept in git history at `6a9343f`.

---

## Payments — Stripe Connect Express

Direct charges on the connected account. The provider is merchant of record, pays Stripe's
processing fee, and funds never touch a platform balance.

**Provider onboarding.** Create an Express account, then an Account Link with `return_url`
and `refresh_url` back into the dashboard. Account Links are single-use and short-lived, so
the refresh path has to mint a fresh one rather than reuse the old URL.

**Gating.** Express verification is not always instant. The public booking link stays
disabled until `charges_enabled` is true, with a "finish Stripe setup" state in the
dashboard — a live link that can't take money is worse than no link. Kept current via the
`account.updated` Connect event.

**Taking the deposit.** Checkout Session created *on the connected account* (`stripeAccount`
request option), with `payment_intent_data.application_fee_amount` for the platform cut and
an idempotency key derived from the pending booking id. Session metadata carries the booking
id so the webhook can resolve it without a lookup table.

**Webhooks.** One endpoint at `src/app/api/stripe/webhook/route.ts`, registered as a
**Connect** endpoint so it receives events from connected accounts (each carries an
`account` field). Raw body via `await req.text()` for signature verification — do not parse
JSON first. Events handled:

| Event | Effect |
|---|---|
| `checkout.session.completed` | booking → `confirmed`, deposit → `paid`, send both emails |
| `checkout.session.expired` | booking → `expired`, releasing the slot |
| `charge.refunded` | deposit → `refunded` |
| `account.updated` | cache `charges_enabled` on the provider |

Handlers are idempotent — Stripe retries, and a duplicate `completed` must not send a second
confirmation email.

**Refunds.** On cancel outside the window, refund on the connected account with
`refund_application_fee: true`, so the platform fee goes back with the deposit rather than
being kept on a booking that didn't happen.

**Local dev.** `stripe listen --forward-to localhost:3000/api/stripe/webhook`. Test mode
throughout; no live keys until the flow is green end to end.

**Self-hosting notes.** The webhook needs a publicly reachable HTTPS URL, which Caddy
provides — this is one thing Vercel gave for free that now needs the reverse proxy up before
Stripe can be wired. Account Link return URLs must be HTTPS too, so `NEXT_PUBLIC_APP_URL`
has to be the real domain, not an IP.

---

## Auth — Better Auth

Note that Better Auth replaces **only** Supabase Auth. Supabase was also the database, so
that half of the swap needs its own answer: Postgres in a container, which the VPS move
makes natural anyway.

- `better-auth` with the `magicLink` plugin and the Drizzle adapter.
- Handler at `src/app/api/auth/[...all]/route.ts` via `toNextJsHandler`.
- Sessions read server-side with `auth.api.getSession({ headers: await headers() })` —
  `headers()` is async in Next 16.
- Magic link emails go through Resend directly from the plugin's `sendMagicLink`. This is
  strictly simpler than the Supabase plan, which needed Supabase's SMTP repointed at Resend
  to dodge its ~3/hour built-in limit. That whole problem disappears.
- Better Auth owns `user`/`session`/`account`/`verification` tables. `providers` becomes a
  profile table keyed by `user.id` rather than carrying its own `email` and identity.

## Database

Postgres 17 as a Compose service, Drizzle for schema and migrations
(`drizzle-kit generate` / `migrate`), run on container start.

Schema is unchanged from the previous plan except that `providers.id` now references
Better Auth's `user.id`:

```
providers          slug unique, timezone IANA, cancellation_window_hours int default 24,
                   buffer_minutes int default 0, user_id -> user.id
services           sort_order, is_active
availability_rules weekday 0-6, start_time/end_time as `time` (wall clock in provider tz)
blackouts          starts_at/ends_at timestamptz UTC
bookings           status: pending|confirmed|cancelled|expired|completed
                   deposit_status: unpaid|paid|refunded|failed
                   + cancel_token uuid, hold_expires_at, provider_payment_ref
```

Two additions I flagged before and still need: `cancel_token` for the cancel link, and
`hold_expires_at` to stop two clients paying for the same slot.

**Row Level Security is dropped** — it was a Supabase-shaped answer. The app connects as a
single owner role and is the only thing touching the database, so RLS adds nothing here.

### Double-booking guard

Unchanged, and still a DB constraint rather than application logic:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings ADD CONSTRAINT no_overlap EXCLUDE USING gist (
  provider_id WITH =,
  tstzrange(starts_at, starts_at + (duration_minutes || ' minutes')::interval) WITH &&
) WHERE (status IN ('pending','confirmed'));
```

Buffer stays out of the constraint and lives in slot generation, so changing
`buffer_minutes` never invalidates existing rows.

## Slot generation — the hard part

Unchanged and rail-independent. Pure function, zero I/O, `src/lib/slots.ts`:

```ts
generateSlots(input: {
  date: string; timezone: string; durationMinutes: number; bufferMinutes: number
  rules: { weekday: number; startTime: string; endTime: string }[]
  bookings: { startsAt: Date; durationMinutes: number }[]
  blackouts: { startsAt: Date; endsAt: Date }[]
  now: Date                 // injected, never Date.now()
}): Date[]                  // UTC instants
```

Tests: exact window fit, trailing partial slot dropped, buffer on both sides of an existing
booking, blackout splitting a day, DST spring-forward (9-5 is 7 real hours), DST fall-back
(9 hours), booking spanning midnight UTC but not local, past slots filtered on today,
multiple rules per weekday, empty rules. Vitest.

## Deployment — Docker on your VPS

```
compose.yaml
  app       Next standalone build, output: 'standalone' in next.config.ts
  db        postgres:17-alpine, named volume
  caddy     reverse proxy, automatic TLS via Let's Encrypt
```

Multi-stage Dockerfile: deps → build → runner on `node:22-alpine`, non-root user, only
`.next/standalone`, `.next/static`, and `public` in the final image.

This move fixes two real problems in the previous plan:

- **Reminders can be exact.** Vercel Hobby crons fire once a day, so the plan was a daily
  batch emailing everyone 24–48h out. With your own box it's a real crontab hitting an
  authenticated endpoint every 15 minutes, so T-24h means T-24h.
- **No idle pausing.** Supabase free tier suspends a project after ~7 days of inactivity,
  which is fatal for a booking link that sits quiet for a week. A container doesn't do that.

It also moves work onto you: **`pg_dump` on a cron plus offsite copy is now yours to own.**
Nobody is backing this database up by default, and it holds money-linked records. I'll
include a backup service in the Compose file, but restores need testing by a human.

The "$0 infrastructure" constraint is now "whatever the VPS costs" — fixed and predictable
rather than usage-scaled, which for this product is probably the better shape.

## Build order

1. **Postgres + Drizzle schema + migrations**, Compose skeleton, seed script.
2. **Slot generation + tests** — pure function first, green suite, no UI.
3. **Public booking `/[slug]`** — header, service list, slot picker, client form, pending
   booking with hold, Checkout redirect, webhook, `/[slug]/confirmed`. Stripe test mode.
4. **Cancel flow** — `/cancel/[token]`, window check, refund with `refund_application_fee`.
5. **Provider onboarding** — magic link, Connect Express onboarding + `charges_enabled`
   gating, services CRUD, weekly hours, settings, "here's your link".
6. **Dashboard** — upcoming bookings list, settings page. No charts.
7. **Emails** — confirmation to both parties, T-24h reminder via cron.
8. **Dockerfile + Compose + Caddy + backup job.**

Nothing is blocked. Small commits, `tsc --noEmit` and `eslint` green before each.

## Env vars

```
DATABASE_URL              # postgres://... pointing at the db service
POSTGRES_USER / POSTGRES_PASSWORD / POSTGRES_DB
BETTER_AUTH_SECRET / BETTER_AUTH_URL
STRIPE_SECRET_KEY / STRIPE_WEBHOOK_SECRET
PLATFORM_FEE_BPS          # basis points, e.g. 200 = 2%. Defaults to 0.
RESEND_API_KEY / EMAIL_FROM
NEXT_PUBLIC_APP_URL       # real HTTPS domain — Stripe return URLs require it
CRON_SECRET
```

## Where I'm guessing, not certain

- **Better Auth API shape.** I'm confident about the magic link plugin, Drizzle adapter and
  `toNextJsHandler`, less so about exact option names. I'll verify against the installed
  package's docs before writing code rather than trusting recall.
- **Platform fee, unanswered across two rounds now.** I need the number, and confirmation
  of who absorbs it. With direct charges the *provider* is merchant of record and pays both
  Stripe's processing fee and your application fee out of their side; if you meant the
  client to cover it, the deposit amount charged has to be grossed up and that's different
  math in the Checkout Session. Shipping with `PLATFORM_FEE_BPS=0` until you say otherwise,
  so nothing is silently skimmed from providers.
- **Stripe account requirements for a platform.** Taking an application fee makes you a
  payment facilitator in Stripe's eyes and their Connect platform review can ask for company
  details before enabling live application fees. Test mode is unaffected, so this won't slow
  the build — but it can slow your first real payment, and it's better known now than in
  week three.

## Non-goals (not building)

Staff/multi-user, recurring appointments, in-app messaging, native app, calendar sync,
custom branding, analytics, provider subscription billing, intake forms, packages.

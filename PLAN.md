# v1 Plan — Deposit-taking booking links for solo providers

Scope: one shareable link where a client picks a service and slot, pays a deposit,
and gets reminders. Two flows only: provider onboarding, public booking.

## Stack

| Layer | Choice |
|---|---|
| App | Next.js 16 App Router + TypeScript + Tailwind v4 |
| Auth | **Better Auth** (magic link plugin) |
| DB | **Postgres 17 in Docker** + Drizzle ORM |
| Payments | **Unresolved — see "Payments" below.** Polar cannot be used. |
| Email | Resend |
| Deploy | **Docker Compose on your VPS**, Caddy in front for TLS |

## Decisions locked

| Question | Decision |
|---|---|
| Slot step | Start at availability window start, step by `duration + buffer` |
| Cancellation | Tokenized cancel link; auto-refund outside window, keep deposit inside |
| Provider auth | Magic link, email sent through Resend |

---

## Payments — the swap that doesn't work

**Polar.sh explicitly prohibits this product.** Its acceptable use policy limits the
platform to digital goods and software, and names the exclusions directly: physical goods
of any kind, "SaaS services requiring fulfillment via physical delivery or human services,"
and "human services such as marketing, design, web development and consulting in general."
It states that if a company's primary offering is human services, the platform "should not
be used."

Every customer in your target list — barbers, mobile detailers, lash techs, dog groomers,
massage therapists, tutors — sells human services fulfilled in person. That is the
prohibited category, not an edge case.

There is a second, independent blocker. Polar is a Merchant of Record for **first-party**
sales: you selling your own products. It has no marketplace primitive for onboarding
third-party merchants who take their own payments — no Connect Express equivalent. Polar
does use Stripe Connect Express internally, but for paying out *its own sellers*, which is
you, not your barbers. Under Polar you would be merchant of record for every haircut
deposit in the system, custodying funds and owning every chargeback. That is the exact
opposite of the "I never custody funds" property you picked Stripe Connect for.

Either issue alone rules Polar out. Together they mean building on it produces a system
that takes money for a while and then gets the account closed.

### What I recommend

**Keep Stripe Connect Express.** It is the only option in reach for a solo dev on a
two-week budget that supports third-party merchant onboarding for in-person services.

If the goal was getting *off Stripe specifically*, the marketplace-capable alternatives are
Mangopay, Adyen for Platforms, Mollie Connect, and PayPal Commerce Platform. All of them
carry heavier onboarding than Stripe — contracts or compliance review before you can take a
live payment — and none is a two-week drop-in. Every merchant-of-record product in Polar's
category (Paddle, Lemon Squeezy, Dodo) has the same digital-goods-only restriction and
fails for the same reason.

**I have not written the payments section of the build below.** Tell me the rail and I'll
fill it in; the rest of the plan is rail-independent.

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
   booking with hold. *Payment step blocked on the rail decision.*
4. **Cancel flow** — `/cancel/[token]`, window check, refund.
5. **Provider onboarding** — magic link, payment onboarding, services CRUD, weekly hours,
   settings, "here's your link".
6. **Dashboard** — upcoming bookings list, settings page. No charts.
7. **Emails** — confirmation to both parties, T-24h reminder via cron.
8. **Dockerfile + Compose + Caddy + backup job.**

Steps 1, 2, 4 (logic), 6 and 8 are unblocked today. Small commits, `tsc --noEmit` and
`eslint` green before each.

## Env vars

```
DATABASE_URL
BETTER_AUTH_SECRET / BETTER_AUTH_URL
RESEND_API_KEY / EMAIL_FROM
NEXT_PUBLIC_APP_URL
CRON_SECRET
# payment rail vars TBD
```

## Where I'm guessing, not certain

- **Better Auth API shape.** I'm confident about the magic link plugin, Drizzle adapter and
  `toNextJsHandler`, less so about exact option names. I'll verify against the installed
  package's docs before writing code rather than trusting recall.
- **Polar's policy could change**, and I could not fetch the policy pages directly — both
  returned 403 to my fetcher, so the quotes above come from search result excerpts of
  Polar's own acceptable use pages, which were consistent across two independent searches.
  If you have a written exception from Polar, that changes the first blocker but not the
  merchant-of-record one.
- **Platform fee, still unanswered from last round.** Whatever rail we land on, I need to
  know the cut and who absorbs it. Defaulting to zero so nothing is silently skimmed.

## Non-goals (not building)

Staff/multi-user, recurring appointments, in-app messaging, native app, calendar sync,
custom branding, analytics, provider subscription billing, intake forms, packages.

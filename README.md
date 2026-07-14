# Minutes — AI meeting notes

Paste a meeting transcript, get a summary and a checklist of action items
back in seconds. Free accounts get 3 write-ups a day; Pro ($15/mo, via
Stripe) is unlimited. Built for the "Zero to Subscriber" hackathon brief —
every non-negotiable in that brief (real auth, real persistence, real Stripe
subscriptions with webhooks, real deploy) is implemented for real, not
mocked.

## Stack

- **Framework:** Next.js 16 (App Router) + TypeScript + Tailwind CSS v4
- **Database:** Postgres, accessed with plain SQL via `pg` (no ORM — see
  [Why no ORM](#why-no-orm-and-no-authjs) below)
- **Auth:** Hand-rolled email/password (bcrypt + signed JWT session cookie)
- **AI:** Anthropic API (`claude-sonnet-5`) with a forced tool call, so the
  response is always structured JSON, never free-form prose
- **Payments:** Stripe Checkout (subscriptions) + Billing Portal + webhooks
- **Fonts:** Fraunces / IBM Plex, self-hosted via `@fontsource` (no Google
  Fonts request at runtime)

## Local setup

You'll need Node 20+ and a Postgres database (local, or a free one from
[Neon](https://neon.tech) or [Supabase](https://supabase.com) — either works
fine for local dev too, so you can skip installing Postgres if you'd rather
not).

```bash
npm install
cp .env.example .env      # then fill in the values (see below)
npm run db:migrate        # creates the users / meetings / action_items tables
npm run dev
```

Open http://localhost:3000.

### Environment variables

See `.env.example` for the full list with explanations. You need, at
minimum, `DATABASE_URL` and `AUTH_SECRET` to sign up and log in.
`ANTHROPIC_API_KEY` is required to actually generate notes; the four
`STRIPE_*` variables are required for the billing flow. The app boots and
the rest of the product works without Stripe/Anthropic configured — those
specific features fail with a clear error message instead of crashing until
you add the keys.

### Stripe setup (test mode)

1. Create an account at [dashboard.stripe.com](https://dashboard.stripe.com)
   and make sure you're in **Test mode** (toggle top-right).
2. **API keys** → copy the secret key into `STRIPE_SECRET_KEY`.
3. **Product catalog** → add a product ("Pro") with a **recurring** price
   (e.g. $15/month) → copy that price's id (`price_...`) into
   `STRIPE_PRO_PRICE_ID`.
4. Webhook, for local dev, via the [Stripe CLI](https://docs.stripe.com/stripe-cli):
   ```bash
   stripe listen --forward-to localhost:3000/api/stripe/webhook
   ```
   It prints a `whsec_...` value — put that in `STRIPE_WEBHOOK_SECRET`.
   (In production you instead create the webhook endpoint in the Dashboard —
   see [Deploying](#deploying) below.)
5. Use [Stripe's test card](https://docs.stripe.com/testing) `4242 4242 4242
   4242`, any future expiry, any CVC, to pay.

## How the non-negotiables are implemented

- **Landing page** (`app/page.tsx`): value prop, feature list, two real
  pricing tiers pulled from `lib/plan.ts`, all CTAs go to `/signup`.
- **Auth** (`lib/auth.ts`, `app/api/auth/*`): signup/login create a signed,
  httpOnly session cookie (30-day expiry), so reloading the page keeps you
  logged in. Passwords are bcrypt-hashed, never stored or returned as-is
  (see `toPublicUser`).
- **The product** (`app/(app)/dashboard/*`, `lib/anthropic.ts`,
  `lib/meetings.ts`): a transcript is sent to Claude, which returns a title,
  summary, and action items via a forced tool call (not parsed from free
  text, so it can't come back malformed). Everything is saved per-user in
  Postgres — refreshing, logging out and back in, or opening a new browser
  all show the same history because it's a real row in a real table, not
  component state.
- **Plan gating** (`lib/plan.ts`): free accounts are capped at 3 write-ups
  per calendar day, enforced server-side in `POST /api/meetings` by counting
  that user's rows created today — not a client-side check you could bypass.
- **Stripe** (`app/api/stripe/*`): `/billing` → Checkout → Stripe redirects
  back → the `checkout.session.completed` / `customer.subscription.*`
  webhooks flip `users.plan` in the database. The billing page reads that
  same column, so it reflects whatever Stripe last told us, not what the
  client hopes is true. Canceling goes through the real Stripe Billing
  Portal.

## Why no ORM, and no Auth.js

Two intentional deviations from the "sane default" stack in the brief, both
made for reliability rather than taste:

- **No Prisma/ORM.** Prisma's CLI needs to download a native engine binary
  on first use; in the sandboxed environment this was built in, that
  download was blocked by network policy, which made it untestable there. A
  few hand-written SQL statements over `pg` for 3 tables is little enough
  code that the ORM wasn't buying much anyway — see `db/schema.sql` and
  `lib/*.ts`. Swap in Prisma/Drizzle if you'd prefer; the schema is plain
  enough to port in a few minutes.
- **No Auth.js/NextAuth.** As of this writing Auth.js v5 is still in beta,
  and a known middleware-bypass class of bug (CVE-2025-29927) means
  middleware-only route protection isn't sufficient on its own. So auth here
  is ~120 lines of bcrypt + `jose` (JWT) + httpOnly cookies (`lib/auth.ts`),
  and — more importantly — every protected page and API route independently
  re-checks the session server-side (see the comments in `proxy.ts` and
  `app/(app)/layout.tsx`) instead of trusting proxy/middleware alone.

## Deploying

1. Push this repo to GitHub.
2. **Database:** create a free Postgres on [Neon](https://neon.tech) or
   [Supabase](https://supabase.com), copy its connection string.
3. **Vercel:** import the repo at [vercel.com/new](https://vercel.com/new),
   add the environment variables from `.env.example` (use your real
   `ANTHROPIC_API_KEY` and Stripe **test-mode** keys; set
   `NEXT_PUBLIC_APP_URL` to the Vercel URL you're given), deploy.
4. Run the migration against the production database once:
   ```bash
   DATABASE_URL="<your prod connection string>" npm run db:migrate
   ```
5. **Stripe webhook:** in the Dashboard → Developers → Webhooks → Add
   endpoint → URL = `https://<your-app>.vercel.app/api/stripe/webhook`,
   events = `checkout.session.completed`, `customer.subscription.updated`,
   `customer.subscription.deleted`. Copy its signing secret into Vercel's
   `STRIPE_WEBHOOK_SECRET` and redeploy.
6. Visit the live URL, sign up, and run through a real test-mode checkout.

## What's verified vs. what needs your keys

Everything in this repo was built and tested against a real local Postgres
instance and a real running server: signup/login/session persistence,
plan-gating math (3-a-day cutoff, reset, Pro override), the full Stripe
webhook handler (signature verification and both the upgrade and
cancellation paths, using locally-signed test events), and the production
build/lint. The one thing that couldn't be exercised end-to-end in the
sandbox this was built in is a live call to the Anthropic API, since doing
so needs a real `ANTHROPIC_API_KEY` — the request was instead checked
directly against the current `@anthropic-ai/sdk` type definitions and
confirmed to pass Anthropic's own request validation (an auth-only error
came back for a deliberately invalid key, rather than a schema error). Add
your key and it should work; if anything doesn't, `lib/anthropic.ts` is
short and worth a read first.

## Recording the demo video

A suggested path for the walkthrough video the brief asks for, in order:
landing page → sign up → paste a transcript and generate notes → check off
an action item → reload the page to show the session and history persist →
hit the free-tier limit on a 4th write-up → `/billing` → upgrade through a
real Stripe Checkout with the test card → show the webhook flipping the
account to Pro and the limit lifting → cancel via the Billing Portal.

---
name: build-fullstack-react-app
description: Scaffold a full-stack booking/reservation web app (tennis courts, doctor appointments, salon slots, equipment rentals, meeting rooms, or any custom domain) using a proven Express + React (Vite) + Postgres + Better Auth stack. Use when the user asks to build, scaffold, or start a booking app, reservation app, appointment-scheduling app, or names a specific booking domain (court booking, salon booking, room booking, rental booking, etc.) and wants a new full-stack project generated.
---

# Build a full-stack booking app

Scaffolds a new full-stack booking/reservation app from a proven blueprint: one Express server
serving both the JSON API and a Vite-built React frontend, Postgres (Neon-friendly) for storage,
Better Auth for email/password auth, and a generalized time-range booking model that prevents
double-booking at the database level regardless of domain.

This skill scaffolds and gets the app running **locally**. It does not provision cloud
infrastructure, write platform-specific deploy configs, or push/deploy anything — deployment is
an explicit non-goal, called out again in the handoff step.

Three reference files carry the actual technical detail — read them at the point they're
referenced below, not all up front:
- `reference/architecture.md` — file-by-file server/frontend blueprint
- `reference/booking-model.md` — the schema, availability computation, conflict handling
- `reference/gotchas.md` — concrete pitfalls to avoid; **check this at every step below**, not
  just once

## Step 0 — Clarify the domain

Ask (via AskUserQuestion, batched into as few rounds as possible):

1. **Domain & terminology**: what's being booked (tennis courts, doctors, salon chairs,
   equipment, meeting rooms, something else)? Pick or confirm the resource noun and booking noun
   (e.g. Court/Reservation, Provider/Appointment) — see the mapping table in
   `reference/booking-model.md` for defaults per domain, or ask if it's a custom domain.
2. **Slot shape**: fixed-duration slots on a schedule (e.g. hourly tennis court slots,
   30-minute doctor appointments) vs. free-form start/end times chosen by the user (e.g.
   multi-day equipment rental). Both use the same schema; only the availability-computation code
   differs (see `reference/booking-model.md`).
3. **Capacity**: can one resource ever hold more than one simultaneous booking (e.g. a 20-seat
   class), or is every resource exclusive-use (default, and the common case)? Default to
   exclusive-use unless told otherwise.
4. **Domain-specific fields** worth capturing in `metadata` (surface type, specialty, hourly
   rate structure, amenities, etc.) — a short free-text answer is fine.
5. **Project location**: what directory/name should the new app go in? If that directory already
   exists and has files in it, stop and confirm with the user before scaffolding into it —
   don't overwrite an existing project silently.

Don't proceed past this step with assumptions about naming — get it settled first, since it
threads through schema comments, route names, and every page's copy.

## Step 1 — Scaffold the project skeleton

Create the new project directory (if it doesn't exist) and write: `package.json`, `vite.config.js`,
`.oxlintrc.json`, `.env.example`, `.gitignore`, `index.html`. Use `reference/architecture.md`'s
file layout and the exact `package.json` scripts/deps listed there (`dev`, `build`, `start` via
`cross-env`, `db:setup`, `lint`). Don't invent alternate script names — the rest of this skill's
guidance assumes those exact commands.

## Step 2 — Server layer

Write, in this order: `server/db.js`, `server/auth.js`, `server/data/<resources>.js` (seed data,
named for the domain's resource noun), `server/db/schema.sql`, `server/db/migrate.js`,
`server/routes/*.js` (resources, availability, bookings, my-bookings), `server/index.js`.

- Follow `reference/architecture.md` for wiring (Express+Vite middleware mode, Better Auth
  mount order, session-check pattern).
- Follow `reference/booking-model.md` exactly for `schema.sql`'s `bookings`/`resources` tables
  and for the availability + booking-creation route logic — the `EXCLUDE USING gist` constraint
  and the `23P01` conflict handling are the load-bearing part of this whole skill; don't
  simplify them away.
- Generate the Better Auth core tables into `schema.sql` with `npx @better-auth/cli generate`
  against the real `server/auth.js` rather than hand-typing them.
- Re-check `reference/gotchas.md` before finishing this step — the wildcard route syntax,
  mount ordering, `dotenv/config` placement, and identifier quoting are all easy to get subtly
  wrong on the first pass.

## Step 3 — Frontend layer

Write `src/main.jsx`, `src/App.jsx`, `src/api.js`, `src/lib/auth-client.js`,
`src/components/{Navbar,RequireAuth}.jsx`, and pages: `ResourceList`, `ResourceDetail` (browse +
availability picker + booking form), `SignIn`, `SignUp`, `MyBookings`. Use the domain's chosen
nouns in component names/copy where natural, but keep the underlying file/route shape from
`reference/architecture.md`.

Only add a persisted, `CartContext`-style multi-item booking context if Step 0 established the
domain needs multiple bookings queued in one visit. Otherwise keep the in-progress slot
selection as local state in `ResourceDetail` — see the "Cart-equivalent state" note at the end of
`reference/architecture.md` for why persisting a stale time-slot selection is actively
misleading, unlike a stale e-commerce cart item.

Browsing (resource list/detail, availability) stays anonymous-friendly; only submitting a
booking and viewing "my bookings" require a session — gate exactly those, not the whole detail
page.

## Step 4 — Wire it up and verify locally

1. `npm install`.
2. Create `.env` from `.env.example`. Ask the user for `DATABASE_URL` (their Postgres/Neon
   connection string) if not already available. Generate `BETTER_AUTH_SECRET` yourself with
   `openssl rand -base64 32` — this is a local, reversible command, fine to run directly without
   asking. Set `BETTER_AUTH_URL=http://localhost:3000`.
3. `npm run db:setup` — applies the schema and seeds resources. If this fails, it's almost always
   a missing/wrong `DATABASE_URL` or a missing `btree_gist` extension permission — surface the
   actual Postgres error to the user rather than guessing.
4. `npm run dev`, then verify the golden path actually works before calling this done (per this
   project's own testing standard — type-checking isn't enough):
   - Resource list loads and shows seeded data.
   - Sign up, then sign in, works end-to-end.
   - Booking an available slot succeeds.
   - Immediately trying to book the *same* slot again returns a 409 — this is the one check that
     proves the double-booking guard actually works, not just that the happy path works.
   - Use the `run` skill if available to drive the app in a real browser; otherwise use `curl`
     against the API routes and manually check the UI.

## Step 5 — Handoff

Summarize for the user: what was scaffolded, which env vars they still need to supply for their
own environment (at minimum `DATABASE_URL`), and state explicitly that **deployment/hosting/CI
was intentionally not done** — this skill stops at a working local dev server. If they want to
deploy, that's a separate follow-up conversation, not part of this skill's output.

## Guardrails

- Don't push anything to a git remote, create cloud resources, or run deploy commands from this
  skill.
- Confirm with the user before installing anything globally or running commands outside the new
  project's own directory.
- This is new scaffolding — don't touch unrelated existing files/apps in the workspace without
  asking first, especially if Step 0's target directory turned out to be non-empty.

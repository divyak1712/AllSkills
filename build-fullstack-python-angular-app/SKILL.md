---
name: build-fullstack-python-angular-app
description: Scaffold a full-stack booking/reservation web app (tennis courts, doctor appointments, salon slots, equipment rentals, meeting rooms, or any custom domain) using a proven FastAPI (Python) + Angular + MySQL stack with hand-rolled, session-cookie auth. Use when the user asks to build, scaffold, or start a booking/reservation/appointment-scheduling app and wants Angular for the frontend, Python for the backend, and/or MySQL for the database — including when they name a booking domain (court booking, salon booking, room booking, rental booking, etc.) and specify this stack, or explicitly ask for "the Python/Angular/MySQL version".
---

# Build a full-stack booking app (FastAPI + Angular + MySQL)

Scaffolds a new full-stack booking/reservation app from a proven blueprint: one FastAPI process
serving a JSON API (with an Angular dev server proxied in front of it in dev, and serving
Angular's built static files itself in production), MySQL 8 for storage, hand-rolled
bcrypt + opaque-session-token auth (no external auth service), and a generalized time-range
booking model that prevents double-booking under concurrent requests regardless of domain.

This is a sibling of the `build-fullstack-react-app` skill — same domain-modeling philosophy,
same booking/race-safety concerns, same step shape and guardrails — retargeted at Angular,
Python, and MySQL. If the user hasn't specified a stack and doesn't care, prefer that skill's
Express/React/Postgres stack for its lower plumbing; reach for *this* skill when Angular, Python,
or MySQL is what's wanted.

This skill scaffolds and gets the app running **locally**. It does not provision cloud
infrastructure, write platform-specific deploy configs, or push/deploy anything — deployment is
an explicit non-goal, called out again in the handoff step.

Four reference files carry the actual technical detail — read them at the point they're
referenced below, not all up front:
- `reference/architecture.md` — file-by-file server/frontend blueprint, and why the dev topology
  differs from a Vite-based stack (two processes + a proxy, not one)
- `reference/booking-model.md` — the schema and the MySQL-specific race-safety mechanism that
  replaces Postgres's `EXCLUDE USING gist` (MySQL has no equivalent constraint type)
- `reference/auth-model.md` — the hand-rolled bcrypt + hashed-session-token auth model that
  replaces Better Auth (no equivalent turnkey library exists in the Python ecosystem)
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
   and which race-safety layer does the real work differ (see `reference/booking-model.md`).
3. **Capacity**: can one resource ever hold more than one simultaneous booking (e.g. a 20-seat
   class), or is every resource exclusive-use (default, and the common case)? Default to
   exclusive-use unless told otherwise.
4. **Domain-specific fields** worth capturing in the resource's `metadata` JSON column (surface
   type, specialty, hourly rate structure, amenities, etc.) — a short free-text answer is fine.
5. **Database access**: do they already have a running MySQL 8+ instance (local, Docker,
   PlanetScale, RDS, etc.), or should the skill suggest the quickest local option (a `docker run
   mysql:8` one-liner)? Either is fine; just settle it before Step 4.
6. **Project location**: what directory/name should the new app go in? If that directory already
   exists and has files in it, stop and confirm with the user before scaffolding into it —
   don't overwrite an existing project silently.

Don't proceed past this step with assumptions about naming — get it settled first, since it
threads through schema comments, route names, and every page's copy.

## Step 1 — Scaffold the project skeleton

Create the new project root (if it doesn't exist). Two subprojects live side by side:

- `server/` — the Python/FastAPI backend (not a Node package; no `package.json` inside it).
- `client/` — an Angular CLI app, created with `ng new client` run from the project root
  (accept routing + CSS defaults; standalone components are the modern Angular default, no
  special flag needed). If `ng` isn't available, `npx @angular/cli new client` works without a
  global install.

At the project root, write a small orchestration `package.json` (scripts: `dev`, `build`,
`start`, `db:setup`, `lint` — see `reference/architecture.md` for the exact script bodies) plus
`.gitignore`. Don't invent alternate script names — the rest of this skill's guidance assumes
those exact commands. This root `package.json` only exists to run both halves with one command
via `concurrently`; it is not the Angular app's `package.json` (that one lives in `client/` and
was generated by `ng new`).

In `server/`, write `requirements.txt`, `.env.example` — see `reference/architecture.md` for the
exact dependency list and env vars.

## Step 2 — Server layer

Write, in this order: `server/db.py`, `server/auth.py`, `server/data/resources.py` (seed data,
named for the domain's resource noun), `server/db/schema.sql`, `server/db/migrate.py`,
`server/routers/*.py` (auth, resources, availability, bookings, my_bookings), `server/main.py`.

- Follow `reference/architecture.md` for wiring (router mount order, static-file fallback for
  the built Angular app, the `{error}`-shaped error contract).
- Follow `reference/booking-model.md` exactly for `schema.sql`'s `bookings`/`resources` tables
  and for the availability + booking-creation route logic — the `SELECT ... FOR UPDATE` locking
  pattern plus the `UNIQUE(resource_id, starts_at)` constraint and MySQL's `1062` duplicate-key
  error handling are the load-bearing part of this whole skill (MySQL's answer to Postgres's
  exclusion constraint); don't simplify them away.
- Follow `reference/auth-model.md` exactly for `auth.py` and `routers/auth.py` — bcrypt hashing,
  opaque random session tokens hashed-at-rest, and the `get_current_user` dependency pattern.
- Re-check `reference/gotchas.md` before finishing this step — MySQL's `DATETIME` vs
  `TIMESTAMP` timezone behavior, raw-SQL JSON column (de)serialization, the multi-statement
  `schema.sql` execution issue, and the FastAPI error-shape override are all easy to get subtly
  wrong on the first pass.

## Step 3 — Frontend layer

Inside `client/src/app/`, write `app.config.ts`, `app.routes.ts`, `app.component.ts`,
`services/api.service.ts`, `services/auth.service.ts`, `guards/auth.guard.ts`, and pages:
`resource-list`, `resource-detail` (browse + availability picker + booking form), `sign-in`,
`sign-up`, `my-bookings`. Use the domain's chosen nouns in component names/copy where natural,
but keep the underlying file/route shape from `reference/architecture.md`. Use standalone
components and the `@if`/`@for` control-flow syntax (not `*ngIf`/`*ngFor` — see
`reference/gotchas.md`).

Only add a persisted, cart-style multi-item booking service if Step 0 established the domain
needs multiple bookings queued in one visit. Otherwise keep the in-progress slot selection as
local component state (a signal) in the resource-detail component — see the "Cart-equivalent
state" note at the end of `reference/architecture.md` for why persisting a stale time-slot
selection is actively misleading, unlike a stale e-commerce cart item.

Browsing (resource list/detail, availability) stays anonymous-friendly; only submitting a
booking and viewing "my bookings" require a session — gate exactly those with the auth guard,
not the whole detail page.

Write `client/proxy.conf.json` so `ng serve` forwards `/api` to the FastAPI backend — this is
what keeps dev same-origin (no CORS, cookies just work) despite being two processes; see
`reference/architecture.md`.

## Step 4 — Wire it up and verify locally

1. Create a Python virtual environment inside `server/` (`python -m venv venv`), activate it,
   and `pip install -r requirements.txt`. Then `npm install` at the project root (for
   `concurrently`) and `npm install --prefix client` for the Angular app's own deps.
2. Create `server/.env` from `server/.env.example`. Ask the user for `DATABASE_URL` if not
   already available (a `mysql+pymysql://user:pass@host:3306/dbname` URL). If they asked for the
   quick local option in Step 0, run the `docker run mysql:8 ...` one-liner from
   `reference/architecture.md` yourself — that's local and reversible, fine to run directly.
3. `npm run db:setup` — applies the schema and seeds resources. If this fails, it's almost
   always a wrong `DATABASE_URL`, a MySQL server that isn't reachable yet, or a missing
   `cryptography` package (needed by PyMySQL for MySQL 8's default auth plugin) — surface the
   actual MySQL error to the user rather than guessing.
4. `npm run dev` (starts FastAPI on :8000 and `ng serve` on :4200 with the proxy config
   together), then verify the golden path actually works before calling this done (per this
   project's own testing standard — type-checking isn't enough):
   - Resource list loads and shows seeded data.
   - Sign up, then sign in, works end-to-end (cookie gets set, `/api/auth/session` reflects it).
   - Booking an available slot succeeds.
   - Immediately trying to book the *same* slot again returns a 409 — this is the one check that
     proves the double-booking guard actually works under a race, not just that the happy path
     works.
   - Use the `run` skill if available to drive the app in a real browser; otherwise use `curl`
     against the API routes and manually check the UI.

## Step 5 — Handoff

Summarize for the user: what was scaffolded, which env vars they still need to supply for their
own environment (at minimum `DATABASE_URL`), and state explicitly that **deployment/hosting/CI
was intentionally not done** — this skill stops at working local dev servers. If they want to
deploy, that's a separate follow-up conversation, not part of this skill's output.

## Guardrails

- Don't push anything to a git remote, create cloud resources, or run deploy commands from this
  skill.
- Confirm with the user before installing anything globally or running commands outside the new
  project's own directory (running the local Docker MySQL one-liner is the one pre-approved
  exception, and only when the user asked for that path in Step 0).
- This is new scaffolding — don't touch unrelated existing files/apps in the workspace without
  asking first, especially if Step 0's target directory turned out to be non-empty.
- Don't reach for Better Auth, Express, Postgres, or Vite here — if the user actually wants that
  stack, that's the `build-fullstack-react-app` skill, not this one.

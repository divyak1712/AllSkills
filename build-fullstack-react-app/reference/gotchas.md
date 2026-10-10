# Gotchas checklist

Concrete corrections learned from a real build (an Express + React + Postgres + Better Auth
e-commerce app). Every one of these was a bug or a deliberate fix, not a style preference.
Check this list while writing server code — it's easy to reintroduce any of these from scratch.

- **Express 5 wildcard routes need a name.** `app.all('/api/auth/*splat', ...)` — not
  `'/api/auth/*'`. A bare `*` throws at startup on Express 5's router (path-to-regexp v6+).
  Same rule for any other catch-all route.

- **Mount Better Auth before `express.json()`, not after.** Better Auth's handler
  (`toNodeHandler(auth)`) reads the raw request stream itself. If `express.json()` has already
  consumed the body, Better Auth's sign-in/sign-up requests silently fail. Order in
  `server/index.js` must be: mount `/api/auth/*splat` → *then* `app.use(express.json())` → then
  everything else.

- **`dotenv/config` must be the first import in every entrypoint.** `server/index.js` and
  `server/db/migrate.js` both start with `import 'dotenv/config'` before any other import,
  because `server/db.js` reads `process.env.DATABASE_URL` at module-load time — if env vars
  aren't loaded yet, the pool is constructed with `undefined`.

- **Neon (and most pooled Postgres providers) need `ssl: { rejectUnauthorized: false }`** on the
  `pg.Pool` config. Without it, connections fail on the provider's certificate chain.

- **Double-quote Better Auth's own identifiers in raw SQL.** Its tables/columns are camelCase
  (`"user"`, `"userId"`, `"createdAt"`, `"expiresAt"`) and the user table is literally named
  `user` — a near-reserved word. Unquoted identifiers get folded to lowercase by Postgres and
  break Better Auth's own queries against them. Generate this part of the schema with
  `npx @better-auth/cli generate` against your actual `server/auth.js` config rather than
  hand-typing it, and keep the quoting exactly as generated.

- **No migration framework — one idempotent `schema.sql`.** Every statement is
  `CREATE TABLE IF NOT EXISTS` / `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`. Schema changes are
  *appended*, never rewritten in place, so `npm run db:setup` stays safe to re-run against a
  database that already has data. Never write a destructive `ALTER`/`DROP` into this file.

- **Seed data's source of truth is a static JS module, not the database.** e.g.
  `server/data/resources.js` exports a plain array; `server/db/migrate.js` reads it and upserts
  with `INSERT ... ON CONFLICT (id) DO NOTHING`. The API always reads from Postgres, never from
  that file directly — the file only exists to make `db:setup` reproducible.

- **Vite runs in middleware mode *inside* the Express process in dev — there is no separate Vite
  dev server.** `server/index.js` does `const { createServer: createViteServer } = await
  import('vite')` (dynamic import, dev-only) and uses `vite.middlewares` plus a manual
  `vite.transformIndexHtml` on `index.html` for every non-API request. `npm run dev` runs
  `node server/index.js`, not `vite`. In production it serves the static `dist/` build with an
  SPA fallback to `dist/index.html` for any non-API route.

- **`cross-env` is required to set `NODE_ENV=production` in the `start` script** — a bare
  `NODE_ENV=production node ...` in `package.json` doesn't work on Windows.

- **Session checks on protected routes follow one pattern:**
  `const session = await auth.api.getSession({ headers: fromNodeHeaders(req.headers) })`
  (both imports from `better-auth/node`); if falsy, `res.status(401).json({ error: '...' })`.
  Never build a separate hand-rolled auth check.

- **Every API error response has the shape `{ error: string }`.** The frontend's thin fetch
  wrapper (`src/api.js`) reads exactly that field to build the thrown `Error`'s message. A route
  that responds with a different error shape breaks error display silently (falls back to a
  generic "Request failed: 4xx" string).

- **`pg` returns `NUMERIC`/`DECIMAL` columns as strings, not numbers.** Always wrap in `Number()`
  before sending JSON (prices, totals) — otherwise the frontend does string concatenation instead
  of arithmetic the first time it adds two of them.

- **Never trust client-supplied price, total, or availability.** The client sends a resource id
  and a requested time range; the server recomputes price/total and re-validates availability
  itself, inside the same transaction that writes the booking. A prior `SELECT`-then-`INSERT`
  check is a race condition under concurrent requests — the database constraint (see
  `booking-model.md`) is the actual source of truth, and the app code just reacts to it.

- **Transactions**: check out a client explicitly (`pool.connect()`), `BEGIN` / `COMMIT` /
  `ROLLBACK` on error, and always `client.release()` in a `finally` — don't run multi-statement
  writes against the shared pool directly.

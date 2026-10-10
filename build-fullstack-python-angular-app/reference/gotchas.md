# Gotchas checklist

Concrete corrections and MySQL/Python/Angular-specific translations of the parent blueprint's
learnings. Every one of these is a real way this stack breaks on a first pass, not a style
preference. Check this list while writing server *and* frontend code — it's easy to reintroduce
any of these from scratch, and several of them fail silently (wrong data, not a crash).

## MySQL / database

- **`DATETIME`, never `TIMESTAMP`, for `starts_at`/`ends_at`.** `TIMESTAMP` is stored as UTC and
  converted to the connection's session `time_zone` on every read — the same booking can render
  at different wall-clock times depending on which connection reads it. `DATETIME` stores
  exactly what was written. Treat every booking time as UTC in application code and never let
  a `TIMESTAMP` column hold one. See `booking-model.md`.
- **There is no `EXCLUDE USING gist` in MySQL.** Double-booking safety comes from `SELECT ...
  FOR UPDATE` row-locking the resource plus a `UNIQUE(resource_id, starts_at)` constraint as a
  second line of defense — not a single declarative constraint. Don't skip the `FOR UPDATE`;
  without it this degrades to the classic unsafe "check then insert" race. See
  `booking-model.md` in full before writing `bookings.py`.
- **MySQL's duplicate-key error code is `1062`, not Postgres's `23P01`.** It surfaces as
  `sqlalchemy.exc.IntegrityError` wrapping a PyMySQL error whose `.args[0]` is `1062` — catch
  that specific code and translate to a 409, the same way the Postgres blueprint catches
  `23P01`.
- **`ENGINE=InnoDB` must be explicit (or at least never accidentally end up as `MyISAM`).**
  InnoDB is MySQL 8's default, but MyISAM has no transactions and no row locks — if a table ever
  ends up on it, every guarantee above silently stops working with no error.
- **Raw `JSON` columns come back as strings, not dicts, through PyMySQL/`text()` queries.**
  Unlike Postgres's `jsonb` (which `pg`/`psycopg` auto-parse), MySQL's `JSON` type read through
  a raw SQL string returns the JSON text itself. Always `json.loads()` on the way out and
  `json.dumps()` on the way in — see `to_resource()` in `architecture.md`. Forgetting this
  doesn't crash; it just serializes a JSON-encoded string inside your JSON response, which the
  Angular side will render as a literal string instead of an object.
- **A `pg`-style multi-statement `pool.query(wholeSchemaFile)` doesn't translate.** SQLAlchemy
  connections (and PyMySQL underneath) generally execute one statement per call. `migrate.py`
  splits `schema.sql` on statement boundaries and executes each individually — see
  `architecture.md`. This is fine because `schema.sql` is simple and fully controlled (no stored
  procedures, no semicolons inside string literals); don't add either of those without revisiting
  the splitter.
- **PyMySQL needs the `cryptography` package installed to connect to MySQL 8 at all**, because
  MySQL 8's default auth plugin (`caching_sha2_password`) needs RSA key exchange that PyMySQL
  delegates to it. Missing it produces an opaque connection error that looks unrelated to auth.
  It's in `requirements.txt` for exactly this reason — don't drop it as "unused."
- **Explicit-ID seed inserts and `AUTO_INCREMENT`**: `server/data/resources.py` seeds fixed
  ids via `INSERT ... ON DUPLICATE KEY UPDATE id = id`. MySQL 8 correctly advances the table's
  auto-increment counter past any explicitly-inserted id (this was actually buggy pre-8.0,
  which is one more reason this blueprint targets MySQL 8+) — but still don't rely on manually
  seeded ids and `AUTO_INCREMENT`-assigned ids ever needing to interleave; keep seed data's ids
  in their own low range if the domain will also let users create resources later.

## Python / FastAPI

- **FastAPI's default error shape is `{"detail": ...}`, not `{"error": ...}`.** Both
  `HTTPException` and `RequestValidationError` need an explicit `@app.exception_handler` to
  reshape their response to `{"error": string}` — see `main.py` in `architecture.md` — so the
  frontend has exactly one error contract to handle, matching the parent blueprint's convention.
  Skipping this means every thrown `HTTPException(409, "...")` reaches the Angular error
  interceptor as `{detail: "..."}` and gets displayed as a generic fallback message instead.
- **Router registration order matters, same as Express middleware order.** API routers must be
  `include_router`'d *before* the catch-all static/SPA mount in `main.py`. Starlette matches
  routes/mounts in registration order — an SPA mount registered first would swallow every
  request, including `/api/*`.
- **`python-dotenv`'s `load_dotenv()` is safe to call more than once** (it's idempotent), so
  `db.py` calls it defensively itself rather than relying on `main.py` importing things in
  exactly the right order. This is a deliberate relaxation of the parent blueprint's stricter
  "`dotenv/config` must be the first import" rule — Python's import-time env reads are the same
  hazard, but `load_dotenv()`'s idempotency gives more slack about *where* it's called from.
- **`pg` returns `NUMERIC` as a string; SQLAlchemy/PyMySQL return MySQL `DECIMAL` as Python
  `Decimal`.** FastAPI's default JSON encoding (`jsonable_encoder`) converts `Decimal` to a
  float automatically when you return a plain dict — so, unlike the Node blueprint, there's
  usually no manual `Number(...)`-equivalent cast needed. Still worth knowing so a `Decimal`
  showing up in a debugger doesn't look like a bug.
- **Never trust client-supplied price, total, or availability.** Same rule as the parent
  blueprint: the client sends a resource id and a requested time range; the server recomputes
  price/total and re-validates availability itself, inside the same transaction that writes the
  booking (see `booking-model.md`).
- **Session checks on protected routes are one pattern: `Depends(get_current_user)`.** Never
  hand-roll a second auth check elsewhere — see `auth-model.md`.
- **Windows note**: activate the venv with `server\venv\Scripts\activate` (not the POSIX
  `source venv/bin/activate`) before `pip install` or running `uvicorn` directly; the root
  `npm run dev`/`server:dev` scripts assume a shell where that's already been done.

## Angular

- **`ng serve` needs `--proxy-config proxy.conf.json` explicitly** — `ng new` does not wire this
  up for you. Without it, any `/api/...` call from the Angular app dev server just 404s (Angular
  tries to resolve it as a client route), which looks like a CORS failure but isn't one — there's
  no CORS involved at all once the proxy is correctly in place, because the browser only ever
  talks to one origin (`localhost:4200`). Wire the flag into `client/package.json`'s `start`
  script, not just remembered as a manual flag, so `npm run dev` (which calls it) always gets it.
- **Use the `@if`/`@for` control-flow syntax**, not the older `*ngIf`/`*ngFor` structural
  directives — it's the current default and most examples generated for this app should use it
  for consistency, even though plenty of code found via search still uses the old syntax.
- **`provideHttpClient()` must be registered in `app.config.ts`**'s `bootstrapApplication`
  providers array. Forgetting it produces a runtime "no provider for HttpClient" error the
  first time any service tries to `inject(HttpClient)` — there's no compile-time check for this.
- **Prefer functional guards (`CanActivateFn`) and functional interceptors**, not the older
  class-based `CanActivate`/`HttpInterceptor` patterns — see `auth.guard.ts` in
  `architecture.md`.
- **Angular's new (v17+) esbuild application builder outputs to `dist/<project>/browser/`**, one
  level deeper than older Angular versions' `dist/<project>/`. `main.py`'s production static-file
  path must match the actual build output exactly, or prod serves 404s for every asset — verify
  the real path with `ng build` once rather than assuming.
- **No `withCredentials: true` and no CORS configuration should be needed anywhere.** If you
  find yourself adding either, something upstream is wrong (most likely the proxy isn't wired
  up, so requests are actually leaving port 4200 for port 8000 directly) — fix that instead of
  patching around it with CORS headers.

# Architecture blueprint

Two-process dev topology, single-process prod: **FastAPI serves the JSON API always; in dev,
Angular's own CLI dev server (`ng serve`) runs alongside it and proxies `/api` requests through
to FastAPI so the browser sees one origin; in production, FastAPI serves Angular's built static
files itself and there's only one process.**

This is a deliberate departure from the Express+Vite blueprint's "always one process," and it's
worth being upfront about why: Vite can run in middleware mode *inside* another Node process,
but Angular's CLI dev server cannot be embedded inside a Python process — it's a standalone Node
tool with its own live-reload/HMR machinery. Rather than fight that, this blueprint leans on
Angular CLI's built-in dev-server proxy (`proxy.conf.json`) to make the two dev processes behave
as one origin from the browser's point of view. The payoff (no CORS, cookies just work) is the
same payoff the single-process approach was chasing — it's just achieved with a proxy instead of
an embedding trick. Production collapses back to one process, same as the Express version.

## File layout

```
package.json            # root — orchestration only: dev, build, start, db:setup, lint
.gitignore
server/
  requirements.txt
  .env.example
  main.py                # entrypoint — FastAPI app, router mounts, static/SPA fallback
  db.py                  # SQLAlchemy engine (MySQL, via PyMySQL)
  auth.py                # bcrypt hashing, session create/verify, get_current_user dependency
  data/
    __init__.py
    resources.py         # seed data source of truth (plain Python list of dicts)
  db/
    __init__.py
    schema.sql            # idempotent, hand-appended
    migrate.py             # applies schema.sql, seeds from data/resources.py
  routers/
    __init__.py
    auth.py                # POST /signup, /signin, /signout, GET /session
    resources.py            # GET /, GET /{id}
    availability.py         # GET /{resource_id}?date=YYYY-MM-DD
    bookings.py               # POST /  (auth required)
    my_bookings.py             # GET /   (auth required)
client/                  # `ng new client` output — its own package.json, angular.json, etc.
  proxy.conf.json        # dev-server proxy: /api -> http://localhost:8000
  src/
    main.ts
    index.html
    app/
      app.config.ts       # providers: HttpClient, Router
      app.routes.ts
      app.component.ts
      services/
        api.service.ts    # thin HttpClient wrapper, mirrors server/api.js's role
        auth.service.ts   # session signal + signIn/signUp/signOut
      guards/
        auth.guard.ts      # functional CanActivateFn
      pages/
        resource-list/
        resource-detail/    # availability picker + "book" form
        sign-in/
        sign-up/
        my-bookings/
```

## Root `package.json`

```json
{
  "name": "<project-name>",
  "private": true,
  "scripts": {
    "dev": "concurrently -n backend,frontend -c blue,green \"npm run server:dev\" \"npm run --prefix client start\"",
    "server:dev": "cd server && uvicorn main:app --reload --port 8000",
    "build": "npm run --prefix client build",
    "start": "cd server && uvicorn main:app --host 0.0.0.0 --port ${PORT:-8000}",
    "db:setup": "cd server && python -m db.migrate",
    "lint": "npm run --prefix client lint && ruff check server"
  },
  "devDependencies": {
    "concurrently": "^9.0.0"
  }
}
```

`npm run dev` assumes the Python virtualenv for `server/` is already active in the current
shell — `uvicorn` needs to resolve to the venv's copy, not a global one. Document this for the
user; it's the Python-side analogue of the parent blueprint's `cross-env` note.

## `client/proxy.conf.json`

```json
{
  "/api": {
    "target": "http://localhost:8000",
    "secure": false,
    "changeOrigin": true
  }
}
```

Wire it into `client/package.json`'s `start` script (`ng serve --proxy-config proxy.conf.json`)
— `ng new` doesn't do this for you, and forgetting it is the single most common way this stack's
dev setup silently breaks (see `gotchas.md`).

## `server/main.py`

```python
from dotenv import load_dotenv
load_dotenv()

import os
from pathlib import Path

from fastapi import FastAPI, HTTPException
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from fastapi.staticfiles import StaticFiles

from routers import auth as auth_router, resources, availability, bookings, my_bookings

IS_PRODUCTION = os.environ.get("ENV") == "production"

app = FastAPI()

@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    # Keep the same {error: string} contract the frontend's api.service.ts expects,
    # instead of FastAPI's default {detail: ...}.
    return JSONResponse(status_code=exc.status_code, content={"error": exc.detail})

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    errors = exc.errors()
    message = errors[0]["msg"] if errors else "Invalid request"
    return JSONResponse(status_code=422, content={"error": message})

# Routers first — they must win over the catch-all static/SPA mount registered below.
app.include_router(auth_router.router, prefix="/api/auth")
app.include_router(resources.router, prefix="/api/resources")
app.include_router(availability.router, prefix="/api/availability")
app.include_router(bookings.router, prefix="/api/bookings")
app.include_router(my_bookings.router, prefix="/api/my-bookings")

if IS_PRODUCTION:
    # Angular's new (v17+) esbuild-based application builder outputs under .../browser/,
    # not directly under dist/<project>/ — check this path against the real build output.
    dist_dir = Path(__file__).resolve().parent.parent / "client" / "dist" / "client" / "browser"

    class SPAStaticFiles(StaticFiles):
        async def get_response(self, path, scope):
            response = await super().get_response(path, scope)
            if response.status_code == 404:
                response = await super().get_response("index.html", scope)
            return response

    app.mount("/", SPAStaticFiles(directory=dist_dir, html=True), name="static")
```

`uvicorn main:app` is run with `cwd=server/`, so `from routers import ...` and `from db import
...` resolve relative to that directory without `server/` itself needing to be an installed
package — only the subpackages it imports (`routers/`, `db/`, `data/`) need an `__init__.py`.

## `server/db.py`

```python
import os
from dotenv import load_dotenv
from sqlalchemy import create_engine

load_dotenv()  # safe to call again here even if main.py already did — idempotent

engine = create_engine(os.environ["DATABASE_URL"], pool_pre_ping=True, pool_recycle=3600)
```

`pool_pre_ping=True` avoids errors from MySQL silently closing idle connections
(`wait_timeout`, default 8 hours) out from under a long-lived pool — the SQLAlchemy analogue of
the Postgres blueprint's Neon SSL note, just for a different failure mode.

## `server/db/migrate.py`

```python
import json
import re
from pathlib import Path

from sqlalchemy import text

from db import engine
from data.resources import RESOURCES

def _statements(sql: str) -> list[str]:
    # schema.sql is fully controlled and simple (no stored procedures, no semicolons inside
    # string literals) so a naive split is safe here — SQLAlchemy/PyMySQL won't run a
    # multi-statement string in one execute() call the way node-postgres can.
    return [s.strip() for s in re.split(r";\s*\n", sql) if s.strip()]

def migrate():
    schema_sql = Path(__file__).parent.joinpath("schema.sql").read_text()
    with engine.begin() as conn:
        for statement in _statements(schema_sql):
            conn.execute(text(statement))

        for r in RESOURCES:
            conn.execute(
                text(
                    """
                    INSERT INTO resources (id, name, category, description, image, price, capacity, metadata)
                    VALUES (:id, :name, :category, :description, :image, :price, :capacity, :metadata)
                    ON DUPLICATE KEY UPDATE id = id
                    """
                ),
                {**r, "metadata": json.dumps(r.get("metadata", {}))},
            )

    print(f"Migrated schema and seeded {len(RESOURCES)} resources.")

if __name__ == "__main__":
    migrate()
```

`ON DUPLICATE KEY UPDATE id = id` is MySQL's shape for "insert, or do nothing if it already
exists" — the direct analogue of Postgres's `ON CONFLICT (id) DO NOTHING`.

## `server/routers/resources.py` (read-only, no auth)

```python
import json
from fastapi import APIRouter, HTTPException
from sqlalchemy import text
from db import engine

router = APIRouter()

def to_resource(row: dict) -> dict:
    return {**row, "metadata": json.loads(row["metadata"])}

@router.get("/")
def list_resources():
    with engine.begin() as conn:
        rows = conn.execute(text("SELECT * FROM resources ORDER BY id")).mappings().all()
    return [to_resource(dict(r)) for r in rows]

@router.get("/{resource_id}")
def get_resource(resource_id: int):
    with engine.begin() as conn:
        row = conn.execute(text("SELECT * FROM resources WHERE id = :id"), {"id": resource_id}).mappings().first()
    if not row:
        raise HTTPException(404, "Resource not found")
    return to_resource(dict(row))
```

`availability.py` and `bookings.py` follow `booking-model.md` exactly — don't improvise the
locking/conflict-handling shape, it's load-bearing. `auth.py` (the router) and `server/auth.py`
(the module) follow `auth-model.md` exactly.

## `client/src/app/services/api.service.ts`

```typescript
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

export interface Resource {
  id: number;
  name: string;
  category: string;
  description: string;
  image: string | null;
  price: number;
  capacity: number;
  metadata: Record<string, unknown>;
}

export interface Slot { startsAt: string; endsAt: string; }

@Injectable({ providedIn: 'root' })
export class ApiService {
  private http = inject(HttpClient);

  listResources(): Observable<Resource[]> {
    return this.http.get<Resource[]>('/api/resources');
  }

  getResource(id: number): Observable<Resource> {
    return this.http.get<Resource>(`/api/resources/${id}`);
  }

  getAvailability(resourceId: number, date: string): Observable<Slot[]> {
    return this.http.get<Slot[]>(`/api/availability/${resourceId}`, { params: { date } });
  }

  createBooking(payload: { resourceId: number; startsAt: string; endsAt: string }) {
    return this.http.post('/api/bookings', payload);
  }

  getMyBookings() {
    return this.http.get('/api/my-bookings');
  }
}
```

Because `proxy.conf.json` makes dev same-origin (and prod is literally same-origin, one
process), no `withCredentials: true` and no `HttpClient` CORS configuration is ever needed —
cookies flow the same way they would for any same-origin fetch. This is the direct payoff of the
proxy approach described above.

## `client/src/app/guards/auth.guard.ts`

```typescript
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { map } from 'rxjs';
import { AuthService } from '../services/auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  return auth.ready$.pipe(
    map(() =>
      auth.session()
        ? true
        : router.createUrlTree(['/login'], { queryParams: { from: state.url } })
    )
  );
};
```

`auth.service.ts` (with the `session` signal and `ready$` observable the guard depends on) is
specified in full in `auth-model.md`.

## `client/src/app/app.routes.ts`

```typescript
import { Routes } from '@angular/router';
import { authGuard } from './guards/auth.guard';

export const routes: Routes = [
  { path: '', loadComponent: () => import('./pages/resource-list/resource-list.component').then(m => m.ResourceListComponent) },
  { path: 'resource/:id', loadComponent: () => import('./pages/resource-detail/resource-detail.component').then(m => m.ResourceDetailComponent) },
  { path: 'login', loadComponent: () => import('./pages/sign-in/sign-in.component').then(m => m.SignInComponent) },
  { path: 'signup', loadComponent: () => import('./pages/sign-up/sign-up.component').then(m => m.SignUpComponent) },
  { path: 'account/bookings', canActivate: [authGuard], loadComponent: () => import('./pages/my-bookings/my-bookings.component').then(m => m.MyBookingsComponent) },
];
```

Browsing resources and checking availability is anonymous-friendly; only the actual booking
submission and "my bookings" require a session — same pattern as the parent blueprint's
"browsing is anonymous, checkout is not." The resource-detail component can let an anonymous
user pick a slot but should route to `/login` (with a `from` query param to return afterward)
when they click "book," rather than gating the whole detail route with `authGuard`.

## Cart-equivalent state: only if the domain needs it

A `BookingCartService` (a small `signal`-backed service, `localStorage`-persisted) only applies
here if the domain interview says a single visit can produce *multiple* bookings queued up
before submission (e.g. renting three pieces of equipment for different windows in one trip).
Most booking domains (one court, one appointment, one room) don't need it — keep the
in-progress selection as a local signal in the resource-detail component, and don't persist it
to `localStorage`: a stale cached time slot is actively misleading (it may no longer be
available), whereas a stale cached cart item in e-commerce is merely inconvenient. If the domain
does need multi-item bookings, still re-validate every item's availability server-side at
submission time — the locking/overlap check in `booking-model.md` is what actually decides,
never the client's cached slot list.

# Hand-rolled auth model (replaces Better Auth)

There's no single dominant, turnkey auth library in the Python ecosystem that plays the same
role Better Auth plays for the Express/React blueprint (batteries-included email/password +
session tables + client hooks, wired in a few lines). The closest candidates
(`fastapi-users`, `authlib`) pull in an ORM/ full ORM model layer and more configuration surface
than this class of app needs. Given this skill's own bias toward raw SQL over an ORM (see
`architecture.md`), this blueprint hand-rolls the same *shape* of thing Better Auth gives you —
bcrypt-hashed passwords, an opaque session token in an `httpOnly` cookie, a `sessions` table,
and a `get_current_user` dependency — in about 80 lines, with no extra dependency risk and full
visibility into what it's doing.

One simplification this buys versus Better Auth: sessions here are **opaque random tokens**,
not JWTs, so there is no `BETTER_AUTH_SECRET`-equivalent signing key to generate or rotate.
Validity is decided by a database lookup, not by verifying a signature — one less secret to
manage, at the cost of one extra query per authenticated request (acceptable for this class of
app; the `sessions` table is tiny and indexed on its primary key).

## Schema

```sql
CREATE TABLE IF NOT EXISTS users (
  id VARCHAR(36) PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  name VARCHAR(255),
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS sessions (
  id VARCHAR(64) PRIMARY KEY,       -- SHA-256 hex digest of the raw token, not the token itself
  user_id VARCHAR(36) NOT NULL,
  expires_at DATETIME NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB;
```

Put both of these *before* `resources`/`bookings` in `schema.sql` — `bookings.user_id`
references `users(id)`.

Sessions store a **hash** of the token, not the raw token: the raw token only ever exists in the
client's cookie and in memory for one request. If the `sessions` table ever leaked, an attacker
would have hashes, not usable session tokens — a small, cheap hardening step Better Auth also
does internally.

## `server/auth.py`

```python
import hashlib
import os
import secrets
from datetime import datetime, timedelta, timezone

import bcrypt
from fastapi import HTTPException, Request, Response
from sqlalchemy import text

from db import engine

SESSION_COOKIE = "session"
SESSION_TTL_DAYS = 30

def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt()).decode()

def verify_password(password: str, password_hash: str) -> bool:
    return bcrypt.checkpw(password.encode(), password_hash.encode())

def _hash_token(token: str) -> str:
    return hashlib.sha256(token.encode()).hexdigest()

def create_session(response: Response, user_id: str) -> None:
    token = secrets.token_urlsafe(32)
    expires_at = datetime.now(timezone.utc) + timedelta(days=SESSION_TTL_DAYS)
    with engine.begin() as conn:
        conn.execute(
            text("INSERT INTO sessions (id, user_id, expires_at) VALUES (:id, :user_id, :expires_at)"),
            {"id": _hash_token(token), "user_id": user_id, "expires_at": expires_at},
        )
    response.set_cookie(
        SESSION_COOKIE,
        token,
        httponly=True,
        samesite="lax",
        secure=os.environ.get("ENV") == "production",
        max_age=SESSION_TTL_DAYS * 24 * 3600,
    )

def clear_session(request: Request, response: Response) -> None:
    token = request.cookies.get(SESSION_COOKIE)
    if token:
        with engine.begin() as conn:
            conn.execute(text("DELETE FROM sessions WHERE id = :id"), {"id": _hash_token(token)})
    response.delete_cookie(SESSION_COOKIE)

def get_current_user(request: Request) -> dict:
    token = request.cookies.get(SESSION_COOKIE)
    if not token:
        raise HTTPException(401, "Sign in required")
    with engine.begin() as conn:
        row = conn.execute(
            text(
                """
                SELECT u.id, u.email, u.name FROM sessions s
                JOIN users u ON u.id = s.user_id
                WHERE s.id = :id AND s.expires_at > UTC_TIMESTAMP()
                """
            ),
            {"id": _hash_token(token)},
        ).mappings().first()
    if not row:
        raise HTTPException(401, "Sign in required")
    return dict(row)

def get_optional_user(request: Request) -> dict | None:
    try:
        return get_current_user(request)
    except HTTPException:
        return None
```

`UTC_TIMESTAMP()` (not `NOW()`) matches the "everything is UTC, always" rule from
`booking-model.md` — `NOW()` returns the session's local time zone, `UTC_TIMESTAMP()` doesn't.

## `server/routers/auth.py`

```python
import uuid
from fastapi import APIRouter, Depends, HTTPException, Request, Response
from pydantic import BaseModel, EmailStr
from sqlalchemy import text
from sqlalchemy.exc import IntegrityError

from db import engine
from auth import clear_session, create_session, get_optional_user, hash_password, verify_password

router = APIRouter()

class SignUpBody(BaseModel):
    email: EmailStr
    password: str
    name: str | None = None

class SignInBody(BaseModel):
    email: EmailStr
    password: str

@router.post("/signup")
def signup(body: SignUpBody, response: Response):
    user_id = str(uuid.uuid4())
    try:
        with engine.begin() as conn:
            conn.execute(
                text(
                    "INSERT INTO users (id, email, password_hash, name) VALUES (:id, :email, :password_hash, :name)"
                ),
                {"id": user_id, "email": body.email, "password_hash": hash_password(body.password), "name": body.name},
            )
    except IntegrityError:
        raise HTTPException(409, "An account with that email already exists")
    create_session(response, user_id)
    return {"id": user_id, "email": body.email, "name": body.name}

@router.post("/signin")
def signin(body: SignInBody, response: Response):
    with engine.begin() as conn:
        row = conn.execute(text("SELECT * FROM users WHERE email = :email"), {"email": body.email}).mappings().first()
    if not row or not verify_password(body.password, row["password_hash"]):
        raise HTTPException(401, "Invalid email or password")
    create_session(response, row["id"])
    return {"id": row["id"], "email": row["email"], "name": row["name"]}

@router.post("/signout")
def signout(request: Request, response: Response):
    clear_session(request, response)
    return {"ok": True}

@router.get("/session")
def session(user: dict | None = Depends(get_optional_user)):
    return user
```

`GET /api/auth/session` returning `null` for "not signed in" (rather than a 401) is
intentional — it's a status read, not a protected action, and `auth.service.ts` below treats a
`null` body as "no session" without needing to catch an error.

## `client/src/app/services/auth.service.ts`

```typescript
import { HttpClient } from '@angular/common/http';
import { Injectable, inject, signal } from '@angular/core';
import { Observable, ReplaySubject, catchError, of, tap } from 'rxjs';

export interface Session { id: string; email: string; name: string | null; }

@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);

  readonly session = signal<Session | null>(null);
  private readonly readySubject = new ReplaySubject<void>(1);
  readonly ready$ = this.readySubject.asObservable();

  constructor() {
    this.refreshSession();
  }

  refreshSession(): void {
    this.http.get<Session | null>('/api/auth/session').pipe(
      catchError(() => of(null))
    ).subscribe((session) => {
      this.session.set(session);
      this.readySubject.next();
    });
  }

  signUp(email: string, password: string, name?: string): Observable<Session> {
    return this.http.post<Session>('/api/auth/signup', { email, password, name }).pipe(
      tap((session) => this.session.set(session))
    );
  }

  signIn(email: string, password: string): Observable<Session> {
    return this.http.post<Session>('/api/auth/signin', { email, password }).pipe(
      tap((session) => this.session.set(session))
    );
  }

  signOut(): Observable<unknown> {
    return this.http.post('/api/auth/signout', {}).pipe(
      tap(() => this.session.set(null))
    );
  }
}
```

`ready$` is a `ReplaySubject(1)` rather than a plain flag so `auth.guard.ts` (in
`architecture.md`) can subscribe to it *after* the initial session check has already resolved
(e.g. on a route the user deep-links straight into) and still get the right answer immediately,
instead of racing the constructor's initial `refreshSession()` call.

## Gotchas specific to this auth model

- **bcrypt has a 72-byte input limit.** Anything past that is silently ignored by most
  implementations. Irrelevant for realistic passwords; worth knowing if someone pastes something
  absurd in testing and wonders why two different long strings hash the same.
- **`secrets.token_urlsafe(32)`, not `uuid4()`, for the session token.** UUIDs are not
  generated with a CSPRNG guarantee suitable for security tokens in every implementation;
  `secrets` is Python's standard library module explicitly designed for this.
- **Never log the raw token or put it anywhere but the `Set-Cookie` header.** Only its SHA-256
  hash should ever reach the database or a log line.
- **Session expiry check happens in SQL (`expires_at > UTC_TIMESTAMP()`), not in Python after
  fetching the row.** Keeps clock-skew and timezone handling in one place.

# The generalized booking data model (MySQL)

One schema shape, reused for every domain (tennis courts, doctor appointments, salon slots,
equipment rentals, meeting rooms, ...). Domain differences live in labels and a `metadata`
column, not in forked schemas — same philosophy as the Postgres blueprint. What's genuinely
different here is *how* double-booking races get closed: MySQL has no range type and no
`EXCLUDE USING gist`-style constraint, so this blueprint uses row locking
(`SELECT ... FOR UPDATE`) plus a `UNIQUE` constraint instead of a single declarative constraint.
Read this whole file before writing `schema.sql` or the booking route — the two layers only work
together.

## Schema

```sql
CREATE TABLE IF NOT EXISTS resources (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  category VARCHAR(255) NOT NULL,
  description TEXT NOT NULL,
  image VARCHAR(1024),
  price DECIMAL(10,2) NOT NULL,
  capacity INT NOT NULL DEFAULT 1,
  metadata JSON NOT NULL
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS bookings (
  id INT AUTO_INCREMENT PRIMARY KEY,
  booking_ref VARCHAR(64) NOT NULL UNIQUE,
  resource_id INT NOT NULL,
  user_id VARCHAR(36) NOT NULL,
  starts_at DATETIME NOT NULL,
  ends_at DATETIME NOT NULL,
  status VARCHAR(32) NOT NULL DEFAULT 'confirmed',
  total DECIMAL(10,2) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (resource_id) REFERENCES resources(id),
  FOREIGN KEY (user_id) REFERENCES users(id),
  UNIQUE KEY uq_resource_slot (resource_id, starts_at),
  INDEX idx_bookings_resource_range (resource_id, starts_at, ends_at)
) ENGINE=InnoDB;
```

`ENGINE=InnoDB` is explicit and load-bearing — it's MySQL 8's default, but if it were ever
`MyISAM` there'd be no transactions and no row locks, silently breaking every guarantee below.

Two deliberate choices worth flagging:

- **`starts_at`/`ends_at` are `DATETIME`, not `TIMESTAMP`.** MySQL's `TIMESTAMP` type is stored
  as UTC internally and *converted to the connection's session `time_zone` on every read* —
  which means the same row can render as a different wall-clock time depending on which
  connection reads it. `DATETIME` stores and returns exactly what was written, no conversion.
  Store and compare every booking time as UTC in application code, and never let a `TIMESTAMP`
  column anywhere near a value that means "this specific instant a court is reserved for" — the
  ambiguity is a correctness bug, not a style choice.
- **`UNIQUE KEY uq_resource_slot (resource_id, starts_at)`** catches exact-duplicate bookings on
  the same resource at the same instant, race-safe by itself (MySQL enforces `UNIQUE` at insert
  time under any isolation level). It is *not* a full overlap guard — two bookings for
  `10:00–11:00` and `10:30–11:30` on the same resource have different `starts_at` values and
  this constraint won't catch the overlap. That's what the locking pattern below is for.

## Availability: compute on request, don't pre-generate a slots table

Given a resource and a date, generate candidate slots from the resource's own metadata
(operating hours + slot duration), then subtract ranges that overlap existing confirmed
bookings — same approach as the Postgres blueprint, just with MySQL's `DATETIME` comparisons
instead of `tstzrange`:

```python
# server/routers/availability.py
from fastapi import APIRouter, HTTPException, Query
from sqlalchemy import text
from db import engine
from lib.slots import build_candidate_slots  # pure function: (date, open_hour, close_hour, slot_minutes) -> [(starts_at, ends_at)]

router = APIRouter()

@router.get("/{resource_id}")
def get_availability(resource_id: int, date: str = Query(...)):
    with engine.begin() as conn:
        resource = conn.execute(text("SELECT * FROM resources WHERE id = :id"), {"id": resource_id}).mappings().first()
        if not resource:
            raise HTTPException(404, "Resource not found")

        metadata = json.loads(resource["metadata"])
        candidates = build_candidate_slots(
            date, metadata.get("openHour", 9), metadata.get("closeHour", 18), metadata.get("slotMinutes", 60)
        )

        booked = conn.execute(
            text(
                """
                SELECT starts_at, ends_at FROM bookings
                WHERE resource_id = :id AND status = 'confirmed'
                  AND starts_at < :window_end AND ends_at > :window_start
                """
            ),
            {"id": resource_id, "window_start": candidates[0][0], "window_end": candidates[-1][1]},
        ).all()

    available = [
        {"startsAt": s.isoformat(), "endsAt": e.isoformat()}
        for s, e in candidates
        if not any(s < b.ends_at and e > b.starts_at for b in booked)
    ]
    return available
```

This works for fixed-duration domains (tennis courts, salon chairs, doctor slots) and
free-form-duration domains alike (for the latter, skip candidate generation and just let the
client propose a `starts_at`/`ends_at`; the overlap check and the locking pattern below don't
care which produced the range).

## Booking creation: lock the resource, recheck for real, then insert

This is the part that replaces Postgres's `EXCLUDE USING gist`. The mechanism: `SELECT ... FOR
UPDATE` takes an exclusive row lock on the resource being booked, for the duration of the
transaction. A second concurrent request trying to book the *same* resource blocks at that same
`SELECT ... FOR UPDATE` line until the first transaction commits or rolls back — so by the time
the second request's own overlap check runs, it is guaranteed to see whatever the first request
just committed. That ordering guarantee is exactly what closes the classic
"both requests pass the availability check, then both insert" race.

```python
# server/routers/bookings.py
import time
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from sqlalchemy import text
from sqlalchemy.exc import IntegrityError
from db import engine
from auth import get_current_user

router = APIRouter()

class BookingCreate(BaseModel):
    resource_id: int
    starts_at: str  # ISO 8601, treated as UTC
    ends_at: str

@router.post("/")
def create_booking(body: BookingCreate, user=Depends(get_current_user)):
    with engine.begin() as conn:  # BEGIN ... COMMIT/ROLLBACK, one connection for the whole thing
        resource = conn.execute(
            text("SELECT * FROM resources WHERE id = :id FOR UPDATE"), {"id": body.resource_id}
        ).mappings().first()
        if not resource:
            raise HTTPException(404, "Resource not found")

        overlap = conn.execute(
            text(
                """
                SELECT id FROM bookings
                WHERE resource_id = :id AND status = 'confirmed'
                  AND starts_at < :ends_at AND ends_at > :starts_at
                LIMIT 1
                """
            ),
            {"id": body.resource_id, "starts_at": body.starts_at, "ends_at": body.ends_at},
        ).first()
        if overlap:
            raise HTTPException(409, "That time was just booked. Please pick another slot.")

        total = compute_total(resource, body.starts_at, body.ends_at)  # server computes price, never trusts the client
        booking_ref = f"BK-{int(time.time() * 1000)}"
        try:
            result = conn.execute(
                text(
                    """
                    INSERT INTO bookings (booking_ref, resource_id, user_id, starts_at, ends_at, total)
                    VALUES (:ref, :resource_id, :user_id, :starts_at, :ends_at, :total)
                    """
                ),
                {
                    "ref": booking_ref, "resource_id": body.resource_id, "user_id": user["id"],
                    "starts_at": body.starts_at, "ends_at": body.ends_at, "total": total,
                },
            )
        except IntegrityError as err:
            # 1062 = duplicate entry — the UNIQUE(resource_id, starts_at) constraint caught an
            # exact-slot race that somehow reached here without FOR UPDATE serializing it
            # (belt-and-suspenders; the FOR UPDATE lock above should already prevent this).
            if getattr(err.orig, "args", [None])[0] == 1062:
                raise HTTPException(409, "That time was just booked. Please pick another slot.")
            raise

    return {"bookingRef": booking_ref, "startsAt": body.starts_at, "endsAt": body.ends_at}
```

Do not skip the `FOR UPDATE`. Without it, this degrades to the exact "`SELECT` then `INSERT`"
race the Postgres blueprint calls out as unsafe — the `UNIQUE` constraint alone only catches
*exact* `starts_at` collisions, not partial overlaps, so a free-form-duration domain would have
no race protection at all without the lock.

## Capacity > 1

Default: model each identical unit as its **own row** in `resources` (e.g. "Court 1", "Court 2",
"Meeting Room A", "Chair 3") rather than a `capacity` column with counting logic. The locking
pattern above then works unmodified — no extra code. Only reach for counting-based capacity
(`SELECT COUNT(*) ... FOR UPDATE` against all of a resource's confirmed bookings in the
requested range, compared to `capacity`) if the user specifically asks for shared-capacity
resources (e.g. a class with 20 seats, one row) — and even then, the `FOR UPDATE` on the
resource row is still what serializes the count-then-insert sequence.

## Domain → terminology mapping

Ask the user (or infer from their description) which row applies, then substitute consistently
across schema comments, route names, and UI copy — the schema and code shape never change.

| Domain | resource noun | booking noun | typical `metadata` fields |
|---|---|---|---|
| Tennis courts | Court | Reservation | `surface`, `indoor`, `slotMinutes` |
| Doctor appointments | Provider | Appointment | `specialty`, `slotMinutes` |
| Salon slots | Stylist / Chair | Appointment | `services`, `slotMinutes` |
| Equipment rentals | Item | Rental | `sku`, `depositAmount` (often free-form start/end, not fixed slots) |
| Meeting rooms | Room | Booking | `capacity`, `amenities` |
| Custom | (ask the user) | (ask the user) | (ask the user) |

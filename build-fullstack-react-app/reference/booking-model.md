# The generalized booking data model

One schema shape, reused for every domain (tennis courts, doctor appointments, salon slots,
equipment rentals, meeting rooms, ...). Domain differences live in labels and a `metadata`
column, not in forked schemas. This is the part of the stack the e-commerce blueprint didn't
need to solve — e-commerce has no concept of two people racing for the same time slot.

## Schema

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE IF NOT EXISTS resources (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  category TEXT NOT NULL,
  description TEXT NOT NULL,
  image TEXT,
  price NUMERIC(10,2) NOT NULL,
  capacity INTEGER NOT NULL DEFAULT 1,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb
);

CREATE TABLE IF NOT EXISTS bookings (
  id SERIAL PRIMARY KEY,
  booking_ref TEXT UNIQUE NOT NULL,
  resource_id INTEGER NOT NULL REFERENCES resources(id),
  user_id TEXT REFERENCES "user"("id"),
  starts_at TIMESTAMPTZ NOT NULL,
  ends_at TIMESTAMPTZ NOT NULL,
  during TSTZRANGE NOT NULL GENERATED ALWAYS AS (tstzrange(starts_at, ends_at, '[)')) STORED,
  status TEXT NOT NULL DEFAULT 'confirmed',
  total NUMERIC(10,2) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT no_double_booking EXCLUDE USING gist (resource_id WITH =, during WITH &&)
    WHERE (status = 'confirmed')
);
```

The `EXCLUDE USING gist` constraint is the whole point: Postgres itself rejects any insert whose
`during` range overlaps an existing *confirmed* booking for the same `resource_id`. This is
race-safe under concurrent requests in a way that an application-level `SELECT` check before
`INSERT` is not — two requests can both pass the `SELECT` and then both `INSERT` before either
commits. Only the database constraint closes that window.

`status` lets you soft-cancel (`UPDATE bookings SET status = 'cancelled' WHERE ...`) without
deleting history — a cancelled booking's range no longer counts against the exclusion constraint
because of the `WHERE (status = 'confirmed')` clause, freeing the slot immediately.

## Availability: compute on request, don't pre-generate a slots table

Given a resource and a date, generate candidate slots from the resource's own metadata
(operating hours + slot duration), then subtract ranges that overlap existing confirmed
bookings:

```js
// server/routes/availability.js
router.get('/:resourceId', async (req, res) => {
  const { date } = req.query // 'YYYY-MM-DD'
  const { rows: [resource] } = await pool.query('SELECT * FROM resources WHERE id = $1', [req.params.resourceId])
  if (!resource) return res.status(404).json({ error: 'Resource not found' })

  const { openHour = 9, closeHour = 18, slotMinutes = 60 } = resource.metadata
  const candidates = buildCandidateSlots(date, openHour, closeHour, slotMinutes) // [{ startsAt, endsAt }]

  const { rows: booked } = await pool.query(
    `SELECT starts_at, ends_at FROM bookings
     WHERE resource_id = $1 AND status = 'confirmed' AND during && tstzrange($2, $3, '[)')`,
    [resource.id, candidates[0].startsAt, candidates[candidates.length - 1].endsAt]
  )

  const available = candidates.filter((slot) =>
    !booked.some((b) => slot.startsAt < b.ends_at && slot.endsAt > b.starts_at)
  )

  res.json(available)
})
```

This works for fixed-duration domains (tennis courts, salon chairs, doctor slots) and
free-form-duration domains alike (for the latter, skip candidate generation and just let the
client propose a `starts_at`/`ends_at`; the availability check and the exclusion constraint don't
care which produced the range).

## Booking creation: recompute, don't trust, and let the constraint decide

```js
// server/routes/bookings.js
router.post('/', async (req, res) => {
  const session = await auth.api.getSession({ headers: fromNodeHeaders(req.headers) })
  if (!session) return res.status(401).json({ error: 'Sign in required to book' })

  const { resourceId, startsAt, endsAt } = req.body
  const client = await pool.connect()
  try {
    await client.query('BEGIN')
    const { rows: [resource] } = await client.query('SELECT * FROM resources WHERE id = $1', [resourceId])
    if (!resource) throw Object.assign(new Error('Resource not found'), { status: 404 })

    const total = computeTotal(resource, startsAt, endsAt) // server computes price, never trusts the client
    const bookingRef = `BK-${Date.now()}`
    const { rows: [booking] } = await client.query(
      `INSERT INTO bookings (booking_ref, resource_id, user_id, starts_at, ends_at, total)
       VALUES ($1, $2, $3, $4, $5, $6) RETURNING id, booking_ref, starts_at, ends_at`,
      [bookingRef, resourceId, session.user.id, startsAt, endsAt, total]
    )
    await client.query('COMMIT')
    res.json(booking)
  } catch (err) {
    await client.query('ROLLBACK')
    if (err.code === '23P01') { // exclusion_violation — the constraint caught a race
      return res.status(409).json({ error: 'That time was just booked. Please pick another slot.' })
    }
    if (err.status) return res.status(err.status).json({ error: err.message })
    throw err
  } finally {
    client.release()
  }
})
```

`23P01` is Postgres's error code for `exclusion_violation`. Catching it and returning 409 is what
turns "the database rejected a write" into a clean user-facing message instead of a 500.

## Capacity > 1

Default: model each identical unit as its **own row** in `resources` (e.g. "Court 1", "Court 2",
"Meeting Room A", "Chair 3") rather than a `capacity` column with counting logic. The exclusion
constraint then works unmodified — no extra code. Only reach for counting-based capacity
(`COUNT(*) FILTER (WHERE during && requested) < capacity`, which needs a `SERIALIZABLE`
transaction or advisory lock to stay race-safe, since `EXCLUDE` can't count) if the user
specifically asks for shared-capacity resources (e.g. a class with 20 seats, one row).

## Domain → terminology mapping

Ask the user (or infer from their description) which row applies, then substitute consistently
across schema comments, route names, and UI copy — the schema and code shape never change.

| Domain | resource noun | booking noun | typical `metadata` fields |
|---|---|---|---|
| Tennis courts | Court | Reservation | `surface`, `indoor`, `slotMinutes` |
| Doctor appointments | Provider | Appointment | `specialty`, `slotMinutes` |
| Salon slots | Stylist / Chair | Appointment | `services[]`, `slotMinutes` |
| Equipment rentals | Item | Rental | `sku`, `depositAmount` (often free-form start/end, not fixed slots) |
| Meeting rooms | Room | Booking | `capacity`, `amenities[]` |
| Custom | (ask the user) | (ask the user) | (ask the user) |

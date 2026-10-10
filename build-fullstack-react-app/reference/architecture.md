# Architecture blueprint

Single-server full-stack shape: **one Express process serves both the JSON API and the
Vite-built React frontend** — no separate frontend dev server, no separate deploy target for
client vs. server. This is deliberate: it collapses CORS, env-var duplication, and "which port is
which" confusion, at the cost of not being able to scale the frontend and API independently
(acceptable for this class of app).

## File layout

```
package.json            # scripts: dev, build, start, db:setup, lint
vite.config.js
index.html
.env.example
.oxlintrc.json
.gitignore
server/
  index.js              # entrypoint — dotenv/config first, then everything else
  db.js                 # pg Pool
  auth.js               # betterAuth() instance
  data/
    resources.js        # seed data source of truth (plain JS array)
  db/
    schema.sql          # idempotent, hand-appended
    migrate.js          # applies schema.sql, seeds from data/resources.js
  routes/
    resources.js         # GET /, GET /:id
    availability.js      # GET /:resourceId?date=YYYY-MM-DD
    bookings.js           # POST /  (auth required), used for "book this slot"
    my-bookings.js         # GET /   (auth required), the signed-in user's bookings
src/
  main.jsx
  App.jsx                # routes + CartProvider-equivalent if needed
  api.js                 # thin fetch wrapper
  lib/
    auth-client.js       # createAuthClient()
  components/
    RequireAuth.jsx
    Navbar.jsx
  pages/
    ResourceList.jsx
    ResourceDetail.jsx    # availability picker + "book" form
    SignIn.jsx
    SignUp.jsx
    MyBookings.jsx
```

## server/index.js

```js
import 'dotenv/config'
import fs from 'node:fs/promises'
import path from 'node:path'
import { fileURLToPath } from 'node:url'
import express from 'express'
import { toNodeHandler } from 'better-auth/node'
import { auth } from './auth.js'
import resourcesRouter from './routes/resources.js'
import availabilityRouter from './routes/availability.js'
import bookingsRouter from './routes/bookings.js'
import myBookingsRouter from './routes/my-bookings.js'

const __dirname = path.dirname(fileURLToPath(import.meta.url))
const root = path.resolve(__dirname, '..')
const isProduction = process.env.NODE_ENV === 'production'
const port = process.env.PORT || 3000

async function createServer() {
  const app = express()

  // Must be mounted before express.json() — Better Auth needs the raw request stream.
  app.all('/api/auth/*splat', toNodeHandler(auth))

  app.use(express.json())

  app.use('/api/resources', resourcesRouter)
  app.use('/api/availability', availabilityRouter)
  app.use('/api/bookings', bookingsRouter)
  app.use('/api/my-bookings', myBookingsRouter)

  if (isProduction) {
    const distDir = path.join(root, 'dist')
    app.use(express.static(distDir))
    app.use((req, res) => {
      res.sendFile(path.join(distDir, 'index.html'))
    })
  } else {
    const { createServer: createViteServer } = await import('vite')
    const vite = await createViteServer({ root, server: { middlewareMode: true }, appType: 'custom' })

    app.use(vite.middlewares)
    app.use(async (req, res, next) => {
      try {
        const url = req.originalUrl
        let template = await fs.readFile(path.join(root, 'index.html'), 'utf-8')
        template = await vite.transformIndexHtml(url, template)
        res.status(200).set({ 'Content-Type': 'text/html' }).end(template)
      } catch (err) {
        vite.ssrFixStacktrace(err)
        next(err)
      }
    })
  }

  app.listen(port, () => {
    console.log(`Server listening on http://localhost:${port} (${isProduction ? 'production' : 'development'})`)
  })
}

createServer()
```

## server/db.js

```js
import { Pool } from 'pg'

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: { rejectUnauthorized: false },
})
```

## server/auth.js

```js
import { betterAuth } from 'better-auth'
import { pool } from './db.js'

export const auth = betterAuth({
  database: pool,
  emailAndPassword: { enabled: true },
  secret: process.env.BETTER_AUTH_SECRET,
  baseURL: process.env.BETTER_AUTH_URL,
  trustedOrigins: [process.env.BETTER_AUTH_URL].filter(Boolean),
})
```

Generate the Better Auth core tables (`"user"`, `"session"`, `"account"`, `"verification"`) with
`npx @better-auth/cli generate` against this config and append the output into `schema.sql`
rather than hand-typing it — see `gotchas.md` on why the quoting matters.

## server/db/migrate.js

```js
import 'dotenv/config'
import fs from 'node:fs/promises'
import path from 'node:path'
import { fileURLToPath } from 'node:url'
import { pool } from '../db.js'
import { RESOURCES } from '../data/resources.js'

const __dirname = path.dirname(fileURLToPath(import.meta.url))

async function migrate() {
  const schema = await fs.readFile(path.join(__dirname, 'schema.sql'), 'utf-8')
  await pool.query(schema)

  for (const resource of RESOURCES) {
    await pool.query(
      `INSERT INTO resources (id, name, category, description, image, price, capacity, metadata)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
       ON CONFLICT (id) DO NOTHING`,
      [resource.id, resource.name, resource.category, resource.description, resource.image, resource.price, resource.capacity ?? 1, resource.metadata ?? {}]
    )
  }

  console.log(`Migrated schema and seeded ${RESOURCES.length} resources.`)
  await pool.end()
}

migrate().catch((err) => {
  console.error(err)
  process.exit(1)
})
```

## server/routes/resources.js (read-only, no auth)

```js
import { Router } from 'express'
import { pool } from '../db.js'

const router = Router()

function toResource(row) {
  return { ...row, price: Number(row.price) }
}

router.get('/', async (req, res) => {
  const { rows } = await pool.query('SELECT * FROM resources ORDER BY id')
  res.json(rows.map(toResource))
})

router.get('/:id', async (req, res) => {
  const { rows } = await pool.query('SELECT * FROM resources WHERE id = $1', [req.params.id])
  if (rows.length === 0) return res.status(404).json({ error: 'Resource not found' })
  res.json(toResource(rows[0]))
})

export default router
```

`availability.js` and `bookings.js` follow `booking-model.md` exactly — don't improvise the
conflict-handling shape, it's load-bearing.

## server/routes/my-bookings.js

```js
import { Router } from 'express'
import { fromNodeHeaders } from 'better-auth/node'
import { pool } from '../db.js'
import { auth } from '../auth.js'

const router = Router()

router.get('/', async (req, res) => {
  const session = await auth.api.getSession({ headers: fromNodeHeaders(req.headers) })
  if (!session) return res.status(401).json({ error: 'Sign in required' })

  const { rows } = await pool.query(
    `SELECT b.id, b.booking_ref, b.starts_at, b.ends_at, b.total, b.status, r.name AS resource_name
     FROM bookings b JOIN resources r ON r.id = b.resource_id
     WHERE b.user_id = $1 ORDER BY b.starts_at DESC`,
    [session.user.id]
  )
  res.json(rows.map((r) => ({ ...r, total: Number(r.total) })))
})

export default router
```

## src/api.js

```js
async function request(url, options) {
  const res = await fetch(url, options)
  if (!res.ok) {
    const body = await res.json().catch(() => ({}))
    throw new Error(body.error || `Request failed: ${res.status}`)
  }
  return res.json()
}

export function fetchResources() { return request('/api/resources') }
export function fetchResource(id) { return request(`/api/resources/${id}`) }
export function fetchAvailability(resourceId, date) { return request(`/api/availability/${resourceId}?date=${date}`) }
export function createBooking(payload) {
  return request('/api/bookings', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) })
}
export function fetchMyBookings() { return request('/api/my-bookings') }
```

## src/lib/auth-client.js

```js
import { createAuthClient } from 'better-auth/react'

export const authClient = createAuthClient() // same-origin, no baseURL needed
export const { useSession, signIn, signUp, signOut } = authClient
```

## src/components/RequireAuth.jsx

```jsx
import { Navigate, useLocation } from 'react-router-dom'
import { useSession } from '../lib/auth-client'

export default function RequireAuth({ children }) {
  const { data: session, isPending } = useSession()
  const location = useLocation()

  if (isPending) return null
  if (!session) return <Navigate to="/login" state={{ from: location.pathname }} replace />
  return children
}
```

## src/App.jsx (routes)

```jsx
import { Routes, Route } from 'react-router-dom'
import Navbar from './components/Navbar'
import RequireAuth from './components/RequireAuth'
import ResourceList from './pages/ResourceList'
import ResourceDetail from './pages/ResourceDetail'
import SignIn from './pages/SignIn'
import SignUp from './pages/SignUp'
import MyBookings from './pages/MyBookings'

export default function App() {
  return (
    <>
      <Navbar />
      <main>
        <Routes>
          <Route path="/" element={<ResourceList />} />
          <Route path="/resource/:id" element={<ResourceDetail />} /> {/* booking itself, not just browsing, still needs a session check inside */}
          <Route path="/login" element={<SignIn />} />
          <Route path="/signup" element={<SignUp />} />
          <Route path="/account/bookings" element={<RequireAuth><MyBookings /></RequireAuth>} />
        </Routes>
      </main>
    </>
  )
}
```

Browsing resources and checking availability is anonymous-friendly; only the actual booking
submission and "my bookings" require a session — same pattern as the e-commerce blueprint's
"browsing is anonymous, checkout is not." `ResourceDetail` can let an anonymous user pick a slot
but should redirect through `/login` (with return-path state, same as `RequireAuth`) when they
click "book," rather than gating the whole detail page.

## Cart-equivalent state: only if the domain needs it

The e-commerce blueprint's `CartContext` (a `useReducer` + `localStorage`-persisted list) only
applies here if the domain interview says a single visit can produce *multiple* bookings queued
up before submission (e.g. renting three pieces of equipment for different windows in one trip).
Most booking domains (one court, one appointment, one room) don't need it — keep the in-progress
selection as local component state in `ResourceDetail`, and don't persist it to `localStorage`:
a stale cached time slot is actively misleading (it may no longer be available), whereas a stale
cached cart item in e-commerce is merely inconvenient. If the domain does need multi-item
bookings, port `CartContext` verbatim but rename to `BookingCartContext` / storage key
`booking_cart`, and still re-validate every item's availability server-side at submission time.

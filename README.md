# 🍕 Pizza Platform

A full-stack pizza ordering and kitchen-inventory platform. Customers assemble a
pizza step by step, pay through Razorpay, and watch their order move through the
kitchen in real time. Staff get a separate portal for the order queue, revenue,
and low-stock alerts over email.

```
pizza-platform/
├── server/     Node + Express + MongoDB + Socket.IO + Razorpay + Gmail SMTP
└── client/     React 18 + Vite + React Router + Socket.IO client
```

---

## Quick start

**Prerequisites:** Node 18+ (developed on 24.21.0), a running MongoDB.

```bash
# 1. Install
cd server && npm install
cd ../client && npm install

# 2. Configure
cd ../server
cp .env.example .env        # then fill in the values below

# 3. Seed the kitchen and the first admin
npm run seed                # 27 ingredients
npm run seed:admin          # admin from ADMIN_EMAIL / ADMIN_PASSWORD

# 4. Run (two terminals)
npm run dev                 # API  → http://localhost:5000
cd ../client && npm run dev # Web  → http://localhost:5173
```

The Vite dev server proxies `/api` and `/socket.io` to port 5000, so the client
needs no configuration.

---

## Configuration

Everything is read from `server/.env` at boot. `.env.example` documents every
key; the ones you must fill in:

| Key | Why |
| --- | --- |
| `MONGO_URI` | Your MongoDB connection string. |
| `JWT_SECRET`, `JWT_REFRESH_SECRET` | Signs access and refresh tokens. Generate with `node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"`. |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Credentials created by `npm run seed:admin`. The password is hashed at seed time and only read then. |
| `SMTP_USER`, `SMTP_PASS` | Gmail address + **App Password**. Needed for verification and password-reset email. |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` | Razorpay **test mode** keys from the Razorpay dashboard. |

### Optional

| Key | Default | Purpose |
| --- | --- | --- |
| `MAIL_LOG_TO_FILE` | `true` | Writes every rendered email to `server/logs/mail.log`. Verification and reset links are logged here when SMTP is not configured, which is what makes local testing possible. |
| `STOCK_ALERT_CRON` | `*/10 * * * *` | How often the low-stock sweep runs. |
| `STOCK_ALERT_ENABLED` | `true` | Set `false` to disable the cron entirely. |
| `DISABLE_RATE_LIMIT` | `false` | Set `true` **only** for automated suites — lifts the `/api/auth` rate limits so a test run is not throttled. |

### Payments without keys

If the Razorpay keys are placeholders the API still works: order creation and
quoting succeed, and `POST /api/orders/verify-payment` returns
`502 PAYMENT_PROVIDER_DOWN`. Checkout shows a clear error instead of failing
silently, so the whole flow is testable before you add keys.

### Email without SMTP

With `MAIL_LOG_TO_FILE=true` every email is written to `server/logs/mail.log`
whether or not SMTP succeeds. Open that file and click the link. This is the
intended local workflow.

---

## Roles

Two roles, enforced on both sides. The API never trusts the client.

| | Customer | Admin |
| --- | --- | --- |
| Register, verify, order, track | ✅ | — |
| `/admin/*` routes and screens | ❌ | ✅ |
| Ordering flow (`/build`, `/checkout`, `/orders`) | ✅ | ❌ redirected to `/admin` |
| Inventory read/write | menu only | full CRUD |

Admin sessions and customer sessions are separate. `/admin/login` hits its own
endpoint (`POST /api/auth/admin/login`), is rate-limited independently, and
cannot be reached by registering — there is no way to self-promote to admin.
Attempting to escalate a customer token is rejected by `requireAdmin` on every
admin route.

---

## API

All responses are `{ success, ... }` or `{ success: false, message, code }`.
Errors carry a stable `code` (`VALIDATION_ERROR`, `UNAUTHORIZED`,
`FORBIDDEN`, `NOT_FOUND`, `RATE_LIMITED`, `CONFLICT`, `PAYMENT_PROVIDER_DOWN`).

### Auth — `/api/auth`

| Method | Path | Notes |
| --- | --- | --- |
| `POST` | `/register` | Returns tokens immediately; verification is emailed. |
| `POST` | `/verify-email` | Idempotent — a double-submitted link still succeeds. |
| `POST` | `/resend-verification` | Always the same response, so accounts cannot be enumerated. |
| `POST` | `/login` | Customer login. |
| `POST` | `/admin/login` | Admin login, separate limiter. |
| `POST` | `/refresh` | Rotates the refresh token. |
| `POST` | `/logout` | Revokes the presented refresh token. |
| `GET` `PATCH` | `/me` | Read / update profile. |
| `POST` | `/change-password` | Revokes all other sessions. |
| `POST` | `/forgot-password` | Never reveals whether the account exists. |
| `POST` | `/reset-password` | Single-use token, 1 hour. |

### Inventory — `/api/inventory`

| Method | Path | Access |
| --- | --- | --- |
| `GET` | `/menu` | public |
| `GET` | `/` | admin |
| `POST` | `/` | admin |
| `PATCH` `DELETE` | `/:id` | admin |
| `POST` | `/:id/restock` | admin |
| `GET` | `/summary` | admin |
| `POST` | `/low-stock/check` | admin |
| `POST` | `/thresholds/bulk` | admin |

### Orders — `/api/orders`

| Method | Path | Access |
| --- | --- | --- |
| `GET` | `/config` | public — Razorpay key id for the browser |
| `POST` | `/quote` | customer — server-priced cart |
| `POST` | `/` | verified customer — creates a `PAYMENT_PENDING` order |
| `POST` | `/verify-payment` | verified customer — signature check, then stock is committed |
| `GET` | `/` `/:id` | customer — own orders only |
| `POST` | `/:id/cancel` | customer — restores stock |
| `GET` | `/admin/all` `/admin/stats` | admin |
| `PATCH` | `/:id/status` | admin — validated against the status graph |

---

## How the tricky parts work

### Prices are computed server-side, always

The client sends ingredient ids and quantity. The server looks up every price
and returns the authoritative total. A tampered client cannot change what is
charged, and the pay button shows the same number the server will verify.

### Stock is decremented at payment verification, not order creation

Creating an order reserves nothing. Stock is committed only after Razorpay's
signature check passes, so an abandoned checkout never starves the kitchen.

Because MongoDB is running standalone here, multi-document transactions are
unavailable. Instead each ingredient line is decremented with a single guarded
`$inc` (`stock: { $gte: qty }`) so an oversell is impossible at the database
level. If any line fails, the lines already decremented are compensated back
exactly, and the order is cancelled — no orphaned `PAYMENT_PENDING` rows.

### Order status is a graph, not a free field

```
PLACED → PREPARING → BAKING → OUT_FOR_DELIVERY → DELIVERED
   └──────────────→ CANCELLED
```

`PATCH /admin/status` rejects any transition not in `STATUS_FLOW`, so an order
cannot jump from `PLACED` straight to `DELIVERED` or move backwards.

### Realtime with a polling fallback

Socket.IO pushes status changes into per-user, per-order and `admins` rooms.
A `useLiveQuery` hook polls as a backstop and backs off when the socket is
disconnected, so tracking still updates if websockets are blocked.

### Low-stock alerts

A cron sweep compares `stock` to `threshold` per ingredient and emails the
configured recipients. Admins can also trigger it on demand and edit thresholds
in bulk. Every email is logged to `server/logs/mail.log`.

### Razorpay errors are not `Error` objects

The SDK rejects with a plain `{ statusCode, error }` object, which a normal
`instanceof Error` check misses. The error middleware detects that shape and
returns `502 PAYMENT_PROVIDER_DOWN` instead of leaking the provider's internals.

---

## Testing

Two suites live outside the repo so they never ship with the app.

```bash
# API — 48 checks
node C:/WINDOWS/TEMP/opencode/apitest/smoke.js

# Browser — 43 checks (Playwright)
# start both dev servers, then:
set DISABLE_RATE_LIMIT=true
node C:/WINDOWS/TEMP/opencode/apitest/ui.cjs
```

Both are green. The browser suite covers registration and email verification,
the 4-step builder, server-priced checkout, the admin portal, role separation,
live badges, password reset, mobile layout, and the 404 page. It writes
screenshots to `C:/WINDOWS/TEMP/opencode/shots/`.

The browser suite needs its own `browser.newContext()` for the admin: sharing a
context shares the cookie jar, so the admin would inherit the customer session.

---

## Scripts

| Where | Command | Does |
| --- | --- | --- |
| `server` | `npm run dev` | API with nodemon |
| `server` | `npm start` | API |
| `server` | `npm run seed` | 27 ingredients (5 bases, 5 sauces, 5 cheeses, 12 vegetables) |
| `server` | `npm run seed:admin` | Admin from env |
| `client` | `npm run dev` | Vite dev server |
| `client` | `npm run build` | Production build |
| `client` | `npm run lint` | ESLint |

---

## Notes and limitations

- **MongoDB must be a replica set to use transactions.** The code works
  standalone via guarded `$inc` plus compensation, which is correct but does not
  give multi-document atomicity. A replica set would let `reserveStock` become a
  single transaction.
- **Razorpay and Gmail are unverified end-to-end** until real keys are pasted
  into `server/.env`. Everything up to the provider call is tested.
- **`server/logs/mail.log` contains live verification and reset links.** It is
  gitignored and must not be committed or shipped.
- The admin account is a single seeded user; there is no admin-user management
  screen.
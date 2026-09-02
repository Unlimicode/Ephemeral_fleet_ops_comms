# SwiftLink

[![CI](https://github.com/Unlimicode/Ephemral_fleet_ops_comms/actions/workflows/ci.yml/badge.svg)](https://github.com/Unlimicode/Ephemral_fleet_ops_comms/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**Privacy-preserving fleet operations and communication platform for small fleet operators.**

SwiftLink lets a fleet operator dispatch trips, assign drivers, and carry
client–driver communication end to end **without ever handing the client's
contact details to the driver**. Messaging runs through a mediated WebSocket
relay that opens when a trip is accepted and closes when it ends, so the driver
has no way to reach the client outside the system and no client data lingers on
a personal phone.

> BSc Computer Science final-year project — Ian Lemashon Sopia (SCT221-0593/2022),
> Jomo Kenyatta University of Agriculture and Technology.

---

## Table of contents

- [Actors](#actors)
- [How the privacy model works](#how-the-privacy-model-works)
- [Tech stack](#tech-stack)
- [Repository layout](#repository-layout)
- [Getting started](#getting-started)
- [Environment variables](#environment-variables)
- [Testing & CI](#testing--ci)
- [Further documentation](#further-documentation)

---

## Actors

| Actor | Interface | Responsibilities |
|-------|-----------|------------------|
| **Fleet manager** | Desktop web dashboard | Owns the platform; assigns drivers and vehicles, manages the roster and vehicle inventory, handles complaints, and has full audit visibility. Signs in via magic link. |
| **Driver** | Mobile-first PWA | Receives trip assignments (pickup, destination, notes — never client contact details), communicates with the client through the relay, and progresses the trip through its lifecycle. Signs in with a password. |
| **Client** | Mobile-first PWA | Corporate trip booker. Submits a booking, tracks status, chats with the assigned driver, and can file a complaint within 24h of completion. Access is via a single-use booking token emailed on submission; the session is an HttpOnly cookie. |

## How the privacy model works

- **Data confinement.** The operator collects only what is operationally
  necessary, and that data stays inside the organisation — it never flows
  downstream to the driver.
- **Ephemeral sessions.** Auth sessions live in Redis with a TTL tied to the
  trip lifecycle. Relay session keys (`session:trip:{id}:driver` /
  `session:trip:{id}:client`) are created when a driver accepts and deleted on
  completion.
- **Mediated messaging.** Client and driver never exchange identifiers. Messages
  pass through a Socket.IO relay namespace and are archived encrypted
  (AES-256-GCM); the archive is only decryptable while a complaint is under
  investigation.
- **Compliance tooling.** The manager dashboard exposes a data-protection
  compliance report (PDF) and an audit-log export (CSV), backed by legal-basis
  and retention-category columns on the audit log.

### Trip lifecycle

```
pending ──▶ accepted ──▶ in_progress ──▶ completed
   │            │
   └─ cancelled ┘   (client may cancel at pending or accepted, not in_progress)
```

## Tech stack

**Backend** — Node.js 20, Express 5, PostgreSQL (`pg`), Redis, Socket.IO,
`jsonwebtoken`, `web-push`, Brevo transactional email API.

**Frontend** — React 19, Vite 7, React Router 7, Tailwind CSS 3, Socket.IO
client, Axios. Driver and client interfaces are installable PWAs.

**Tooling** — GitHub Actions CI (backend tests + frontend lint/build), ngrok for
local device testing via the root `start-dev.js`.

## Repository layout

```
.
├── start-dev.js            Boots backend + frontend + an ngrok tunnel together
├── implementation_plan.md  Sprint-by-sprint engineering log (append-only)
├── docs/                   User guide spec, dissertation notes, outstanding work
│
├── backend/
│   ├── server.js           Express entry point
│   ├── config/             db, redis, redisHelpers (TTL utils), mailer, webpush
│   ├── middleware/          auth.js (manager/driver JWT), clientAuth.js (client cookie)
│   ├── routes/              auth, bookings, complaints, contact, dashboard,
│   │                       drivers, driverTrips, messages, push, roster, trips,
│   │                       vehicles, flights  (mounted by index.js)
│   ├── socket/             io.js (relay namespace), dashboardNamespace.js
│   ├── utils/              encryption, push-notification dispatchers
│   ├── database/           schema.sql, migrations/, seed.js, apply-migrations.js
│   └── tests/              14 Jest suites (run in CI)
│
└── frontend/
    └── src/
        ├── api/            Axios instance + Bearer interceptor
        ├── context/        AuthContext (token / role / user)
        ├── hooks/          useChat, usePushNotifications, useOnlineStatus, …
        ├── components/     Shared UI; components/layout/ holds the role shells
        ├── pages/          Route components — manager/, driver/, booking/,
        │                   plus public LoginPage / SwiftlinkHomePage
        ├── styles/         tokens.css (design tokens), animations.css
        └── utils/          ripple, compliancePdf
```

## Getting started

### Prerequisites

- Node.js 20 (`.nvmrc`)
- PostgreSQL 14+
- Redis 6+

### 1. Install

```bash
# backend
cd backend && npm install

# frontend
cd ../frontend && npm install

# root (dev tunnel helper only)
cd .. && npm install
```

### 2. Configure

```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

Fill in `backend/.env` — at minimum `DATABASE_URL`, `REDIS_URL`, and
`JWT_SECRET`. See [Environment variables](#environment-variables).

### 3. Create and seed the database

```bash
cd backend
psql "$DATABASE_URL" -f database/schema.sql
node database/apply-migrations.js 001_driver_notifications.sql \
  002_audit_log_compliance.sql 003_add_trips_created_at.sql \
  004_add_additional_info_eta_vehicle_details.sql \
  005_add_cancelled_trip_status.sql 006_add_client_push_subscriptions.sql \
  007_enquiries.sql 008_direct_messages.sql 009_global_direct_messages.sql \
  010_driver_password_resets.sql
npm run seed
```

### 4. Run

```bash
# backend  (http://localhost:3001)
cd backend && npm run dev

# frontend (http://localhost:5173)
cd frontend && npm run dev
```

Or run everything plus an ngrok tunnel for phone testing:

```bash
npm run dev        # from the repo root — needs NGROK_AUTHTOKEN in backend/.env
```

The seed script prints the manager magic-link and the driver / client test
credentials.

## Environment variables

All backend configuration lives in `backend/.env` (see `backend/.env.example`
for the full list with comments).

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis connection string |
| `JWT_SECRET`, `JWT_EXPIRES_IN` | Session token signing and default TTL |
| `CLIENT_ORIGIN` | Allowed CORS origin for the frontend |
| `BREVO_API_KEY`, `MAIL_FROM` | Transactional email (all mail via `config/mailer.js`) |
| `CONTACT_ENQUIRY_EMAIL` | Destination for corporate enquiry submissions |
| `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_MAILTO` | Web Push |
| `TEST_MANAGER_EMAIL`, `TEST_MANAGER_PASSWORD` | Seed / test credentials |
| `AVIATIONSTACK_KEY` | Optional — flight-number ETA enrichment on bookings |
| `NGROK_AUTHTOKEN` | Only used by the root `start-dev.js` tunnel |

The frontend reads `VITE_API_URL` and `VITE_WS_URL` (see
`frontend/.env.example`).

## Testing & CI

```bash
cd backend && npm test          # 14 Jest suites, run with --runInBand
cd frontend && npm run lint     # ESLint
cd frontend && npm run build    # production build
```

GitHub Actions (`.github/workflows/ci.yml`) runs the backend suite and the
frontend lint + build on every push and pull request to `main` / `develop`.
Backend test secrets are supplied through repository secrets.

## Further documentation

| File | Contents |
|------|----------|
| [`implementation_plan.md`](implementation_plan.md) | Chronological sprint log — every change, its files, and its rationale |
| [`docs/outstanding_system.md`](docs/outstanding_system.md) | Remaining gaps and planned follow-up work |
| [`docs/USER_GUIDE.md`](docs/USER_GUIDE.md) | Spec for the in-system help feature (source of truth for the help UI) |
| [`docs/chapter4_annotated.md`](docs/chapter4_annotated.md) | Requirement-to-code and claim-to-test mapping for the dissertation |

## License

Released under the [MIT License](LICENSE).

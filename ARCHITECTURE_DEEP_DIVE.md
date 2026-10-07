# Job Application Tracker - Architecture Deep Dive

This document explains how the project is built, why each part exists, and how a junior developer could build the same system from zero.

## 1. What This App Does

The app is a full-stack tracker for a user's job pipeline:
- user authentication (register/login/session)
- protected app routes
- CRUD on job applications
- filter + sort + pagination
- workflow transitions (`applied -> interview -> offer/rejected`)
- dashboard metrics

## 2. Tech Stack and Why

### Backend
- Node + Express: fast API iteration, simple middleware model
- SQLite via `better-sqlite3`: zero external infra needed, fast local persistence
- JWT: stateless auth for API routes
- Zod: reliable runtime validation

### Frontend
- React + Vite: simple SPA setup
- React Router: route-level auth gating
- Zustand: lightweight global session state
- React Hook Form + Zod: form validation with minimal boilerplate
- Axios: shared API client + auth header interceptor

## 3. High-Level System Design

### Request flow
1. User authenticates (`/api/auth/register` or `/api/auth/login`).
2. Server returns JWT + user payload.
3. Client stores token in localStorage + Zustand.
4. Axios attaches `Authorization: Bearer <token>`.
5. Protected routes call API; backend middleware validates token.

### Data ownership
- Backend is source of truth for:
  - auth validation
  - workflow transition rules
  - query filtering/sorting/pagination
- Frontend is source of truth for:
  - current UI state (filters, modal open/close)
  - pending async loading/error states

## 4. Backend Structure

### Entry point (`server/src/server.js`)
Responsibilities:
- load env
- assert `JWT_SECRET`
- initialize DB schema (`require('./db')`)
- register middleware (`cors`, `json`, `morgan`)
- mount domain routes:
  - `/api/auth`
  - `/api/applications`
  - `/api/dashboard`

Why this order matters:
- DB schema must exist before any route handler runs
- JSON parsing middleware must be active before POST/PUT handlers

### DB bootstrap (`server/src/db.js`)
Creates tables and indexes once at startup:
- `users`
- `applications`
Indexes:
- user-scoped lookups
- status filters
- date sorting

Why create schema in code:
- easy onboarding
- no manual migration step for small project

### Validation (`server/src/validation.js`)
All request contracts are centrally defined:
- auth payloads
- application create/update payloads
- query params

Why centralize:
- avoids validation logic being scattered across routes
- easier to update contract in one place

### Auth middleware (`server/src/middleware/auth.js`)
- extracts bearer token
- verifies JWT with server secret
- attaches decoded user to `req.user`

Why middleware:
- all protected routes share one gate
- keeps handlers focused on business logic

### Application routes (`server/src/routes/applications.js`)
Core behaviors:
- list with filter/search/sort/page
- create, read single, update, delete
- explicit transition endpoint (`PATCH /:id/transition`)

Important design choices:
- all queries are user-scoped (`user_id = req.user.id`)
- workflow transitions validated server-side (never trust UI only)
- partial update salary check runs after merge to prevent invalid min/max combinations

### Dashboard route (`server/src/routes/dashboard.js`)
- grouped counts by status
- derived KPIs (`active`, `conversionRate`)

Why precompute on server:
- keeps frontend simple
- avoids recomputing from full dataset client-side

## 5. Frontend Structure

### Router and auth boot (`client/src/App.jsx`)
- `AuthBootstrap` runs once to hydrate session
- public routes: `/login`, `/register`
- protected app shell at `/`

Why hydrate before rendering protected content:
- prevents flash of unauthorized routes
- ensures stale tokens are cleaned by `/auth/me` validation

### Session store (`client/src/store/authStore.js`)
State:
- `user`, `token`, `isReady`
Actions:
- `hydrate()`
- `setSession()`
- `logout()`

Why `isReady` exists:
- UI can distinguish "still checking session" vs "not logged in"

### Applications page (`client/src/pages/ApplicationsPage.jsx`)
Major patterns:
- debounced search text -> API params
- `loadApplications()` reused after every mutation
- modal reused for create and edit
- transition buttons map to allowed next states

Why reuse one loader:
- every mutation gets fresh data from source of truth
- avoids client-side patch bugs on table rows

### Form modal (`client/src/components/ApplicationFormModal.jsx`)
- zod schema for field-level constraints
- supports both create and edit initial values

Why one modal component:
- avoids duplicate form logic and drift

## 6. Build-From-Scratch Plan (Junior-Friendly)

If building from zero, do this order:

1. Scaffold backend and frontend.
2. Add DB with `users` and `applications` tables.
3. Implement auth endpoints and JWT middleware.
4. Build protected `/applications` read endpoint first.
5. Add create/update/delete endpoints.
6. Add status transition rules and endpoint.
7. Add dashboard aggregate endpoint.
8. Build frontend auth pages and route guard.
9. Build list view (read-only first).
10. Add filters/sort/page query wiring.
11. Add create/edit modal forms.
12. Add delete + transition actions.
13. Add dashboard cards.
14. Add polish: loading/errors/empty states.

## 7. How Pieces Fit Together (Concrete Example)

Example: moving `applied -> interview`

1. User clicks "Move to Interview".
2. UI calls `PATCH /api/applications/:id/transition`.
3. Backend checks:
   - user owns row
   - current status is `applied`
   - `interview` is allowed transition
4. Backend updates row + `updated_at`.
5. UI reloads list using `loadApplications()`.
6. Dashboard stats reflect new counts on next fetch.

## 8. Security and Correctness Notes

- Auth token is always server-verified (`/auth/me`) during hydrate.
- All app rows are user-scoped in SQL.
- Workflow enforcement is backend-only source of truth.
- Query param validation limits invalid sorting/filtering fields. `sort_by` is interpolated into SQL (`server/src/routes/applications.js:51`), which is safe only because `querySchema` restricts it to a fixed enum (`server/src/validation.js:68`). Keep that enum if you touch the query.

### Known gaps (verified against the source)

These were checked against the code in this snapshot on 2026-10-06. The maintained repository, [elirc/job-track](https://github.com/elirc/job-track), has since fixed the server-side causes of the first three in its root app.

- **No optional field can be cleared.** The server turns `""` into `undefined` for every optional field (`server/src/validation.js:24,32,38,40-41,45`), and the update merge keeps the old value whenever a field is `undefined` (`server/src/routes/applications.js:144-151`). The client's `toPayload` also turns every blank optional field into `undefined` (`client/src/components/ApplicationFormModal.jsx:71-85`). So a location, URL, salary, next-step date or note, once saved, can be changed but never removed, through the UI or the API. The `|| null` branches in the merge never see an empty string.
- **A blank salary sent to the API is stored as `0`.** `z.coerce.number()` runs before the `z.literal('')` branch (`server/src/validation.js:40-41`), and `Number("")` is `0`. The UI avoids this only because it never sends blank salaries.
- **Dates are checked for shape only.** The regex at `server/src/validation.js:21,42` accepts `2026-02-30`.
- **Duplicate registration can crash the server.** `/register` checks for the email (`server/src/routes/auth.js:18`), then `await`s bcrypt (line 24), then inserts (lines 26-28). Two simultaneous sign-ups with the same email can both pass the check; the second insert hits the `UNIQUE` constraint (`server/src/db.js:20`) and throws inside an `async` handler. Express 4 (`server/package.json`: `^4.21.2`) does not forward a rejected promise to the error handler at `server/src/server.js:39-42`, so it becomes an unhandled rejection, which ends the process by default on Node 22. A fix is try/catch around the insert, returning 409.
- **Login timing differs by account existence.** The messages match (`server/src/routes/auth.js:57,62`), but a missing email returns before `bcrypt.compare` runs (line 60). Response time can therefore reveal which emails are registered.
- **The transition table exists twice.** `client/src/utils/status.js` `TRANSITIONS` copies `server/src/constants.js` `ALLOWED_TRANSITIONS`. They match today; nothing enforces that.

## 9. Common Mistakes to Avoid

- trusting front-end-only transition validation
- forgetting `user_id` in WHERE clauses
- updating list UI optimistically without reconciling server truth
- not validating partial updates after merged state

## 10. Next Improvements (If Extending)

- refresh tokens and rotating JWT secrets
- per-field audit history (status changed by/date)
- stronger pagination (cursor-based)
- test suite (API + component tests); this snapshot has no automated tests and no `test` script
- role-based collaboration (team view)

## 11. File Map to Study

Backend:
- `server/src/server.js`
- `server/src/db.js`
- `server/src/middleware/auth.js`
- `server/src/routes/auth.js`
- `server/src/routes/applications.js`
- `server/src/routes/dashboard.js`
- `server/src/validation.js`
- `server/src/constants.js` (statuses and `ALLOWED_TRANSITIONS`)
- `server/src/utils.js` (`parseOrRespond`)

Frontend:
- `client/src/App.jsx`
- `client/src/store/authStore.js`
- `client/src/lib/api.js`
- `client/src/pages/ApplicationsPage.jsx`
- `client/src/components/ApplicationFormModal.jsx`
- `client/src/pages/DashboardPage.jsx`

`client/src/main.ts`, `client/src/counter.ts` and `client/src/style.css` are leftovers from the Vite TypeScript template. `client/index.html` loads `/src/main.jsx`, so they never run.

# Job Application Tracker

Full-stack JavaScript app for tracking job applications with auth, protected routes, validation, CRUD, filtering/sorting, workflow transitions, and dashboard stats.

## Stack
- Frontend: React + Vite + React Router + Zustand + React Hook Form + Zod
- Backend: Node.js + Express + SQLite (better-sqlite3) + JWT auth

## Run
```bash
npm install
npm run install:all
npm run dev
```

- Frontend: http://localhost:5173
- Backend: http://localhost:4000

## Demo flow
1. Register a new user.
2. Create applications (default status `applied`).
3. Move statuses with workflow buttons:
   - applied -> interview -> offer/rejected
   - applied -> rejected
4. Filter/search/sort in Applications.
5. View stats and recent updates in Dashboard.

Create `server/.env` with a `JWT_SECRET` before `npm run dev`; the server throws at startup without it (`server/src/server.js:8-10`). This copy has no automated tests. In this snapshot its source matches the root app; see section 0 of `ARCHITECTURE_DEEP_DIVE.md` for the known gaps they share.

## API endpoints
- `POST /api/auth/register`
- `POST /api/auth/login`
- `GET /api/auth/me`
- `GET /api/dashboard/stats`
- `GET /api/applications`
- `GET /api/applications/:id`
- `POST /api/applications`
- `PUT /api/applications/:id`
- `PATCH /api/applications/:id/transition`
- `DELETE /api/applications/:id`
# Job Application Tracker

Full-stack JavaScript app for tracking job applications with auth, protected routes, validation, CRUD, filtering/sorting, workflow transitions, and dashboard stats.

## Stack
- Frontend: React + Vite + React Router + Zustand + React Hook Form + Zod
- Backend: Node.js + Express + SQLite (better-sqlite3) + JWT auth

## Run
Create `server/.env` with at least `JWT_SECRET=<any long random string>` first: the server refuses to start without it (`server/src/server.js:8-10`). This snapshot ships no `.env.example`.

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

## About this snapshot

This repository is a snapshot of [elirc/job-track](https://github.com/elirc/job-track) from before its upskilling courses and fixes were added. There are no automated tests yet. `job-tracker-next/` has the same server and client source as the root app here. `ARCHITECTURE_DEEP_DIVE.md` section 8 lists the known gaps.

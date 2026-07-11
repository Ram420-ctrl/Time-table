# Latticework — Dynamic Timetable Management System

A full-stack academic scheduling platform that automatically generates conflict-free class and faculty timetables, manages workloads, and handles emergency rescheduling in real time.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — run the API server (port 8080)
- `pnpm --filter @workspace/latticework run dev` — run the frontend (port 20892)
- `pnpm run typecheck` — full typecheck across all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from the OpenAPI spec
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- Required env: `DATABASE_URL` — Postgres connection string

## Stack

- pnpm workspaces, Node.js 24, TypeScript 5.9
- Frontend: React 18 + Vite, Wouter routing, Tailwind CSS, Recharts, Framer Motion
- API: Express 5 (`artifacts/api-server`)
- DB: PostgreSQL + Drizzle ORM (`lib/db`)
- Validation: Zod (`zod/v4`), `drizzle-zod`
- API codegen: Orval (from OpenAPI spec at `lib/api-spec/openapi.yaml`)
- Build: esbuild (CJS bundle)

## Where things live

- `lib/api-spec/openapi.yaml` — single source of truth for all API contracts
- `lib/db/src/schema/` — Drizzle table definitions (faculty, subjects, classes, rooms, assignments, temp_overrides, conflict_logs)
- `artifacts/api-server/src/lib/scheduler.ts` — core constraint-satisfaction scheduling algorithm
- `artifacts/api-server/src/routes/` — Express route handlers (faculty, subjects, classes, rooms, timetable, reports, dashboard)
- `artifacts/latticework/src/` — React frontend (pages, components)

## Architecture decisions

- **Contract-first OpenAPI**: All API types are generated from `lib/api-spec/openapi.yaml` — never hand-write types that codegen produces.
- **Scheduler algorithm**: Most-constrained-first (MCF) greedy CSP. Sessions are sorted by ascending qualified-faculty count; least-loaded available faculty is picked per slot. Hard conflicts logged when no slot is found; soft conflicts logged for overload.
- **Minimal-disruption reschedule**: `rescheduleFaculty()` removes only the affected faculty's assignments and replaces them, leaving all other assignments intact.
- **Temporary overrides**: Stored as a separate `temp_overrides` table; timetable view endpoints apply active overrides (by date range) at query time — the base `assignments` table is never mutated.
- **PostgreSQL JSONB**: `availableDays`, `qualifiedSubjectIds`, `onLeaveDates`, `subjectIds` stored as JSONB arrays for schema simplicity.

## Product

- **Dashboard** — live stats (faculty, classes, sessions placed, conflicts, utilization %)
- **Setup** — CRUD for Faculty, Subjects, Classes, Rooms
- **Timetable Generate** — one-click scheduling with before/after results
- **Class / Faculty Timetable** — weekly grid view with active override highlighting
- **Temporary Overrides** — substitute faculty with expiry date, no base schedule mutation
- **Workload Report** — bar chart of hours vs capacity per faculty, overload badges
- **Conflicts Report** — hard/soft conflict log with resolution status

## User preferences

_Populate as you build — explicit user instructions worth remembering across sessions._

## Gotchas

- After any `lib/*` schema change, run `pnpm run typecheck:libs` before checking artifact packages — stale declarations cause false TS errors.
- After any OpenAPI spec change, re-run `pnpm --filter @workspace/api-spec run codegen` before touching routes or frontend.
- JSONB columns from Drizzle come back as `unknown` at runtime — cast with `as number[]` / `as string[]` where needed in routes and scheduler.
- The `conflictLogsTable.type` column is typed `"hard" | "soft"` — values from `InsertConflictLog` must be cast with `as "hard" | "soft"` before inserting.

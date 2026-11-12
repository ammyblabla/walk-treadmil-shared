# Task list

**Last updated:** 2026-11-10
**Sources:** [01 introduction](requirement/01-introduction.md), [02 DB schema and API](requirement/02-db-schema-and-api.md), [03 frontend requirement](requirement/03-frontend-requirement.md), [ADR-0001](adr/0001-tech-stack.md).

## How to use this list

- Work top to bottom. Each task is one small slice: **one endpoint**, **one piece of logic**, or **one frontend page**.
- Each task is one commit on `main` (trunk-based) and leaves `main` deployable. Unfinished features stay unwired.
- Tick a box only when the task is on `main`, tests and lint pass, and (for server tasks) it is deployed.
- Tags: **[server]** = `walk-treadmil-server`, **[client]** = `walk-treadmil-client`.
- "Spec" means [requirement/02-db-schema-and-api.md](requirement/02-db-schema-and-api.md).

## Progress

| # | Milestone | What you can do when it's finished | Done |
|---|---|---|---|
| 0 | Foundation | Backend deploys to Cloud Run with DB, auth and profile API | 9 / 9 |
| 1 | Walking skeleton | Open the app on the phone, sign in, see the (mock) home screen | 2 / 9 |
| 2 | Profile and preferences | Edit profile, treadmill limits and defaults in the app | 0 / 5 |
| 3 | First real plan | Pick a time, get a real steady plan, see it as a chart and step list | 0 / 7 |
| 4 | All styles and conditions | All 7 styles; sleep, period and injury change the plan | 0 / 14 |
| 5 | Run and summary | Run a plan offline, upload it, see planned vs actual | 0 / 13 |
| 6 | Adjust the plan | Quick chips, change inputs, version history | 0 / 8 |
| 7 | Real home | Home shows a real suggestion, last workout and week | 0 / 7 |
| 8 | History | Browse, add, fix and delete workouts | 0 / 7 |
| 9 | Chat | Free-text plan changes in a bottom sheet | 0 / 6 |
| 10 | Import and analysis | Screenshot or link becomes a workout with feedback | 0 / 7 |
| 11 | Garmin sync (Phase 1.5) | Connect Garmin and import history | 0 / 10 |
| 12 | Hardening | Safe retries, account deletion, CI guards | 0 / 7 |
| 13 | Audio guide (Phase 2) | The app tells you when to change speed or incline | 0 / 3 |
| 14 | Tech tasks | Safer, more automated delivery | 1 / 3 |

Milestones 1–8 are the usable MVP without an LLM. 9–11 complete Phase 1 and 1.5.

---

## 0. Foundation — done

- [x] **0.1 [server]** FastAPI app with `GET /health`
- [x] **0.2 [server]** `GET /api/v1/home` returning mock data
- [x] **0.3 [server]** Dockerfile and `docker compose` (API + Postgres 18)
- [x] **0.4 [server]** Terraform for Cloud Run, Artifact Registry, budget; CI tests and deploys `main`
- [x] **0.5 [server]** SQLAlchemy, Alembic, local Postgres
- [x] **0.6 [server]** ORM models for all 13 tables and the initial migration
- [x] **0.7 [server]** `GET/PUT /me/profile` with computed HR zones (`planner/hr_zones.py`)
- [x] **0.8 [server]** Firebase ID token check, just-in-time user creation, `AUTH_MODE=dev` for local
- [x] **0.9 [server]** Production database: Neon Postgres 18 (`aws-ap-southeast-1`), URLs in Secret Manager, wired into Cloud Run; smoke test checks `/me/profile` returns 401

## 1. Walking skeleton — client on the phone

Goal: the thinnest path from phone to deployed backend, before any real feature.

- [x] **1.1 [client]** Scaffold Vite + React 19 + TypeScript, Tailwind, shadcn/ui, Vitest, oxlint (Vite's template default, chosen over ESLint). One placeholder page
- [x] **1.2 [client]** GitHub Actions: typecheck, lint, test, build
- [ ] **1.3 [client]** Deploy to Cloudflare Pages on push to `main`
- [ ] **1.4 [server]** CORS: allow the Pages origin and `http://localhost:5173` (from config)
- [ ] **1.5 [client]** `npm run gen:api` (`openapi-typescript` + `openapi-fetch`) and an API client that reads the base URL from env
- [ ] **1.6 [client]** Sign-in page with Firebase Google sign-in; route guard; send the ID token as `Bearer` on every call. First, register a **Web app** in Firebase project `walk-treadmil` (none exists yet) for the client config, and add the Pages domain to Auth → **Authorized domains**
- [ ] **1.7 [client]** App shell: React Router, bottom navigation (Home, History, Profile), theme ported from `walk-treadmil-client/mockup/index.html`
- [ ] **1.8 [client]** Home page that renders `GET /home`: plan card with a small chart, Start button, quick chips (not wired yet), week summary
- [ ] **1.9 [client]** PWA: manifest, icons, service worker (`vite-plugin-pwa`). Check "Install app" on the phone

## 2. Profile and preferences

- [ ] **2.1 [client]** Profile page: view and edit weight, birth year, max/resting HR, HR ceiling; show computed HR zones
- [ ] **2.2 [client]** Profile page, treadmill section: min/max speed, max incline, speed and incline steps
- [ ] **2.3 [server]** `PreferenceSet` Pydantic model and `GET/PUT /me/preferences`
- [ ] **2.4 [client]** Preferences section: default duration, favourite styles, warm-up/cool-down minutes, comfortable speed range, max incline, track period
- [ ] **2.5 [server]** `GET /me` (user + profile + preferences + integration status) and `PATCH /me` (`display_name`, `timezone`)

## 3. First real plan — steady style, end to end

- [ ] **3.1 [server]** Planner types (`ResolvedInput`, `Plan`, `PlanStep`) and `planner.generate()` for style `steady`: warm-up, steady block, cool-down. Table-driven tests
- [ ] **3.2 [server]** Planner hard limits: clamp to treadmill min/max and round to the speed and incline steps
- [ ] **3.3 [server]** `PreferenceResolver`: system defaults < preferences < session overrides, with precedence tests
- [ ] **3.4 [server]** `POST /sessions`: save session, plan v1 and steps; return session, plan and `resolved_input`
- [ ] **3.5 [server]** `GET /sessions/{id}`: session with its current plan and steps (404 for other users)
- [ ] **3.6 [client]** Set up plan page, step "time": slider and chips, then `POST /sessions` (style fixed to steady for now)
- [ ] **3.7 [client]** Plan preview page: speed and incline over time chart (Recharts), step list, Start button (not wired yet)

## 4. All styles and conditions

Planner — one style per task, each with table-driven tests:

- [ ] **4.1 [server]** Style `easy`
- [ ] **4.2 [server]** Style `flat` (no incline)
- [ ] **4.3 [server]** Style `hill`
- [ ] **4.4 [server]** Style `intervals`
- [ ] **4.5 [server]** Style `hiit`
- [ ] **4.6 [server]** Style `pyramid`

Planner — conditions and outputs:

- [ ] **4.7 [server]** Condition rule: low sleep / low energy lowers intensity, and writes the reason into `rationale`
- [ ] **4.8 [server]** Condition rule: period day lowers targets
- [ ] **4.9 [server]** Condition rule: injury (`avoid: incline / speed`, severity) caps the plan
- [ ] **4.10 [server]** HR targets per step from the computed zones, capped by `hr_ceiling`
- [ ] **4.11 [server]** Estimates: `est_distance_m` and `est_kcal` (uses weight)
- [ ] **4.12 [server]** Infeasible input returns 422 `PLAN_INFEASIBLE` with a suggestion; all errors use `application/problem+json`

Setup flow:

- [ ] **4.13 [server]** `GET /catalog`: styles, condition questions, quick chips
- [ ] **4.14 [client]** Set up plan page, steps "conditions" (sleep slider, period toggle only when `track_period`, injury picker) and "style" (cards from `/catalog`)

## 5. Run and summary

- [ ] **5.1 [server]** Session state machine (`planning → ready → in_progress → completed`, any → `abandoned`); invalid change returns 409 `INVALID_STATE`
- [ ] **5.2 [server]** `POST /sessions/{id}/start`
- [ ] **5.3 [client]** Run mode page: very large speed and incline, time remaining, countdown to the next change, next step shown. Timer from timestamps, not tick counting
- [ ] **5.4 [client]** Run mode controls: pause, skip, stop (with confirm); record actual segments locally
- [ ] **5.5 [client]** Screen Wake Lock during a run
- [ ] **5.6 [client]** Offline: persist the active session, plan and run state to IndexedDB (TanStack Query persist). A reload mid-run resumes
- [ ] **5.7 [server]** `POST /sessions/{id}/complete`: create the workout and segments from the upload
- [ ] **5.8 [server]** `Idempotency-Key` support on `complete` (same key returns the same response; different body returns 422)
- [ ] **5.9 [client]** Upload on finish with an idempotency key; queue and retry when back online
- [ ] **5.10 [server]** `POST /sessions/{id}/abandon`, called by the client on Stop
- [ ] **5.11 [server]** Comparison logic: planned vs actual in 10 s buckets, `adherence_pct`, skipped steps, time delta. Exposed in `GET /workouts/{id}`
- [ ] **5.12 [server]** `POST` and `GET /workouts/{id}/feedback` (perceived exertion 1–10, notes)
- [ ] **5.13 [client]** Summary page: planned vs actual on one chart, totals, exertion picker and notes

## 6. Adjust the plan

- [ ] **6.1 [server]** `PlanAdjustment` model and `planner.adjust(plan, adjustment)` returning a new version. Hard limits still clamp
- [ ] **6.2 [server]** `POST /sessions/{id}/adjust` for chips `shorter`, `easier`, `no_incline` and structured adjustments. Only in `planning` or `ready`
- [ ] **6.3 [client]** Quick-adjust chips on plan preview; the chart updates to the new version
- [ ] **6.4 [server]** `PATCH /sessions/{id}`: change time, style or conditions and generate a new version
- [ ] **6.5 [server]** `GET /sessions/{id}/plans` (version history)
- [ ] **6.6 [server]** `POST /sessions/{id}/plans/{plan_id}/restore`
- [ ] **6.7 [client]** Version history on plan preview with "undo" (restore)
- [ ] **6.8 [server]** `GET /plans/{id}` with immutable cache headers

## 7. Real home

Replace the mock in `GET /home` one field at a time.

- [ ] **7.1 [server]** `/home`: `greeting_name`, `today` in the user's timezone, `needs_profile` from the database
- [ ] **7.2 [server]** `/home`: `suggestion` computed by the planner from preferences (not saved)
- [ ] **7.3 [server]** `/home`: `active_session` (latest `ready` or `in_progress` session)
- [ ] **7.4 [server]** `/home`: `last_workout` and `week` totals
- [ ] **7.5 [server]** Suggestion uses recent history for style rotation and writes a `reason`
- [ ] **7.6 [server]** `POST /sessions` with `from_suggestion: true`
- [ ] **7.7 [client]** Home wired up: Start and chips create a session from the suggestion; "resume" card for an active session; prompt to fill the profile when `needs_profile`

## 8. History

- [ ] **8.1 [server]** `GET /workouts` with cursor pagination and `from`, `to`, `source` filters
- [ ] **8.2 [client]** History page: list of workouts with infinite scroll
- [ ] **8.3 [client]** Workout detail page (reuses the summary chart)
- [ ] **8.4 [server]** `POST /workouts` (manual entry)
- [ ] **8.5 [server]** `PATCH /workouts/{id}` and `DELETE /workouts/{id}`
- [ ] **8.6 [client]** Manual entry form; edit and delete on the detail page
- [ ] **8.7 [server]** `GET /sessions` (paginated, `?status=`)

## 9. Chat

- [ ] **9.1 [server]** `LlmPort` interface with a fake for tests; text → validated `PlanAdjustment`
- [ ] **9.2 [server]** Gemini Flash adapter behind `LlmPort` (structured output, retries, timeout, key from Secret Manager). Sends the message text only, no profile or history
- [ ] **9.3 [server]** `POST /sessions/{id}/messages`: store both turns, apply the adjustment, return the reply and the new plan
- [ ] **9.4 [server]** `GET /sessions/{id}/messages`
- [ ] **9.5 [client]** Chat bottom sheet (shadcn Drawer) on plan preview: history, composer, plan updates in place
- [ ] **9.6 [server + client]** LLM down returns 503 `ASSISTANT_UNAVAILABLE`; the sheet shows the chips as a fallback

## 10. Import and analysis

- [ ] **10.1 [server]** `WorkoutSourcePort` interface with a fake for tests
- [ ] **10.2 [server]** `POST /workouts/import` with a screenshot: the LLM extracts metrics into a draft; the image is not stored. Idempotent
- [ ] **10.3 [server]** `POST /workouts/import` with a Garmin activity link
- [ ] **10.4 [client]** Import flow: pick a screenshot or paste a link, review the draft, confirm
- [ ] **10.5 [client]** Link an imported workout to a session (`PATCH /workouts/{id}`) to get planned vs actual
- [ ] **10.6 [server]** `POST /workouts/{id}/feedback?analyze=true`: assistant analysis stored in `feedback.analysis`
- [ ] **10.7 [client]** Show the analysis on the summary and workout detail pages

## 11. Garmin sync (Phase 1.5)

- [ ] **11.1 [server]** Spike (half a day): `garth` login and activity fetch against the real account. Write down what works
- [ ] **11.2 [server]** Fernet encryption helper for `integrations.credentials_enc` with `key_version`
- [ ] **11.3 [server]** `GarminAdapter` behind `WorkoutSourcePort`
- [ ] **11.4 [server]** `PUT /integrations/garmin` (email + password, then MFA code). Password never stored
- [ ] **11.5 [server]** `GET /integrations` and `DELETE /integrations/garmin`
- [ ] **11.6 [server]** `POST /integrations/garmin/sync`: idempotent inserts, `needs_reauth` handling, 30 activities / 20 s cap
- [ ] **11.7 [client]** Garmin section on the profile page: connect, MFA step, status, disconnect
- [ ] **11.8 [client]** "Sync now" on the history page
- [ ] **11.9 [server]** `/home`: real `integrations` status
- [ ] **11.10 [server]** Fill `is_period` from Garmin cycle data when available, and skip the question (decision D3). Depends on the 11.1 spike showing that the data is reachable

## 12. Hardening

- [ ] **12.1 [server]** `ETag` and `If-Match` on `PUT /me/profile` and `PUT /me/preferences` (412 on stale)
- [ ] **12.2 [server]** `DELETE /me` (cascade), plus a confirm dialog on the profile page **[client]**
- [ ] **12.3 [server]** Structured JSON logs with `user_id` and `request_id`, never request bodies
- [ ] **12.4 [server]** Purge `idempotency_keys` older than 7 days
- [ ] **12.5 [client]** CI check that fails when the generated API client is out of date
- [ ] **12.6 [client]** One Playwright test for the run-mode flow (start, step change, pause, finish)
- [ ] **12.7 [server]** API tests for tenancy: every resource returns 404 for another user

## 13. Audio guide (Phase 2)

- [ ] **13.1 [server]** Planner fills `plan_steps.cue` for every step
- [ ] **13.2 [client]** Run mode speaks the cue at each change (`speechSynthesis`), with a short warning before it
- [ ] **13.3 [client]** Audio on/off toggle in run mode and in preferences

## 14. Tech tasks

- [x] **14.1 [server]** Migrations run in CI before each deploy (`alembic upgrade head` on the runner against Neon's direct endpoint; a failure keeps the old revision serving)
- [ ] **14.2 [server]** Terraform in CI. Today CI runs `fmt -check` and `validate`, and `apply` is manual. Next: `plan` on push, and `apply` via a manual "Run workflow" button with a separate Terraform service account
- [ ] **14.3 [server + client]** Separate dev and prod: a Neon `dev` branch, a second Cloud Run service or project, and a separate Firebase project or tenant

---

## Not planned (designed for only)

Reminder notifications, real-time HR display, HR-driven guidance, energy budget, Health Connect, treadmill sync, plan sharing. See the spec §1 "Designed for, not built".

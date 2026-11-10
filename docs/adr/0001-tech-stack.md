# ADR-0001: Tech stack for the treadmill workout planner (client + server)

**Status:** Proposed
**Date:** 2026-11-09
**Deciders:** Owner

## Context

Sources: [01 introduction](../requirement/01-introduction.md), [03 frontend requirement](../requirement/03-frontend-requirement.md), `walk-treadmil-client/mockup/index.html`.

| Force | What it means for the stack |
|---|---|
| Budget about $0 | Free hosting tiers, apps that can shut down when idle, no paid services |
| Owner is not strong in frontend | Use the most mainstream framework (most examples and the best AI help) and generate the API client instead of writing it |
| **Must run on the owner's Android phone** | An installable PWA in Chrome. No native build or Play Store. Offline during a run, plus wake lock, plus speech for Phase 2 |
| Garmin Connect import (Phase 1.5) | **No official API for individuals.** Garmin's Developer Program needs business approval. The realistic route is the unofficial Python libraries (`garminconnect` / `garth`) |
| Health Connect | **Out of scope for now.** It's an Android on-device API with no web access, so it would need a native shell. Deferred (see Consequences) |
| Chat, screenshot or link analysis | Needs an LLM. Health data is private, so keep what is sent to it to a minimum |
| Plan generation as its own module | Pure, deterministic code that's easy to unit test |
| More users later | Real auth and a `user_id` on every table from day one |

## Decision

```
┌────────────── Android phone (Chrome) ─────┐
│ Installed PWA: React + TS + Vite          │──── Cloudflare Pages (free, HTTPS)
│ + vite-plugin-pwa (offline run mode)      │
└──────────────┬────────────────────────────┘
               │ REST + OpenAPI (generated TS client)
               │ Firebase ID token (Bearer)
┌──────────────▼────────────────────────────┐
│ Python 3.14 + FastAPI  (Docker)           │──── Google Cloud Run (free tier)
│  api/ → services/ → planner/ (pure rules) │
│  adapters/: garmin, llm                   │
└───────┬──────────────────────┬────────────┘
        │                      │
  Postgres (Neon free)    Gemini API (chat → structured
  Docker Postgres locally   adjustments; screenshot parsing)
```

### Backend

| Layer | Choice |
|---|---|
| Language/framework | **Python 3.14 + FastAPI**, Pydantic v2 |
| ORM / migrations | SQLAlchemy 2.0 (or SQLModel) + Alembic |
| Package manager | `uv` |
| DB | **PostgreSQL**: Docker locally, **Neon** free tier in production |
| API | REST with OpenAPI generated automatically by FastAPI. No gRPC, because browsers need a grpc-web proxy and it buys nothing at this scale |
| Auth | **Firebase Auth** (Google sign-in, free). The backend checks ID tokens with `firebase-admin`. No passwords stored, and it already supports multiple users |
| Plan generation | A `planner/` package with pure functions: `(Profile, ResolvedPreferences, SessionInput) -> Plan`. It has no I/O |
| LLM | Gemini Flash behind an `LlmPort` adapter. It **only** turns free text into a structured `PlanAdjustment` (e.g. `{"intensity": -1, "reason": "slept 4h"}`) and reads screenshots into workout metrics. It never builds the plan itself |
| Integrations | `GarminAdapter` (with `garth`) behind a `WorkoutSourcePort`, so another source can be added later without touching the core |
| Secrets | Environment variables locally, Secret Manager on Cloud Run. Garmin tokens are encrypted at the application level (Fernet) before they are stored |
| Tests | pytest, table-driven tests for the planner (poor sleep, injury, short time), and httpx with a Postgres test container for API integration tests |

### Frontend

| Layer | Choice |
|---|---|
| Framework | **React 19 + TypeScript + Vite** (single-page app, not Next.js because server rendering isn't needed) |
| PWA / offline | `vite-plugin-pwa` (Workbox): a web manifest, a service worker, and install to the home screen. TanStack Query with a persisted cache, so a loaded plan survives going offline |
| UI | Tailwind + **shadcn/ui** (the Drawer component covers the chat bottom sheet). Port the Barlow look from the mockup into a theme |
| Charts | Recharts (plan preview, planned vs actual) |
| Routing | React Router |
| API client | `openapi-typescript` + `openapi-fetch`, generated from the server's `/openapi.json` |
| Run mode | Screen Wake Lock API. Timers based on timestamps, not `setInterval` counting. `speechSynthesis` for the Phase 2 audio guide |
| Tests | Vitest + Testing Library. One Playwright test for the run-mode flow |

### Running on the Android phone

Service workers, wake lock and installing a PWA all need a **secure context** (HTTPS, or `localhost`).

- **Day-to-day use:** open the Cloudflare Pages URL in Chrome on Android, then choose **⋮ → Add to Home screen / Install app**. It opens full screen, and once a plan is loaded it works offline.
- **Development on the phone:** connect over USB with debugging on and run `adb reverse tcp:5173 tcp:5173` (and `tcp:8000` for the API). Then open `http://localhost:5173` on the phone. It counts as a secure context, so the PWA features work. Debug from `chrome://inspect` on the laptop.
- **Not used:** opening `http://<laptop-LAN-IP>:5173` directly. It isn't a secure context, so the service worker and wake lock won't work.

### Deploy and CI

- **Frontend:** Cloudflare Pages, free with unlimited bandwidth, HTTPS, and a preview build for every branch.
- **Backend:** a Docker image on **Google Cloud Run**. It shuts down when idle, with a cold start of about 1 to 3 seconds. It needs a billing account, so set a $1 budget alert.
  - Option without a card: **Render** free tier. The cost is a cold start of 30 seconds or more after 15 minutes idle.
- **DB:** Neon free tier (Postgres in production and locally).
- **Local:** `docker compose up` runs api and postgres.
- **CI:** GitHub Actions. Server: lint, type check, tests, image build. Client: typecheck, tests, build. Plus a check that fails if the generated API client is out of date with `openapi.json`.

## Options Considered

### Backend language

| | **Python + FastAPI** ✅ | Node + NestJS | Java 17 + Spring Boot | Go + gin |
|---|---|---|---|---|
| Garmin libraries | Mature unofficial libraries | Weaker ports | None | None |
| LLM SDKs and parsing | Best | Good | OK | OK |
| Generated OpenAPI | Built in | Built in (decorators) | springdoc | Manual or swag |
| Cold start on free hosts | Fast | Fast | Slow, high memory | Fastest |
| Complexity | Low | Medium | High | Low to medium |

**Why Python:** Garmin access is the hardest integration, and it only really exists in Python. FastAPI also generates the OpenAPI spec.
**Cost:** weaker static typing. Mitigate with pyright or mypy in CI and Pydantic models as the contract.

### Frontend

| | **React + Vite PWA** ✅ | Vue/Svelte PWA | Flutter | Native Kotlin |
|---|---|---|---|---|
| Learning curve for a non-frontend developer | Medium, but the most examples and the best AI help | Lower, but a smaller ecosystem | New language (Dart) | High |
| Runs on Android without a native build | Yes (install as PWA) | Yes | No | No |
| Offline and installable | Yes | Yes | Yes | Yes |

**Why React:** the deepest ecosystem (shadcn, Recharts, TanStack). Health Connect is out of scope for now, so no native build is needed.
**Cost:** more boilerplate than Svelte.

### Database

- **Postgres (Neon)** ✅: one engine everywhere and real support for multiple users.
- **SQLite + Litestream:** simplest locally, but free container hosts have temporary disks.
- **Supabase:** a good bundle (database and auth), but free projects pause after a week of inactivity.

### Auth

- **Firebase Auth** ✅: Google sign-in, free, multi-user. It sees identity only, never health data.
- **Supabase Auth:** about as good. Pick it if Supabase is chosen for the database.
- **Self-built JWT with a password:** adds password storage and resets. Not worth it.

## Trade-off Analysis

1. **Deterministic planner plus an LLM that only translates.** The LLM maps "I only slept 4 hours" to a typed adjustment, and the rules engine applies it within the owner's heart-rate and treadmill limits. This keeps the health logic testable, prevents unsafe plans the LLM might invent, and limits what reaches Gemini: send the request text and screenshot, not the profile or history.
   - ⚠️ On the Gemini **free tier, Google may use inputs to improve its products.** The `LlmPort` adapter allows a switch to the paid tier or a local model through Ollama without other changes.
2. **Unofficial Garmin access** can break, and it uses the Garmin login and session tokens. Keep it behind an adapter, encrypt the tokens, and keep a fallback from day one: a screenshot or link plus manual entry.
3. **PWA instead of a native app:** no build, no Play Store, one codebase. The limit: browsers throttle a PWA's audio and timers once the screen locks, so run mode keeps the screen awake with a wake lock.
4. **Free hosts shut down when idle.** Expect a cold start of about 2 seconds on Cloud Run. The aggregated `/home` endpoint (one call) plus the cached plan offline hides most of it.

## Consequences

- **Easier:** testing planner rules, swapping LLMs, adding users (auth and `user_id` scoping are already in place), shipping updates (a deploy updates the installed PWA).
- **Harder:** keeping types in sync across two repos, which the generated client and the CI check handle. Garmin auth may break when Garmin changes things.
- **Revisit:**
  - **Health Connect, or audio with the screen off:** wrap the same PWA in **Capacitor** to get a native Android shell, then add a Health Connect plugin that pushes data to a new adapter endpoint. No frontend rewrite needed.
  - Gemini privacy terms, before real health context goes into prompts.

## Action Items

1. [ ] Server: scaffold with `uv`, FastAPI, SQLAlchemy and Alembic, `docker-compose.yml`, and the `planner/` package with table-driven tests.
2. [ ] Server: define the entities (User, Profile, Preference, Session, Plan, PlanStep, Workout, Feedback), all scoped by `user_id`.
3. [ ] Server: Firebase token check as a dependency, and `LlmPort` / `WorkoutSourcePort` interfaces with fake implementations for tests.
4. [ ] Client: Vite with the React and TS template, Tailwind, shadcn/ui, vite-plugin-pwa (manifest and icons), and an `npm run gen:api` script.
5. [ ] Client: port the mockup screens, building Home and Run mode first. Check installation and offline run mode on the phone with `adb reverse`.
6. [ ] Deploy: Neon database, a Cloud Run service with a budget alert, a Cloudflare Pages project, and GitHub Actions in both repos.
7. [ ] Spike (half a day): test `garth` login and activity import against the owner's Forerunner 165 account before committing to Phase 1.5.

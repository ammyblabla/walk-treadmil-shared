# 02 — Database schema and REST API

**Status:** Draft for review
**Date:** 2026-11-09
**Builds on:** [01-introduction.md](01-introduction.md), [ADR-0001](../adr/0001-tech-stack.md) (FastAPI, PostgreSQL on Neon, Firebase Auth)

---

## 1. Requirements recap

### Functional, Phase 1 (what the schema and API must support)

1. Store a profile: weight, max HR (HR zones are computed from it), treadmill limits.
2. Store user preferences, and let one session override them.
3. Create a session from time, conditions, injury, goal, and style, which generates a plan of speed and incline over time.
4. Adjust the plan with quick chips (shorter, easier, no incline) or free-text chat, and keep the history of versions.
5. Run the plan offline on the phone, then upload what was actually done.
6. Import workouts from a Garmin link, a screenshot, or a Garmin sync (Phase 1.5).
7. Store feedback and analysis per workout. Compare planned with actual.
8. Return one aggregated home endpoint.

### Designed for, not built

These have tables, columns, or extension points already reserved:

- Audio cues (Phase 2): `plan_steps.cue`.
- Heart-rate guidance and real-time display: `workout_samples`.
- Post-workout summary that feeds back into future plans: `feedback`, plus `workouts.plan_id`.
- Energy budget: `sessions.conditions` and workout history.
- Sharing plans: `plans.visibility`.

### Non-functional

| Concern | Target |
|---|---|
| Load | 1 user now, under 50 later. Fewer than 1k requests a day |
| Latency | `GET /home` under 300 ms when warm. Plan generation under 100 ms, because it's rules only. Chat adjustment under 5 s, because an LLM is involved |
| Availability | Best effort on a free tier. Once a plan is loaded, the client runs the workout fully offline |
| Storage | Neon free tier, 0.5 GB. Phase 1 data is a few MB a year (see §6) |
| Privacy | Health data is scoped by `user_id` on every query. Third-party tokens are encrypted. Screenshots are not kept after processing |

---

## 2. High-level design

```
             PWA (Android)
                  │  Bearer <Firebase ID token>
                  ▼
┌──────────────────────── FastAPI ─────────────────────────┐
│ api/ (routers, schemas)                                  │
│   │                                                      │
│   ▼                                                      │
│ services/                                                │
│   ├── PreferenceResolver  defaults < prefs < session     │
│   ├── PlanService ──► planner/ (pure, rules engine)      │
│   ├── ChatService ──► LlmPort  (text → PlanAdjustment)   │
│   ├── WorkoutService ─► WorkoutSourcePort (Garmin, OCR)  │
│   └── HomeService (aggregation)                          │
│   │                                                      │
│   ▼                                                      │
│ repositories/ (SQLAlchemy) ───────────────► PostgreSQL   │
└──────────────────────────────────────────────────────────┘
```

### Plan generation flow

```
SessionInput ─┐
Preferences ──┼─► PreferenceResolver ─► ResolvedInput ─► planner.generate()
Profile ──────┘                           (hard limits)        │
                                                               ▼
                                              Plan + PlanSteps (version n)
Chip / chat ─► (LLM if text) ─► PlanAdjustment ─► planner.adjust(plan n) ─► version n+1
```

**Precedence, lowest to highest:** system defaults, then user `preferences`, then `sessions.overrides`, then the chat or chip `PlanAdjustment`.

**Hard limits always clamp the result**, whatever the precedence: treadmill max speed and incline, the HR ceiling, and injury rules.

**The LLM never writes plan steps.** It only produces a validated `PlanAdjustment`.

---

## 3. Data model

### 3.1 ER overview

```
users 1─1 profiles
users 1─1 preferences
users 1─* integrations
users 1─* sessions 1─* plans 1─* plan_steps
                 │        ▲
                 │        └── plans.parent_plan_id (version chain)
                 ├─* chat_messages ──(produced)──► plans
                 └─0..1 workouts 1─* workout_segments
                                 1─* workout_samples   (later)
                                 1─* feedback
users 1─* workouts   (imported workouts may have no session)
```

### 3.2 Conventions

- **Primary keys:** `uuid` with the default `gen_random_uuid()`. If Neon runs PG 18+, switch to `uuidv7()` for keys that sort by time.
- **Times:** `timestamptz` (UTC) everywhere. A `created_at` and `updated_at` pair on every mutable table, with `updated_at` set by the ORM.
- **Enums:** `text` plus a `CHECK` constraint instead of PG `ENUM`, so adding a value is a one-line migration.
- **Units are canonical:**
  - speed in **km/h** as `numeric(4,1)`
  - incline in **%** as `numeric(4,1)`
  - durations in **seconds** as `int`
  - distance in **metres** as `int`
  - weight in **kg** as `numeric(5,2)`

  Speed is **km/h only**, both stored and displayed (decision D1), so there's no unit conversion.
- **Tenancy:** every top-level table has `user_id` with `ON DELETE CASCADE`. Every repository query filters by the authenticated `user_id`.
- **Flexible JSONB:** used only for shapes that are versioned and validated by Pydantic: preferences, conditions, overrides, adjustments. Anything queried or used in a join is a real column.

### 3.3 DDL

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;  -- gen_random_uuid() on PG < 13 safety

-- ───────────── identity ─────────────
CREATE TABLE users (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  firebase_uid  text NOT NULL UNIQUE,
  email         text,
  display_name  text,
  timezone      text NOT NULL DEFAULT 'Asia/Bangkok',   -- for "today" on /home
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now()
);

-- Body + equipment facts. Hard limits for the planner.
CREATE TABLE profiles (
  user_id            uuid PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  weight_kg          numeric(5,2) CHECK (weight_kg BETWEEN 20 AND 300),
  birth_year         smallint,
  -- HR zones are COMPUTED, not stored (decision D2) — see §3.5.
  max_hr             smallint CHECK (max_hr BETWEEN 100 AND 230),  -- NULL ⇒ estimate from birth_year
  resting_hr         smallint CHECK (resting_hr BETWEEN 30 AND 120),  -- set ⇒ Karvonen (HR reserve)
  hr_ceiling         smallint,                 -- never plan above this
  treadmill_max_speed_kmh   numeric(4,1) NOT NULL DEFAULT 12.0,
  treadmill_min_speed_kmh   numeric(4,1) NOT NULL DEFAULT 1.0,
  treadmill_max_incline_pct numeric(4,1) NOT NULL DEFAULT 12.0,
  treadmill_speed_step      numeric(3,1) NOT NULL DEFAULT 0.1,  -- round plan to what you can dial
  treadmill_incline_step    numeric(3,1) NOT NULL DEFAULT 1,
  updated_at         timestamptz NOT NULL DEFAULT now()
);

-- Per-user defaults. Shape = Pydantic `PreferenceSet` (schema_version for migrations).
--   { "default_duration_min": 40, "favorite_styles": ["hill","intervals"],
--     "warmup_min": 5, "cooldown_min": 5, "comfortable_speed_kmh": [4.5, 6.5],
--     "max_incline_pct": 10, "track_period": true }   -- track_period ⇒ ask "On your period?" in session setup
CREATE TABLE preferences (
  user_id        uuid PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  schema_version smallint NOT NULL DEFAULT 1,
  data           jsonb NOT NULL DEFAULT '{}'::jsonb,
  updated_at     timestamptz NOT NULL DEFAULT now()
);

-- Third-party connections. Secrets encrypted in-app (Fernet) before insert.
CREATE TABLE integrations (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id          uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider         text NOT NULL CHECK (provider IN ('garmin')),
  status           text NOT NULL DEFAULT 'connected'
                   CHECK (status IN ('connected','needs_reauth','error','disconnected')),
  credentials_enc  bytea NOT NULL,          -- encrypted OAuth/session tokens, never plaintext
  key_version      smallint NOT NULL DEFAULT 1,  -- for key rotation
  last_sync_at     timestamptz,
  last_error       text,
  created_at       timestamptz NOT NULL DEFAULT now(),
  updated_at       timestamptz NOT NULL DEFAULT now(),
  UNIQUE (user_id, provider)
);

-- ───────────── planning ─────────────
-- A Session = one intended treadmill workout ("today, 30 min, slept badly").
CREATE TABLE sessions (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  status          text NOT NULL DEFAULT 'planning'
                  CHECK (status IN ('planning','ready','in_progress','completed','abandoned')),
  available_min   smallint NOT NULL CHECK (available_min BETWEEN 5 AND 240),
  style           text NOT NULL CHECK (style IN
                  ('steady','easy','flat','hill','hiit','intervals','pyramid')),
  goal            text,                     -- free text or tag: 'fat_burn','endurance','recovery'
  -- Today's conditions. Pydantic `SessionConditions`:
  --   { "sleep_hours": 4, "energy": 2, "is_period": true,
  --     "injury": {"area":"left_knee","severity":2,"avoid":["incline"]} }
  -- MVP: is_period is a user-tapped boolean. Later: auto-filled from Garmin cycle data (D3).
  conditions      jsonb NOT NULL DEFAULT '{}'::jsonb,
  -- Overrides of PreferenceSet keys for this session only.
  overrides       jsonb NOT NULL DEFAULT '{}'::jsonb,
  current_plan_id uuid,                     -- FK added below (circular)
  started_at      timestamptz,
  ended_at        timestamptz,
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX sessions_user_created_idx ON sessions (user_id, created_at DESC);

-- Immutable plan versions. Every adjustment = new row; history is free.
CREATE TABLE plans (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id          uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  session_id       uuid REFERENCES sessions(id) ON DELETE CASCADE,  -- NULL = saved template
  parent_plan_id   uuid REFERENCES plans(id) ON DELETE SET NULL,
  version          smallint NOT NULL DEFAULT 1,
  title            text NOT NULL,           -- "30-min rolling hills"
  rationale        text,                    -- why: "eased 15% for 4h sleep"
  total_duration_s int  NOT NULL,
  est_distance_m   int,
  est_kcal         int,
  generator        text NOT NULL,           -- 'rules@1.2.0' — which planner version made it
  adjustment       jsonb,                   -- PlanAdjustment that produced this version
  visibility       text NOT NULL DEFAULT 'private'
                   CHECK (visibility IN ('private','link','public')),   -- Dream: sharing
  created_at       timestamptz NOT NULL DEFAULT now(),
  UNIQUE (session_id, version)
);
CREATE INDEX plans_session_idx ON plans (session_id, version DESC);

ALTER TABLE sessions
  ADD CONSTRAINT sessions_current_plan_fk
  FOREIGN KEY (current_plan_id) REFERENCES plans(id) ON DELETE SET NULL;

CREATE TABLE plan_steps (
  plan_id        uuid NOT NULL REFERENCES plans(id) ON DELETE CASCADE,
  seq            smallint NOT NULL,
  start_offset_s int NOT NULL,              -- denormalized for O(1) "what now?" in run mode
  duration_s     int NOT NULL CHECK (duration_s > 0),
  phase          text NOT NULL CHECK (phase IN ('warmup','work','recover','steady','cooldown')),
  speed_kmh      numeric(4,1) NOT NULL CHECK (speed_kmh > 0),
  incline_pct    numeric(4,1) NOT NULL CHECK (incline_pct >= 0),
  target_hr_min  smallint,
  target_hr_max  smallint,
  cue            text,                      -- Phase 2 audio: "Hill! Incline to 8"
  PRIMARY KEY (plan_id, seq)
);

-- Chat is scoped to a session; each assistant turn may produce a plan version.
CREATE TABLE chat_messages (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id          uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  session_id       uuid NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  role             text NOT NULL CHECK (role IN ('user','assistant','system')),
  content          text NOT NULL,
  adjustment       jsonb,                   -- parsed PlanAdjustment (assistant turns)
  produced_plan_id uuid REFERENCES plans(id) ON DELETE SET NULL,
  created_at       timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX chat_session_idx ON chat_messages (session_id, created_at);

-- ───────────── execution / history ─────────────
CREATE TABLE workouts (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  session_id     uuid UNIQUE REFERENCES sessions(id) ON DELETE SET NULL,
  plan_id        uuid REFERENCES plans(id) ON DELETE SET NULL,  -- the version actually run
  source         text NOT NULL CHECK (source IN ('app','garmin','screenshot','link','manual')),
  external_id    text,                      -- Garmin activityId
  started_at     timestamptz NOT NULL,
  duration_s     int NOT NULL,
  distance_m     int,
  avg_hr         smallint,
  max_hr         smallint,
  kcal           int,
  elevation_gain_m int,
  -- Minimal provider payload we chose to keep (whitelisted fields, not the whole blob)
  source_meta    jsonb,
  created_at     timestamptz NOT NULL DEFAULT now(),
  updated_at     timestamptz NOT NULL DEFAULT now(),
  UNIQUE (user_id, source, external_id)    -- idempotent Garmin sync
);
CREATE INDEX workouts_user_started_idx ON workouts (user_id, started_at DESC);

-- What was actually dialled in (from run mode, or Garmin laps). Drives planned-vs-actual.
CREATE TABLE workout_segments (
  workout_id     uuid NOT NULL REFERENCES workouts(id) ON DELETE CASCADE,
  seq            smallint NOT NULL,
  plan_step_seq  smallint,                  -- which planned step this maps to (NULL = unplanned)
  start_offset_s int NOT NULL,
  duration_s     int NOT NULL,
  speed_kmh      numeric(4,1),
  incline_pct    numeric(4,1),
  avg_hr         smallint,
  event          text CHECK (event IN ('step','pause','skip','extend')),
  PRIMARY KEY (workout_id, seq)
);

-- LATER (HR guidance / real-time): ~1 row/s. One JSONB array per workout keeps row count low.
CREATE TABLE workout_samples (
  workout_id  uuid PRIMARY KEY REFERENCES workouts(id) ON DELETE CASCADE,
  interval_s  smallint NOT NULL DEFAULT 1,
  hr          smallint[],                   -- index = t / interval_s
  speed_kmh   real[],
  incline_pct real[]
);

CREATE TABLE feedback (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  workout_id  uuid NOT NULL REFERENCES workouts(id) ON DELETE CASCADE,
  author      text NOT NULL CHECK (author IN ('user','assistant')),
  perceived_exertion smallint CHECK (perceived_exertion BETWEEN 1 AND 10),  -- RPE: Rate of Perceived Exertion, 1 = very easy, 10 = max effort
  notes       text,
  -- Assistant analysis: {"too_hard_steps":[4,5], "suggest":{"intensity":-1}, "summary":"..."}
  analysis    jsonb,
  created_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX feedback_workout_idx ON feedback (workout_id);

-- ───────────── API support ─────────────
-- Offline client retries POST /complete etc. Same key ⇒ same response.
CREATE TABLE idempotency_keys (
  user_id        uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  key            text NOT NULL,
  request_hash   text NOT NULL,
  response_code  smallint NOT NULL,
  response_body  jsonb NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, key)
);   -- purge rows older than 7 days on startup / daily
```

### 3.4 Why the schema looks like this

| Decision | Why | What it costs |
|---|---|---|
| Plans can't be changed. Each adjustment adds a new row with `parent_plan_id` | Plan history and undo come for free, and chat can explain what changed. The workout also points to the exact version that was run | More rows. That's fine at a few KB per plan |
| `sessions.current_plan_id` stores the latest version | `/home` and run mode find the active plan with one indexed lookup | A circular foreign key, added with `ALTER` |
| Preferences, conditions and overrides are JSONB, validated by Pydantic | The question set will change often during Phase 1, and resolution is a plain dict merge | The database can't check the shape. Mitigation: `schema_version` plus Pydantic on both read and write |
| Profile limits are typed columns | The planner always uses them as hard limits, and their meaning is stable | Adding a field needs a migration |
| `plan_steps.start_offset_s` is stored, not computed | Run mode can find "the current step" directly, and steps are never edited after creation | Stored data could drift from durations. Prevented because plans can't be changed |
| One `workouts` table for every source, with `UNIQUE(user_id, source, external_id)` | Running Garmin sync twice is safe. History reads from one place | Some columns are NULL depending on the source |
| `workout_samples` as arrays, one row per workout | A one-hour walk at one sample per second is about 3,600 values in one row, not 3,600 rows | Can't query individual samples with SQL. They aren't needed |
| Screenshots aren't stored | Privacy, and Neon's 0.5 GB limit. Only the extracted metrics go into `workouts` | You can't re-process an old screenshot |
| `is_period` lives in `sessions.conditions`, not on the profile | It changes daily and is only asked about when `track_period` is true | Cycle trends need a JSONB query. Acceptable |
| HR zones aren't stored | One source of truth (`max_hr` and `resting_hr`). Changing max HR updates every zone automatically | No per-zone manual tweak. Add an override column if one is ever needed |

**Sensitive fields:** period status and injury. Neon already encrypts data at rest. If more protection is needed later, move `conditions` into an encrypted `bytea`, the same way as `integrations.credentials_enc`. The cost is losing SQL queries over it.

### 3.5 Computed HR zones (D2)

This is a pure function in `planner/hr_zones.py`, unit-tested, and never stored.

1. **Max HR:** `profiles.max_hr` if it's set. Otherwise estimate it with the Tanaka formula: `208 − 0.7 × age`, with age from `birth_year`. If neither is set, the profile is incomplete and `/home` returns `needs_profile: true`.
2. **Zone bounds** (5 zones, the same split as Garmin's default):

   | Zone | % | Use in plans |
   |---|---|---|
   | 1 | 50–60 | warm-up, cool-down, recovery |
   | 2 | 60–70 | easy, steady, long walks |
   | 3 | 70–80 | hills and tempo walks |
   | 4 | 80–90 | HIIT and interval work |
   | 5 | 90–100 | not used for walking plans |

   - **If `resting_hr` is set:** use **Karvonen**, `rest + pct × (max − rest)`. It's more personal.
   - **Otherwise:** use `pct × max`.
3. **Ceiling:** `hr_ceiling`, if set, caps every zone. The planner never sets a target above it.
4. **Today's conditions lower the targets:** a period day or poor sleep takes 5 bpm off, or one zone. These rules live in the planner, not in this function.

---

## 4. REST API

### 4.1 Conventions

- **Base path:** `/api/v1`. OpenAPI is served at `/api/v1/openapi.json`. The TypeScript client is generated from it.
- **Auth:** `Authorization: Bearer <Firebase ID token>` on every endpoint except `/health`.
  - The first authenticated call creates the `users` row (just-in-time provisioning).
  - There is no `user_id` in any URL. Everything under the API belongs to the caller.
- **JSON:** `snake_case`. Times use ISO-8601 UTC. Units are canonical (km/h, %, s, m).
- **Errors:** RFC 9457 `application/problem+json`:
  ```json
  { "type": "https://walk.app/errors/plan-limit", "title": "Speed exceeds treadmill max",
    "status": 422, "detail": "Requested 14.0 km/h; max is 12.0", "code": "PLAN_LIMIT" }
  ```
- **Pagination:** cursor based, `?limit=20&cursor=<opaque>`. The response looks like `{ "items": [...], "next_cursor": "..." | null }`.
- **Idempotency:** writes that an offline client may retry (`POST .../complete` and `POST /workouts/import`) accept an `Idempotency-Key: <uuid>` header.
- **Concurrency:** `PUT /me/profile` and `PUT /me/preferences` return an `ETag` and accept `If-Match`. A stale write gets 412. This matters once two devices are in use.

### 4.2 Endpoint list

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Liveness check (no auth) |
| **Me** |||
| GET | `/me` | User, profile, preferences, and integration status in one call |
| PATCH | `/me` | Update `display_name` and `timezone` |
| GET / PUT | `/me/profile` | Body and treadmill limits |
| GET / PUT | `/me/preferences` | `PreferenceSet` |
| DELETE | `/me` | Delete the account and all data (cascades) |
| **Home** |||
| GET | `/home` | Aggregated home screen (§4.3) |
| **Catalog** |||
| GET | `/catalog` | Workout styles, condition questions, and quick-adjust chips, so the UI is driven by the server |
| **Sessions and plans** |||
| POST | `/sessions` | Create a session from inputs. Generates plan v1 and returns the session and the plan |
| GET | `/sessions` | List sessions (paginated, `?status=`) |
| GET | `/sessions/{id}` | Session plus its current plan with steps |
| PATCH | `/sessions/{id}` | Change inputs (time, style, conditions). Generates a new plan version |
| POST | `/sessions/{id}/adjust` | A quick chip `{ "chip": "shorter" }` or a structured adjustment. No LLM involved, so it's deterministic and fast |
| GET | `/sessions/{id}/plans` | Version history |
| POST | `/sessions/{id}/plans/{plan_id}/restore` | Make an older version current (adds a new version that copies it) |
| GET | `/sessions/{id}/messages` | Chat history |
| POST | `/sessions/{id}/messages` | Free text. The LLM turns it into a `PlanAdjustment` and the planner applies it. Returns the reply and maybe a new plan |
| POST | `/sessions/{id}/start` | Status becomes `in_progress`. Freezes `current_plan_id` for this run |
| POST | `/sessions/{id}/complete` | Upload the actual segments recorded offline. Creates the workout. **Idempotent** |
| POST | `/sessions/{id}/abandon` | Status becomes `abandoned` |
| GET | `/plans/{id}` | A plan with its steps (also used for sharing later) |
| **Workouts** |||
| GET | `/workouts` | History (paginated, `?from=&to=&source=`) |
| GET | `/workouts/{id}` | Detail, including segments and the planned-vs-actual comparison |
| POST | `/workouts` | Manual entry |
| POST | `/workouts/import` | `multipart` screenshot **or** `{ "url": "https://connect.garmin.com/..." }`. Returns the extracted draft for confirmation. **Idempotent** |
| PATCH | `/workouts/{id}` | Fix extracted values, or link the workout to a session |
| DELETE | `/workouts/{id}` | |
| GET | `/workouts/{id}/feedback` | |
| POST | `/workouts/{id}/feedback` | The user's perceived exertion (1–10) and notes. `?analyze=true` also asks the assistant for analysis |
| **Integrations (Phase 1.5)** |||
| GET | `/integrations` | Status of each provider. Credentials are never returned |
| PUT | `/integrations/garmin` | Connect `{ email, password }`, or `{ mfa_code }` for step two. Tokens are stored encrypted and the password is thrown away |
| POST | `/integrations/garmin/sync` | Pull recent activities, either `?since=` or from the last sync. Inserts are idempotent |
| DELETE | `/integrations/garmin` | Disconnect and delete the tokens |

### 4.3 Key payloads

**`GET /home`** is the only call the home screen makes:

```json
{
  "greeting_name": "Am",
  "today": "2026-11-09",
  "active_session": null,
  "suggestion": {
    "style": "hill", "available_min": 30,
    "plan": { "title": "30-min rolling hills", "total_duration_s": 1800,
              "est_distance_m": 2900, "steps": [ /* PlanStep[] */ ] },
    "reason": "Last 2 sessions were flat; you prefer hills on Mondays."
  },
  "quick_chips": ["shorter", "easier", "no_incline"],
  "last_workout": { "id": "…", "started_at": "…", "duration_s": 2400, "distance_m": 3800,
                    "plan_adherence_pct": 92 },
  "week": { "workouts": 3, "duration_s": 6300, "distance_m": 10200 },
  "integrations": { "garmin": "connected" },
  "needs_profile": false
}
```

The suggestion is **computed, not saved**. Pressing Start or a chip calls `POST /sessions` with `{ "from_suggestion": true, ... }`, and that persists it.

**`POST /sessions`**

```json
// request
{
  "available_min": 30,
  "style": "intervals",
  "goal": "endurance",
  "conditions": { "sleep_hours": 4, "is_period": true,
                  "injury": { "area": "left_knee", "severity": 2, "avoid": ["incline"] } },
  "overrides": { "max_incline_pct": 3 }
}
// 201 response
{
  "session": { "id": "s_…", "status": "ready", "current_plan_id": "p_…", ... },
  "plan": {
    "id": "p_…", "version": 1, "title": "30-min easy intervals",
    "rationale": "Intensity -1 for 4h sleep; incline capped at 3% for knee.",
    "total_duration_s": 1800, "generator": "rules@1.0.0",
    "steps": [
      { "seq": 1, "start_offset_s": 0,   "duration_s": 300, "phase": "warmup",
        "speed_kmh": 4.5, "incline_pct": 1.0, "target_hr_min": 100, "target_hr_max": 120,
        "cue": "Warm up, 4.5 at 1%" },
      { "seq": 2, "start_offset_s": 300, "duration_s": 120, "phase": "work",
        "speed_kmh": 6.2, "incline_pct": 2.0, ... }
    ]
  },
  "resolved_input": { /* the merged prefs+overrides the planner actually used — for transparency */ }
}
```

**`POST /sessions/{id}/messages`**

```json
// request
{ "text": "I don't like this, too many changes. Fewer, longer intervals." }
// 200 response
{
  "user_message":      { "id": "…", "role": "user", "content": "…" },
  "assistant_message": { "id": "…", "role": "assistant",
                         "content": "Merged into 3 longer intervals of 4 min.",
                         "adjustment": { "interval_count": 3, "interval_len_s": 240 } },
  "plan": { /* new version, or null if nothing changed */ }
}
```

`PlanAdjustment` is a closed Pydantic model and is the LLM's structured output schema. Any field outside it is rejected. Fields:

- `duration_delta_min`
- `intensity` (−2…+2)
- `max_incline_pct`
- `max_speed_kmh`
- `style`
- `interval_count`
- `interval_len_s`
- `no_incline`
- `note`

**`POST /sessions/{id}/complete`** (from run mode, possibly sent long after the run while offline)

```json
// headers: Idempotency-Key: 6f0c…
{
  "plan_id": "p_…",
  "started_at": "2026-11-09T11:02:10Z",
  "ended_at":   "2026-11-09T11:33:40Z",
  "segments": [
    { "seq": 1, "plan_step_seq": 1, "start_offset_s": 0, "duration_s": 300,
      "speed_kmh": 4.5, "incline_pct": 1.0, "event": "step" },
    { "seq": 2, "plan_step_seq": null, "start_offset_s": 300, "duration_s": 60, "event": "pause" }
  ],
  "perceived_exertion": 6
}
// 201 → { "workout": {...}, "comparison": {...} }
```

**Planned vs actual** is in `GET /workouts/{id}` under `comparison`:

```json
{
  "plan_id": "p_…",
  "series": [ { "t": 0, "planned_speed": 4.5, "actual_speed": 4.5,
                "planned_incline": 1.0, "actual_incline": 1.0 }, ... ],   // 10 s buckets
  "adherence_pct": 92,
  "skipped_steps": [5],
  "time_delta_s": 90
}
```

### 4.4 Status codes

| Case | Code |
|---|---|
| Created a session, workout, or feedback | 201 |
| Validation failed (Pydantic) | 422 |
| The planner can't satisfy the constraints (e.g. 5 min of HIIT with a knee injury) | 422 with `code: PLAN_INFEASIBLE` and a suggestion |
| Invalid state change (`complete` on an `abandoned` session) | 409 `INVALID_STATE` |
| Idempotency key reused with a different body | 422 `IDEMPOTENCY_MISMATCH` |
| Stale `If-Match` | 412 |
| Resource missing or belongs to another user | **404**, so other users' IDs aren't revealed |
| LLM unavailable or timed out | 503 `ASSISTANT_UNAVAILABLE`. The client falls back to the chips |
| Garmin auth expired | 409 `INTEGRATION_NEEDS_REAUTH`, and the integration status is updated |
| Garmin rate-limited | 503 with `Retry-After` |

---

## 5. Cross-cutting concerns

**Caching**
- The server doesn't cache anything, because there's one user and the data is tiny.
- The client caches with TanStack Query. The active session and plan are persisted to IndexedDB, so run mode works offline.
- `GET /plans/{id}` sends `Cache-Control: private, max-age=31536000, immutable`, which is safe because plans can't be changed.

**Background work**
- There is no queue in Phase 1.
- Garmin sync runs inside the request, limited to about 30 activities with a 20 s timeout.
- If sync gets slow, move it to a Cloud Scheduler job that calls `POST /internal/sync` (free tier: 3 jobs).

**Retries**
- **Garmin and LLM adapters:** up to 3 attempts with exponential backoff and jitter, and only for 5xx or timeouts.
- **Client:** retries writes only when they carry an `Idempotency-Key`.

**Run-state machine** (enforced in `SessionService`):

```
planning ─► ready ─► in_progress ─► completed
   │          │           │
   └──────────┴───────────┴──► abandoned
```

Plans can only be adjusted in the `planning` and `ready` states.

**Monitoring:** basic and free.
- Cloud Run's request logs and metrics.
- Structured JSON logs carrying `user_id` and `request_id`, but never request bodies, because they contain health data.
- `/health`.
- Optionally Sentry's free tier, with PII scrubbing.

**Testing hooks**
- `planner/` is pure, so tests are table-driven: poor sleep, injury, short time, treadmill limits, and rounding to the treadmill's increments.
- `PreferenceResolver` gets precedence tests.
- API tests run against a Postgres test container, with fake implementations of `LlmPort` and `WorkoutSourcePort`.

---

## 6. Load and storage estimate

| Item | Per workout | Per user per year (5 workouts a week) |
|---|---|---|
| session + 3 plan versions × ~12 steps | ~6 KB | ~1.5 MB |
| chat messages (~6) | ~3 KB | ~0.8 MB |
| workout + segments | ~2 KB | ~0.5 MB |
| samples (later, 1 Hz × 40 min) | ~25 KB | ~6.5 MB |
| **Total** | | **~10 MB** |

That's about 50 user-years before Neon's 0.5 GB limit. Requests stay under 100 a day per user. There's no scaling work to do. Scale up the single Cloud Run container before considering anything else.

---

## 7. Trade-offs and what to revisit

| Choice now | Revisit when |
|---|---|
| JSONB for preferences and conditions | The question set stabilizes, or analytics needs real columns. Then promote the hot keys to columns |
| Suggestion on `/home` computed on the fly | Generation needs history or LLM input and gets slow. Then cache it once a day in a `daily_suggestions` table |
| Garmin sync inside the request | It takes more than about 10 s, or you want automatic sync. Then use Cloud Scheduler and an internal endpoint |
| `workout_samples` as arrays | You need per-sample queries or the live HR display. Then consider a narrow table or TimescaleDB |
| Visibility flag on `plans` only | Sharing (Dream). Then add a `plan_shares(plan_id, token, expires_at)` table and a public `GET /shared/{token}` route |
| Account deletion cascades and removes data permanently | You need an audit or export. Then add `GET /me/export` before `DELETE /me` |
| Firebase UID as the external identity | A second provider is added. Then split it into a `user_identities(provider, subject)` table |

---

## 8. Decisions (answers to review questions, 2026-11-09)

| # | Topic | Decision | Effect on the design |
|---|---|---|---|
| D1 | Speed units | **km/h only** | Dropped `profiles.speed_unit`. No mph conversion anywhere |
| D2 | HR zones | **Computed** | Dropped `profiles.hr_zones`. Zones come from `max_hr` and `resting_hr` (§3.5) and are returned read-only in `GET /me/profile` as `hr_zones` |
| D3 | Cycle | **MVP: boolean `is_period`** in `sessions.conditions`, asked when `preferences.track_period` is true. **Ideal: import from Garmin** | The planner only needs a yes or no on whether it's a period day. When Garmin cycle import works, the `GarminAdapter` fills `is_period` in automatically during session setup, and the question is skipped. Needs a spike first: the unofficial Garmin API may or may not expose women's-health data |
| D4 | Garmin login | **Log in with the Garmin account** (email, password, MFA if enabled) | Keep `PUT /integrations/garmin` as designed. The password is used once to get tokens and **never stored**. Only the encrypted tokens are kept (`credentials_enc`). Expired tokens cause `needs_reauth` and the user logs in again |
| D5 | Chat scope | **Only inside a session** | Keep `chat_messages.session_id NOT NULL`. No general chat endpoint |
| D6 | Treadmill increments and limits | **Start with the defaults; edit them in the app** | Defaults: speed step 0.1 km/h, incline step 1%, speed 1.0–12.0 km/h, incline 0–12%. They can be changed through `PUT /me/profile`, and the planner rounds every step to these increments |

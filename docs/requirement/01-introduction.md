# treadmill workout planner
I want to build a treadmill workout planner. It will use my Garmin Connect / Health Connect history and my goals to recommend speed and incline, and it will be operated through a chat interface. Please help me design it and then implement it in phases.

## Product overview
A personal treadmill planning assistant. I tell it how long I have, how I feel, and what style I want. It recommends a workout plan (speed and incline over time), and I can adjust the plan in chat. The treadmill has no Bluetooth or Wi-Fi, so I set speed and incline manually, and the system only needs to tell me what to set. 

Originally I prompt to Gemini then it responses me the plan. After I workout I send a garmin activity link to gemini for feedback

## Users and roles
- Owner (me): primary user, sets profile and preferences
- Future: other users can use the system too (Dream phase), so the design should not hard-code a single user

## Constraints
- Budget: as close to zero as possible, using free tools and services only
- Ease of use: minimal manual process. Chat is the main interface for customizing plans.
- Developer skill: I can write code, but I am not strong in frontend
- Mobile: Android phone
- Watch: Garmin Forerunner 165
- Treadmill: no connectivity, all settings are manual
- Data sources: Garmin Connect (Phase 1.5) and Health Connect on Android

## Phase 1 scope (must have)
- Ask for the session time and conditions, then recommend speed and incline. Conditions include menstrual cycle, injury, and sleep.
- Ask only the questions that are needed before a workout. Some answers are per user (stored as preferences) and some are per session (overrides for today).
- User profile: body weight, maximum heart rate or the heart-rate zones I can tolerate, and the treadmill's maximum speed and incline (Flexible)
- Session inputs: available time, injury or pain, and today's goal
- Workout styles I can choose from: one long steady session, easy/relaxed, no incline, hill-style, HIIT-style, intervals, and pyramid
- Customize a plan in chat, for example "I don't like this plan" or "I only slept 4 hours"
- Show the plan as a clear step-by-step list of speed and incline by time, so I can set it manually
- Accept a screenshot or a link from a workout result and use it as input for analysis
- Preference resolution: session preferences override user preferences when they exist
- Connect to Garmin Connect to import workout history

## Phase 2
- Audio guide that tells me when to change speed or incline

## Later phases (design for these, do not build yet)
- Reminder notifications for when I want to walk
- Real-time workout display (distance, heart rate, time)
- Post-workout summary that compares the completed workout with the plan and feeds back into future plans
- Heart-rate-driven guidance that adjusts instructions when heart rate goes above the target zone
- Daily energy budget that estimates recovery from sleep and recent activity and recommends a plan

## Dreams
- Start the workout in sync with the treadmill
- Share plans with other people

## Backend requirements
- Language/framework: [choose a free option and explain the trade-off, e.g. Java 17 + Spring Boot, or Node.js + NestJS, Go + gin or Python + FastAPI]
- Database: [free option, e.g. PostgreSQL or SQLite for Phase 1]
- Authentication: simple and free, enough for one user now and multiple users later
- API style: REST, documented with OpenAPI (gRPC better if fronted support)
- Core entities: User, Profile, Preference, Session, Plan, PlanStep, Workout, Feedback
- Plan generation should be a separate module, so the rules can change without touching the API
- Integrations (Garmin, Health Connect) should sit behind adapters so they can be added in Phase 1.5 without rewriting the core

## Frontend requirements
- Accessible from an Android phone (a mobile web app / PWA is acceptable if it avoids a native build)
- Simple screens: current plan (large, easy to read while walking), profile and preferences
- Plan view must work with one hand and show the next change clearly
- Keep the frontend simple and framework-standard, since frontend is not my strength

## Non-functional requirements
- Single-user load for now, designed so it can scale to a small number of users later
- Security: secrets not in code, data stored securely, third-party tokens protected
- Deployment: free hosting or local-first, with Docker for local runs
- Testing: unit tests for plan generation rules, integration tests for the API, and a few test cases for typical conditions (poor sleep, injury, short time)
- Health data is personal. Keep it private and do not send it to unnecessary third parties.

## Working process
1. First, write the requirements spec and architecture: components, data model, API boundaries, plan-generation approach, and the integration plan for Garmin and Health Connect. Recommend a free tech stack and explain the trade-offs. Do not write code yet.
2. Note any Garmin or Health Connect API access constraints (approval, free tier limits, Android-only features) that affect the design.
3. Stop and wait for my review before continuing.
4. After approval, build the backend skeleton, data model, plan-generation module, and API, with tests.
5. Then build the chat frontend and the Android-friendly plan view.
6. Keep each step small and explain any decision that involves a trade-off.

Start with step 1.
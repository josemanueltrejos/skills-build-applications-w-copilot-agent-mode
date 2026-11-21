# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

- Presentation tier decision: Implement React route components in `octofit-tracker/frontend/src/components` and make every API call through `src/api/client.ts`. Keep presentation responsible for loading, empty, success, and accessible error states while leaving persistence and business rules to the backend.
- Logic tier decision: Use explicit Express routes for `/api/*` and delegate validation and behavior to functions in `octofit-tracker/backend/src/services/octofitService.ts`. Return deliberate 4xx responses for bad IDs and unknown resources, and keep HTTP error handling in the routes.
- Data tier decision: Define required, typed Mongoose fields in the models. Store `User.team`, `Team.members`, and activity references as ObjectIds; keep seed fixtures stable and idempotent with upserts and no collection-wide deletion.
- Activity tracking approach: Validate the submitted IDs with `mongoose.Types.ObjectId.isValid`, confirm both referenced records exist, and reject missing or malformed references with 4xx. Create a lean activity record from the request, with positive points and `occurredAt` computed in the service.
- Team management approach: Require a non-empty `name` and `mascot` in both the Team schema and team creation service. Preserve submitted members and expose team member ObjectIds through both create and list responses.
- Leaderboard approach: Aggregate activity `points` by team, sum points for each team, sort descending, and assign a 1-based rank in the returned `{ teamName, totalPoints, rank }` entries.
- Workout suggestions approach: Resolve the decoded email to a seeded or created User, derive the user's recent activity pattern or fitness goal, and select matching workouts. Return a 404-style 4xx response when the email has no matching user and include a reason based on the matching goal or activity.

## Test plan

- Install dependencies:
  - `npm install --prefix octofit-tracker/backend`
  - `npm install --prefix octofit-tracker/frontend`
  - There is no root `package.json`, so run install commands from this repository only with `--prefix`.
- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`
  - Use this first to verify Mongoose required fields and idempotent seeding. A validation assertion failure means a model is missing required schema rules; a seeding failure means fixtures or team membership reconciliation is incorrect.
- API command: `npm run test:api --prefix octofit-tracker/backend`
  - Run after the data models pass. A 501 response means activity or team behavior is still pending; malformed/missing references must return 4xx rather than cast-error 500s. Leaderboard and workout failures indicate incorrect service aggregation or matching.
- Frontend command: `npm test --prefix octofit-tracker/frontend`
  - Run after the API contract passes. Failures showing `Implementation pending` mean a capability has not fetched or rendered data; missing status/alert roles mean loading or error states are incomplete.
- Backend build command: `npm run build --prefix octofit-tracker/backend`
  - Run after API implementation to catch Express, Mongoose, and TypeScript type errors.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`
  - Run after presentation implementation to verify React, Router, and Vite composition.
- Smallest implementation order:
  1. Complete Mongoose validation and seed behavior.
  2. Implement teams and activities in routes/services.
  3. Implement leaderboard aggregation and workout matching.
  4. Implement the shared presentation states and capability components.
  5. Re-run data tests, API tests, frontend tests, and both builds.

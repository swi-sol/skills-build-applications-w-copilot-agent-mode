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

- Presentation tier: Keep data access in the shared API client and rendering in the existing React components. Use relative `/api` paths and expose accessible loading, empty, and error states on each capability page.
- Logic tier: Keep Express routes thin and put validation, persistence, aggregation, and recommendation logic in `octofitService`. Return deliberate 4xx responses for invalid input or missing resources; do not let malformed IDs become 500 errors.
- Data tier: Enforce required fields, references, and sensible numeric constraints in the Mongoose models. Keep seeding idempotent with upserts, and preserve records that are not part of the demo fixtures.
- Activity tracking: Validate the activity type, positive minutes, and well-formed IDs that reference existing users and teams. Store those references with the submitted type and minutes, award one point per activity minute, and set `occurredAt` when the activity is created.
- Team management: Require a non-empty name and mascot, default members to an empty array, and validate supplied member IDs against existing users. Preserve team names, mascots, and members in create and list responses.
- Leaderboard: Sum activity points by team, order teams by total points descending (using team name as a stable tie-breaker), then assign 1-based ranks.
- Workout suggestions: Find users by email, prefer a workout matching their fitness goal, and use recent activity as recommendation context or a fallback focus. Include a human-readable reason for the first suggestion and return a 4xx response for an unknown email.

## Test plan

- Data tier: `npm run test:data --prefix octofit-tracker/backend` checks model validation/defaults and repeatable, non-destructive seeding. A failure points to schema or seed behavior.
- API logic: `npm run test:api --prefix octofit-tracker/backend` checks request validation, persistence/listing, leaderboard aggregation, and workout recommendations. A failure points to an endpoint or service contract.
- Presentation: `npm test --prefix octofit-tracker/frontend` checks API-driven rendering and accessible loading, empty, and error states. A failure points to frontend data handling or UI behavior.
- Backend build: `npm run build --prefix octofit-tracker/backend` type-checks backend code; a failure indicates a TypeScript/build issue.
- Frontend build: `npm run build --prefix octofit-tracker/frontend` type-checks and bundles the frontend; a failure indicates a TypeScript or Vite build issue.
- Baseline before implementation: data tests had 4 failures and 1 pass; API tests had 4 failures; frontend tests had 15 failures. Rerun the affected suite after each tier change, then run both builds.

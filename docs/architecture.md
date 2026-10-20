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

Record your plan here before editing code.

- Presentation tier decision: Keep routing and presentation in the existing React components. Use the shared API client for relative `/api` requests and show accessible loading, error, and empty states alongside returned data.
- Logic tier decision: Keep request handling in Express routes and capability behavior in `octofitService.ts`. Validate input and references before writes, return deliberate 4xx errors for invalid or missing resources, and leave persistence details to Mongoose.
- Data tier decision: Define required fields, references, and defaults in the four Mongoose models. Keep the seed representative, repeatable, and non-destructive so it preserves unrelated records and existing user/team relationships.
- Activity tracking approach: Validate `userId`, `teamId`, activity type, and positive minutes; confirm referenced user and team exist; save their IDs as `user` and `team` with computed points and an `occurredAt` timestamp. List persisted activities through the same service.
- Team management approach: Require a team name, retain the submitted name and mascot, and support member IDs using the model's User references. Reject invalid member data with a deliberate 4xx response.
- Leaderboard approach: Aggregate activity points by team, include teams in descending total-points order, and assign 1-based ranks in the service response.
- Workout suggestions approach: Find the user by email, use their fitness goal and recent activity to select matching workout records, and include a concise reason for the leading suggestion. Return a deliberate 4xx response when no user matches.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` - Baseline: 4 failed, 1 passed. The four model validation checks fail because required fields are not enforced; the idempotent seed test passes.
- API command: `npm run test:api --prefix octofit-tracker/backend` - Baseline: 4 failed. Activity/team creation returns `501` instead of validating requests, the leaderboard is empty, and workout suggestions omit the expected reason.
- Frontend command: `npm test --prefix octofit-tracker/frontend` - Baseline: 15 failed. Components still report implementation pending, so data rendering, the user-specific workout request, and loading, empty, and accessible error states are missing.
- Backend build command: `npm run build --prefix octofit-tracker/backend` - Not run for the collected baseline; run after backend changes to check TypeScript compilation.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` - Not run for the collected baseline; run after frontend changes to check TypeScript compilation and the Vite build.

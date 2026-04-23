# sensai-frontend

Next.js 15 (App Router) frontend for the sensai LMS. Pairs with the Python `sensai-backend`.

## Tech stack

- Next.js 15.2.3 + React 19 + TypeScript 5
- Dev server uses **Turbopack** (`next dev --turbopack`)
- Node pinned to **22.12** (Dockerfile)
- Tailwind CSS **v4** (`@tailwindcss/postcss`), dark mode via class, theme colors as HSL CSS vars in `src/index.css`
- UI primitives: Radix UI, Headless UI, custom shadcn-style components
- Auth: `next-auth` 4 with Google provider
- Rich content: BlockNote (`@blocknote/core|mantine|react`), Monaco, `react-pdf`, `react-markdown`, `@udus/notion-renderer`, `@notionhq/client`
- Error tracking: `@sentry/nextjs`
- Tests: Jest 29 + `@testing-library/react` 16, `jsdom`

No Redux / Zustand / React Query / SWR — state is React Context + local hooks + raw `fetch`.

## Running locally

```bash
cp .env.example .env.local   # fill in values
npm ci
npm run dev                  # http://localhost:3000
```

Backend (`sensai-backend`) must be running and reachable at `BACKEND_URL` / `NEXT_PUBLIC_BACKEND_URL` (default `http://localhost:8001`).

Scripts: `dev`, `build`, `start`, `lint`, `test`, `test:watch`, `test:coverage`, `test:ci`.

## Layout

```
src/
  app/                     # App Router pages + API routes
    layout.tsx             # Root; wraps app in SessionProvider + IntegrationProvider
    page.tsx               # Dashboard / course list
    login/                 # OAuth login screen
    school/                # Learner + admin routes, nested [id]/[cohortId]/[courseId]
    api/
      auth/[...nextauth]/  # NextAuth route + JWT/session callbacks
      code/                # Judge0 proxy (submit + status)
      integrations/        # Notion OAuth callback + page/block fetchers
    global-error.tsx       # Sentry-reporting error boundary
  middleware.ts            # Auth gate — redirects unauthenticated users to /login
  components/              # ~63 components (editors, dialogs, views, integrations)
  context/
    IntegrationContext.tsx # Notion integration state (token, pages, OAuth flow)
    EditorContext.tsx
  lib/
    api.ts                 # Client hooks: useCourses, useSchools, getCourseModules, getCompletionData
    server-api.ts          # Server-only fetch (uses BACKEND_URL, not NEXT_PUBLIC_)
    auth.ts                # useAuth() hook on top of next-auth's useSession
    course.ts              # Milestone → module transform
    utils.ts               # cn() = clsx + tailwind-merge
    utils/                 # localStorage, indexedDB, blockUtils, dateFormat, urlUtils, audioUtils, scorecardValidation
  providers/SessionProvider.tsx
  types/                   # TS interfaces (course, quiz, next-auth augmentation)
  instrumentation.ts       # Sentry server init
  instrumentation-client.ts# Sentry client init
  __tests__/               # Jest tests, mirrors src/
```

## Routing

- File-based via App Router. Dynamic segments with `[param]`.
- `src/middleware.ts` protects everything except `/login`, `/api/auth/*`, and static assets; unauthenticated requests redirect to `/login` with a callback param.
- Pages are Server Components unless they have `"use client"` at the top.

## Data fetching / API client

- Plain `fetch` — no client library or wrapper.
- Client hooks live in `src/lib/api.ts`. They do not cache: every mount refetches. Caller owns loading + error UI.
- Server-side fetching uses `src/lib/server-api.ts` with `BACKEND_URL`.
- Client-side fetching uses `NEXT_PUBLIC_BACKEND_URL`. Don't confuse the two.

## Auth flow

- `src/app/api/auth/[...nextauth]/route.ts` — Google provider + JWT/session callbacks.
- On first Google sign-in, the route calls backend `/auth/login` to upsert the user and stores the backend user id on `token.userId`; it's surfaced on `session.user.id` via the session callback.
- `useAuth()` in `src/lib/auth.ts` wraps `useSession()` and exposes `{ isAuthenticated, isLoading, user }`. It also mutates a module-level `globalAuthState` singleton to log-once per user id.

## Styling

- Tailwind v4 via PostCSS plugin (`postcss.config.mjs`).
- Directives + theme vars in `src/index.css`.
- `cn()` in `src/lib/utils.ts` for class merging.
- Plugins: `tailwindcss-animate`, `tailwind-scrollbar`. Custom keyframes (accordion, gradient-fade, celebration-reveal, etc.) in `tailwind.config.js`.

## External integrations

| Integration | Where | Env vars |
|---|---|---|
| Google OAuth | `src/app/api/auth/[...nextauth]/route.ts` | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `NEXTAUTH_URL`, `NEXTAUTH_SECRET` |
| Backend API | `src/lib/api.ts`, `src/lib/server-api.ts` | `BACKEND_URL` (server), `NEXT_PUBLIC_BACKEND_URL` (client) |
| Judge0 (code exec) | `src/app/api/code/*` proxy | `JUDGE0_API_URL` |
| Notion | `src/app/api/integrations/*`, `src/context/IntegrationContext.tsx` | `NOTION_CLIENT_ID`, `NOTION_CLIENT_SECRET`, `NEXT_PUBLIC_NOTION_CLIENT_ID` |
| Sentry | `src/instrumentation*.ts`, `next.config.ts`, `src/app/global-error.tsx` | `NEXT_PUBLIC_SENTRY_DSN`, `NEXT_PUBLIC_SENTRY_ENVIRONMENT`, `SENTRY_AUTH_TOKEN` |

Also referenced: `NEXT_PUBLIC_APP_URL`.

## Deployment

- `Dockerfile` — multi-stage, Node 22.12 Alpine, env vars wired as build ARGs then ENV, `npm ci --include=dev`, `npm run build`, `npm start` on port 3000.
- `docker-compose.yml` — single service, maps `$PORT` → 3000.
- CI (`.github/workflows/`):
  - `codecov.yml` — Jest + coverage upload on push/PR touching `src/`.
  - `deploy-staging.yml` / `deploy-only-staging.yml` / `deploy-only-prod.yml` — build image, push to Docker Hub, SSH-deploy via docker-compose.

## Tests

```bash
npm run test              # once
npm run test:watch
npm run test:coverage
npm run test:ci
```

- `jest.config.js`: `jsdom` env, `ts-jest` with `tsconfig.jest.json`, extensive `transformIgnorePatterns` for ESM deps (react-markdown, remark-*, @blocknote, etc.).
- Heavy modules are mocked: `react-pdf`, `react-markdown`, `@blocknote/*`, `react-datepicker`, `canvas-confetti` (see `moduleNameMapper` in `jest.config.js:31-41`). Editor tests do not exercise editor internals.
- `jest.setup.js` mocks `next/navigation`, `next/image`, `next-auth/react`; sets `NEXT_PUBLIC_APP_URL`; polyfills `TextEncoder`/`TextDecoder`.

## Known-fragile / non-obvious

- **No client-side caching or dedup.** `useCourses` / `useSchools` refetch on every mount. Re-renders that change the deps (`user?.id`, `isAuthenticated`, `authLoading`) re-trigger the fetch.
- **`BACKEND_URL` vs `NEXT_PUBLIC_BACKEND_URL`** — server-only code must use `BACKEND_URL`; anything that runs in the browser must use the `NEXT_PUBLIC_` one. Mixing them silently breaks in one environment.
- **Global auth singleton** (`globalAuthState` in `src/lib/auth.ts`) persists across re-mounts for the lifetime of the page. Log-once behavior is intentional but can mask state changes when debugging.
- **JWT callback swallows errors.** If the backend `/auth/login` call fails, the NextAuth JWT callback logs to console and returns the token anyway (`src/app/api/auth/[...nextauth]/route.ts` ~L35). Sessions may land without a `userId`.
- **`/api/code/*` proxy has no auth or rate limiting** — it forwards request bodies straight to Judge0.
- **Notion OAuth `state` param is used as a post-callback redirect target.** No CSRF token layered on top.
- **BlockNote bundle is large** and imports CSS directly in components. Be mindful when adding more editor surface area.
- **Tailwind v4**, not v3 — config and plugin APIs differ. Don't copy v3 snippets from the internet blindly.
- **Turbopack in dev.** Most things work, but some webpack-specific plugins / loaders won't. If a dep misbehaves in dev only, try `next dev` without `--turbopack` to isolate.

## What to check before a change

- **`"use client"` or not?** Pages and components default to Server Components. Adding hooks/state/event handlers requires `"use client"` at the top.
- **Calling the backend?** Use `NEXT_PUBLIC_BACKEND_URL` from client code and `BACKEND_URL` from server code (`src/lib/server-api.ts`). Never import `server-api.ts` into a client component.
- **New page or route?** Check `src/middleware.ts` — everything is auth-gated except `/login`, `/api/auth/*`, and static assets. If a new route must be public, add it to the matcher.
- **New data hook?** Remember hooks in `src/lib/api.ts` don't cache or dedup. Keep the dep array tight (`[user?.id, isAuthenticated, authLoading]`-style) and let the caller own loading/error UI.
- **Touching auth?** The backend user id is written to `token.userId` in the NextAuth JWT callback. If the `/auth/login` call fails, the callback swallows the error — sessions can land without `userId`. Don't assume it's present.
- **Env-var change?** Update `.env.example`. Decide client vs server (`NEXT_PUBLIC_` prefix vs not) and don't read server-only vars from client code.
- **Styling?** Tailwind **v4** — config and plugin syntax differ from v3. Merge classes with `cn()` from `src/lib/utils.ts`.
- **Adding editor features?** BlockNote, `react-pdf`, `react-markdown`, and friends are mocked in Jest (`jest.config.js:31-41`). Don't rely on editor behavior in tests; test what you wrap around it.
- **Run checks:** `npm run lint` and `npm run test` before pushing. If you hit a dev-only issue, retry `next dev` without `--turbopack` to isolate.

## Domain glossary

Core hierarchy: **School (= Organization) → Cohort → User** (with role) and **School → Course → Module → ModuleItem**. Backend names in parentheses.

- **School** (backend: `organizations`) — the top-level tenant. All routes under `/school/...` operate on one. The backend always calls this an "organization"; the rename is frontend-only (routes, UI copy).
- **Cohort** — a group of users inside a school who take courses together. Users in a cohort have a role of `learner` or `mentor`. Admin UI under `/school/admin/[id]/cohorts/[cohortId]`, learner view under `/school/[id]/cohort/[cohortId]`.
- **Batch** — a subdivision of a cohort used for finer-grained grouping and analytics. Batches belong to cohorts, not schools.
- **Role** (`learner` / `mentor`) — assigned **per cohort**, not per school. The same user can be a learner in one cohort and a mentor in another. (School-level admin/owner roles exist separately on the backend.)
- **Course** — owned by a school. Contains milestones and tasks.
- **Milestone** (backend) ↔ **Module** (frontend) — same concept, renamed in `src/lib/course.ts` via `transformMilestonesToModules`. The backend never says "module"; the frontend rarely says "milestone" in user-visible text.
- **ModuleItem** — a frontend-only wrapper around a backend **Task**. Typed as `material`, `quiz`, or `assignment`, matching the backend's `TaskType` (`LEARNING_MATERIAL` / `QUIZ` / `ASSIGNMENT`).
- **Task** (backend term) — the unit of learning work. Three kinds:
  - **Learning material** — static content to read/watch.
  - **Quiz** — container for **Questions** (graded).
  - **Assignment** — single submission (code/text/audio) evaluated against criteria/scorecard.
- **Question** — lives only inside a quiz task. Not a standalone entity.
- **Block** — a rich-content node authored in the **BlockNote** editor. Persisted as JSON on the task/question/assignment. There is no separate "blocks" resource.
- **Scorecard** — a school-level rubric (criteria with min/max/pass scores) attached to questions or assignments for AI-assisted grading.
- **Chat history** — the learner's conversation with the AI coach for a given task or question. Rendered by `ChatHistoryView`.
- **Code draft** — in-progress code for a (user, question) pair, saved via the Monaco editor flow. Backend upserts on save.
- **Integration** — a per-user OAuth connection to an external service. **Notion** is the only one wired up end-to-end (`src/context/IntegrationContext.tsx`, `src/app/api/integrations/*`). Used to import pages as learning material.
- **Course / task generation job** — async AI-driven generation kicked off from the admin UI. Progress is streamed to the frontend over a WebSocket (see course-editor views).
- **HVA** — **HyperVerge Academy**, a specific hardcoded school on the backend with dedicated `/hva` endpoints. Not a general concept — treat it as a one-off when you see it.

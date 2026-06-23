# Squad Up — Youth Soccer Management Platform

> Full-stack web application for managing youth soccer teams — games, players, training, and analytics.

![Node.js](https://img.shields.io/badge/Node.js-18%2B-339933?logo=node.js&logoColor=white)
![Express](https://img.shields.io/badge/Express-4.x-000000?logo=express&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-Atlas-47A248?logo=mongodb&logoColor=white)
![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)
![Vite](https://img.shields.io/badge/Vite-5.x-646CFF?logo=vite&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind-CSS-06B6D4?logo=tailwindcss&logoColor=white)
![Jest](https://img.shields.io/badge/Jest-98%20tests%20passing-C21325?logo=jest&logoColor=white)

---

<!-- Replace this line with your demo GIF once captured:
![Squad Up demo](media/demo.gif)
-->

> **Demo screenshots below.** A screen recording will be added here shortly.

---

## What it is

Squad Up is a management platform for youth soccer development departments — built around a problem
that most sports tech tools don't solve well: **how do you turn subjective coaching observations into
meaningful, long-term insight?**

Every match assessment, player rating, and coach note entered into the system is inherently
subjective — one coach's read of a player's tactical performance on a given day. On its own, a
single rating means little. But when those observations are recorded consistently, game after game
and season after season, a pattern emerges. Over time, the accumulated data allows clubs to move
from gut feeling to an evidence-based picture of how individual players are actually developing.

This is especially relevant in **youth development and younger age groups**, where professional
analytics infrastructure (GPS tracking, video analysis, performance science teams) is rarely
available. Squad Up fills that gap with a structured, low-friction tool that coaches can realistically
use on the sideline and in the locker room — capturing the kind of qualitative data that would
otherwise live only in a coach's memory or a spreadsheet.

The result is a platform where the value compounds over time: the longer it's used, the richer
the development picture becomes for every player in the organization.

Built end-to-end as a solo project, using a **spec-driven AI-augmented development workflow** throughout.

---

## Highlights

- **Architected and built end-to-end** — owned every layer from MongoDB schema design and REST API
  architecture through React frontend and production deployment.

- **Engineered a transactional match-event engine** — a `Scheduled → Played → Done` game lifecycle
  state machine with multi-collection MongoDB transactions, a business-rule eligibility validator
  (player-state-at-minute, rolling substitutions, card progression + future-consistency checks), and
  a MongoDB-backed background job queue with atomic locking and exponential-backoff retries that
  decouples heavy analytics recalculations from API response time.

- **Built a modular React 18 frontend in Feature-Sliced Design** — with ESLint-enforced module
  boundaries, a drag-and-drop tactical lineup board, debounced autosave for drafts, React Query
  server-state caching, organization-scoped feature flags, and a player development analytics
  dashboard that turns raw match data into coaching insights.

---

## Tech stack

| Layer | Technology |
|-------|-----------|
| Backend | Node.js 18 + Express 4 |
| Database | MongoDB Atlas (18 collections) |
| Auth | JWT + bcrypt (4-role RBAC) |
| Frontend | React 18 + Vite 5 |
| Styling | Tailwind CSS |
| State | React Context + custom hooks |
| Testing | Jest (backend, 98 tests) + Playwright (E2E) |
| Logging | pino structured JSON logging |
| Architecture | MVC + Domain-Driven Design (backend), Feature-Sliced Design (frontend) |

---

## Features

| Feature | What it does |
|---------|-------------|
| **Game Scheduling** | Create and manage fixtures with opponent, date, location, match type (league / cup / friendly), and a pre-game difficulty assessment (opponent strength, match importance, external conditions) |
| **Game Execution** | The core live-match cockpit — drag-and-drop tactical board for lineup planning across three formations, minute-by-minute recording of goals (with scorer, assister, and goal type), substitutions (with reason), and disciplinary cards with automatic progression rules |
| **Post-Match Reporting** | After the final whistle, coaches rate every player across four development pillars — physical, technical, tactical, and mental — add personal notes, record a team summary, and optionally fill in detailed per-player stats (fouls, shots, passes, duels). Everything autosaves as a draft during the match so nothing is lost |
| **Player Management** | Full player profiles with a development timeline showing game-by-game rating history, season-aggregated stats, minutes played, goals, assists, and disciplinary record |
| **Player Analytics** | Automatically derived season insights: goal partnerships, minutes distribution, form trends across last N games, and consistency scores — recalculated in the background every time a game is finalized |
| **Training Management** | Plan training sessions, assign drills from the library, add session notes and objectives, and track attendance over time |
| **Drill System** | A reusable drill library that coaches can build up over time — tag drills by category, duration, and focus area, then attach them directly to training session plans |
| **Team Management** | Manage team rosters, season metadata, and a tactical board for visualizing formations outside of match context |
| **Scout Reports** | Attach free-form scouting observations and timeline events to any player profile — useful for tracking players being considered for promotion or transfer |
| **User Management + Auth** | JWT authentication with four access levels: Admin (full system), Department Manager (all teams), Division Manager (assigned teams), and Coach (own team only). All data queries are role-scoped on the backend |
| **Settings / Org Config** | Per-organization feature flags that allow each club to enable or disable specific features (e.g. pre-game difficulty assessment, shot tracking, detailed disciplinary stats) |

---

## What's coming in v2

The next major version will introduce an **AI assistant layer** embedded directly in the product.

Because Squad Up already holds a rich history of structured data — match reports, per-player ratings, coach notes, training records, and season stats — the platform is well-positioned to go beyond displaying data and start interpreting it.

The planned AI assistant will be able to:

- **Summarize a player's season** in natural language — pulling together rating trends, standout games, areas of improvement, and disciplinary history into a concise development report that a coach or parent can read in 30 seconds
- **Summarize a match** — combining the coach's post-game notes, final score, key events (goals, red cards, tactical changes), and player ratings into a structured post-match brief
- **Surface patterns a coach might miss** — for example, flagging that a player's tactical ratings consistently drop in the second half, or that a player performs significantly better in cup games than league games
- **Draft end-of-season player evaluations** — generating a first-draft performance summary per player based on the full season's recorded data, ready for the coach to review and personalize

This builds directly on the core value of the platform: the longer an organization uses Squad Up, the more raw material the AI has to work with — and the more meaningful and accurate its summaries become.

---

## Key engineering decisions

### 1. Game lifecycle as a state machine

Every game moves through a strict status sequence: `Scheduled → Played → Done` (or `Postponed`).
Each transition is guarded server-side:

- **Scheduled → Played:** requires exactly 11 players in valid formation positions with a goalkeeper; creates `GameRoster` records in a MongoDB transaction; enqueues `recalc-minutes` job.
- **Played → Done:** server calculates `minutesPlayed`, `goals`, and `assists` from events (clients cannot send these); persists all player reports; triggers goal and substitution analytics recalculation; enqueues `recalc-player-season-insights` job with 10-second delay.

See [docs/prd.md](docs/prd.md) for all 26 business rules enforced by the system.

### 2. MongoDB-backed job queue

Heavy recalculations (player minutes, season insights, goal analytics) are decoupled from API
responses via a MongoDB-based job queue:

- **Atomic locking:** `findOneAndUpdate` atomically claims jobs (`pending → running`) — no double-processing even under concurrent workers.
- **Deduplication:** `recalc-minutes` jobs use `$setOnInsert` upsert so rapid event mutations collapse into one pending job, not N.
- **Reliability:** failed jobs are retried up to 3 times with exponential backoff; completed jobs are auto-deleted after 30 days via TTL index.

See [docs/architecture.md](docs/architecture.md) for the full architecture and sequence diagrams.

### 3. Feature-Sliced Design with ESLint enforcement

The React frontend is organized by business domain (features), not by technical layer. An ESLint
plugin enforces that features cannot import from other features — all shared code must go through
`shared/`. This means:

- Adding or changing a feature never accidentally breaks another
- The dependency graph stays flat and predictable
- Component decomposition is enforced: the `game-execution` feature's root container wires
  together 17 custom hooks and 5 layout modules, with zero business logic in the layout layer

See [docs/frontend.md](docs/frontend.md) for the full component hierarchy and hook breakdown.

### 4. Spec-driven AI-augmented development

This project was built using a structured AI-augmented workflow where each feature domain has a
co-located spec bundle (PRD + architecture doc + AGENTS navigation map) that keeps AI tooling
grounded in the real codebase. This prevents context collapse as the system grows and allows
reliable implementation of complex business rules.

See [docs/ai-workflow.md](docs/ai-workflow.md) for a detailed explanation of the workflow.

---

## Screenshots

> Screenshots will be added here once captured from the running application.
> Planned: tactical board, live game view, player development timeline, analytics dashboard.

<!--
![Tactical Board](media/screenshots/tactical-board.png)
![Game Details](media/screenshots/game-details.png)
![Analytics Dashboard](media/screenshots/analytics-dashboard.png)
![Player Development](media/screenshots/player-development.png)
-->

---

## Documentation

| Document | Contents |
|----------|----------|
| [docs/prd.md](docs/prd.md) | Product requirements — lifecycle, business rules, feature flags |
| [docs/architecture.md](docs/architecture.md) | Backend internals — module map, request flows, job queue design |
| [docs/frontend.md](docs/frontend.md) | Frontend architecture — FSD structure, hooks, state management |
| [docs/ai-workflow.md](docs/ai-workflow.md) | AI-augmented development workflow |

---

## Source code

The source code is private. Available on request for serious inquiries — feel free to reach out.

---

## License

Copyright (c) 2026 Tal Solomon. All Rights Reserved. See [LICENSE](LICENSE).

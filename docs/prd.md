# Games — Product Requirements Document

> **Audience:** Anyone asking "what does this feature do?"

---

## What it does

Squad Up turns a scheduled fixture into a fully analyzed match. Coaches schedule
a game against an opponent, plan their lineup on a tactical board with drag-and-drop,
then run the live match — recording goals, substitutions and cards minute-by-minute.
After the final whistle, the coach captures per-player ratings, team summaries and
optional detailed stats, then finalizes the game so it counts toward season insights
and analytics.

The feature spans two surfaces: **Game Scheduling** (the list and creation flow) and
**Game Execution** (the large per-game cockpit). Behind both, the same backend manages
the game's lifecycle, the events on it, and the post-match reports.

---

## User roles and access

| Role | What they can do with games |
|------|-----------------------------|
| `Admin` | Full access to every game in the system |
| `Department Manager` | Full access to every game in the system |
| `Division Manager` | Access games belonging to teams they manage |
| `Coach` | Access games belonging to the team they coach only |

All API access requires a JWT bearer token; scope checks then narrow the visible set per role.

---

## Lifecycle / status model

```
┌──────────────┐  start game   ┌────────┐  submit final report  ┌──────┐
│  Scheduled   │ ────────────► │ Played │ ────────────────────► │ Done │
└──────┬───────┘               └────────┘                       └──┬───┘
       │                            ▲                              │
       │                            └──────── edit report ─────────┘
       │                            (frontend-only: clears local read-only flag)
       │
       │  postpone
       ▼
┌──────────────┐
│  Postponed   │  (terminal — navigation redirects out)
└──────────────┘
```

| Status | Meaning |
|--------|---------|
| `Scheduled` | Game created but not yet played. Lineup draft is editable; no events recorded yet. Difficulty assessment can be set, edited, or cleared. |
| `Played` | Match started. Goals, substitutions, and cards are recorded. Report draft autosaves until the coach finalizes. |
| `Done` | Match finalized. Per-player reports and team summaries are persisted. Read-only in the UI; analytics jobs trigger. |
| `Postponed` | Match cancelled or rescheduled. Terminal — the UI redirects to the schedule. |

Only Admin/Department Manager (and team-scoped Division Manager/Coach) can drive these
transitions. "Edit report" on a Done game is a client-side convenience that re-opens the
form locally; it does not change the server status.

---

## Business rules

### Scheduling and game creation

1. Every game belongs to exactly one **team** and one **season**. When a coach creates a game, the system copies the team's current season and team name onto the game so the record stays stable even if the team's metadata changes later.
2. A game's title is always displayed as `"<teamName> vs <opponent>"`; this is a derived field, not stored separately.
3. Match type is one of `league`, `cup`, or `friendly`; default is `league`.

### Lineup planning (Scheduled games)

4. Starting the match requires **exactly 11 players** assigned to formation positions, with a **goalkeeper** in the `gk` slot. These are hard blocks.
5. A bench of at least **7 players** is recommended; the system warns but does not block when there are fewer.
6. The lineup draft autosaves with a **2.5-second** debounce while the game is Scheduled.
7. Three formations are supported out of the box: `1-4-4-2`, `1-4-3-3`, `1-3-5-2`.

### Match events (Played games)

8. **Goals** come in two flavors: team goals (with scorer, optional assister, and goal involvement) and opponent goals (minute + goal type only). A scorer can score for the opponent only via an explicit own-goal.
9. A team goal's scorer must be **on the pitch** at the goal's minute; the assister (if any) must also be on the pitch; scorer and assister cannot be the same player.
10. **Cards** progress in a strict order: `yellow` → `second-yellow` → `red`. A player who is already sent off cannot receive another card. Bench players may receive cards.
11. Red and second-yellow cards trigger a **future-consistency check**: the system rejects the card if any later event in the game already involves that player.
12. **Substitutions** support rolling subs — a player who was substituted out can return later. The system rejects subs that violate ON_PITCH / BENCH / SENT_OFF state at the substitution minute.
13. Substitution `reason` defaults to `tactical` and is one of: `tactical`, `tired`, `injury`, `yellow-card-risk`, `poor-performance`, `other`.

### Reporting and finalization

14. The report draft autosaves with a **2.5-second** debounce while the game is Played.
15. Every starting-lineup player must have all four ratings (`rating_physical`, `rating_technical`, `rating_tactical`, `rating_mental`) filled before the game can be submitted Done. Each rating is **1 to 5** with default **3**.
16. Players who did not play (bench players never substituted in) are excluded from the persisted report set.
17. The server calculates `minutesPlayed`, `goals`, and `assists` from match events — clients cannot send these fields.
18. Player match stats (fouls / shooting / passing / duels) are stored as integer ratings **0 to 5**; non-integer or out-of-range values are rejected.

### Match duration

19. Match duration is always three components: `regularTime` (default **90**), `firstHalfExtraTime` (default **0**), `secondHalfExtraTime` (default **0**), with `totalMatchDuration` calculated as their sum.
20. Extra time cannot be negative; values above 15 minutes are warned about.

### Difficulty assessment

21. The pre-game difficulty assessment captures three 1–5 ratings: `opponentStrength`, `matchImportance`, `externalConditions`.
22. The overall score is computed as `(opponentStrength × 0.40) + (matchImportance × 0.35) + (externalConditions × 0.25)`, rounded to one decimal.
23. Difficulty can be set, updated, or cleared only while the game is **Scheduled** — once kickoff happens, the assessment is locked.

### Post-match side effects

24. When a game transitions to **Played**, the system enqueues a deduplicated `recalc-minutes` job so player minutes and `playedInGame` flags stay accurate.
25. When a game transitions to **Done**, the system recalculates goal analytics (sequential goal numbers + match state per goal) and substitution analytics (match state per substitution), then enqueues a `recalc-player-season-insights` job with a **10-second delay** to let final writes settle.
26. Most event mutations (goal/card/sub/report changes) also enqueue a `recalc-player-season-insights` job, but the job creator skips creation unless the game is currently Done.

---

## Feature flags

| Flag | Effect |
|------|--------|
| `gameDifficultyAssessmentEnabled` | When enabled, the difficulty assessment card appears on the game details page and the user is warned (but not blocked) at game-start if no assessment is set. When disabled, the feature is hidden entirely. |

Feature flags are read from `organization_configs` via the `useFeature` hook; they are scoped per organization.

---

## Out of scope

The Games feature intentionally does **not** do these things today:

- **Multi-match-day tournaments** — there is no concept of a tournament or bracket above the single fixture.
- **Live spectator view** — there is no public/read-only URL for parents or fans; access is staff-only.
- **Maximum substitution limit** — the system records however many substitutions are entered; competition rules are not enforced.
- **Backend-side video / clip linking** — no media attached to games or events.

# AI-Augmented Development Workflow

This document describes how Squad Up was built using a structured, spec-driven
AI-augmented workflow — where the developer's role was **architect and decision-maker**,
and AI was used as a force multiplier to accelerate execution.

---

## The core challenge with AI-assisted development

The most common failure mode when building with AI coding assistants is **context collapse**:
the AI loses track of the overall architecture as the codebase grows, starts producing code
that contradicts earlier decisions, and creates inconsistencies that are hard to untangle later.

The workflow built for this project directly solves that problem.

---

## The solution: spec-driven context grounding

Every feature domain in the codebase has a **spec bundle** that lives alongside its code.
Before any implementation begins, a spec bundle is generated from the existing codebase
(or designed from scratch). This bundle becomes the single source of truth that every
AI interaction is grounded in.

### What a spec bundle contains

```
backend/src/services/<domain>/
  AGENTS.md              ← navigation map: what's here, how parts connect
  <domain>-prd.md        ← product view: what it does and why
  <domain>-architecture.md ← backend internals: how the parts interact
  <domain>-tech-profile.yaml ← machine-readable facts (endpoints, job types, etc.)
  flows/                 ← per-entry-point flow narratives

frontend/src/features/<feature>/
  AGENTS.md              ← navigation map
  <feature>-ui-architecture.md ← component hierarchy, hooks, state, API integration
```

### Why this works

An AI coding assistant reads these files at the start of any session and immediately
has a complete, accurate map of the domain. It knows:
- Which files exist and what each one owns
- The exact lifecycle/state machine the domain enforces
- Which business rules are hard blocks vs soft warnings
- What jobs are enqueued and when
- Every cross-feature dependency

This is the difference between an AI that builds **coherently** (because it has context)
and one that makes one-off decisions that conflict with the surrounding system.

---

## AGENTS.md — the navigation layer

Every feature directory contains an `AGENTS.md`. This file is explicitly written for AI
agent consumption (as well as human engineers). It contains:

1. **File index** — every file in the domain with its role and a line-level anchor (e.g. `gameService.js:140`)
2. **Connection map** — a Mermaid diagram showing how requests flow through the layers
3. **Entry points** — every HTTP endpoint and worker job entry point, with exact function anchors
4. **Cross-feature dependencies** — what this domain imports from outside, and what imports from it

The result: any AI session that reads the `AGENTS.md` first can navigate the codebase
accurately without hallucinating file names or function signatures.

---

## Role personas — specialized review quality

The `.cursor/rules/` directory contains two custom AI personas:

### Sports Data Strategist
A specialized analytical persona that:
- Understands every collection and relationship in the Squad Up data model
- Produces metric definitions, aggregation strategies, and coach-facing insight recommendations
- Follows a structured framework (Data Inventory → Metric Generation → Insight Context → Presentation Strategy)
- Stays grounded in existing fields — never invents new data that doesn't exist in the schema

This persona was used for designing the player development analytics and goal partnership features.

### Product Design Lead
A specialized design persona that:
- Produces UI/UX direction aligned to the existing component stack (React 18, Tailwind, FSD)
- Enforces Feature-Sliced Design boundaries in design recommendations
- Follows mobile-first, accessibility-first principles
- Produces implementation specs (component selection, interaction states, acceptance criteria) — not just ideas

This persona was used when building the Game Execution cockpit and the player development timeline.

---

## Concrete example: the Games domain

The `games` domain (the most complex in the project) was built with this exact workflow:

1. **PRD first** — `games-prd.md` was written to capture all product behavior: lifecycle model, 26 business rules, feature flags, out-of-scope decisions.
2. **Architecture doc** — `games-architecture.md` captured the full module map, the request flow (with Mermaid sequence diagram), and the job queue design.
3. **AGENTS.md** — generated to provide AI-navigable anchors for every file, endpoint, and cross-feature dependency.
4. **Implementation** — each service, controller, and hook was implemented with AI assistance, grounded in the spec. The AI could reference exact rule numbers ("Rule 11: future-consistency check") when writing validation logic.
5. **Spec stays in sync** — after significant changes, the spec bundle is updated to reflect the new state. This means future AI sessions always have accurate context.

The result is docs like [prd.md](prd.md) and [architecture.md](architecture.md) in this repository — public artifacts from the spec bundle that demonstrate the quality and depth of the engineering process.

---

## What this demonstrates

This workflow shows:

- **Architectural discipline** — the developer defined domain boundaries, state machines, business rules, and cross-feature dependencies before writing implementation code
- **AI orchestration skill** — using structured context injection, specialized personas, and spec-driven grounding to get consistent, high-quality output from AI tools
- **Documentation as engineering** — the spec bundle is not written after the fact; it drives the implementation forward

In 2026, the ability to build complex systems effectively with AI assistance is a genuine,
in-demand engineering skill. This workflow is a practical example of how to do it at scale.

---

## Tools used

| Tool | Role |
|------|------|
| [Cursor](https://cursor.sh) | AI-native IDE — agent mode, inline completions, per-project rules |
| Claude (Anthropic) | Primary AI model for implementation, spec generation, code review |
| Custom Cursor skills | Reusable agent workflows (spec-module, feature-brief) |
| Custom Cursor rules | Domain-specific AI personas (data analyst, UI/UX designer) |

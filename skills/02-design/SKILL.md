---
name: design
description: Executes the DESIGN half of a development phase for web/frontend projects by dispatching a fresh-context design-implementer per task with two-stage review (visual fidelity + code quality). Builds components with hardcoded / placeholder data — the downstream feature pipeline (`features`) wires them to real data later in the same phase. Atomic commits, `BLOCKED`/`NEEDS_CONTEXT` escalation, no guessing. Does NOT auto-dispatch impeccable — if a design brief is thin, the design-implementer BLOCKS and the user may invoke `impeccable:shape <surface>` manually before resuming. Calls `close-design-phase` at the end (which does NOT push or PR — the feature pipeline closes the single PR per phase). Runs BEFORE `features` in every phase that has design tasks. Requires `setup` to have run.
---

# Execute Design — Web

Execute the DESIGN half of a development phase from TASKS.md using subagent-driven development. Each design-domain task gets a fresh design-implementer with clean context, followed by two-stage review (visual fidelity + code quality). The controller (you) orchestrates — subagents implement.

This is the first half of the per-phase pipeline. The design half commits visual components with hardcoded data; the feature half (`features` → `close-feature-phase`) wires them to real data and opens the single PR.

## Paths

All arsenal artifacts live under `.arsenal/` at the project root.

| What | Path | Notes |
|---|---|---|
| Strategy archive (denied during build) | `.arsenal/strategy/` | MVP_SPEC.md, mockup-briefs/, GTM_STRATEGY.md, REVENUE_MODEL.md, research/{MARKET_RESEARCH,RESEARCH_PLAN}.md |
| Feature specs | `.arsenal/FEATURES.md` (single-mode) or `.arsenal/features/<slug>.md` (split-mode) | Gated per phase via `.claude/settings.json` |
| Project anchor docs | `.arsenal/{ARCHITECTURE,CONVENTIONS,TASKS}.md` | Always readable during build |
| Design reference set | `.arsenal/design/{UX,DESIGN,DESIGN_SYSTEM}.md` + `.arsenal/design/mockups/` | Always readable during build |
| Per-task briefs + ephemera | `.arsenal/tasks/phase-N/`, `.arsenal/tasks/parallel/`, `.arsenal/tasks/archive/` | Gitignored; phase-N gated per active phase |

**Configuration:** `.arsenal/config.yaml` may override the root location, but defaults work for nearly all projects. File names are not configurable.

**Gating:** `expand-phase` writes baseline denies and per-phase allow rules to `.claude/settings.json`. `close-feature-phase` reverts at phase end. Strategy stays fully denied throughout build.

## The phase pipeline — strict sequence per phase

```
design   →   close-design-phase   →   features   →   close-feature-phase
   (design tasks)         (design-pipeline wrap)        (feature tasks)            (phase-final wrap + PR)
```

Both pipelines share **one branch** (`phase-N/short-description`) and **one PR** per phase. This skill creates / checks out the branch on its first run. `close-design-phase` (invoked at Step 4) does NOT push or PR — it returns the branch to the orchestrator for the feature pipeline to pick up. The feature pipeline's `close-feature-phase` opens the single PR at the end.

Design close must complete before any feature task starts; the feature pipeline relies on design-pipeline-committed components being present in git history.

## The hardcoded data contract — load-bearing rule

Design tasks build visual components with **hardcoded / placeholder data**. Real data wiring is the feature pipeline's job and runs later in the same phase.

Specifically:

- Components take literal, realistic values for demo purposes (`name="Sample User"`, `count={3}`, `items={SAMPLE_ITEMS}` where `SAMPLE_ITEMS` is a hardcoded constant).
- Every state from the design brief's variant coverage (loading, empty, error, success, hover/focus/active, dark theme, mobile, etc.) is rendered — typically via Storybook stories or top-level prop toggles.
- No data-fetching utilities, no server actions, no auth checks.
- If a component genuinely needs a derived value, the implementer hardcodes a representative one and leaves a `// TODO(feature-pipeline):` comment.

The rule is enforced by `run-task-design`'s visual fidelity reviewer (CRITICAL fail if data wiring is detected). This orchestrator documents the rule and trusts the per-task pipeline.

The component-boundary rule (enforced in the feature pipeline) protects this work: the feature pipeline may extend (new props/variants/states for data) but not redesign.

## Impeccable is optional and manual — never automatic

This orchestrator does **not** dispatch `impeccable:shape`, `impeccable:craft`, or any other impeccable command. If the design brief for a task is thin (sparse mockup, missing variant coverage, novel IA), the design-implementer BLOCKS. The user decides whether to invoke `impeccable:shape <surface>` (or any impeccable command) manually, update the brief or DESIGN_SYSTEM, and then re-run with `--force`.

The user can also invoke an impeccable audit pass at `close-design-phase` as a soft-gateable phase audit — but that's also user-opted, not automatic.

## Philosophy

- **Fresh context per task.** Subagents never inherit your session history.
- **Two-stage review.** Visual fidelity catches "shipped but doesn't match the design brief" (the design pipeline's spec-compliance equivalent). Code quality catches "right design but sloppy code." Both required.
- **Never guess when stuck.** Subagents report BLOCKED or NEEDS_CONTEXT.
- **Atomic commits.** Every completed task gets its own commit.
- **No auto-impeccable.** Optional and manual at every point.

## Prerequisites

| File | Used For |
|------|----------|
| `.arsenal/TASKS.md` | Phase block in concrete-tasks state with `### Design tasks` and `### Feature tasks` subsections (this skill iterates the former) |
| `.arsenal/design/UX.md` | UX/IA, page sections, components per page — excerpted into design briefs |
| `.arsenal/design/DESIGN.md` | Brand spec, do-not-edit — cited from design briefs |
| `.arsenal/design/DESIGN_SYSTEM.md` | Stack-specific implementation, token map, primitive cites — excerpted into design briefs; declares component-root path |
| `.arsenal/CONVENTIONS.md` | View-structure patterns — excerpted into design briefs |
| `.arsenal/ARCHITECTURE.md` | High-level skim only (the design pipeline doesn't need full architecture) |
| `.arsenal/FEATURES.md` (single mode) or `.arsenal/features/<slug>.md` files (split mode) | Per-feature acceptance criteria, states, copy locks, anti-patterns |
| `.arsenal/design/mockups/<screen>.{jsx,tsx,html,png,figma-export.json}` (optional but recommended) | Design briefs translate mockup regions to component implementations |

If TASKS.md / UX.md / DESIGN_SYSTEM.md don't exist, tell the user to run `/arsenal-build:setup` first.

`impeccable` is **not** a prerequisite. If a brief comes back thin, the user may invoke impeccable manually — but the orchestrator never preflights it.

## Workflow

### Step 1: Phase Selection & Setup

**Identify the phase and scope:**
- "design phase 1" or "build the visual components for phase 1" → phase 1, full design-domain scope
- "begin work on phase N" without specifying → if the phase has unchecked design tasks, this skill handles them first; if the design half is done, route the user to `features`
- Scope can narrow ("design phase 1, just the hero") — resolve here, pass the structured `--scope` arg downstream.

**Confirm there are design tasks:** Read `TASKS.md` for the target phase. If `### Design tasks` is empty / placeholder (`_None — pure feature-domain phase._`), tell the user: "Phase N has no design tasks. Skip to `/arsenal-build:features N`."

**Set up the workspace:**
- **Git initialization:** If no git repo, run `git init` and ask the user for the remote URL.
- Create the phase branch if it doesn't exist: `git checkout -b phase-N/short-description`. **Use the same branch the feature pipeline will share.**
- Verify tests pass on the current branch baseline (if tests exist).

**Context protection:**

If an `.arsenal/` directory exists, set up Read deny rules so strategy and feature spec docs aren't loaded during development:

- **Strategy archive:** always deny `Read(.arsenal/strategy/**)` during build.
- **Single mode** (`.arsenal/FEATURES.md` exists): suggest `Read(.arsenal/FEATURES.md)` deny; per-phase allow rules are added by `expand-phase`.
- **Split mode** (`.arsenal/features/` directory exists): suggest `Read(.arsenal/features/*)` deny but **allow** `Read(.arsenal/features/README.md)`.

**Create the phase working directory:** `.arsenal/tasks/phase-N/` for this phase's working files (context briefs, design briefs, research files).

### Step 2: Expand Tasks & Generate Design Briefs

Step 2 has two parts: **expand** the placeholder phase into concrete tagged tasks (handed off to `expand-phase`), then **generate design briefs** for each design-domain task (handed off to `generate-design-briefs`).

#### Expand the phase (hand-off)

If the target phase has placeholder tasks, hand off to `expand-phase`:

```
/arsenal-build:expand-phase --phase N [--scope full | feature=<slug> | user-story=<id> | ux-section=<name>] [--force]
```

That skill writes the concrete tagged task list back to `TASKS.md` with `### Design tasks` and `### Feature tasks` subsections. This orchestrator iterates only the design subsection.

If the phase already has concrete tasks, `expand-phase` no-ops; proceed directly to the brief-generation hand-off.

#### Generate design briefs (hand-off)

After `expand-phase` returns, hand off design-brief generation to `generate-design-briefs`:

```
/arsenal-build:generate-design-briefs --phase N [--task <N>] [--force]
```

That skill writes per-task context briefs to `.arsenal/tasks/phase-N/task-N-context.md` (≤3k tokens) AND per-task design briefs to `.arsenal/tasks/phase-N/task-N-design.md` (≤1.7k tokens) for every `domain: design` task. Idempotent by default (L1 contract).

If the design brief subagent reports a thin brief or a `MOCKUP_DS_GAP_BLOCKING`, this orchestrator surfaces the gap to the user and pauses. The user decides whether to invoke `impeccable:shape <surface>` manually (or update the brief / DESIGN_SYSTEM by hand) and then re-run with `--force`. **This orchestrator never auto-dispatches impeccable.**

The controller (this orchestrator) does **not** read brief contents after `generate-design-briefs` reports DONE — `run-task-design` reads briefs by path during Step 3 dispatch.

After both `expand-phase` and `generate-design-briefs` return, **drop UX.md and DESIGN_SYSTEM.md from controller context** — the briefs and the now-concrete TASKS.md are the working artifacts.

See `skills/expand-phase/SKILL.md` and `skills/generate-design-briefs/SKILL.md` for the full pipelines.

### Step 3: Design task execution loop (hand-off)

Iterate over `- [ ]` tasks in the phase's `### Design tasks` subsection, in declaration order. For each task, hand off to `run-task-design`:

```
/arsenal-build:run-task-design --phase N --task N
```

That skill runs the per-task pipeline: researcher (if `research: yes`) → design-implementer (hardcoded data discipline, no auto-impeccable) → visual fidelity review (static analysis: token map adherence, state coverage, mockup ↔ code match) → code quality review → atomic commit + flip `[ ]` to `[x]` in TASKS.md.

**Sequential only — never dispatch two `run-task-design` calls in parallel.** Tasks within a phase often touch overlapping files; parallel execution creates merge conflicts.

**Idempotence (M1):** if a task is already `[x]`, `run-task-design` no-ops. To re-run a completed task, invoke directly with `--force`.

**Skip non-design tasks.** This loop iterates only `### Design tasks`. Feature tasks are the feature pipeline's responsibility (later in the same phase).

After every `- [ ]` task in `### Design tasks` has flipped to `[x]`, proceed to Step 4.

See `skills/run-task-design/SKILL.md` for the full per-task pipeline, including the BLOCKED escalations for thin design briefs (where the user may invoke impeccable manually).

### Step 4: Close the design half (hand-off)

Once every design task is committed and `[x]`-marked, hand off to `close-design-phase`:

```
/arsenal-build:close-design-phase N
```

That skill runs the design-pipeline wrap: design QA review (visual fidelity at scale across surfaces shipped) → optional impeccable audit (gateable; soft-prompt user — never automatic) → docs update (if scope drifted) → trim the design-task block in TASKS.md / annotate `.arsenal/tasks/phase-N/` for the feature pipeline.

**`close-design-phase` does NOT push and does NOT open a PR.** It returns the branch to the orchestrator. The feature pipeline's `close-feature-phase` opens the single PR per phase at the end.

After `close-design-phase` returns DONE, this orchestrator reports completion to the user and recommends: "Design half of phase N complete. Run `/arsenal-build:features N` to wire the components to real data."

See `skills/close-design-phase/SKILL.md` for the full gate specifications.

## Handling Edge Cases

**Design brief comes back thin (NEEDS_USER_RESOLUTION):**
`generate-design-briefs` surfaces the unresolved cells. This orchestrator pauses and tells the user: "Design brief for task N is under-specified on <X>. Options: (a) invoke `impeccable:shape <surface>` manually to fill the gap, (b) update DESIGN_SYSTEM.md / the mockup directly, (c) defer this task. After fixing, re-run with `--force --task N`."

**Mockup ↔ DESIGN_SYSTEM gap blocks brief generation:**
`generate-design-briefs` reports `MOCKUP_DS_GAP_BLOCKING`. Same pattern as above — surface to the user, do not pick canonical unilaterally.

**A task wants to wire real data (CRITICAL visual fidelity failure):**
`run-task-design`'s visual fidelity reviewer flags it. The fix subagent reverts the data layer code and uses hardcoded values. The orchestrator surfaces the violation so the user knows the design pipeline overreached.

**User wants to change scope mid-phase:**
Pause execution. Update TASKS.md. Resume from where you left off.

**Tests don't exist yet (new project):**
Skip the "verify clean baseline" step. The first task might add the test setup.

**Context window getting large:**
Checkpoint and suggest fresh session.

## Anti-Patterns — Never Do These

(Per-task subagent disciplines moved to `skills/run-task-design/SKILL.md`. Brief generation disciplines moved to `skills/generate-design-briefs/SKILL.md`. Wrap-gate disciplines moved to `skills/close-design-phase/SKILL.md`. The disciplines below are orchestrator-level.)

- **Don't auto-dispatch impeccable.** Never preflight, never auto-shape, never auto-craft. The user invokes impeccable manually when a brief comes back thin.
- **Don't push or open a PR from this skill.** That's `close-feature-phase`'s terminus, after the feature pipeline runs. The design pipeline's terminus is "all design tasks `[x]` and `close-design-phase` has reported DONE."
- **Don't iterate `### Feature tasks`.** This loop processes only `### Design tasks`. Feature tasks are the feature pipeline's responsibility.
- **Don't wire data.** If a design task accidentally wires real data, the visual fidelity reviewer flags it as CRITICAL — revert and use hardcoded values.
- **Don't drop UX.md or DESIGN_SYSTEM.md into controller context after Step 2.** Briefs are the working artifacts thereafter.
- **Don't build on a failing test suite.** Verify a clean baseline at Step 1 preflight.
- **Don't loop tasks in parallel.** Step 3 hands off to `run-task-design` sequentially.

## Session Management

**Starting a session (fresh or resumed):**
- Read TASKS.md to find where you left off.
- Find the first phase whose `### Design tasks` has unchecked `- [ ]` items. That's where this skill resumes.
- If a phase's design half is done but feature half isn't, route the user to `features`.

**Pausing mid-phase:**
- Ensure all current work is committed.
- TASKS.md reflects progress (`[x]` for completed design tasks).
- Tell the user where you stopped: "Completed design tasks 1-3 of Phase 2. Next up: design task 4 (Hero component). Safe to resume in a new session."

**Resuming:**
- Read TASKS.md, find the active phase, find the first incomplete `### Design tasks` task.
- Continue the execution loop.

## Integration with Other Skills

| Skill | Relationship |
|-------|-------------|
| `/arsenal-planning:mvp`, `/arsenal-planning:features`, `/arsenal-planning:ux-*`, `/arsenal-planning:design` | Upstream planning — produce the artifacts `setup` consumes. |
| `/arsenal-build:setup` | Creates the docs this skill reads. Must run before this skill. |
| `/arsenal-build:expand-phase` | **Invoked at Step 2 (first half)** as a sub-skill (shared with the feature orchestrator). |
| `/arsenal-build:generate-design-briefs` | **Invoked at Step 2 (second half)** as a sub-skill. Writes per-task context + design briefs. |
| `/arsenal-build:run-task-design` | **Invoked at Step 3** as a sub-skill, once per `- [ ]` task in `### Design tasks`. Runs researcher → design-implementer → visual fidelity review → quality review → atomic commit + `[x]`. |
| `/arsenal-build:close-design-phase` | **Invoked at Step 4** as a sub-skill. Runs design-pipeline wrap (no push, no PR — returns the branch for the feature pipeline). |
| `/arsenal-build:features` | **Runs after this skill in every phase that has design tasks.** Sibling orchestrator. Wires the components this skill committed to real data. Shares the same phase branch. |
| `/arsenal-build:close-feature-phase` | Runs after `features`. Opens the single PR per phase. |
| `impeccable` | **Not invoked from this skill, ever.** Available to the user as a manual escape hatch when a design brief is thin — user invokes `impeccable:shape <surface>` manually, updates the brief or DESIGN_SYSTEM, then re-runs this skill with `--force`. |

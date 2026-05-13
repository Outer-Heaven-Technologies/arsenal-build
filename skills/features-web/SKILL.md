---
name: features-web
description: Executes the FEATURE half of a development phase for web/frontend projects (Next.js, Astro, Vite, Node, Python, etc.) by dispatching a fresh-context feature-implementer per task with two-stage review (spec compliance + code quality). Atomic commits, `BLOCKED`/`NEEDS_CONTEXT` escalation, no guessing. The component-boundary rule enforces extend-not-redesign on design-pipeline-committed components. Does NOT call impeccable. Runs AFTER `design-web` + `close-design-phase-web` have completed for the same phase. Calls `close-feature-phase-web` at the end. Requires `anchor-files` to have run. Use when the design half of a phase is committed and you're ready to wire the components to real data. For native iOS, use `features-ios`.
---

# Execute Features — Web

Execute the FEATURE half of a development phase from TASKS.md using subagent-driven development. Each feature-domain task gets a fresh feature-implementer with clean context, followed by two-stage review (spec compliance + code quality). The controller (you) orchestrates — subagents implement.

This is the second half of the per-phase pipeline. The design half (`design-web` → `close-design-phase-web`) commits the visual components first; this skill wires them to real data.

## Paths

Tracked artifacts use these default locations (override via `.arsenal/config.yaml` at the project root):

| Variable | Default | Holds |
|---|---|---|
| `paths.planning` | `planning/` | MARKET_RESEARCH.md, MVP_SPEC.md, FEATURES.md (or features/*.md), GTM_STRATEGY.md, REVENUE_MODEL.md, RESEARCH_PLAN.md |
| `paths.docs` | `docs/` | UX.md, DESIGN.md, DESIGN_SYSTEM.md, ARCHITECTURE.md, CONVENTIONS.md, TASKS.md |
| `paths.mockups` | `docs/mockups/` | Mockup files (PNG, HTML, TSX, Figma exports) |
| `paths.mockup_briefs` | `planning/mockup-briefs/` | Mockup briefs |

**Preflight (every run):** before reading or writing a tracked artifact, check for `.arsenal/config.yaml` at the project root. If present, parse `paths.*` and use those values; otherwise use defaults silently — do not prompt the user just to confirm defaults. File names (e.g. `MVP_SPEC.md`) are not configurable; only their wrapping directory is.

**Consuming an artifact from another skill:** if config (or defaults) point to a location where the expected artifact is missing, ask the user where to find it instead of failing.

## The phase pipeline — strict sequence per phase

```
design-web   →   close-design-phase-web   →   features-web   →   close-feature-phase-web
   (design tasks)         (design-pipeline wrap)        (feature tasks)            (phase-final wrap + PR)
```

Both pipelines share **one branch** (`phase-N/short-description`) and **one PR** per phase. The design-pipeline close does NOT push or PR — it returns the branch to the orchestrator. This skill runs second and `close-feature-phase-web` opens the single PR at the end. Design close must complete before any feature task starts; the feature pipeline relies on design-pipeline-committed components being present in git history.

If invoked before the design half completes, this skill stops with `NEEDS_CONTEXT` and tells the caller to run `design-web` first (or to confirm that this phase has no design tasks — see "Pure feature-domain phases" below).

## The component boundary — load-bearing rule

The feature pipeline wires existing components (committed by `design-web` earlier in the same phase) to real data. It does not do design work.

Specifically:

- **Components committed under the project's component root** (typically `components/ui/` and `components/shell/` — exact path declared by `DESIGN_SYSTEM.md`) are **read-only design surfaces** to the feature pipeline.
- **Extensions are allowed** — new props, new variants, or new states required by the data layer.
- **Visual changes are NOT allowed** — visual treatment, spacing, typography, color, or motion changes go through the design pipeline (a `--force` re-expansion of the design half, or a future phase).
- **Any commit that modifies a file under the project's component root must include a `Component extended: <path> — <one-line why>` note in the commit message body.** Enforced by `run-task-feature-web`'s spec reviewer.
- **Purely visual changes BLOCK** the feature-implementer with redirect to the design pipeline.

The rule is enforced at the per-task level (in `run-task-feature-web` and its feature-implementer prompt). This orchestrator documents the rule and trusts the per-task pipeline to apply it.

## Philosophy

- **Fresh context per task.** Subagents never inherit your session history. Prevents context rot.
- **Two-stage review.** Spec compliance first (also enforces the component-boundary rule); code quality second.
- **Never guess when stuck.** Subagents report BLOCKED or NEEDS_CONTEXT.
- **Atomic commits.** Every completed task gets its own commit.
- **Impeccable is gone.** This orchestrator does not preflight, dispatch, or even mention impeccable. The feature pipeline never invokes it.

## Prerequisites

| File | Used For |
|------|----------|
| `docs/TASKS.md` | Phase block in concrete-tasks state with `### Design tasks` and `### Feature tasks` subsections (this skill iterates the latter) |
| `docs/ARCHITECTURE.md` | System overview, data flow, schema — excerpted into the per-task feature briefs |
| `docs/CONVENTIONS.md` | Code patterns, library idioms — quality standards for review |
| `docs/DESIGN_SYSTEM.md` | Declares the project's component-root path; used by the feature-implementer to know which files are design surfaces |
| `planning/FEATURES.md` (single mode) or `planning/features/<slug>.md` files (split mode) | Per-feature acceptance criteria, states, data lifecycle — drives feature-domain task expansion |
| Phase branch with design-pipeline commits | The components this skill's tasks may wire to must be present in git history |

If TASKS.md / ARCHITECTURE.md / CONVENTIONS.md don't exist, tell the user to run `/anchor-files` first. If the design half hasn't completed for this phase, run `/arsenal-build:design-web N` first.

## Runtime Environment

**This skill is designed for Claude Code** (or any environment with subagent/Task tool support).

**Available tools:**
- **GitHub MCP:** Use for creating branches, PRs, and issues programmatically during phase completion.
- **Web search / Firecrawl / Jina:** When a subagent is BLOCKED or NEEDS_CONTEXT due to a library API question — research the answer before escalating to the user.

## Workflow

### Step 1: Phase Selection & Setup

**Identify the phase and scope:**
- "begin work on phase 1" → phase 1, full feature-domain scope
- "start building" without specifying → read TASKS.md, find the first phase whose design half is complete but whose feature half is not, confirm with the user
- Scope can narrow (*"expand phase 1, just the &lt;feature-name&gt; feature"*) — resolve here, pass the structured `--scope` arg downstream.

**Confirm the design half is done.** Read `TASKS.md` for the target phase:
- If `### Design tasks` still has unchecked `[ ]` items, STOP. Tell the user: "Phase N has uncompleted design tasks. Run `/arsenal-build:design-web N` first."
- If `### Design tasks` is empty / placeholder (pure feature-domain phase), proceed directly — no design half needed.
- If `### Design tasks` are all `[x]` and `close-design-phase-web` has committed any design-pipeline cleanup, proceed.

**Set up the workspace:**
- **Git initialization:** If the project doesn't have a git repo yet, run `git init` and ask the user for the remote URL.
- The phase branch (`phase-N/short-description`) should already exist from `design-web` (the two pipelines share the branch). If it doesn't exist (pure feature-domain phase, or this is being run standalone), create it.
- Verify tests pass on the current branch baseline (if tests exist).

**Context protection:**

If a `planning/` directory exists, set up Read deny rules so planning docs aren't loaded during development:

- **Single mode** (`planning/FEATURES.md` exists): suggest `Read(planning/*)` deny.
- **Split mode** (`planning/features/` directory exists): suggest `Read(planning/features/*)` deny but **allow** `Read(planning/features/README.md)`.

Example `.claude/settings.json` for split mode:
```json
{
  "permissions": {
    "deny": ["Read(planning/features/*)"],
    "allow": ["Read(planning/features/README.md)"]
  }
}
```

**Create / verify the phase working directory:** `.tasks/phase-N/` should exist from the design half. Confirm it does; if not (pure feature-domain phase), create it.

### Step 2: Generate feature briefs (hand-off)

The design half already expanded the phase via `expand-phase` and wrote design-domain briefs via `generate-design-briefs`. This skill needs feature-domain briefs.

If `expand-phase` hasn't run yet for this phase (rare — usually run during the design half), invoke it first:

```
/arsenal-build:expand-phase --phase N --surface web [--scope full | feature=<slug> | user-story=<id> | ux-section=<name>] [--force]
```

Then hand off to `generate-feature-briefs`:

```
/arsenal-build:generate-feature-briefs --phase N --surface web [--task <N>] [--force]
```

That skill writes per-task context briefs to `.tasks/phase-N/task-N-context.md` (≤3k tokens, every feature task). Each brief carries an `## Available components` section enumerating the components the design pipeline committed earlier in this phase (sourced via `git log` on the phase branch). Idempotent by default (L1 contract).

The controller (this orchestrator) does **not** read brief contents after `generate-feature-briefs` reports DONE — `run-task-feature-web` reads briefs by path during Step 3 dispatch.

See `skills/generate-feature-briefs/SKILL.md` for the full briefing pipeline.

After both `expand-phase` (if invoked) and `generate-feature-briefs` return, **drop feature specs and UX.md from controller context** — the briefs and the now-concrete TASKS.md are the working artifacts.

### Step 3: Feature task execution loop (hand-off)

Iterate over `- [ ]` tasks in the phase's `### Feature tasks` subsection, in declaration order. For each task, hand off to `run-task-feature-web`:

```
/arsenal-build:run-task-feature-web --phase N --task N
```

That skill runs the per-task pipeline: researcher (if `research: yes`) → feature-implementer (single path, no impeccable, enforces component boundary) → spec compliance review (enforces `Component extended:` commit rule) → code quality review → atomic commit + flip `[ ]` to `[x]` in TASKS.md.

**Sequential only — never dispatch two `run-task-feature-web` calls in parallel.** Tasks within a phase often touch overlapping files; parallel execution creates merge conflicts.

**Idempotence (M1):** if a task is already `[x]`, `run-task-feature-web` no-ops. To re-run a completed task, invoke directly with `--force`.

**Skip non-feature tasks.** This loop iterates only `### Feature tasks`. Tasks under `### Design tasks` should be `[x]` already (the design pipeline completed them). If any design task is still `[ ]`, that's a state inconsistency — stop and tell the user.

After every `- [ ]` task in `### Feature tasks` has flipped to `[x]`, proceed to Step 4.

See `skills/run-task-feature-web/SKILL.md` for the full per-task pipeline.

### Step 4: Close the phase (hand-off)

Once every feature task is committed and `[x]`-marked, hand off to `close-feature-phase-web`:

```
/arsenal-build:close-feature-phase-web N
```

That skill runs the phase-final gate sequence: final integration test → Playwright test coverage (if configured) → docs update (if scope drifted) → CodeRabbit review (hard gate, runs across the entire phase including design-pipeline commits) → trim `TASKS.md` + archive `.tasks/phase-N/` → push branch + open PR.

CodeRabbit covers the full phase (design + feature commits together) — this is why it runs at the feature close, not the design close. Single CodeRabbit pass per PR.

See `skills/close-feature-phase-web/SKILL.md` for the full gate specifications.

## Pure feature-domain phases

A phase may have no design tasks (`### Design tasks` is just the `_None — pure feature-domain phase._` placeholder). In that case:

- Skip the "confirm design half is done" check in Step 1 (there's nothing to confirm).
- The phase branch is created by this skill, not by `design-web`.
- `close-design-phase-web` is not invoked.
- Otherwise behavior is identical.

This pattern fits backend-only phases (auth, payments webhook, migrations) and infrastructure phases (Phase 0 setup).

## Handling Edge Cases

**Design tasks still unchecked when this skill is invoked:**
Stop. Tell the user: "Phase N has uncompleted design tasks at <line numbers>. Run `/arsenal-build:design-web N` first." Do not proceed; the available-components manifest in feature briefs would be empty or inconsistent.

**A feature task wants to redesign a component:**
The per-task `run-task-feature-web` BLOCKS with `reason: purely visual change required — redirect to design pipeline`. This orchestrator surfaces the block to the user and recommends either (a) re-expand the design half with `expand-phase --force` to add a design task for the visual change, or (b) defer the visual change to a future phase.

**Subagent produces code that conflicts with another task's work:**
This is why tasks run sequentially. If you encounter a conflict with previously completed work in this phase, dispatch a fix subagent that understands both tasks' requirements.

**User wants to change scope mid-phase:**
Pause execution. Update TASKS.md with the user's changes. Resume from where you left off.

**Tests don't exist yet (new project, Phase 0):**
Skip the "verify clean baseline" step. The implementer prompt still asks for TDD, but if the project has no test infrastructure yet, the first task should set it up.

**Context window getting large (long phase):**
Checkpoint: summarize progress, note remaining tasks, suggest the user start a fresh session. The TASKS.md checkboxes preserve state across sessions.

## Anti-Patterns — Never Do These

(Per-task subagent disciplines moved to `skills/run-task-feature-web/SKILL.md`. Brief generation moved to `skills/generate-feature-briefs/SKILL.md`. Wrap-gate disciplines moved to `skills/close-feature-phase-web/SKILL.md`. The disciplines below are orchestrator-level.)

- **Don't invoke impeccable.** The feature pipeline never calls impeccable. If a task seems to need design judgement, the design pipeline missed something — surface to the user.
- **Don't run before the design half completes.** If `### Design tasks` has unchecked items, stop and tell the user. The component-boundary rule depends on design-pipeline commits being present.
- **Don't iterate `### Design tasks`.** This loop processes only `### Feature tasks`. Design tasks are the design pipeline's responsibility.
- **Don't push or open a PR from this skill.** That's `close-feature-phase-web`'s terminus. This orchestrator's terminus is "all feature tasks `[x]` and the close skill has reported DONE."
- **Don't drop feature specs or UX.md into controller context after Step 2.** Briefs are the working artifacts thereafter.
- **Don't read archived task lists speculatively.** Completed phases get trimmed in `TASKS.md`.
- **Don't build on a failing test suite.** Verify a clean baseline at Step 1 preflight.
- **Don't loop tasks in parallel.** Step 3 hands off to `run-task-feature-web` sequentially.

## Session Management

**Starting a session (fresh or resumed):**
- Read TASKS.md to find where you left off.
- Phases marked `✅ Completed` are trimmed stubs — skip them.
- Find the first phase whose `### Feature tasks` has unchecked `- [ ]` items AND whose `### Design tasks` are all `[x]`. That's where this skill resumes.
- If a phase's design half is still in progress, route the user to `design-web` instead.

**Pausing mid-phase:**
- Ensure all current work is committed.
- TASKS.md reflects progress (`[x]` for completed feature tasks).
- Tell the user where you stopped: "Completed feature tasks 1-3 of Phase 2. Next up: feature task 4 (user authentication). Safe to resume in a new session."

**Resuming:**
- Read TASKS.md, find the active phase, find the first incomplete `### Feature tasks` task.
- Verify the branch and test state.
- Continue the execution loop.

## Integration with Other Skills

| Skill | Relationship |
|-------|-------------|
| `/arsenal-planning:mvp`, `/arsenal-planning:features`, `/arsenal-planning:ux-*`, `/arsenal-planning:design` | Upstream planning — produce the artifacts `anchor-files` consumes. |
| `/arsenal-build:anchor-files` | Creates the docs this skill reads. Must run before this skill. |
| `/arsenal-build:design-web` | **Runs before this skill in every phase that has design tasks.** Sibling orchestrator. Commits the visual components this skill's tasks may wire to. Shares the same phase branch. |
| `/arsenal-build:close-design-phase-web` | Runs after `design-web` and before this skill. Returns the branch (no push, no PR). |
| `/arsenal-build:expand-phase` | Invoked when needed at Step 2 (usually already run by `design-web`). |
| `/arsenal-build:generate-feature-briefs` | **Invoked at Step 2** as a sub-skill. Writes per-task context briefs with the available-components manifest. |
| `/arsenal-build:run-task-feature-web` | **Invoked at Step 3** as a sub-skill, once per `- [ ]` task in `### Feature tasks`. Runs researcher → feature-implementer → spec review (with `Component extended:` enforcement) → quality review → atomic commit + `[x]`. |
| `/arsenal-build:close-feature-phase-web` | **Invoked at Step 4** as a sub-skill. Runs the phase-final gates (tests → Playwright → docs → CodeRabbit → trim+archive → push+PR). Opens the single PR per phase. |
| `/arsenal-planning:gtm` | The next step after all phases are complete. |
| `impeccable` | **Not invoked from this skill, ever.** |

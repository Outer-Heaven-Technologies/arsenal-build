---
name: run-task-feature-ios
description: Runs a single `domain: feature` task from a native iOS project's `TASKS.md` phase through the full per-task pipeline — optional researcher (if `research: yes`), feature-implementer dispatch (single path, no design judgement, no visual fidelity gate, no finishers), spec compliance review (enforces the `Component extended:` commit-message rule), code quality review, atomic commit, and `[x]` mark in `TASKS.md`. Does NOT call impeccable, ever. The implementer treats design-pipeline-committed views under the project's view-layer module as read-only design surfaces that may be **extended** (new `@Bindable` parameters, state, or variants the data layer needs) but never **redesigned** (visual treatment, spacing, typography, color, motion). Invoked by `features-ios` once per feature task in a phase loop, after `close-design-phase-ios` has committed the phase's views. Idempotent by default; `--force` re-runs an already-`[x]` task. Does NOT auto-fire on "begin work on phase N" phrasing — that's the orchestrator's territory.
---

# Run Task — Feature, iOS

Execute one `domain: feature` task from a native iOS project's `TASKS.md` phase through the per-task pipeline:

1. Optional researcher (when the task is tagged `research: yes`)
2. Feature-implementer dispatch — with mandatory simulator handshake (carried in the implementer prompt template)
3. Handle implementer response (DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED)
4. Spec compliance review (fix-and-re-review loop; enforces the `Component extended:` commit-message rule)
5. Code quality review (fix-and-re-review loop; reads project `CLAUDE.md` § Hard rules and § Voice + tone; hex literals outside the theme module are CRITICAL hard-fail)
6. Atomic commit + `[x]` in TASKS.md

The pipeline runs **sequentially per task** — never dispatch two feature-implementers in parallel. The orchestrator loops this sub-skill per `- [ ]` task under the phase's `### Feature tasks` subsection; this sub-skill never iterates across multiple tasks itself.

**No visual fidelity gate, no finisher pass.** Both belong to the design pipeline (`run-task-design-ios`). Feature-domain iOS tasks are data-layer work: services, `@Model` types, queries, mutations, and view extensions for data wiring.

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

## The component boundary — load-bearing rule

The feature pipeline wires existing views (committed by the design pipeline earlier in the same phase) to real data. It does not do design work.

Specifically:

- **Views and components committed under the project's view-layer module** (typically `Views/` or the project's feature folder — the exact path is declared by `DESIGN_SYSTEM.md`) are **read-only design surfaces** to the feature pipeline.
- **Extensions are allowed** — new `@Bindable` parameters, new `@State`, new variants, or new props that the data layer requires.
- **Visual changes are NOT allowed** — visual treatment, spacing, typography, color, or motion changes to an existing view are blocked.
- **Any commit that modifies a file under the project's view-layer module must include a `Component extended: <path> — <one-line why>` note in the commit message body.** The spec reviewer enforces this at Step 4.
- **Purely visual changes BLOCK** — if the implementer needs to change visual treatment to complete the task, it reports `BLOCKED` with `reason: purely visual change required — redirect to design pipeline`.

See `references/feature-implementer-prompt.md` (carries the rule) and `references/spec-reviewer-prompt.md` (enforces the rule).

## Prerequisites

| Upstream artifact | Source | Used for |
|---|---|---|
| `docs/TASKS.md` with phase block in concrete-tasks state | `expand-phase` upstream | Read the task line and tags (`domain: feature`, `research:`); flip `[ ]` → `[x]` after success |
| `.tasks/phase-{N}/task-{N}-context.md` | `generate-feature-briefs` upstream | Feature-implementer reads it by path; carries the available-components manifest |
| `docs/CONVENTIONS.md`, `docs/ARCHITECTURE.md`, `docs/DESIGN_SYSTEM.md` | `anchor-files` upstream | Excerpted into the context brief upstream — this skill does not re-read them in full |
| `CLAUDE.md` (root, optional) | Project | Code quality reviewer reads § Hard rules + § Voice + tone as additional review criteria |
| Phase branch checked out (`phase-N/short-description`) | Orchestrator Step 1 | All commits land on this branch; design-pipeline commits are already present |
| Xcode open + one simulator booted | Orchestrator Step 1 preflight | Required for build/test calls — this skill never boots/opens silently |
| Design-pipeline commits already on the phase branch | `close-design-phase-ios` upstream | The views this task may wire to are reachable in git history |

If any upstream artifact is missing, report `NEEDS_CONTEXT` and surface which one.

`impeccable` is **not** a prerequisite. The feature pipeline never invokes it (finishers belong to the design pipeline only).

## Inputs

Caller passes:

- `--phase <N>` — phase number (required)
- `--task <N>` — task identifier within the phase (required)
- `--force` — optional. Re-run the full pipeline even when the task is already `[x]`.

The task's tags (`domain:`, `research:`) are re-read from `docs/TASKS.md` directly. If the task's `domain:` is not `feature`, report `WRONG_DOMAIN` — that task belongs to `run-task-design-ios`.

## Workflow

### Step 1: Validate inputs and check idempotence

Read `docs/TASKS.md` for phase `<N>`. Locate the `### Feature tasks` subsection. Confirm task `<N>` exists there.

- Task lives under `### Design tasks` instead: `WRONG_DOMAIN`.
- Task is `[x]` and `--force` NOT set: `SKIPPED — task already complete`.
- Task is `[x]` and `--force` IS set: continue.
- Task is `[ ]`: continue.

Confirm `.tasks/phase-{N}/task-{N}-context.md` exists. If missing, report `NEEDS_CONTEXT` and tell the caller to run `/arsenal-build:generate-feature-briefs --phase N --task N --surface ios` first.

Confirm iOS tooling state: Xcode is open (`mcp__xcode__XcodeListWindows`), a simulator is booted (`xcrun simctl list devices booted`). If either is missing, report `NEEDS_CONTEXT` — do not auto-boot or auto-open.

### Step 2: Dispatch researcher subagent (if `research: yes`)

Ask the user: "Task N flagged for research (reason: [Family Controls / CloudKit / StoreKit 2 / scenePhase / SwiftData migration / etc.]). Run research first?" Default: yes.

If yes, dispatch using `references/researcher-prompt.md`. The researcher writes `.tasks/phase-{N}/task-{N}-research.md` and terminates. This skill confirms the file was written but does not read its contents.

### Step 3: Dispatch feature-implementer subagent

Dispatch using `references/feature-implementer-prompt.md`. Substitute every bracketed placeholder before dispatch.

The implementer reads its briefs (context + research if applicable) and the available-components section of the context brief. The template carries the component-boundary rule, the simulator handshake, and iOS-specific contracts (concurrency, state management, etc.).

**Model selection:**
- Mechanical (1-2 files, clear spec) → Sonnet/Haiku
- Integration (multi-file, schema migration, service wiring) → Sonnet
- Architectural (novel state machine, scenePhase coordination, CloudKit boundary) → Opus

### Step 4: Handle implementer response

| Status | Action |
|--------|--------|
| **DONE** | Proceed to spec review (Step 5). |
| **DONE_WITH_CONCERNS** | Read concerns; address correctness/scope before review; note observations. |
| **NEEDS_CONTEXT** | Provide missing context; re-dispatch. If unknown, surface to user. |
| **BLOCKED** | Assess and recommend one of five responses, then wait for user direction: (1) Missing context → provide it. (2) Needs more reasoning → upgrade model. (3) Task too large → recommend `expand-phase --force` to split. (4) Plan is wrong → escalate. (5) **Purely visual change required** → escalate; recommend deferring the visual change to the design pipeline. **Never force the same model to retry without changes.** |

### Step 5: Dispatch spec compliance reviewer

Dispatch using `references/spec-reviewer-prompt.md`. The reviewer checks spec compliance AND enforces the `Component extended:` rule (every commit that touches a file under the project's view-layer module must carry the note).

**If the spec reviewer finds issues** (including missing `Component extended:` notes or purely visual changes disguised as extensions):

- Dispatch a fix subagent (feature-implementer template) with the specific issues.
- Spec reviewer re-reviews.
- Repeat until approved.

A purely visual change to an existing view is a CRITICAL spec failure: the fix subagent reverts the visual portion and the user is notified.

### Step 6: Dispatch code quality reviewer

Only after spec compliance passes. Dispatch using `references/quality-reviewer-prompt.md`. Reads the project's `CLAUDE.md` (if present) and applies § Hard rules + § Voice + tone as additional review criteria.

**Hard-fail criteria (CRITICAL):** hex literal in feature code (outside theme module), `Any` or `as!` in new code, Combine in new code, missing `@MainActor` on UI mutations, force-quit-resistant write order violated, memory cycle in escaping closures.

**IMPORTANT severity:** view body > 50 lines, `ObservableObject` in new code, `.onAppear { Task { } }`, missing accessibility identifier, hardcoded numeric literals, `Task { ... }` in view body without `.task`, mocked `@Model` types in tests, missing `final` on `@Model`, `@AppStorage` for non-debug user content, dead code introduced by this task.

**MINOR severity** can be deferred to the per-task report.

**If the quality reviewer finds CRITICAL or IMPORTANT issues:** dispatch a fix subagent, re-review. MINOR can be deferred.

### Step 7: Complete the task

- Ensure all commits landed with semantic messages. Component-touching commits include the `Component extended:` note.
- Mark the task as complete in `TASKS.md`: change `- [ ]` to `- [x]` under `### Feature tasks`.
- Report DONE to the caller.

### Step 8: Return to caller

Report:

- **Status:** DONE | SKIPPED (M1 idempotence) | WRONG_DOMAIN (route to `run-task-design-ios`) | NEEDS_CONTEXT (cite which brief / tool / simulator state is missing) | BLOCKED (cite reason)
- **Phase:** N
- **Task:** N
- **Researcher ran:** yes | no
- **Implementer model:** Sonnet | Haiku | Opus
- **Spec review:** passed | passed after N fix cycles
- **Quality review:** passed | passed after N fix cycles
- **Components extended (if any):** list of `<path> — <one-line why>` from commit messages
- **Deferred minor issues:** list (aggregate into the phase summary at `close-feature-phase-ios`)
- **Commits added:** list of semantic messages
- **TASKS.md:** task N flipped to `[x]` under `### Feature tasks`

## Reference files

| File | Used in | Purpose |
|------|---------|---------|
| `references/researcher-prompt.md` | Step 2 | Researcher — Apple HIG, SwiftUI 18 patterns, library API drift |
| `references/feature-implementer-prompt.md` | Step 3 (and fix dispatches in Steps 5/6) | Feature-implementer — carries iOS tooling prefs, simulator handshake, component-boundary rule, `Component extended:` commit format |
| `references/spec-reviewer-prompt.md` | Step 5 | Spec compliance reviewer — checks spec AND enforces `Component extended:` rule |
| `references/quality-reviewer-prompt.md` | Step 6 | Code quality reviewer — Swift/SwiftUI specific; reads project `CLAUDE.md` |

Each template has bracketed placeholders. Substitute every one before dispatch — never ship a template with stubs intact.

## Anti-patterns — never do these

- **Don't let subagents inherit this skill's session context.** They read briefs by path — that's the only handoff.
- **Don't paste full canonical docs into subagent prompts.** Briefs at `.tasks/phase-{N}/task-{N}-*.md` were generated upstream by `generate-feature-briefs`. Reference them by path.
- **Don't run visual fidelity or finishers.** Both belong to the design pipeline. If a feature task seems to need visual judgement, it was misclassified.
- **Don't skip either review stage.** Spec compliance catches "well-written but wrong" + missing `Component extended:` notes. Quality catches "correct but sloppy."
- **Don't ship hex literals outside the project's theme module.** Quality reviewer fails on this (CRITICAL).
- **Don't boot simulators or open Xcode silently.** If the orchestrator's preflight didn't satisfy this, surface NEEDS_CONTEXT and stop.
- **Don't redesign views.** The feature pipeline extends views for the data layer, never restyles them. Visual changes BLOCK with redirect to the design pipeline.
- **Don't omit the `Component extended:` note.** Every commit modifying a view-layer file carries the note. Spec reviewer enforces this.
- **Don't dispatch multiple feature-implementers in parallel.** They'll create merge conflicts.
- **Don't ignore BLOCKED or NEEDS_CONTEXT.** Something needs to change.
- **Don't invoke impeccable.** The feature pipeline never calls impeccable.
- **Don't re-run a task that's already `[x]` without `--force`.**

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:features-ios` | Invokes this skill in a loop, one call per `- [ ]` task in the phase's `### Feature tasks` subsection. Runs after `close-design-phase-ios` has returned. |
| `/arsenal-build:expand-phase` | Runs **before** this skill (via the orchestrator). |
| `/arsenal-build:generate-feature-briefs` | Runs **before** this skill (via the orchestrator). |
| `/arsenal-build:close-design-phase-ios` | Runs **before** the orchestrator invokes this skill. Commits the views this skill's tasks may wire to. |
| `/arsenal-build:close-feature-phase-ios` | Runs **after** all feature tasks in the phase complete. Aggregates deferred Minor issues, runs `RunAllTests` + Periphery + snapshot verify + CodeRabbit + push + PR. |
| `mcp__xcode__*` MCP | Step 3 implementer uses Build, RunSomeTests/RunAllTests, GetBuildLog. |
| `mcp__ios-simulator__*` MCP | Step 3 implementer for simulator handshake + any test runs that need the booted device. |
| `impeccable` | **Not invoked from this skill, ever.** |

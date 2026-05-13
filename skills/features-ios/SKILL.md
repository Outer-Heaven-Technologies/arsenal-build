---
name: features-ios
description: Executes the FEATURE half of a development phase for a native iOS project (SwiftUI + Xcode). Dispatches a fresh-context feature-implementer per task with two-stage review (spec compliance + code quality). Atomic commits, `BLOCKED`/`NEEDS_CONTEXT` escalation. The component-boundary rule enforces extend-not-redesign on design-pipeline-committed views. Does NOT call impeccable. No visual fidelity gate, no finisher pass (both belong to the design pipeline). Runs AFTER `design-ios` + `close-design-phase-ios` have completed for the same phase. Calls `close-feature-phase-ios` at the end. Requires `anchor-files` to have run. Use when the design half of a phase is committed and you're ready to wire views to real data.
---

# Execute Features — iOS

Execute the FEATURE half of a development phase from `TASKS.md` for a native iOS project. Each feature-domain task gets a fresh feature-implementer with clean context, followed by two-stage review (spec compliance + code quality). The controller (you) orchestrates — subagents implement.

This is the second half of the per-phase pipeline. The design half (`design-ios` → `close-design-phase-ios`) commits the SwiftUI views first; this skill wires them to real data via `@Query`, services, `@Environment`, etc.

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

## When to use this skill vs. `features-web`

| Project signal | Skill |
|---|---|
| `*.xcodeproj` or app-target `Package.swift` at root | **`features-ios`** |
| `package.json` (Next.js, Astro, Vite, etc.) | `features-web` |
| `Cargo.toml`, `go.mod`, server-only repo | `features-web` |
| Cross-platform (RN, Flutter, Tauri) | Ask the user |

## The phase pipeline — strict sequence per phase

```
design-ios   →   close-design-phase-ios   →   features-ios   →   close-feature-phase-ios
   (design tasks)         (design-pipeline wrap)        (feature tasks)            (phase-final wrap + PR)
```

Both pipelines share **one branch** (`phase-N/short-description`) and **one PR** per phase. The design close does NOT push or PR — it returns the branch. The feature close opens the single PR.

Design close must complete before any feature task starts; the available-components manifest in feature briefs comes from the design-pipeline's commits on the branch.

If invoked before the design half completes, this skill stops with `NEEDS_CONTEXT` and tells the caller to run `design-ios` first (unless the phase is pure feature-domain — see below).

## The component boundary — load-bearing rule

The feature pipeline wires existing views (committed by `design-ios` earlier in the same phase) to real data. It does not do design work.

Specifically:

- **Views and components committed under the project's view-layer module** (the exact path is declared by `DESIGN_SYSTEM.md`) are **read-only design surfaces** to the feature pipeline.
- **Extensions are allowed** — new `@Bindable` parameters, new `@State`, new variants, or new props the data layer requires.
- **Visual changes are NOT allowed** — visual treatment, spacing, typography, color, or motion changes go through the design pipeline (a `--force` re-expansion or a future phase).
- **Any commit that modifies a file under the project's view-layer module must include a `Component extended: <path> — <one-line why>` note in the commit body.** Enforced by `run-task-feature-ios`'s spec reviewer.
- **Purely visual changes BLOCK** the feature-implementer with redirect to the design pipeline.

The rule is enforced at the per-task level (in `run-task-feature-ios` and its feature-implementer prompt).

## Philosophy

- **Fresh context per task.** Subagents never inherit your session history.
- **Two-stage review.** Spec compliance + code quality. No visual fidelity gate — that's design-pipeline territory.
- **Mockups are pixel truth** — but that's the design pipeline's concern. The feature pipeline takes the design-committed views as given.
- **Tokens are mandatory** — but hex-literal violations should already be caught by the design pipeline. The feature pipeline's quality reviewer still hard-fails on hex literals (defense in depth).
- **Project-specific rules come from `CLAUDE.md`, not the skill.**
- **Never guess when stuck.** Subagents report BLOCKED or NEEDS_CONTEXT.
- **Atomic commits.** Every completed task gets its own commit.
- **Impeccable is gone.** The feature pipeline never invokes it (finishers belong to the design pipeline only).

## Prerequisites

| File | Used For |
|------|----------|
| `docs/TASKS.md` | Phase block in concrete-tasks state with `### Design tasks` and `### Feature tasks` subsections (this skill iterates the latter) |
| `docs/ARCHITECTURE.md` | SwiftData `@Model` types, CloudKit boundaries, schema — excerpted into feature briefs |
| `docs/CONVENTIONS.md` | Swift patterns, theme primitive rules, concurrency model |
| `docs/DESIGN_SYSTEM.md` | Declares the project's view-layer module path and theme module path; used by the feature-implementer to know which files are design surfaces |
| `docs/DESIGN.md` | Brand spec, do-not-edit (cited only) |
| `planning/features/<slug>.md` (split mode) or `planning/FEATURES.md` (single mode) | Per-feature specs |
| `CLAUDE.md` (root, optional) | Project-specific hard rules + voice/tone — read by the quality reviewer |
| Phase branch with design-pipeline commits | The views this skill's tasks may wire to must be present in git history |

If TASKS.md / ARCHITECTURE.md / CONVENTIONS.md / DESIGN_SYSTEM.md don't exist, tell the user to run `/arsenal-build:anchor-files` first. If the design half hasn't completed for this phase, run `/arsenal-build:design-ios N` first.

`impeccable` is NOT a prerequisite (finishers belong to the design pipeline only).

## Required tooling

| Tool | Purpose | Fallback |
|---|---|---|
| `mcp__xcode__*` (Apple Xcode MCP) | Build, test, project tree | `xcodebuild` via Bash |
| `mcp__ios-simulator__*` | Launch app, run tests on a booted device | `xcrun simctl` |
| Test framework (XCTest) | Unit tests for services, view models, `@Model` types | None |

The skill prefers MCP tools over raw shell.

**Pre-session contract:** before feature tasks dispatch, the project must be open in Xcode and at least one simulator must be booted. The skill does **not** boot simulators or open Xcode silently — it prompts the user.

## Workflow

### Step 1: Phase Selection & Setup

**Identify the phase and scope.** "begin work on phase 1" → phase 1 full feature-domain scope; "start building" → first incomplete phase, confirm.

**Confirm the design half is done.** Read `TASKS.md` for the target phase:
- If `### Design tasks` still has unchecked `[ ]` items, STOP. Tell the user: "Phase N has uncompleted design tasks. Run `/arsenal-build:design-ios N` first."
- If `### Design tasks` is the `_None — pure feature-domain phase._` placeholder, proceed.
- If `### Design tasks` are all `[x]` and `close-design-phase-ios` has returned DONE for this phase, proceed.

**Set up the workspace:**
- The phase branch (`phase-N/short-description`) should already exist from `design-ios`. If not (pure feature-domain phase or standalone run), create it.
- Verify build clean: `mcp__xcode__BuildProject`. Don't build on a broken foundation.
- Verify tests pass: `mcp__xcode__RunAllTests` if a test target exists.

**Pre-flight (iOS-specific):**
- Confirm Xcode is open (`mcp__xcode__XcodeListWindows`).
- Confirm a simulator is booted (`xcrun simctl list devices booted`).
- If either is missing, prompt the user — never boot/open silently.

**Context protection:**

If a `planning/` directory exists, set up Read deny rules. Single mode: deny `Read(planning/*)`. Split mode: deny `Read(planning/features/*)` but allow `Read(planning/features/README.md)`.

**Create / verify the phase working directory:** `.tasks/phase-N/` should exist from the design half. Confirm it does; if not (pure feature-domain phase), create it.

### Step 2: Generate feature briefs (hand-off)

The design half already expanded the phase and wrote design-domain briefs. This skill needs feature-domain briefs.

If `expand-phase` hasn't run yet (rare), invoke it first:

```
/arsenal-build:expand-phase --phase N --surface ios [--scope full | feature=<slug> | user-story=<id> | ux-section=<name>] [--force]
```

Then hand off to `generate-feature-briefs`:

```
/arsenal-build:generate-feature-briefs --phase N --surface ios [--task <N>] [--force]
```

That skill writes per-task context briefs to `.tasks/phase-N/task-N-context.md` (≤3k tokens, every feature task). Each brief carries an `## Available components` section enumerating the views the design pipeline committed earlier in this phase.

The controller does **not** read brief contents after `generate-feature-briefs` reports DONE — `run-task-feature-ios` reads briefs by path.

After both `expand-phase` (if invoked) and `generate-feature-briefs` return, **drop feature specs and UX.md from controller context** — the briefs and TASKS.md are the working artifacts.

See `skills/generate-feature-briefs/SKILL.md` for the full pipeline.

### Step 3: Feature task execution loop (hand-off)

Iterate over `- [ ]` tasks in the phase's `### Feature tasks` subsection, in declaration order. For each task, hand off to `run-task-feature-ios`:

```
/arsenal-build:run-task-feature-ios --phase N --task N
```

That skill runs the per-task pipeline: researcher (if `research: yes`) → feature-implementer (single path, simulator handshake, no visual fidelity gate, no finishers, no impeccable) → spec compliance review (enforces `Component extended:` rule) → code quality review (reads project `CLAUDE.md`) → atomic commit + flip `[ ]` to `[x]`.

**No visual fidelity gate, no finisher pass** at this stage. Both ran in the design pipeline.

**Sequential only — never dispatch two `run-task-feature-ios` calls in parallel.** iOS tasks often touch the same SwiftUI files; parallel execution creates merge conflicts.

**Idempotence (M1):** if a task is already `[x]`, `run-task-feature-ios` no-ops. `--force` re-runs.

**Skip non-feature tasks.** This loop iterates only `### Feature tasks`. Design tasks should be `[x]` from the design pipeline.

After every `- [ ]` task in `### Feature tasks` has flipped to `[x]`, proceed to Step 4.

See `skills/run-task-feature-ios/SKILL.md` for the full per-task pipeline.

### Step 4: Close the phase (hand-off)

Once every feature task is committed and `[x]`-marked, hand off to `close-feature-phase-ios`:

```
/arsenal-build:close-feature-phase-ios N
```

That skill runs the phase-final gate sequence: `RunAllTests` integration check → Periphery dead-code scan → snapshot test verification (against any theme primitives modified by the design pipeline) → docs update (if scope drifted) → CodeRabbit review (hard gate, runs across the entire phase including design-pipeline commits) → trim TASKS.md + archive `.tasks/phase-N/` (with PNG cleanup) → push branch + open PR.

CodeRabbit covers the full phase (design + feature commits together) — single CodeRabbit pass per PR. Snapshot test verification at this gate catches regressions if any feature-task touched theme module values.

See `skills/close-feature-phase-ios/SKILL.md` for the full gate specifications.

## Pure feature-domain phases

A phase may have no design tasks (`### Design tasks` is just the `_None — pure feature-domain phase._` placeholder). In that case:

- Skip the "confirm design half is done" check in Step 1.
- The phase branch is created by this skill, not by `design-ios`.
- `close-design-phase-ios` is not invoked.
- Otherwise behavior is identical.

This pattern fits data-layer phases (schema migration, CloudKit sync, services) and infrastructure phases.

## Handling Edge Cases

**Design tasks still unchecked when this skill is invoked:** Stop. Tell the user to run `/arsenal-build:design-ios N` first.

**A feature task wants to redesign a view:** `run-task-feature-ios` BLOCKS with `reason: purely visual change required — redirect to design pipeline`. Surface to the user; recommend re-expanding the design half or deferring to a future phase.

**Snapshot test baseline drift:** never auto-accept. Prompt: "Update snapshot baseline for `<Test>`? Y/N." Commit baseline updates as `test(phase-N): update snapshot baseline for [primitive]`. (Note: snapshot verification gate runs at `close-feature-phase-ios`, not in this skill directly.)

**Simulator state corruption:** prompt the user to reset (`xcrun simctl shutdown all && xcrun simctl boot ...`). Don't silently restart.

**Build fails mid-phase:** capture `mcp__xcode__GetBuildLog`, dispatch debug subagent with log + recently changed files. Don't stack new tasks on a broken build.

**`xcodebuild` keeps timing out:** switch to `mcp__xcode__BuildProject` calls. If still timing out, ask the user to clean DerivedData.

**Task depends on something not yet built:** reorder the queue. If the dependency is in a different phase, flag and ask.

**User changes scope mid-phase:** pause, update `TASKS.md`, resume.

**Context window getting large:** checkpoint and suggest fresh session.

## Anti-Patterns — Never Do These

(Per-task subagent disciplines moved to `skills/run-task-feature-ios/SKILL.md`. Brief generation moved to `skills/generate-feature-briefs/SKILL.md`. Wrap-gate disciplines moved to `skills/close-feature-phase-ios/SKILL.md`. The disciplines below are orchestrator-level.)

- **Don't invoke impeccable.** The feature pipeline never calls impeccable. If a task seems to need design judgement or polish (finishers), the design pipeline missed something — surface to the user.
- **Don't run before the design half completes.** If `### Design tasks` has unchecked items, stop.
- **Don't iterate `### Design tasks`.** This loop processes only `### Feature tasks`.
- **Don't push or open a PR from this skill.** That's `close-feature-phase-ios`'s terminus.
- **Don't run visual fidelity or finisher gates.** Both belong to the design pipeline.
- **Don't boot simulators or open Xcode silently.** Step 1 preflight prompts the user.
- **Don't loop tasks in parallel.** Sequential execution is deliberate.
- **Don't read archived task lists speculatively.**
- **Don't drop feature specs or UX.md into controller context after Step 2.**

## Session Management

**Starting (fresh or resumed):**
- Confirm Xcode open + simulator booted.
- Read `TASKS.md`. Trimmed `✅ Completed` phases skip.
- First phase whose `### Feature tasks` has unchecked items AND whose `### Design tasks` are all `[x]` → continue execution loop here.
- If a phase's design half is still in progress, route the user to `design-ios` instead.

**Pausing mid-phase:**
- Ensure all current work is committed.
- `TASKS.md` reflects progress.
- Tell the user where you stopped.

**Resuming:**
- Re-confirm tooling state.
- Read `TASKS.md`, find the active phase, continue.

## Integration with Other Skills

| Skill | Relationship |
|---|---|
| `/arsenal-planning:mvp`, `/arsenal-planning:features`, `/arsenal-planning:ux-ios`, `/arsenal-planning:design` | Upstream planning. |
| `/arsenal-build:anchor-files` | Creates `TASKS.md`, `ARCHITECTURE.md`, `CONVENTIONS.md`, `DESIGN_SYSTEM.md`, `CLAUDE.md`. Must run before this skill. |
| `/arsenal-build:design-ios` | **Runs before this skill in every phase that has design tasks.** Sibling orchestrator. Commits the views this skill's tasks may wire to. Shares the same phase branch. |
| `/arsenal-build:close-design-phase-ios` | Runs after `design-ios` and before this skill. Returns the branch (no push, no PR). |
| `/arsenal-build:expand-phase` | Invoked when needed at Step 2 (usually already run by `design-ios`). |
| `/arsenal-build:generate-feature-briefs` | **Invoked at Step 2** as a sub-skill. Writes per-task context briefs. |
| `/arsenal-build:run-task-feature-ios` | **Invoked at Step 3** as a sub-skill, once per `- [ ]` task in `### Feature tasks`. |
| `/arsenal-build:close-feature-phase-ios` | **Invoked at Step 4** as a sub-skill. Runs the phase-final gates and opens the single PR per phase. |
| `coderabbit:review` | Dispatched by `close-feature-phase-ios`. Hard gate. |
| `impeccable` | **Not invoked from this skill, ever.** |

## What's iOS-specific in this skill (vs upstream `features-web`)

| Element | iOS variant |
|---|---|
| Tooling | Xcode MCP, iOS Simulator MCP for tests |
| Implementer rules | Hex literal lockdown, accessibility identifiers, `@Observable` macro, Swift Concurrency only, `@MainActor` on UI mutations, `final class` on `@Model` — carried in `run-task-feature-ios`'s implementer prompt |
| Phase wrap | Periphery dead-code scan + snapshot test verification before CodeRabbit (in `close-feature-phase-ios`) |

Everything else (subagent dispatch, brief budget, two-stage review structure, archive trim, anti-patterns around context bloat) is shared philosophy with upstream `features-web`.

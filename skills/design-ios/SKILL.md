---
name: design-ios
description: Executes the DESIGN half of a development phase for a native iOS project (SwiftUI + Xcode). Dispatches a fresh-context design-implementer per task with three-stage review (visual fidelity → spec compliance → code quality) plus auto-dispatched finishers (harden, animate, clarify, typeset, arrange) per the task's `finishers: [...]` tag. Builds SwiftUI views with hardcoded data — the downstream feature pipeline (`features-ios`) wires them to real data later in the same phase. Atomic commits, `BLOCKED`/`NEEDS_CONTEXT` escalation. Does NOT auto-dispatch `impeccable:shape` or `impeccable:craft` — if a design brief is thin, the design-implementer BLOCKS and the user may invoke impeccable manually. Finishers (impeccable sub-skills harden/animate/clarify/typeset/arrange) ARE auto-dispatched as per-task polish. Calls `close-design-phase-ios` at the end (which does NOT push or PR — the feature pipeline closes the single PR). Runs BEFORE `features-ios` in every phase that has design tasks.
---

# Execute Design — iOS

Execute the DESIGN half of a development phase from `TASKS.md` for a native iOS project. Each design-domain task gets a fresh design-implementer with clean context, followed by three-stage review (visual fidelity → spec compliance → code quality) plus a finisher pass. The controller (you) orchestrates — subagents implement.

This is the first half of the per-phase pipeline. The design half commits SwiftUI views with hardcoded data; the feature half (`features-ios` → `close-feature-phase-ios`) wires them to real data and opens the single PR.

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
design-ios   →   close-design-phase-ios   →   features-ios   →   close-feature-phase-ios
   (design tasks)         (design-pipeline wrap)        (feature tasks)            (phase-final wrap + PR)
```

Both pipelines share **one branch** (`phase-N/short-description`) and **one PR** per phase. This skill creates / checks out the branch. `close-design-phase-ios` (invoked at Step 4) does NOT push or PR — it returns the branch for the feature pipeline. `close-feature-phase-ios` opens the single PR.

## The hardcoded data contract — load-bearing rule

Design tasks build SwiftUI views with **hardcoded / placeholder data**. Real data wiring (`@Query`, services, `@Environment(\.modelContext)`) is the feature pipeline's job.

Specifically:

- View parameters take literal, realistic values for demo purposes.
- Every state from the design brief's variant coverage is rendered (typically via `#Preview` macros with multiple variants).
- No `@Query`, no service injection, no `@Environment(\.modelContext)` beyond state demos.
- If a view needs a derived value, hardcode a representative one and leave a `// TODO(feature-pipeline):` comment.

The rule is enforced by `run-task-design-ios`'s visual fidelity reviewer (CRITICAL fail if real data wiring is detected). The component-boundary rule (enforced in the feature pipeline) protects this work.

## Impeccable: shape and craft are user-manual; finishers are auto-dispatched

Two impeccable surfaces apply, with different rules:

- **`impeccable:shape <surface>` and `impeccable:craft <surface>` are NEVER auto-dispatched.** If a design brief is thin, the design-implementer BLOCKS and the user decides whether to invoke `impeccable:shape <surface>` manually, update the brief / DESIGN_SYSTEM by hand, and re-run with `--force`.

- **Finishers ARE auto-dispatched per the task's `finishers: [list]` tag.** Finishers are impeccable sub-skills (`harden`, `animate`, `clarify`, `typeset`, `arrange`) that run as a per-task post-implementation polish pass inside `run-task-design-ios` (Step 8 of that skill). They're scoped to the files this task modified — never sibling tasks' files.

The distinction: shape/craft are design-judgement commands the user opts into; finishers are scoped polish passes the brief pre-commits to via `expand-phase`'s auto-flag heuristics.

`impeccable:audit` and `impeccable:polish` may also run at `close-design-phase-ios` as a soft-gateable phase audit — but those are user-opted, not automatic.

## Philosophy

- **Fresh context per task.** Subagents never inherit your session history.
- **Three-stage review.** Visual fidelity catches "shipped but doesn't match the mockup." Spec compliance catches "well-structured but wrong behavior" (especially async/await ordering and scenePhase). Code quality catches "right behavior but sloppy code." All required for design-domain iOS tasks.
- **Mockups are pixel truth.** When `docs/mockups/<screen>.{jsx,tsx,html,png,figma-export.json}` exists for a surface, it overrides any narrative description.
- **Tokens are mandatory.** Hex literals are illegal outside the project's theme module.
- **Project-specific rules come from `CLAUDE.md`.**
- **Never guess when stuck.** Subagents report BLOCKED or NEEDS_CONTEXT.
- **Atomic commits.**

## Prerequisites

| File | Used For |
|------|----------|
| `docs/TASKS.md` | Phase block in concrete-tasks state with `### Design tasks` and `### Feature tasks` subsections (this skill iterates the former) |
| `docs/ARCHITECTURE.md` | High-level skim |
| `docs/CONVENTIONS.md` | Swift patterns, view-structure rules, concurrency model |
| `docs/DESIGN_SYSTEM.md` | SwiftUI implementation of the brand spec — token map, primitives, declared theme module path |
| `docs/DESIGN.md` | Brand spec, do-not-edit |
| `docs/UX.md` | Screen inventory, sections per screen, anti-patterns |
| `docs/mockups/` | High-fidelity mockups — pixel/copy/layout truth when present |
| `planning/features/<slug>.md` (split mode) or `planning/FEATURES.md` (single mode) | Per-feature specs — design-relevant excerpts (states, copy locks, anti-patterns) |
| `CLAUDE.md` (root, optional) | Project-specific rules — read by the quality reviewer |

If `TASKS.md` / `ARCHITECTURE.md` / `CONVENTIONS.md` / `DESIGN_SYSTEM.md` don't exist, tell the user to run `/arsenal-build:anchor-files` first. Mockups are optional but materially improve visual fidelity.

`impeccable` is **not** a preflight requirement for `shape`/`craft` (the design pipeline doesn't auto-dispatch them). It IS required at runtime if any task has a non-empty `finishers: [...]` list (the design-implementer's finisher pass at Step 8 dispatches harden/animate/clarify/etc.). If `impeccable` is missing when a finisher would run, `run-task-design-ios` reports `NEEDS_CONTEXT` — the user installs impeccable and resumes with `--force`.

## Required tooling

| Tool | Purpose | Fallback |
|---|---|---|
| `mcp__xcode__*` | Build, test, SwiftUI preview render | `xcodebuild` via Bash |
| `mcp__ios-simulator__*` | Screenshot, a11y tree, navigation per design brief | `xcrun simctl` for status only |
| `swift-snapshot-testing` (PointFree) | Visual regression on theme primitives (for `snapshot: yes` tasks) | None — fail visual gate |

**Pre-session contract:** before design tasks dispatch, the project must be open in Xcode and at least one simulator must be booted. The skill does **not** boot simulators or open Xcode silently.

## Workflow

### Step 1: Phase Selection & Setup

**Identify the phase and scope.** "design phase 1" or "build the visual components for phase 1" → phase 1 full design scope. Sub-scope allowed.

**Confirm there are design tasks.** Read `TASKS.md`. If `### Design tasks` is `_None — pure feature-domain phase._`, tell the user: "Phase N has no design tasks. Skip to `/arsenal-build:features-ios N`."

**Set up the workspace:**
- Create the phase branch if it doesn't exist: `git checkout -b phase-N/short-description`. Use the same branch the feature pipeline will share.
- Verify build clean: `mcp__xcode__BuildProject`.
- Verify tests pass: `mcp__xcode__RunAllTests` if a test target exists.

**Pre-flight (iOS-specific):**
- Confirm Xcode is open (`mcp__xcode__XcodeListWindows`).
- Confirm a simulator is booted (`xcrun simctl list devices booted`).
- If either is missing, prompt the user — never boot/open silently.

**Context protection:** Single mode → deny `Read(planning/*)`. Split mode → deny `Read(planning/features/*)` but allow `Read(planning/features/README.md)`.

**Create the phase working directory:** `.tasks/phase-N/` (briefs, research files, screenshots — gitignore `*.png` here).

### Step 2: Expand Tasks & Generate Design Briefs

#### Expand the phase (hand-off)

If the phase has placeholder tasks, hand off to `expand-phase`:

```
/arsenal-build:expand-phase --phase N --surface ios [--scope full | feature=<slug> | user-story=<id> | ux-section=<name>] [--force]
```

That skill reads the phase header, relevant UX sections, feature specs, `docs/mockups/`, and skims of ARCHITECTURE / CONVENTIONS / DESIGN_SYSTEM. iOS tag set: `domain:`, `research:`, plus `finishers:` and `snapshot:` on design-domain tasks only.

If concrete tasks already exist, `expand-phase` no-ops.

#### Generate design briefs (hand-off)

After `expand-phase` returns, hand off to `generate-design-briefs`:

```
/arsenal-build:generate-design-briefs --phase N --surface ios [--task <N>] [--force]
```

That skill writes per-task context briefs to `.tasks/phase-N/task-N-context.md` (≤3k tokens) AND per-task design briefs to `.tasks/phase-N/task-N-design.md` (≤1.7k tokens) for every `domain: design` task. Idempotent by default (L1 contract).

If a brief comes back thin (`NEEDS_USER_RESOLUTION`) or with `MOCKUP_DS_GAP_BLOCKING`, surface to the user and pause. The user decides whether to invoke `impeccable:shape <surface>` manually, update the mockup or DESIGN_SYSTEM by hand, or skip the task. **This orchestrator never auto-dispatches impeccable.**

The controller does **not** read brief contents after `generate-design-briefs` reports DONE.

After both `expand-phase` and `generate-design-briefs` return, **drop feature specs and UX.md from controller context.**

See `skills/expand-phase/SKILL.md` and `skills/generate-design-briefs/SKILL.md` for the full pipelines.

### Step 3: Design task execution loop (hand-off)

Iterate over `- [ ]` tasks in the phase's `### Design tasks` subsection, in declaration order. For each task, hand off to `run-task-design-ios`:

```
/arsenal-build:run-task-design-ios --phase N --task N
```

That skill runs the per-task pipeline: researcher (if `research: yes`) → design-implementer (simulator handshake, token preload, hardcoded data) → visual fidelity gate (build → snapshot → token scan → a11y → screenshot → mockup compare → reduced motion) → spec compliance review → code quality review → finisher pass (per `finishers: [...]`) → atomic commit + `[x]`.

**Sequential only — never dispatch two `run-task-design-ios` calls in parallel.** iOS tasks often touch the same SwiftUI files; parallel execution creates merge conflicts.

**Idempotence (M1):** if a task is already `[x]`, `run-task-design-ios` no-ops. `--force` re-runs.

**Skip non-design tasks.** This loop iterates only `### Design tasks`. Feature tasks are the feature pipeline's responsibility.

After every `- [ ]` task in `### Design tasks` has flipped to `[x]`, proceed to Step 4.

See `skills/run-task-design-ios/SKILL.md` for the full per-task pipeline, including the BLOCKED escalations for thin design briefs.

### Step 4: Close the design half (hand-off)

Once every design task is committed and `[x]`-marked, hand off to `close-design-phase-ios`:

```
/arsenal-build:close-design-phase-ios N
```

That skill runs the design-pipeline wrap: visual phase audit (optional impeccable `audit` + `polish`, gateable — never automatic) → docs update (if scope drifted) → trim the design-task block in TASKS.md / annotate `.tasks/phase-N/` for the feature pipeline.

**`close-design-phase-ios` does NOT push and does NOT open a PR.** It returns the branch to the orchestrator. The feature pipeline's `close-feature-phase-ios` opens the single PR (and runs the snapshot test verification + Periphery + CodeRabbit gates).

After `close-design-phase-ios` returns DONE, this orchestrator reports completion and recommends: "Design half of phase N complete. Run `/arsenal-build:features-ios N` to wire the views to real data."

See `skills/close-design-phase-ios/SKILL.md` for the full gate specifications.

## Handling Edge Cases

**Mockup doesn't exist for a `design: yes` task:** the design-brief subagent runs the fallback shape interview, flags the missing mockup. Implementation proceeds with a thin brief — but if the brief is too thin to implement, the design-implementer BLOCKS for user resolution.

**Mockup ↔ DESIGN_SYSTEM disagree:** brief generation flags and pauses. User picks canonical.

**Snapshot test baseline drift:** never auto-accept. Prompt: "Update snapshot baseline for `<Test>`? Y/N." Commit baseline updates as `test(phase-N): update snapshot baseline for [primitive]`.

**Simulator state corruption:** prompt the user to reset. Don't silently restart.

**Build fails mid-phase:** capture `mcp__xcode__GetBuildLog`, dispatch debug subagent. Don't stack new tasks on a broken build.

**Task depends on something not yet built:** reorder the queue.

**User changes scope mid-phase:** pause, update `TASKS.md`, resume.

**Context window getting large:** checkpoint and suggest fresh session.

## Anti-Patterns — Never Do These

(Per-task disciplines moved to `skills/run-task-design-ios/SKILL.md`. Brief generation moved to `skills/generate-design-briefs/SKILL.md`. Wrap-gate disciplines moved to `skills/close-design-phase-ios/SKILL.md`. The disciplines below are orchestrator-level.)

- **Don't auto-dispatch `impeccable:shape` or `impeccable:craft`.** Finishers ARE auto-dispatched per the `finishers:` tag (at the per-task level, inside `run-task-design-ios`). Shape and craft are user-manual.
- **Don't push or open a PR from this skill.** That's `close-feature-phase-ios`'s terminus.
- **Don't iterate `### Feature tasks`.** This loop processes only `### Design tasks`.
- **Don't wire data.** Visual fidelity reviewer flags real data wiring as CRITICAL.
- **Don't boot simulators or open Xcode silently.** Step 1 preflight prompts the user.
- **Don't loop tasks in parallel.**
- **Don't read archived task lists speculatively.**
- **Don't drop feature specs or UX.md into controller context after Step 2.**

## Session Management

**Starting (fresh or resumed):**
- Confirm Xcode open + simulator booted.
- Read `TASKS.md`. Trimmed `✅ Completed` phases skip.
- First phase whose `### Design tasks` has unchecked items → continue execution loop here.
- If a phase's design half is done, route the user to `features-ios`.

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
| `/arsenal-build:anchor-files` | Creates the docs this skill reads. Must run before this skill. |
| `/arsenal-build:expand-phase` | **Invoked at Step 2 (first half)** as a sub-skill (shared with the feature orchestrator). |
| `/arsenal-build:generate-design-briefs` | **Invoked at Step 2 (second half)** as a sub-skill. Writes per-task context + design briefs (no `impeccable:shape` gate). |
| `/arsenal-build:run-task-design-ios` | **Invoked at Step 3** as a sub-skill, once per `- [ ]` task in `### Design tasks`. Runs researcher → design-implementer → visual fidelity gate → spec review → quality review → finisher pass → atomic commit + `[x]`. |
| `/arsenal-build:close-design-phase-ios` | **Invoked at Step 4** as a sub-skill. Runs design-pipeline wrap (optional impeccable audit + polish, gateable). No push, no PR. |
| `/arsenal-build:features-ios` | **Runs after this skill in every phase that has design tasks.** Sibling orchestrator. Wires the views this skill committed to real data. Shares the same phase branch. |
| `/arsenal-build:close-feature-phase-ios` | Runs after `features-ios`. Opens the single PR per phase, runs snapshot verify + Periphery + CodeRabbit. |
| `impeccable` (specifically finisher sub-skills: harden, animate, clarify, typeset, arrange) | Auto-dispatched at the per-task level inside `run-task-design-ios` (Step 8), per the `finishers:` tag. `impeccable:shape` and `impeccable:craft` are NOT auto-dispatched — user-manual only. `impeccable:audit` and `:polish` may run at `close-design-phase-ios` as a soft-gateable audit. |
| `mcp__xcode__*` MCP | Step 3 implementer + Step 5 visual gate (inside `run-task-design-ios`). |
| `mcp__ios-simulator__*` MCP | Same. |

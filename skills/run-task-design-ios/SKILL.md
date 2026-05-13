---
name: run-task-design-ios
description: Runs a single `domain: design` task from a native iOS project's `TASKS.md` phase through the full per-task pipeline — optional researcher (if `research: yes`), design-implementer dispatch (with simulator handshake + token preload), visual fidelity gate (build → snapshot → token scan → a11y → screenshot → mockup compare → reduced motion), spec compliance review, code quality review, finisher pass (per `finishers: [...]`), atomic commit, and `[x]` mark in `TASKS.md`. Does NOT auto-dispatch `impeccable:shape` or `impeccable:craft` — if the design brief is under-specified, the design implementer BLOCKS and the user may invoke impeccable manually before resuming. Finishers (harden, animate, clarify, typeset, arrange) ARE auto-dispatched per the task's `finishers: [...]` tag — they're the design pipeline's post-implementation polish, scoped to files this task modified. Views are built with hardcoded data; data wiring is the feature pipeline's job. Invoked by `design-ios` once per design task in a phase loop. Idempotent by default; `--force` re-runs an already-`[x]` task. Does NOT auto-fire on "begin work on phase N" phrasing — that's the orchestrator's territory.
---

# Run Task — Design, iOS

Execute one `domain: design` task from a native iOS project's `TASKS.md` phase through the per-task pipeline:

1. Optional researcher (when the task is tagged `research: yes`)
2. Design-implementer dispatch — with mandatory simulator handshake and token-source preload
3. Visual fidelity gate — build → snapshot test → token-discipline scan → a11y identifier check → screenshot → mockup compare → Reduced Motion check
4. Spec compliance review (fix-and-re-review loop)
5. Code quality review (fix-and-re-review loop; reads project `CLAUDE.md` § Hard rules and § Voice + tone)
6. Finisher pass (per tag in `finishers: [...]`, dispatch the matching impeccable sub-skill scoped to files this task modified)
7. Atomic commit + `[x]` in TASKS.md

The pipeline runs **sequentially per task** — never dispatch two design-implementers in parallel. The orchestrator loops this sub-skill per `- [ ]` task under the phase's `### Design tasks` subsection.

**Three-stage review catches three failure modes** (visual fidelity → spec → quality). Visual fidelity catches "shipped but doesn't match the mockup." Spec compliance catches "well-structured but wrong behavior" (especially async/await ordering and scenePhase concerns). Code quality catches "right behavior but sloppy code."

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

## Hardcoded data — the design pipeline's contract

Design tasks build SwiftUI views with **hardcoded / placeholder data** that demonstrate every state from the design brief's variant coverage. Real data wiring (`@Query`, services, `@Environment(\.modelContext)`) is the feature pipeline's job and runs later in the same phase.

The design-implementer prompt template enforces this: view parameters take literal values, `#Preview` macros render each state, no `@Query` / service injection / real APIs. If a view genuinely needs a derived value to render, the implementer hardcodes a representative value and leaves a `// TODO(feature-pipeline): derive from <source>` comment.

The component-boundary rule (enforced in the feature pipeline) protects this work: the feature pipeline may extend (new params/state/variants for data) but not redesign.

## Impeccable: shape and craft are user-manual; finishers are auto-dispatched

Two impeccable surfaces apply to this skill, with different rules:

- **`impeccable:shape <surface>` and `impeccable:craft <surface>` are NEVER auto-dispatched** from this skill. If the design brief is thin (sparse mockup, missing variant coverage, novel IA the brief couldn't resolve), the design-implementer BLOCKS. The user decides whether to invoke `impeccable:shape <surface>` manually, update the brief or DESIGN_SYSTEM, and then re-run this skill with `--force`.

- **Finishers ARE auto-dispatched** per the task's `finishers: [list]` tag (set by `expand-phase`). Finishers are impeccable sub-skills (`harden`, `animate`, `clarify`, `typeset`, `arrange`) that run as a post-implementation polish pass in Step 6, scoped to files this task modified. The mapping lives in `references/finishers-table.md`.

The distinction: shape/craft are design-judgement commands the user opts into; finishers are scoped polish passes the brief pre-commits to.

## Prerequisites

| Upstream artifact | Source | Used for |
|---|---|---|
| `docs/TASKS.md` with phase block in concrete-tasks state | `expand-phase` upstream | Read the task line and tags (`domain: design`, `research:`, `finishers:`, `snapshot:`); flip `[ ]` → `[x]` after success |
| `.tasks/phase-{N}/task-{N}-context.md` | `generate-design-briefs` upstream | Design-implementer reads it by path |
| `.tasks/phase-{N}/task-{N}-design.md` | `generate-design-briefs` upstream | Design-implementer + visual-fidelity reviewer read it; navigation script drives simulator nav |
| `docs/CONVENTIONS.md`, `docs/ARCHITECTURE.md`, `docs/DESIGN_SYSTEM.md` | `anchor-files` upstream | Excerpted into the briefs upstream — this skill does not re-read them in full |
| `docs/mockups/<screen>.{jsx,tsx,html,png,figma-export.json}` (when applicable) | Project | Visual fidelity gate compares the simulator screenshot to the mockup at the cited region |
| `CLAUDE.md` (root, optional) | Project | Code quality reviewer reads § Hard rules + § Voice + tone as additional review criteria |
| Phase branch checked out | Orchestrator Step 1 | All commits land on this branch |
| Xcode open + one simulator booted | Orchestrator Step 1 preflight | `mcp__xcode__*` and `mcp__ios-simulator__*` tools require this — this skill never boots/opens silently |
| `impeccable` skill installed | Orchestrator Step 1 preflight | Required for the finisher pass (Step 6) only. If absent and the task has a non-empty `finishers: [...]` list, the orchestrator should have caught this — but this skill rechecks before Step 6 and reports `NEEDS_CONTEXT` if impeccable is missing. |

If any upstream artifact is missing, report `NEEDS_CONTEXT` and surface which one. The design brief is required — don't attempt the implementer dispatch without it.

## Inputs

Caller passes:

- `--phase <N>` — phase number (required)
- `--task <N>` — task identifier within the phase (required)
- `--force` — optional. Re-run the full pipeline even when the task is already `[x]`.

The task's tags (`domain:`, `research:`, `finishers:`, `snapshot:`) are re-read from `docs/TASKS.md` directly. If the task's `domain:` is not `design`, report `WRONG_DOMAIN` — that task belongs to `run-task-feature-ios`.

## Workflow

### Step 1: Validate inputs and check idempotence

Read `docs/TASKS.md` for phase `<N>`. Locate the `### Design tasks` subsection. Confirm task `<N>` exists there.

- Task lives under `### Feature tasks` instead: `WRONG_DOMAIN`.
- Task is `[x]` and `--force` NOT set: `SKIPPED — task already complete`.
- Task is `[x]` and `--force` IS set: continue.
- Task is `[ ]`: continue.

Confirm both briefs exist:

- `.tasks/phase-{N}/task-{N}-context.md` — required
- `.tasks/phase-{N}/task-{N}-design.md` — required (design tasks always have a design brief)

If either is missing: `NEEDS_CONTEXT`; tell the caller to run `/arsenal-build:generate-design-briefs --phase N --task N --surface ios` first.

Confirm iOS tooling state: Xcode open, simulator booted. If either is missing, `NEEDS_CONTEXT` — do not auto-boot or auto-open.

### Step 2: Dispatch researcher subagent (if `research: yes`)

Ask the user: "Task N flagged for research (reason: [novel SwiftUI pattern / Family Controls authorization / SwiftData migration / etc.]). Run research first?" Default: yes.

If yes, dispatch using `references/researcher-prompt.md`. The researcher writes `.tasks/phase-{N}/task-{N}-research.md` and terminates. This skill does not read the contents.

### Step 3: Dispatch design-implementer subagent

Dispatch using `references/design-implementer-prompt.md`. Substitute every bracketed placeholder before dispatch.

The implementer reads its briefs (context + design + research if applicable) and the mockup at the cited region. The template carries the simulator handshake, token-source preload, hardcoded data discipline, hard rules (token discipline, a11y identifiers, concurrency, state, view structure), and the BLOCKED escalations for thin design briefs.

**Model selection:**
- Mechanical (atom-level view, clear translation plan, sibling pattern) → Sonnet/Haiku
- Integration (multi-state view, multiple variants, mockup ↔ DS reconciliation) → Sonnet
- Architectural (novel surface, sparse brief, broad DS implications) → Opus

### Step 4: Handle implementer response

| Status | Action |
|--------|--------|
| **DONE** | Proceed to visual fidelity gate (Step 5). |
| **DONE_WITH_CONCERNS** | Read concerns; address correctness/scope; note observations. |
| **NEEDS_CONTEXT** | Provide missing context; re-dispatch. |
| **BLOCKED** | Assess and recommend, then wait for user direction: (1) Missing context → provide it. (2) Needs more reasoning → upgrade model. (3) Task too large → recommend `expand-phase --force` to split. (4) Plan is wrong → escalate. (5) **Design brief under-specified** → escalate; recommend the user invoke `impeccable:shape <surface>` manually (or update the brief / DESIGN_SYSTEM by hand) and re-run `/arsenal-build:generate-design-briefs --phase N --task N --surface ios --force`, then resume this task with `--force`. **Never auto-dispatch impeccable; never force the same model to retry without changes.** |

### Step 5: Visual fidelity gate

Dispatch using `references/visual-fidelity-reviewer-prompt.md`. The subagent runs in fixed order, stopping on first hard fail:

1. **Build** — `mcp__xcode__BuildProject` against the simulator destination
2. **Snapshot test** (if `snapshot: yes`) — `mcp__xcode__RunSomeTests` filtered to the snapshot test the implementer added
3. **Token-discipline scan** — grep modified files for `#[0-9A-Fa-f]{6}` patterns outside the theme module; any match in feature code is a TOKEN_VIOLATION
4. **Accessibility identifiers** — `mcp__ios-simulator__ui_describe_all` after navigating to the implemented screen; check coverage and uniqueness
5. **Screenshot capture** — `.tasks/phase-{N}/task-{N}-after.png`
6. **Mockup comparison + regression check** — compare against the mockup cited region. If a before-screenshot exists, also check the change didn't shift adjacent surfaces
7. **Reduced Motion verification** (if the design brief specifies motion) — toggle Reduce Motion ON, re-run, confirm fallback honored

Result categories: PASS / BUILD_FAILED / SNAPSHOT_FAIL / SNAPSHOT_MISSING / TOKEN_VIOLATION / A11Y_MISSING / A11Y_DUPLICATE / DRIFT / GAP / REGRESSION / REDUCED_MOTION_FAIL.

On any failure: dispatch fix subagent (design-implementer template) with the specific findings, then re-run the gate from stage 1. No "close enough."

### Step 6: Dispatch spec compliance reviewer

Dispatch using `references/spec-reviewer-prompt.md`. Job: did the code match the spec? Code quality is the next stage's concern.

**For iOS specifically:** the reviewer pays attention to async/await ordering. A spec like "write before animate" can be silently violated by `Task { ... }` ordering or actor isolation — trace the actual call path.

**If issues found:** dispatch fix subagent, re-review until approved.

### Step 7: Dispatch code quality reviewer

Only after spec compliance passes. Dispatch using `references/quality-reviewer-prompt.md`. Reads project `CLAUDE.md` (if present) and applies § Hard rules + § Voice + tone.

**Hard-fail criteria (CRITICAL):** hex literal in feature code (outside theme module), `Any` or `as!` in new code, Combine in new code, missing `@MainActor` on UI mutations, force-quit-resistant write order violated, memory cycle in escaping closures.

**IMPORTANT severity:** view body > 50 lines, `ObservableObject` in new code, `.onAppear { Task { } }`, missing accessibility identifier, hardcoded numeric literals, mocked `@Model` types in tests, missing `final` on `@Model`, dead code introduced by this task.

**MINOR severity** can be deferred to the per-task report.

**If CRITICAL or IMPORTANT issues:** dispatch a fix subagent, re-review. MINOR can be deferred.

### Step 8: Finisher pass (per tag in `finishers: [...]`)

Per tag in the task's `finishers: [...]` array, dispatch a finisher subagent using the matching impeccable skill. The mapping lives in `references/finishers-table.md`. Scope: only files the implementer modified for this task — finishers never touch sibling tasks' files.

**Default order when multiple finishers apply:** harden → animate → typeset → arrange → clarify. Harden first because correctness invariants matter more than aesthetics; clarify last because copy is the easiest to iterate on.

**Sequential, not parallel** — finishers modify the same files; running them in parallel creates conflicts.

Common iOS finishers: `harden` (force-quit resistance, error states), `animate` (Reduced Motion gating, transition timing), `clarify` (UX copy, microcopy), `typeset` (typography), `arrange` (spacing/rhythm).

Skipped by default: `adapt` (single device family). Surface-by-surface skills (`colorize`, `bolder`, `quieter`, `delight`, `overdrive`) are hand-flag only.

**A finisher reporting "no changes needed" is a valid result and should be commit-skipped** — don't create empty commits.

**If a finisher proposes changes that would touch a primitive in the project's theme module,** stop and ask the user. Theme changes need broader thought (snapshot test impact, brand spec alignment) than a per-task finisher should make alone.

Each finisher that does propose changes commits as `polish(phase-N): [skill-name] pass on T-N`.

### Step 9: Complete the task

- Ensure all commits from the implementer + reviewers + finishers landed with semantic messages.
- Mark the task as complete in `TASKS.md`: change `- [ ]` to `- [x]` under `### Design tasks`.
- Report DONE to the caller.

### Step 10: Return to caller

Report:

- **Status:** DONE | SKIPPED (M1 idempotence) | WRONG_DOMAIN (route to `run-task-feature-ios`) | NEEDS_CONTEXT (cite which brief / tool / simulator state is missing) | BLOCKED (cite reason, including "design brief under-specified" cases that the user may resolve via manual impeccable invocation)
- **Phase:** N
- **Task:** N
- **Researcher ran:** yes | no
- **Implementer model:** Sonnet | Haiku | Opus
- **Visual fidelity:** PASS (cite which stages ran)
- **Spec review:** passed | passed after N fix cycles
- **Quality review:** passed | passed after N fix cycles
- **Finishers run:** comma list with each one's result ("harden: changes committed", "animate: no changes needed")
- **Deferred minor issues:** list (aggregate into the phase summary at `close-design-phase-ios`)
- **Commits added:** list of semantic messages
- **Screenshots:** `task-{N}-before.png` (if captured), `task-{N}-after.png`
- **View path:** the canonical path where the new view lives
- **TASKS.md:** task N flipped to `[x]` under `### Design tasks`

## Reference files

| File | Used in | Purpose |
|------|---------|---------|
| `references/researcher-prompt.md` | Step 2 | Researcher — Apple HIG, SwiftUI 18 patterns, library API drift |
| `references/design-implementer-prompt.md` | Step 3 (and fix dispatches in Steps 5/6/7) | Design-implementer — carries simulator handshake, token preload, hardcoded data discipline, hard rules, BLOCKED reasons for thin briefs |
| `references/visual-fidelity-reviewer-prompt.md` | Step 5 | Visual gate — runs on simulator, compares to mockup, validates token discipline, a11y, Reduced Motion |
| `references/spec-reviewer-prompt.md` | Step 6 | Spec compliance reviewer — async/await ordering, behavioral correctness |
| `references/quality-reviewer-prompt.md` | Step 7 | Code quality reviewer — Swift/SwiftUI specific; reads project `CLAUDE.md` |
| `references/finishers-table.md` | Step 8 | Maps `finishers:` task tags → impeccable sub-skills (harden, animate, clarify, etc.) |

Each template has bracketed placeholders. Substitute every one before dispatch.

## Anti-patterns — never do these

- **Don't auto-dispatch `impeccable:shape` or `impeccable:craft`.** Finishers (harden/animate/clarify/etc.) ARE auto-dispatched per the `finishers:` tag. Shape and craft are user-manual.
- **Don't let subagents inherit this skill's session context.** They get a system prompt + the explicit files they're told to read.
- **Don't paste full canonical docs into subagent prompts.** Reference briefs by path.
- **Don't wire real data.** Design tasks use hardcoded / placeholder values. Importing `@Query`, services, or `@Environment(\.modelContext)` beyond demo-state toggles is a CRITICAL visual fidelity failure — that belongs to the feature pipeline.
- **Don't skip any of the three review stages.** Visual fidelity catches "doesn't match mockup", spec catches "well-structured but wrong", quality catches "right but sloppy."
- **Don't ship hex literals outside the project's theme module.** Quality reviewer + visual fidelity gate both fail on this (CRITICAL).
- **Don't use coordinate-based simulator taps.** Always use `ui_find_element` against an accessibility identifier from the design brief.
- **Don't auto-accept snapshot test baselines.** Drift is a signal. If a baseline change is intentional, prompt the user explicitly.
- **Don't boot simulators or open Xcode silently.**
- **Don't use `xcodebuild` via Bash when `mcp__xcode__*` is available.**
- **Don't dispatch multiple implementers (or finishers) in parallel.**
- **Don't ignore `BLOCKED` or `NEEDS_CONTEXT`.**
- **Don't run finishers across sibling tasks' files.** Each finisher is scoped to the files modified by THIS task.
- **Don't auto-update a theme-module primitive from a finisher pass.** Surface the proposed change to the user and pause.
- **Don't re-run a task that's already `[x]` without `--force`.**

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:design-ios` | Invokes this skill in a loop, one call per `- [ ]` task in the phase's `### Design tasks` subsection. Hand-off: phase number + task identifier. |
| `/arsenal-build:expand-phase` | Runs **before** this skill (via the orchestrator). Writes the concrete tagged task list (including iOS-specific `finishers:` and `snapshot:` tags on design-domain tasks). |
| `/arsenal-build:generate-design-briefs` | Runs **before** this skill (via the orchestrator). Writes the briefs this skill's subagents read by path. |
| `/arsenal-build:close-design-phase-ios` | Runs **after** all design tasks in the phase complete. Aggregates deferred Minor issues this skill reported per task; runs the optional `impeccable:audit` + `impeccable:polish` gate, then hands the branch off to the feature pipeline. |
| `/arsenal-build:run-task-feature-ios` | Sibling skill. Runs **after** the entire design half of the phase completes. References the views this skill committed. |
| `impeccable` (and per-finisher sub-skills) | Dispatched in Step 8 per `finishers:` tag. `impeccable:shape` and `impeccable:craft` are NOT auto-dispatched — those are user-manual escape hatches when a brief is thin. |
| `mcp__xcode__*` MCP | Step 3 implementer + Step 5 visual gate use Build, RunSomeTests, RenderPreview, GetBuildLog. |
| `mcp__ios-simulator__*` MCP | Step 3 implementer (pre-change screenshot) + Step 5 visual gate use `get_booted_sim_id`, `launch_app`, `ui_describe_all`, `screenshot`, `ui_find_element`, `ui_tap`. |

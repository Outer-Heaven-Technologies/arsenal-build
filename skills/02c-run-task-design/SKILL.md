---
name: run-task-design
description: Runs a single `domain: design` task from a web/frontend project's `TASKS.md` phase through the full per-task pipeline — optional researcher (if `research: yes`), design-implementer dispatch (single path), visual fidelity review (static analysis — design-system adherence, state coverage, mockup ↔ code match; no live-browser gate), code quality review, atomic commit, and `[x]` mark in `TASKS.md`. Does NOT auto-dispatch impeccable. If the design brief is under-specified, the design implementer BLOCKS and the user may invoke impeccable manually before resuming. Components are built with hardcoded / placeholder data; data wiring is the feature pipeline's job. Invoked by `design` once per design task in a phase loop. Idempotent by default; `--force` re-runs an already-`[x]` task. Does NOT auto-fire on "begin work on phase N" phrasing — that's the orchestrator's territory.
---

# Run Task — Design, Web

Execute one `domain: design` task from a web/frontend project's `TASKS.md` phase through the per-task pipeline:

1. Optional researcher (when the task is tagged `research: yes`)
2. Design-implementer dispatch (single path)
3. Handle implementer response (DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED)
4. Visual fidelity review (fix-and-re-review loop; static analysis — token map adherence, state coverage, mockup ↔ code match)
5. Code quality review (fix-and-re-review loop; Critical/Important must fix; Minor can defer)
6. Atomic commit + `[x]` in TASKS.md

The pipeline runs **sequentially per task** — never dispatch two design-implementers in parallel. The orchestrator loops this sub-skill per `- [ ]` task under the phase's `### Design tasks` subsection; this sub-skill never iterates across multiple tasks itself.

## Paths

All arsenal artifacts live under `.arsenal/` at the project root.

| What | Path | Notes |
|---|---|---|
| Strategy archive (denied during build) | `.arsenal/strategy/` | MARKET_RESEARCH.md, RESEARCH_PLAN.md, MVP_SPEC.md, mockup-briefs/, GTM_STRATEGY.md, REVENUE_MODEL.md |
| Feature specs | `.arsenal/FEATURES.md` (single-mode) or `.arsenal/features/<slug>.md` (split-mode) | Gated per phase via `.claude/settings.json` |
| Project anchor docs | `.arsenal/{ARCHITECTURE,CONVENTIONS,TASKS}.md` | Always readable during build |
| Design reference set | `.arsenal/design/{UX,DESIGN,DESIGN_SYSTEM}.md` + `.arsenal/design/mockups/` | Always readable during build |
| Per-task briefs + ephemera | `.arsenal/tasks/phase-N/`, `.arsenal/tasks/parallel/`, `.arsenal/tasks/archive/` | Gitignored; phase-N gated per active phase |

**Configuration:** `.arsenal/config.yaml` may override the root location, but defaults work for nearly all projects. File names are not configurable.

**Gating:** `expand-phase` writes baseline denies and per-phase allow rules to `.claude/settings.json`. `close-feature-phase` reverts at phase end. Strategy stays fully denied throughout build.

## Two-stage review for the design pipeline

The design pipeline preserves the two-stage review invariant, with the first stage retuned to design concerns:

- **Stage 1 — Visual fidelity review** replaces spec compliance as the first stage. Reviews against the design brief (which is the spec for a design task): token map adherence, locked-primitive citation, state coverage from the variant matrix, mockup ↔ code match, gap detection.
- **Stage 2 — Code quality review** unchanged: readability, naming, prop typing, file structure, accessibility patterns, no premature abstraction.

Visual fidelity is **static analysis** on web — the reviewer reads the design brief + the implemented code + the mockup and checks adherence without a live browser. (Cross-surface design checks are deferred to `close-design-phase`'s optional impeccable audit, which the user opts into at phase end.)

## Hardcoded data — the design pipeline's contract

Design tasks build visual components with **hardcoded / placeholder data**. Real data wiring (queries, mutations, server actions, state management) is the feature pipeline's job and runs later in the same phase.

Specifically:

- A component built by this skill takes props with realistic but **literal** values (e.g., `name="Sample User"`, `count={3}`, `status="active"`).
- States from the design brief (loading, empty, error, success, hover/focus/active) are all rendered, typically via a Storybook-style mechanism or top-level prop toggles.
- The implementer **does not** import data-fetching utilities, server actions, or stateful hooks beyond what's needed to demo states.
- If a component genuinely needs data to render (e.g., a derived value), the implementer hardcodes a representative value and notes it in the commit message.

The feature pipeline later wires these components to real data, extending the component's props or adding new variants as the data layer requires. The component-boundary rule (extend not redesign) protects the design work this skill commits.

## Impeccable is optional and manual — never automatic

This skill does **not** dispatch `impeccable:shape`, `impeccable:craft`, or any other impeccable command. If the design brief is thin (sparse mockup, missing variant coverage, novel IA the brief couldn't resolve), the design-implementer BLOCKS with `reason: design brief under-specified on <X>` and waits for user direction.

The user decides whether to invoke `impeccable:shape <surface>` (or any other impeccable command) manually, update the brief or DESIGN_SYSTEM, and then re-run this skill with `--force`. This skill never makes that call automatically.

## Prerequisites

This sub-skill assumes its upstream contracts are satisfied:

| Upstream artifact | Source | Used for |
|---|---|---|
| `.arsenal/TASKS.md` with phase block in concrete-tasks state | `expand-phase` upstream | Read the task line and tags (`domain: design`, `research:`); flip `[ ]` → `[x]` after success |
| `.arsenal/tasks/phase-{N}/task-{N}-context.md` | `generate-design-briefs` upstream | Design-implementer reads it by path |
| `.arsenal/tasks/phase-{N}/task-{N}-design.md` | `generate-design-briefs` upstream | Design-implementer + visual-fidelity reviewer read it; the brief carries the token map, variant matrix, and mockup pointer |
| `.arsenal/CONVENTIONS.md`, `.arsenal/ARCHITECTURE.md`, `.arsenal/design/DESIGN_SYSTEM.md`, `.arsenal/design/DESIGN.md` | `anchor-files` upstream | Excerpted into the briefs upstream — this skill does not re-read them in full |
| `.arsenal/design/mockups/<screen>.{jsx,tsx,html,png,figma-export.json}` (when applicable) | Project | Visual fidelity reviewer compares the implemented code to the mockup at the cited region |
| Phase branch checked out (`phase-N/short-description`) | Orchestrator Step 1 | All commits land on this branch |

If any upstream artifact is missing, report `NEEDS_CONTEXT` and surface which one. The design brief is required for design tasks — don't attempt the implementer dispatch without it.

`impeccable` is **not** a prerequisite. The design pipeline never auto-invokes it.

## Inputs

Caller passes:

- `--phase <N>` — phase number (required)
- `--task <N>` — task identifier within the phase (required)
- `--force` — optional. Re-run the full pipeline even when the task is already `[x]` in `TASKS.md`. Without `--force`, this skill is idempotent per the M1 contract: `[x]` tasks no-op pass-through.

The task's tags (`domain:`, `research:`) are re-read from `.arsenal/TASKS.md` directly so direct invocation works without orchestrator state. If the task's `domain:` is not `design`, report `WRONG_DOMAIN` and stop — that task belongs to `run-task-feature`.

## Workflow

### Step 1: Validate inputs and check idempotence

Read `.arsenal/TASKS.md` for phase `<N>`. Locate the `### Design tasks` subsection. Confirm task `<N>` exists in the subsection.

- If the task is not in the `### Design tasks` subsection: report `WRONG_DOMAIN` or `NEEDS_CONTEXT` accordingly.
- If the task is `[x]` and `--force` is NOT set: report `SKIPPED — task already complete`. No-op.
- If the task is `[x]` and `--force` IS set: continue (will re-run the full pipeline).
- If the task is `[ ]`: continue.

Parse the task's tags (`domain: design`, `research: yes/no`). Confirm both briefs exist:

- `.arsenal/tasks/phase-{N}/task-{N}-context.md` — required
- `.arsenal/tasks/phase-{N}/task-{N}-design.md` — required (design tasks always have a design brief)

If either brief is missing, report `NEEDS_CONTEXT` and tell the caller to run `/arsenal-build:generate-design-briefs --phase N --task N` first.

### Step 2: Dispatch researcher subagent (if `research: yes`)

Ask the user: "Task N is flagged for research (reason: [novel CSS pattern / accessibility concern / animation library API / etc., from the task's research-flag reason]). Run research pass first, or skip?" Default suggestion: yes for flagged tasks.

If yes, spawn a fresh-context researcher subagent using the prompt template at `references/researcher-prompt.md`. Substitute the bracketed placeholders for this task's specifics. The researcher writes `.arsenal/tasks/phase-{N}/task-{N}-research.md` and terminates. This skill does not read the contents.

### Step 3: Dispatch design-implementer subagent

Spawn a design-implementer subagent using the prompt template at `references/design-implementer-prompt.md`. Substitute every bracketed placeholder before dispatch.

The implementer reads its briefs (context + design + research if applicable) and the mockup at the cited region. It does NOT read full canonical docs. The design brief is the authoritative source of truth for the surface — the implementer translates it into code with hardcoded data and the states listed in the brief's variant coverage.

**Model selection:**

- Mechanical tasks (1-2 files, clear translation plan, no novel design moves) → fast/cheap model (Sonnet or Haiku)
- Integration tasks (multi-state component, multiple variants, mockup ↔ DS reconciliation) → standard model (Sonnet)
- Architectural tasks (novel surface, sparse design brief, broad design-system implications) → most capable model (Opus)

Default to cheaper models when the design brief is well-specified; upgrade when needed.

### Step 4: Handle implementer response

| Status | Action |
|--------|--------|
| **DONE** | Proceed to visual fidelity review (Step 5). |
| **DONE_WITH_CONCERNS** | Read the concerns. If they're about correctness or scope, address before review. If they're observations, note them and proceed. |
| **NEEDS_CONTEXT** | Provide the missing context if available; otherwise surface the gap to the user and pause. Re-dispatch with additional context. |
| **BLOCKED** | Assess and recommend one of five responses, then wait for user direction: (1) Missing context → provide it, re-dispatch same model. (2) Needs more reasoning → upgrade model. (3) Task too large → recommend `/arsenal-build:expand-phase --phase N --force` to split. (4) Plan is wrong → escalate. (5) **Design brief under-specified** → escalate; recommend the user invoke `impeccable:shape <surface>` manually (or update the brief / DESIGN_SYSTEM by hand) and re-run `/arsenal-build:generate-design-briefs --phase N --task N --force`, then resume this task with `--force`. **Never auto-dispatch impeccable; never force the same model to retry without changes.** |

### Step 5: Dispatch visual fidelity reviewer

Spawn a reviewer subagent using the prompt template at `references/visual-fidelity-reviewer-prompt.md`. The job is binary: did the code match the design brief?

The reviewer checks:

- **Token map adherence** — every value in the brief's token map appears as the cited token / class in the code. No raw hex literals outside the project's theme module (path declared in DESIGN_SYSTEM.md).
- **Locked-primitive citation** — primitives the brief cited as locked (e.g., "Button § 3.2") are imported and used, not rebuilt inline.
- **State coverage** — every cell the brief listed under "Cells the mockup locks" and "Cells the implementer derives from DESIGN_SYSTEM" is rendered, typically as a state toggle or Storybook story.
- **Mockup ↔ code match** — for surfaces with a mockup, the implemented code matches the cited region (layout, copy, typography, spacing). The reviewer reads the mockup file directly.
- **Net-new design** — anything in the brief's "Net-new design" section is implemented as described (with the variant-matrix cells derived from DESIGN_SYSTEM where applicable).
- **Hardcoded data discipline** — the implementer used hardcoded values, not data-fetching utilities. If the implementer reached for real data wiring, that's a CRITICAL fidelity failure (data wiring belongs to the feature pipeline).

**If the visual fidelity reviewer finds issues:**

- Dispatch a fix subagent (use the design-implementer template at `references/design-implementer-prompt.md` with the specific issues as the task body).
- Visual fidelity reviewer re-reviews.
- Repeat until approved — no "close enough."

### Step 6: Dispatch code quality reviewer

Only after visual fidelity passes. Spawn a reviewer subagent using the prompt template at `references/quality-reviewer-prompt.md`.

The quality reviewer covers code structure concerns: prop typing, file organization, naming, accessibility patterns (semantic HTML, ARIA, keyboard navigation), error handling for the hardcoded states (e.g., an Error state renders gracefully even with a missing prop), no premature abstraction, no over-engineering.

**If the quality reviewer finds issues:**

- Dispatch a fix subagent (design-implementer template) with the specific issues.
- Quality reviewer re-reviews.
- Critical and Important must be fixed. Minor can be noted and deferred (record in the per-task report).

### Step 7: Complete the task

Once both reviews pass:

- Ensure all changes are committed with semantic messages: `feat(phase-N): [task summary]` or the appropriate type. The implementer makes these commits during its dispatch; this skill verifies the working tree is clean before flipping the checkbox.
- Mark the task as complete in `TASKS.md`: change `- [ ]` to `- [x]` for this task (under the `### Design tasks` subsection).
- Report DONE to the caller.

### Step 8: Return to caller

Report:

- **Status:** DONE | SKIPPED (M1 idempotence) | WRONG_DOMAIN (task is `domain: feature`, route to `run-task-feature`) | NEEDS_CONTEXT (cite which brief is missing) | BLOCKED (cite the implementer's BLOCKED reason and your assessment, including "design brief under-specified" cases that the user may resolve via manual impeccable invocation)
- **Phase:** N
- **Task:** N
- **Researcher ran:** yes | no
- **Implementer model:** Sonnet | Haiku | Opus
- **Visual fidelity review:** passed | passed after N fix cycles
- **Quality review:** passed | passed after N fix cycles
- **Deferred minor issues:** list (these aggregate into the phase summary at `close-design-phase`)
- **Commits added:** list of semantic messages
- **Component created at:** path declared in the context brief / DESIGN_SYSTEM
- **TASKS.md:** task N flipped to `[x]` under `### Design tasks`

## Reference files

| File | Used in | Purpose |
|------|---------|---------|
| `references/researcher-prompt.md` | Step 2 | Researcher subagent template (when task is `research: yes`) |
| `references/design-implementer-prompt.md` | Step 3 (and fix dispatches in Steps 5/6) | Design-implementer subagent template — translates the design brief into a component with hardcoded data + state coverage |
| `references/visual-fidelity-reviewer-prompt.md` | Step 5 | Visual fidelity reviewer template — static analysis against the design brief and mockup |
| `references/quality-reviewer-prompt.md` | Step 6 | Code quality reviewer template |

Each template has bracketed placeholders. Substitute every one before dispatch — never ship a template with stubs intact.

## Anti-patterns — never do these

- **Don't auto-dispatch impeccable.** This skill never invokes `impeccable:shape`, `impeccable:craft`, or any other impeccable command. If the design brief is thin, BLOCK and let the user decide whether to invoke impeccable manually.
- **Don't let subagents inherit this skill's session context.** Subagents read briefs by path — that's the only handoff.
- **Don't paste entire docs into subagent prompts.** Briefs at `.arsenal/tasks/phase-{N}/task-{N}-*.md` were generated upstream. Reference them by path.
- **Don't wire real data.** Design tasks use hardcoded / placeholder values. Importing data-fetching utilities, server actions, or stateful hooks beyond demo-state toggles is a CRITICAL visual fidelity failure — that belongs to the feature pipeline.
- **Don't skip the visual fidelity stage.** It's the design pipeline's spec-compliance equivalent. Skipping it collapses the two-stage review.
- **Don't accept "close enough" on visual fidelity.** Token map, state coverage, and mockup match are all binary checks. Issues get fixed and re-reviewed.
- **Don't ship hex literals outside the project's theme module.** Visual fidelity reviewer fails on this (CRITICAL). Add a token to the theme module (or flag the gap to the user) — never inline a raw hex.
- **Don't dispatch multiple design-implementers in parallel.** They'll create merge conflicts. Sequential execution is deliberate.
- **Don't ignore BLOCKED or NEEDS_CONTEXT.** Something needs to change — more context, a richer design brief (possibly via manual impeccable), a different model, a smaller task, or human input.
- **Don't re-run a task that's already `[x]` without `--force`.** No-op pass-through with a clear report.

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:design` | Invokes this skill in a loop, one call per `- [ ]` task in the phase's `### Design tasks` subsection. Hand-off: phase number + task identifier. |
| `/arsenal-build:expand-phase` | Runs **before** this skill (via the orchestrator). Writes the concrete tagged task list this skill reads from `TASKS.md`. |
| `/arsenal-build:generate-design-briefs` | Runs **before** this skill (via the orchestrator). Writes the context + design briefs this skill's subagents read by path. |
| `/arsenal-build:close-design-phase` | Runs **after** all design tasks in the phase complete. Aggregates deferred Minor issues, runs the optional `claude-in-chrome` live audit + optional impeccable audit, then hands the branch off to the feature pipeline (no push, no PR — feature close opens the single PR per phase). |
| `/arsenal-build:run-task-feature` | Sibling skill. Runs **after** the entire design half of the phase (including `close-design-phase`) completes. References the components this skill committed. |
| `impeccable` | **Not invoked from this skill.** Available to the user as a manual escape hatch when the design brief is thin — user invokes manually, this skill resumes after the user updates the brief and reruns with `--force`. |

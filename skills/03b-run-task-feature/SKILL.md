---
name: run-task-feature
description: Runs a single `domain: feature` task from a web/frontend project's `TASKS.md` phase through the full per-task pipeline — optional researcher (if `research: yes`), feature-implementer dispatch (single path, no design judgement), spec compliance review (enforces the `Component extended:` commit-message rule), code quality review, atomic commit, and `[x]` mark in `TASKS.md`. Does NOT call impeccable, ever. The implementer treats design-pipeline-committed components under the project's component root as read-only design surfaces that may be **extended** (new props, variants, or states the data layer needs) but never **redesigned** (visual treatment, spacing, typography, color). Invoked by `features` once per feature task in a phase loop, after `close-design-phase` has committed the phase's visual components. Idempotent by default; `--force` re-runs an already-`[x]` task. Does NOT auto-fire on "begin work on phase N" phrasing — that's the orchestrator's territory.
---

# Run Task — Feature, Web

Execute one `domain: feature` task from a web/frontend project's `TASKS.md` phase through the per-task pipeline:

1. Optional researcher (when the task is tagged `research: yes`)
2. Feature-implementer dispatch (single path — no `impeccable:craft` routing, no design judgement)
3. Handle implementer response (DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED)
4. Spec compliance review (fix-and-re-review loop; enforces the `Component extended:` commit-message rule)
5. Code quality review (fix-and-re-review loop; Critical/Important must fix; Minor can defer)
6. Atomic commit + `[x]` in TASKS.md

The pipeline runs **sequentially per task** — never dispatch two feature-implementers in parallel. The orchestrator loops this sub-skill per `- [ ]` task under the phase's `### Feature tasks` subsection; this sub-skill never iterates across multiple tasks itself.

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

## The component boundary — load-bearing rule

The feature pipeline wires existing components (committed by the design pipeline earlier in the same phase) to real data. It does not do design work.

Specifically:

- **Components committed under the project's component root** (typically `components/ui/` and `components/shell/` for web — the exact path is declared by `DESIGN_SYSTEM.md`) are **read-only design surfaces** to the feature pipeline.
- **Extensions are allowed** — new props, new variants, or new states required by the data layer (e.g., a loading state for async data, a new prop for the entity ID, a disabled state for permission gating).
- **Visual changes are NOT allowed** — visual treatment, spacing, typography, color, or motion changes to an existing component are blocked. Those go through the design pipeline as a follow-up phase or a `--force` re-expansion.
- **Any commit that modifies a file under the project's component root must include a `Component extended: <path> — <one-line why>` note in the commit message body.** The spec reviewer enforces this at Step 5.
- **Purely visual changes BLOCK** — if the implementer needs to change a component's visual treatment to complete the task, it reports `BLOCKED` with `reason: purely visual change required — redirect to design pipeline` and waits for user direction.

The feature-implementer prompt template carries this rule verbatim and the spec reviewer enforces it. See `references/feature-implementer-prompt.md` and `references/spec-reviewer-prompt.md`.

## Prerequisites

This sub-skill assumes its upstream contracts are satisfied:

| Upstream artifact | Source | Used for |
|---|---|---|
| `.arsenal/TASKS.md` with phase block in concrete-tasks state | `expand-phase` upstream | Read the task line and tags (`domain: feature`, `research:`); flip `[ ]` → `[x]` after success |
| `.arsenal/tasks/phase-{N}/task-{N}-context.md` | `generate-feature-briefs` upstream | Feature-implementer reads it by path; carries available-components manifest |
| `.arsenal/CONVENTIONS.md`, `.arsenal/ARCHITECTURE.md`, `.arsenal/design/DESIGN_SYSTEM.md` | `anchor-files` upstream | Excerpted into the context brief upstream — this skill does not re-read them |
| Phase branch checked out (`phase-N/short-description`) | Orchestrator Step 1 | All commits land on this branch; design-pipeline commits are already present |
| Design-pipeline commits already on the phase branch | `close-design-phase` upstream | The components this task may wire to are reachable in git history; the context brief's available-components manifest enumerates them |

If any upstream artifact is missing, report `NEEDS_CONTEXT` and surface which one. Don't fabricate briefs or guess what the upstream skill would have produced.

`impeccable` is **not** a prerequisite. The feature pipeline never invokes it.

## Inputs

Caller passes:

- `--phase <N>` — phase number (required)
- `--task <N>` — task identifier within the phase (required)
- `--force` — optional. Re-run the full pipeline even when the task is already `[x]` in `TASKS.md`. Without `--force`, this skill is idempotent per the M1 contract: `[x]` tasks no-op pass-through.

The task's tags (`domain:`, `research:`) are re-read from `.arsenal/TASKS.md` directly so direct invocation works without orchestrator state. If the task's `domain:` is not `feature`, report `WRONG_DOMAIN` and stop — that task belongs to `run-task-design`.

## Workflow

### Step 1: Validate inputs and check idempotence

Read `.arsenal/TASKS.md` for phase `<N>`. Locate the `### Feature tasks` subsection. Confirm task `<N>` exists in the subsection.

- If the task is not in the `### Feature tasks` subsection (it lives under `### Design tasks` or isn't present at all): report `WRONG_DOMAIN` or `NEEDS_CONTEXT` accordingly.
- If the task is `[x]` and `--force` is NOT set: report `SKIPPED — task already complete`. No-op.
- If the task is `[x]` and `--force` IS set: continue (will re-run the full pipeline; any new commits will stack on top of the original).
- If the task is `[ ]`: continue.

Parse the task's tags (`domain: feature`, `research: yes/no`). Confirm the context brief exists:

- `.arsenal/tasks/phase-{N}/task-{N}-context.md` — required, always

If the brief is missing, report `NEEDS_CONTEXT` and tell the caller to run `/arsenal-build:generate-feature-briefs --phase N --task N` first.

### Step 2: Dispatch researcher subagent (if `research: yes`)

Ask the user: "Task N is flagged for research (reason: [Stripe webhook semantics / Next.js App Router pattern / etc., from the task's research-flag reason]). Run research pass first, or skip?" Default suggestion: yes for flagged tasks.

If yes, spawn a fresh-context researcher subagent using the prompt template at `references/researcher-prompt.md`. Substitute the bracketed placeholders for this task's specifics.

The researcher writes `.arsenal/tasks/phase-{N}/task-{N}-research.md` and terminates. **This skill does not read the contents of this file** — it only confirms the file was written. The feature-implementer subagent reads it via the brief reference.

**Research isolation rules (enforce strictly):**

- Research files live only at `.arsenal/tasks/phase-{N}/task-{N}-research.md` — never shared paths.
- Subagents read only their own task's research file, by exact path (no globs, no "read everything in the phase directory").
- This skill doesn't accumulate research file contents in its own context.
- Research is per-task, not cached across tasks.

### Step 3: Dispatch feature-implementer subagent

Spawn a feature-implementer subagent using the prompt template at `references/feature-implementer-prompt.md`. Substitute every bracketed placeholder before dispatch — never ship a template with stubs intact.

The implementer reads its briefs (context + research if applicable) by path. **Never paste entire canonical docs into the prompt.** The context brief budget is ≤3k tokens; if you find yourself wanting to include more, the task is too big — recommend running `/arsenal-build:expand-phase --phase N --force` to split it before continuing.

The feature-implementer prompt template carries the component-boundary rule (extend not redesign; `Component extended:` note required; BLOCKED on visual changes) and reminds the implementer to consult the context brief's `## Available components` section before touching any component file.

**Model selection:**

- Mechanical tasks (1-2 files, clear spec) → fast/cheap model (Sonnet or Haiku)
- Integration tasks (multi-file, pattern matching across the codebase) → standard model (Sonnet)
- Architectural tasks (design judgment, broad understanding) → most capable model (Opus)

Default to cheaper models when the plan is well-specified; upgrade when needed.

### Step 4: Handle implementer response

| Status | Action |
|--------|--------|
| **DONE** | Proceed to spec review (Step 5). |
| **DONE_WITH_CONCERNS** | Read the concerns. If they're about correctness or scope, address before review. If they're observations (e.g., "this file is getting large"), note them in the final report and proceed to review. |
| **NEEDS_CONTEXT** | Provide the missing context if you have it. If you don't, surface the gap to the user and pause. Re-dispatch the subagent with the additional context. |
| **BLOCKED** | Assess and recommend one of five responses, then wait for user direction: (1) Missing context → provide it, re-dispatch same model. (2) Needs more reasoning → upgrade model and re-dispatch. (3) Task too large → recommend `/arsenal-build:expand-phase --phase N --force` to split. (4) Plan is wrong → escalate to user. (5) **Purely visual change required** → escalate to user; recommend deferring the visual change to the design pipeline (a `--force` re-expansion of the design half, or a future phase). **Never force the same model to retry without changes.** |

### Step 5: Dispatch spec compliance reviewer

Spawn a reviewer subagent using the prompt template at `references/spec-reviewer-prompt.md`. The job is binary: did the code match the spec, AND did every commit that modified a component file include the `Component extended:` note?

**If the spec reviewer finds issues** (including a missing `Component extended:` note on a component-touching commit):

- Dispatch a fix subagent (use the feature-implementer template at `references/feature-implementer-prompt.md` with the specific issues as the task body).
- Spec reviewer re-reviews.
- Repeat until approved — no "close enough."

If the spec reviewer determines the implementer made a **purely visual change** to a design-pipeline-committed component (a change that goes beyond the extend-not-redesign rule), that's a CRITICAL spec failure. The reviewer flags it; the fix subagent reverts the visual portion and surfaces the change to the user for redirect to the design pipeline.

### Step 6: Dispatch code quality reviewer

Only after spec compliance passes. Spawn a reviewer subagent using the prompt template at `references/quality-reviewer-prompt.md`.

**If the quality reviewer finds issues:**

- Dispatch a fix subagent (feature-implementer template) with the specific issues.
- Quality reviewer re-reviews.
- Critical and Important must be fixed. Minor can be noted and deferred (record in the per-task report so they're not lost; the orchestrator's final phase summary aggregates them).

### Step 7: Complete the task

Once both reviews pass:

- Ensure all changes are committed with semantic messages: `feat(phase-N): [task summary]` or the appropriate type (`fix`, `refactor`, `test`, etc.). Component-touching commits must include the `Component extended:` note. The implementer makes these commits during its dispatch; this skill verifies the working tree is clean before flipping the checkbox.
- Mark the task as complete in `TASKS.md`: change `- [ ]` to `- [x]` for this task (under the `### Feature tasks` subsection).
- Report DONE to the caller.

### Step 8: Return to caller

Report:

- **Status:** DONE | SKIPPED (M1 idempotence) | WRONG_DOMAIN (task is `domain: design`, route to `run-task-design`) | NEEDS_CONTEXT (cite which brief is missing) | BLOCKED (cite the implementer's BLOCKED reason and your assessment)
- **Phase:** N
- **Task:** N
- **Researcher ran:** yes | no (with reason — skipped, no flag, etc.)
- **Implementer model:** Sonnet | Haiku | Opus
- **Spec review:** passed | passed after N fix cycles
- **Quality review:** passed | passed after N fix cycles
- **Components extended (if any):** list of `<path> — <one-line why>` extracted from commit messages
- **Deferred minor issues:** list (these aggregate into the phase summary at `close-feature-phase`)
- **Commits added:** list of semantic messages
- **TASKS.md:** task N flipped to `[x]` under `### Feature tasks`

## Reference files

| File | Used in | Purpose |
|------|---------|---------|
| `references/researcher-prompt.md` | Step 2 | Researcher subagent template (when task is `research: yes`) |
| `references/feature-implementer-prompt.md` | Step 3 (and fix dispatches in Steps 5/6) | Feature-implementer subagent template — carries the component-boundary rule, the `Component extended:` commit-message format, and the BLOCKED escalation for purely visual changes |
| `references/spec-reviewer-prompt.md` | Step 5 | Spec compliance reviewer template — checks spec compliance AND enforces the `Component extended:` note on component-touching commits |
| `references/quality-reviewer-prompt.md` | Step 6 | Code quality reviewer template |

Each template has bracketed placeholders. Substitute every one before dispatch — never ship a template with stubs intact.

## Anti-patterns — never do these

- **Don't let subagents inherit this skill's session context.** Subagents get their own system prompt plus basic environment details — that's it. Construct their prompts with exactly what they need.
- **Don't paste entire docs into subagent prompts.** Briefs at `.arsenal/tasks/phase-{N}/task-{N}-*.md` were generated upstream by `generate-feature-briefs`. Reference them by path.
- **Don't read full canonical docs in subagent prompts.** Subagents read their per-task brief and (if applicable) their per-task research file. They never read FEATURES.md / `.arsenal/features/`, UX.md, or full ARCHITECTURE.md/CONVENTIONS.md/DESIGN_SYSTEM.md directly.
- **Don't share research files across tasks.** Each task gets its own `.arsenal/tasks/phase-{N}/task-{N}-research.md`.
- **Don't accumulate research file contents in this skill's context.** When a researcher subagent writes a file, this skill confirms the file was written but does not read its contents.
- **Don't skip either review stage.** Spec compliance catches "well-written but wrong" and missing `Component extended:` notes. Quality catches "correct but sloppy." Both matter.
- **Don't dispatch multiple feature-implementers in parallel.** They'll create merge conflicts. This sub-skill always runs one task at a time.
- **Don't accept "close enough" from reviewers.** If issues were found, they get fixed and re-reviewed.
- **Don't ignore BLOCKED or NEEDS_CONTEXT.** Something needs to change — more context, a different model, a smaller task, or human input. Never force the same model to retry without changes.
- **Don't invoke impeccable.** The feature pipeline never calls impeccable. If a feature task seems to need design judgement, it was misclassified — surface to the user.
- **Don't redesign components.** The feature pipeline extends components for the data layer, never restyles them. Purely visual changes BLOCK with redirect to the design pipeline.
- **Don't omit the `Component extended:` note.** Every commit that modifies a file under the project's component root carries the note. Spec reviewer enforces this — bypassing it is a defect.
- **Don't re-run a task that's already `[x]` without `--force`.** No-op pass-through with a clear report.

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:features` | Invokes this skill in a loop, one call per `- [ ]` task in the phase's `### Feature tasks` subsection. Hand-off: phase number + task identifier. Runs after `close-design-phase` has returned. |
| `/arsenal-build:expand-phase` | Runs **before** this skill (via the orchestrator). Writes the concrete tagged task list this skill reads from `TASKS.md`. |
| `/arsenal-build:generate-feature-briefs` | Runs **before** this skill (via the orchestrator). Writes the briefs this skill's subagents read by path; the available-components manifest in each brief comes from this skill. |
| `/arsenal-build:close-design-phase` | Runs **before** the orchestrator invokes this skill. Commits the components this skill's tasks may wire to. |
| `/arsenal-build:close-feature-phase` | Runs **after** all feature tasks in the phase complete. Aggregates deferred Minor issues this skill reported per task, runs the test suite + CodeRabbit + push + PR. |
| `impeccable` | **Not invoked from this skill, ever.** |

---
name: generate-feature-briefs
description: Writes per-task context briefs for every `domain: feature` task in a phase. The brief (≤3,000 tokens / ~12,000 chars) carries the task description plus scope-relevant excerpts from the feature spec, ARCHITECTURE, CONVENTIONS, DESIGN_SYSTEM component cites, and the **available component paths** the feature task may wire to (read-only inputs committed earlier in the phase by the design pipeline). Does NOT call impeccable. Does NOT write design briefs. Invoked by `features` after the design pipeline closes its half of the phase; can be invoked directly to regenerate feature briefs for a phase whose feature spec or architecture changed. Idempotent by default; `--force` regenerates.
---

# Generate Feature Briefs

Generate the per-task context briefs that downstream `run-task-feature` subagents read. Briefs are the filesystem hand-off — this skill writes them to disk; the controller never reads brief contents after this skill reports DONE.

This skill is normally invoked by `features` after `close-design-phase` has committed the phase's visual components. It can also be invoked directly to regenerate feature briefs for a phase whose feature spec, ARCHITECTURE, or available-components set changed after the original generation — `--force` regenerates; default is idempotent.

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

## When this skill writes vs. no-ops

| State of `.tasks/phase-{N}/task-{N}-context.md` | Behavior |
|---|---|
| Does not exist | Generate the brief from disk reads. |
| Exists and `--force` not set | Skip with status `SKIPPED — brief exists`. Mirrors L1 idempotence. |
| Exists and `--force` set | Regenerate, overwriting in place. |

This skill only processes tasks tagged `domain: feature`. Design-domain tasks are skipped entirely — they're the design pipeline's responsibility via `generate-design-briefs`.

## Inputs

Caller passes:

- `--phase <N>` — phase number (required)
- `--task <N>` — optional. Generate the brief only for the named task. When omitted, brief every `domain: feature` task in the phase that doesn't already have a context brief on disk (or every feature task if `--force`).
- `--force` — optional. Regenerate briefs even when files already exist.

Files read from disk (read-only):

- `docs/TASKS.md` — to discover task tags (`domain:`, `research:`) and locate each `domain: feature` task under the phase's `### Feature tasks` subsection. Re-read here rather than trusting orchestrator state, so direct invocation works.
- `planning/FEATURES.md` (single mode) or `planning/features/<slug>.md` (split mode, by exact path) — scope-relevant feature spec section per task.
- `docs/ARCHITECTURE.md` — data flow / integration / schema excerpts per task.
- `docs/CONVENTIONS.md` — 2–4 patterns most relevant to each task (data-fetch, mutation, server-handler, etc.).
- `docs/DESIGN_SYSTEM.md` — primitive cites by `§ X.Y` for components this task wires to; never paste content, only cite. Also read for the project's component-path convention (e.g., `components/ui/`).
- **Available component paths** — components this task may wire to. Source in priority order:
  1. Git log on the current phase branch (`git log <phase-base>..HEAD -- 'components/**'` or similar) to enumerate paths the design pipeline committed earlier this phase.
  2. If git history isn't reliable (mid-refactor, no merge base resolved), fall back to a `find components -name '*.{tsx,jsx,vue,svelte}'` style enumeration scoped to the project's declared component root.
  3. If neither yields anything, note the section as `_No design-pipeline components committed yet for this phase._` and proceed — the implementer will BLOCK if it tries to wire to a missing component.

Files written:

- `.tasks/phase-{N}/task-{N}-context.md` per `domain: feature` task in scope — context brief, ≤3,000 tokens (~12,000 chars).

## Workflow

### Step 1: Read TASKS.md and resolve scope

Read `docs/TASKS.md` for phase `<N>`. Locate the `### Feature tasks` subsection. If the subsection is missing or contains only the placeholder line (`_None — pure feature-domain phase._` or `Tasks not yet generated`), report `NEEDS_EXPANSION` and stop — the orchestrator should run `expand-phase` first.

For each task in the subsection (or just `--task <N>` if specified):

1. Confirm the task's `domain:` tag is `feature`. If not, skip with a defect warning (expand-phase emitted a misplaced task).
2. Parse `research: yes/no`.
3. Check whether `.tasks/phase-{N}/task-{N}-context.md` already exists.
   - Exists and `--force` not set → skip, mark `SKIPPED — brief exists`.
   - Exists and `--force` set → continue, will overwrite.
   - Does not exist → continue.

### Step 2: Enumerate available components for the phase

Once per phase invocation (not per task), build the list of available component paths the feature pipeline may wire to:

1. Identify the phase branch's merge base (typically `main`).
2. List paths under the project's declared component root committed since the merge base. The convention is `components/ui/` and `components/shell/` — read DESIGN_SYSTEM.md's declared path if defined; otherwise default to `components/`.
3. Group paths by component for readability (e.g., `Hero.tsx`, `EmailCaptureForm.tsx`).

Cache this list across all feature tasks in the phase — every task's context brief references the same available-component manifest.

If no design-pipeline commits exist on this branch (the design pipeline hasn't run yet for this phase, or this is a pure feature-domain phase), the manifest is empty. Note this in each brief; the implementer treats components as needing to be discovered from existing project state.

### Step 3: Write the per-task context brief

For each in-scope task, write the brief at `.tasks/phase-{N}/task-{N}-context.md`. **Brief budget: ≤3,000 tokens (~12,000 characters).** Summarize down rather than paste full sections. **If a task genuinely needs more context than 3k tokens, that's a signal the task is too big — report `TASK_TOO_BIG` and recommend the caller re-run `expand-phase --force` to split it.**

The context brief contains only:

- The task description verbatim from `TASKS.md` (including its inline tag line)
- The relevant feature spec section (or AC subset) — sourced from `FEATURES.md` (single mode) or `planning/features/<slug>.md` (split mode)
- The 2–4 `CONVENTIONS.md` patterns most relevant to this task (data fetching, mutation, server handlers, etc.)
- `ARCHITECTURE.md` excerpts relevant to this task (data flow, integration, schema)
- `DESIGN_SYSTEM.md` component cites by `§ X.Y` for primitives this task may reference — these are read-only inputs.
- **`## Available components`** section — the phase-scoped manifest from Step 2, with a note: *"These components were committed earlier in this phase by the design pipeline. Treat them as read-only design surfaces. You may extend a component (new props, variants, or states the data layer needs) but never redesign one (visual treatment, spacing, typography, color). Any commit that extends a component must include a `Component extended: <path> — <one-line why>` note in the message. Purely visual changes are BLOCKED — surface them so the design pipeline can address them in a later phase."*
- Pointer to the task's research file path (if `research: yes`) — but only the path, never the contents. The downstream researcher subagent will write it before the implementer reads it.

The context brief does **not** include any design brief. Design briefs only exist for `domain: design` tasks, written by `generate-design-briefs`.

Reference these files in downstream subagent prompts by exact path. Subagents read their own brief — never the canonical docs directly.

### Step 4: Report to caller

Once all in-scope feature tasks have been briefed (or skipped via L1), report:

- **Status:** DONE | NEEDS_EXPANSION (phase has placeholder feature-task line) | TASK_TOO_BIG (cite which task exceeded budget) | NEEDS_CONTEXT
- **Phase:** N
- **Feature tasks processed:** count
- **Context briefs written:** count
- **Skipped (L1 idempotence):** count, with task IDs
- **Available components manifest:** count of design-pipeline-committed components included in briefs (or `empty — design pipeline has not run for this phase`)

The caller's next step is to execute each feature task via `run-task-feature`; this skill does not run tasks itself.

## Anti-patterns — never do these

- **Don't write design briefs.** This skill writes only context briefs. Design briefs are the design pipeline's concern (`generate-design-briefs`).
- **Don't process design-domain tasks.** Locate the `### Feature tasks` subsection and stay within it. A misplaced task (e.g., `domain: design` appearing under `### Feature tasks`) is a defect — flag it, don't silently brief it.
- **Don't call impeccable.** Feature briefs are about data, architecture, and component-wiring. Impeccable is the design pipeline's tool. If a feature task seems to need design judgement, it was misclassified — surface to the user and don't brief it.
- **Don't read brief contents after writing them.** This skill confirms the file exists and reports DONE. Downstream `run-task-feature` subagents read briefs by path. Accumulating brief contents in this skill's context defeats the filesystem hand-off pattern.
- **Don't regenerate briefs without `--force`.** Default behavior is idempotent per L1: skip per-task if the context-brief file already exists. Silent regeneration clobbers any hand-tuned briefs.
- **Don't paste full canonical docs into briefs.** The 3k-token budget is a forcing function. If you can't fit the task within budget, the task is too big — report `TASK_TOO_BIG` and recommend re-expansion.
- **Don't omit the available-components manifest.** Every feature brief includes the section, even if empty. The implementer needs to know what surfaces it may wire to and which boundary rules apply.

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:features` | Invokes this skill at Step 2 after `close-design-phase` returns. Hand-off: phase number. |
| `/arsenal-build:expand-phase` | Runs **before** this skill (via the orchestrator). Writes the concrete tagged task list with `domain:` tags this skill reads from `TASKS.md`. |
| `/arsenal-build:generate-design-briefs` | Sibling skill. Runs **before** this skill in the phase (during the design half of the pipeline). Writes context + design briefs for design-domain tasks. This skill never touches those tasks. |
| `/arsenal-build:run-task-feature` | Runs **after** this skill, per task. Reads briefs from `.tasks/phase-{N}/` by path. |
| `/arsenal-build:close-design-phase` | Runs **before** this skill in the phase pipeline. Commits the components this skill manifests as available inputs. |

---
name: generate-design-briefs
description: Writes per-task context briefs AND design briefs for every `domain: design` task in a phase. Context brief (≤3,000 tokens / ~12,000 chars) carries the task description plus excerpts from UX.md, the design-relevant portions of the feature spec, DESIGN_SYSTEM cites, and view-structure conventions. Design brief (≤7,000 chars / ~1,700 tokens) is a translation plan from mockup (when present) or written design direction to the implementation surface — token map, locked primitives, variant coverage, mockup ↔ DESIGN_SYSTEM gap flags. Does NOT auto-dispatch impeccable. If the design brief is under-specified, the design implementer can BLOCK and the user may invoke impeccable manually. Invoked by `design` after `expand-phase`. Idempotent by default; `--force` regenerates.
---

# Generate Design Briefs

Generate the per-task context briefs (every design-domain task) and design briefs (also every design-domain task) for the design half of a phase. Briefs are the filesystem hand-off — downstream sub-skills (`run-task-design`) read them by path; the controller never receives their contents through prompts.

This skill is normally invoked by `design` immediately after `expand-phase` has written the concrete tasks to `TASKS.md`. It can also be invoked directly to regenerate briefs for a phase whose UX, DESIGN_SYSTEM, or mockup changed after the original generation — `--force` regenerates; default is idempotent.

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

## Why design briefs are front-loaded

All design briefs generate **here**, not lazily per-task during the design execution loop. The reason: design decisions, variant coverage, and mockup ↔ DESIGN_SYSTEM gaps should surface **before** any code is written. If briefing reveals a blocking gap (mockup uses a color the design system doesn't have, a primitive isn't declared, motion behavior is unspecified, etc.), the user resolves it once and implementation runs uninterrupted afterward.

## Impeccable is optional and manual — never automatic

This skill does **not** dispatch `impeccable:shape`, `:craft`, or any other impeccable command.

If the design source for a task is thin (no mockup, sparse DESIGN_SYSTEM coverage for the surface, novel IA), the design-brief subagent writes whatever it can and reports `NEEDS_USER_RESOLUTION` for the unresolved cells. The user decides whether to invoke `impeccable:shape <surface>` manually to produce a richer design source, then re-run this skill with `--force --task <N>`. The skill stays out of that decision.

## When this skill writes vs. no-ops

| State of `.arsenal/tasks/phase-{N}/task-{N}-{context,design}.md` | Behavior |
|---|---|
| Both files do not exist | Generate both. |
| Context exists, design does not | Generate the design brief only. |
| Design exists, context does not | Generate the context brief only. |
| Both exist and `--force` not set | Skip with status `SKIPPED — briefs exist`. Mirrors L1 idempotence. |
| Either exists and `--force` set | Regenerate the affected file(s), overwriting in place. |

This skill only processes tasks tagged `domain: design`. Feature-domain tasks are skipped entirely — they're the feature pipeline's responsibility via `generate-feature-briefs`.

## Inputs

Caller passes:

- `--phase <N>` — phase number (required)
- `--task <N>` — optional. Generate briefs only for the named design task. When omitted, brief every `domain: design` task in the phase that doesn't already have both briefs on disk (or every design task if `--force`).
- `--force` — optional. Regenerate briefs even when files already exist.

Files read from disk (read-only):

- `.arsenal/TASKS.md` — to discover task tags (`domain:`, `research:`) and locate each `domain: design` task under the phase's `### Design tasks` subsection. Re-read here rather than trusting orchestrator state.
- `.arsenal/design/UX.md` — the relevant page/section per task (IA truth, anti-patterns, conversion or engagement model).
- `.arsenal/design/DESIGN_SYSTEM.md` — token + primitive cites by `§ X.Y` per task; never paste, only cite. Read for the project's variant axes and component-path convention.
- `.arsenal/design/DESIGN.md` — brand spec, read-only reference. Cite rarely.
- `.arsenal/CONVENTIONS.md` — 2–4 view-structure patterns most relevant to each task (component composition, styling idioms, accessibility patterns).
- `.arsenal/ARCHITECTURE.md` — high-level skim only (the design pipeline doesn't need full architecture context — only enough to know where components live).
- `.arsenal/FEATURES.md` (single mode) or `.arsenal/features/<slug>.md` (split mode, by exact path) — the design-relevant portions of the feature spec: states, anti-patterns, copy locks.
- `.arsenal/design/mockups/<screen>.{jsx,tsx,html,png,figma-export.json}` — read by the design-brief subagent, not by this skill's controller; cited from the brief by path + region.

Files written:

- `.arsenal/tasks/phase-{N}/task-{N}-context.md` per `domain: design` task in scope — context brief, ≤3,000 tokens (~12,000 chars).
- `.arsenal/tasks/phase-{N}/task-{N}-design.md` per `domain: design` task in scope — design brief, ≤7,000 chars (~1,700 tokens).

## Workflow

### Step 1: Read TASKS.md and resolve scope

Read `.arsenal/TASKS.md` for phase `<N>`. Locate the `### Design tasks` subsection. If the subsection is missing or contains only the placeholder line, report `NEEDS_EXPANSION` and stop.

For each task in the subsection (or just `--task <N>` if specified):

1. Confirm the task's `domain:` tag is `design`. If not, skip with a defect warning.
2. Parse `research: yes/no`.
3. Check whether each brief file already exists; apply the idempotence table above.

### Step 2a: Write the context brief (controller-written, no subagent)

For each in-scope task, write the per-task context brief at `.arsenal/tasks/phase-{N}/task-{N}-context.md`. **Brief budget: ≤3,000 tokens (~12,000 characters).** If a task genuinely needs more, that's a signal the task is too big — report `TASK_TOO_BIG` and recommend the caller re-run `expand-phase --force` to split it.

The context brief for a design task contains only:

- The task description verbatim from `TASKS.md` (including its inline tag line)
- The relevant `UX.md` page/section (IA, anti-patterns, conversion or engagement model)
- The design-relevant portions of the feature spec — states, anti-patterns, copy locks — from `FEATURES.md` or `.arsenal/features/<slug>.md`
- The 2–4 `CONVENTIONS.md` view-structure patterns most relevant to this task
- `DESIGN_SYSTEM.md` cites by `§ X.Y` for primitives this task uses or extends — these are the design-system surfaces being built or referenced
- Pointer to the mockup file + region if one exists for this surface (the full translation plan lives in the design brief — Step 2b — not here)
- Pointer to the task's research file path (if `research: yes`)
- **Component creation path** — declared by `DESIGN_SYSTEM.md` or by project convention (e.g., `components/ui/<ComponentName>.tsx`). The design implementer writes the component at this path; downstream feature tasks reference it as a read-only input.

The context brief is the design implementer's "what + why"; the design brief (Step 2b) is the "how to translate the design source to the implementation surface."

### Step 2b: Write the design brief (subagent-dispatched)

Dispatch a fresh-context design-brief subagent using `references/design-brief-prompt.md`.

Substitute every bracketed placeholder before dispatch — never ship a template with stubs intact. The subagent reads the design sources (mockup, DESIGN_SYSTEM, DESIGN, UX, feature spec) and writes a translation plan to `.arsenal/tasks/phase-{N}/task-{N}-design.md`. Budget: ≤7,000 characters (~1,700 tokens).

The controller (this skill) does **not** read the design brief contents after the subagent writes it — only confirms the file exists. Downstream `run-task-design` reads the brief via path reference.

**Mockup ↔ DESIGN_SYSTEM gap detection:** if a mockup uses a value the DESIGN_SYSTEM doesn't have, the subagent **flags it** rather than picking. The user resolves which is canonical. If the subagent reports `MOCKUP_DS_GAP_BLOCKING`, surface the gap to the user and pause subsequent task briefs until resolved.

**Thin design source:** when the surface has no mockup and DESIGN_SYSTEM coverage is sparse, the subagent writes what it can and reports `NEEDS_USER_RESOLUTION`. This skill surfaces the gap and pauses. The user decides whether to invoke `impeccable:shape <surface>` manually (or otherwise fill the gap) and re-run with `--force --task <N>`. Never auto-dispatch impeccable.

### Step 3: Report to caller

Once all in-scope design tasks have been briefed (or skipped via L1), report:

- **Status:** DONE | NEEDS_EXPANSION (phase has placeholder design-task line) | TASK_TOO_BIG (cite which task exceeded budget) | MOCKUP_DS_GAP_BLOCKING (cite task) | NEEDS_USER_RESOLUTION (cite task and the unresolved cells)
- **Phase:** N
- **Design tasks processed:** count
- **Context briefs written:** count
- **Design briefs written:** count
- **Skipped (L1 idempotence):** count, with task IDs
- **Blocking gaps requiring user resolution:** any `MOCKUP_DS_GAP_BLOCKING` or `NEEDS_USER_RESOLUTION` reports, cited per task

The caller's next step is to execute each design task via `run-task-design`; this skill does not run tasks itself.

## Reference files

| File | Used in | Purpose |
|---|---|---|
| `references/design-brief-prompt.md` | Step 2b | Template for the design-brief subagent — translates mockup + DESIGN_SYSTEM into a token-mapped technical plan with variant matrix. |

The template has bracketed placeholders. Substitute every one before dispatch — never ship a template with stubs intact.

## Anti-patterns — never do these

- **Don't auto-dispatch impeccable.** This skill never invokes `impeccable:shape`, `:craft`, or any other impeccable command. If a brief comes back thin, surface to the user and let them decide whether to invoke impeccable manually.
- **Don't write feature briefs.** This skill writes only context + design briefs for `domain: design` tasks. Feature-domain tasks are the feature pipeline's concern (`generate-feature-briefs`).
- **Don't process feature-domain tasks.** Locate the `### Design tasks` subsection and stay within it.
- **Don't read brief contents after writing them.** Controller confirms the design brief subagent's file exists and reports DONE. Downstream `run-task-design` reads briefs by path.
- **Don't regenerate briefs without `--force`.** Default behavior is idempotent per L1: skip per-task when the brief files already exist.
- **Don't paste full canonical docs into briefs.** The 3k-token context budget and 1.7k-token design budget are forcing functions. If the task can't fit, report `TASK_TOO_BIG` and recommend re-expansion.
- **Don't proceed past a `MOCKUP_DS_GAP_BLOCKING` or `NEEDS_USER_RESOLUTION` report.** Surface the gap to the user and pause. Resolving the gap (updating DESIGN_SYSTEM, updating the mockup, or running impeccable manually) is a separate concern that must commit first.
- **Don't cross-pollinate briefs.** Each `.arsenal/tasks/phase-{N}/task-{N}-*.md` file is isolated to its task. Subagents read only their own task's briefs by exact path — no globs, no cross-task accumulation.

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:design` | Invokes this skill at Step 2 after `expand-phase`. Hand-off: phase number. |
| `/arsenal-build:expand-phase` | Runs **before** this skill. Writes the concrete tagged task list this skill reads from `TASKS.md`. |
| `/arsenal-build:generate-feature-briefs` | Sibling skill. Runs **after** this skill **and after** `close-design-phase` — the feature briefs reference the components this design pipeline commits. This skill never touches feature-domain tasks. |
| `/arsenal-build:run-task-design` | Runs **after** this skill, per task. Reads briefs from `.arsenal/tasks/phase-{N}/` by path. |
| `impeccable` (any sub-command) | **Not invoked from this skill.** Available to the user as a manual escape hatch when a brief comes back thin — user invokes `impeccable:shape <surface>` etc., then re-runs this skill with `--force`. |

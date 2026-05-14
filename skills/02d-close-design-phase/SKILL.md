---
name: close-design-phase
description: Closes the DESIGN half of a development phase for web/frontend projects by running two gates in order — design QA review across surfaces shipped (optional impeccable audit, gateable; never automatic) → docs update (if scope drifted). Writes a brief design-pipeline summary to `.arsenal/tasks/phase-N/design-summary.md` for the feature pipeline's PR body to reference. Does NOT push, does NOT open a PR, does NOT trim TASKS.md, does NOT run CodeRabbit (all of that happens at `close-feature-phase`, the phase terminus). Invoked by `design` at the end of the design half, or directly to wrap up a design half whose per-task work already landed. Use when the design tasks are all `[x]` and the user is about to hand off to the feature pipeline.
---

# Close Design Phase — Web

Run the design-pipeline wrap for a completed design half of a phase. **Two gates**, both optional in execution semantics (each may no-op if conditions don't apply) but always invoked in order. This skill returns the phase branch to the orchestrator for the feature pipeline to pick up — it does NOT push, PR, or trim TASKS.md.

This skill is normally invoked by `design` after every design-domain task in the phase is committed and `[x]`-marked. It can also be invoked directly to re-run the wrap on a branch whose design-task commits already landed.

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

## Where this fits in the phase pipeline

```
design   →   close-design-phase  ← YOU ARE HERE   →   features   →   close-feature-phase
   (design tasks)         (design-pipeline wrap)                        (feature tasks)            (phase-final wrap + PR)
```

This skill's terminus is "design-summary.md written and the branch is ready for the feature pipeline." Nothing is pushed; no PR is opened. The phase isn't done — only the design half is.

## Why no CodeRabbit, no push, no PR

CodeRabbit runs once per PR. The PR opens at `close-feature-phase` (end of phase). Running CodeRabbit here would either (a) duplicate the cost, or (b) miss the feature-pipeline commits — neither is desirable. Single CodeRabbit pass per phase, covering both domains.

Similarly, the trim/archive of TASKS.md happens at `close-feature-phase` because the phase isn't complete until both halves are. Design tasks have already been individually `[x]`-marked by `run-task-design`; the structural cleanup waits for the feature half.

## Why impeccable is OPTIONAL here (never automatic)

Audit and polish are useful as a cross-surface phase audit, but only when the project actually wants them. This skill **prompts the user** before invoking; never automatic.

The web design pipeline doesn't use per-task finishers; the optional audit here is the cross-surface design check.

## Prerequisites

| File / state | Used for |
|---|---|
| `.arsenal/TASKS.md` | Phase block with `### Design tasks` all `[x]`. This skill does not modify the file (trimming is `close-feature-phase`'s job). |
| `.arsenal/tasks/phase-N/` | Working dir; this skill writes `.arsenal/tasks/phase-N/design-summary.md` for the feature pipeline to read later. |
| Phase branch checked out (`phase-N/short-description`) | All design-pipeline commits already on it. |
| `.arsenal/design/DESIGN_SYSTEM.md` | Read for the project's component-root path and token convention (for the optional audit). |
| `impeccable` skill installed (OPTIONAL) | Required only if the user opts in to Gate 1's impeccable audit. Soft-prompt; never block on absence. |

## Inputs (for direct invocation)

When invoked directly, the caller passes the phase number (e.g., `N=1`). All other state is read from disk:

- `.arsenal/TASKS.md` (read at Gate 1 to enumerate surfaces shipped)
- `.arsenal/tasks/phase-N/task-N-design.md` files (read at Gate 1 to aggregate variant coverage / mockup pointers across surfaces)
- Phase branch must already be checked out

## The two gates

### Gate 1 of 2 — Design QA review across surfaces shipped this phase

**Step A — Aggregate the design output.**

For each `domain: design` task in this phase (`### Design tasks` subsection of TASKS.md, all `[x]`):

- Read `.arsenal/tasks/phase-N/task-N-design.md` to identify the surface implemented, the mockup pointer, and the variant coverage that was supposed to land.
- Note the implemented component path.

Build a phase-level summary: which surfaces shipped, which mockup regions were translated, which states were rendered. This summary feeds Step B (optional impeccable audit) and the `design-summary.md` artifact.

**Step B — Optional impeccable audit (prompt the user).**

Prompt the user:

> "Phase N design half is complete. Would you like to run an optional impeccable audit + polish pass across all surfaces shipped this phase? This catches design-system drift and cross-surface inconsistencies that per-task review may have missed. [Y / N — default N]
>
> Surfaces shipped this phase:
> - [list of components committed under the project's component root]
>
> If yes, this skill will dispatch `impeccable:audit <surface>` and `impeccable:polish <surface>` for each. The audit produces P0–P3 findings; the user triages fix/dismiss/discuss before this skill exits. Audit fixes commit as `fix(phase-N): <description>` on the same branch."

- **If N (default):** skip the audit. Note in `design-summary.md`: `Impeccable audit: skipped at user request.`
- **If Y:** dispatch `impeccable:audit <surface>` per shipped component / page. Aggregate findings into a triage list. For each fix-list item, dispatch a fix subagent (using the design-implementer template at `run-task-design/references/design-implementer-prompt.md`) and commit as `fix(phase-N): <description>`. After fixes, dispatch `impeccable:polish <surface>` per shipped surface for final spacing/alignment micro-detail. Loop until findings are fixed or explicitly dismissed.

**If `impeccable` is not installed:**

Tell the user: "impeccable skill not detected. The audit step is optional anyway — skipping. If you'd like to run it, install impeccable and re-invoke this skill with `--force`." Do not block.

**Multi-surface audits:**

If the phase shipped 2+ surfaces, audit each separately and aggregate findings into a single triage list. Don't double-audit the same surface that was touched by multiple tasks — audit once against the final state.

### Gate 2 of 2 — Update docs (if scope drifted)

**Only update docs if scope changed or a spec drifted during the design half.** If the phase shipped exactly what was planned, this gate is a no-op pass-through.

When updates are warranted:
- Note any design decisions made during implementation in the Key Decisions Log.
- If a mockup ↔ DESIGN_SYSTEM gap was resolved (DESIGN_SYSTEM updated to match a mockup value), confirm the update committed and note it in the summary.
- If a feature spec's design-relevant section drifted from what was built, flag for a follow-up `.arsenal/features/` update — don't edit specs inline.

### After both gates: write the design-summary.md and report

Write `.arsenal/tasks/phase-N/design-summary.md` for the feature pipeline (and ultimately the PR body at `close-feature-phase` Gate 6) to read:

```markdown
# Phase N — Design half summary

**Surfaces shipped:** [comma-separated list of component paths]
**States rendered (aggregated):** default, loading, empty, error, hover/focus, mobile, dark
**Impeccable audit:** [run — N findings, M fixed | skipped at user request | skill not installed]
**Cross-surface findings:** [any notable cross-surface concerns, or "none"]
**Mockup ↔ DESIGN_SYSTEM gaps resolved this half:** [list, or "none"]
**TODOs left for the feature pipeline:** [list of `// TODO(feature-pipeline):` comments aggregated from design-pipeline commits]
```

This file is read by `close-feature-phase` Gate 6 (PR body composition) so the GitHub reviewer sees a full picture of the phase.

Return to the caller:

- **Status:** DONE | NEEDS_CONTEXT (cite which artifact is missing)
- **Phase:** N
- **Surfaces audited (if any):** count
- **Audit findings:** Critical: A, Important: B, Minor: C — or "skipped"
- **Audit fixes committed:** count
- **Docs updated:** yes | no
- **`design-summary.md`:** written to `.arsenal/tasks/phase-N/design-summary.md`
- **Branch state:** clean and ready for the feature pipeline

## Anti-patterns — never do these

- **Don't push, don't open a PR.** That's `close-feature-phase`'s terminus.
- **Don't trim TASKS.md.** The phase isn't complete until the feature half lands.
- **Don't run CodeRabbit.** Single CodeRabbit pass per PR, at the feature close.
- **Don't run a final test suite.** That's a feature-pipeline concern (data-layer correctness).
- **Don't auto-dispatch impeccable.** Always prompt the user first.
- **Don't fail when impeccable is missing.** It's optional here. Note it in the summary and proceed.
- **Don't read archived task lists speculatively.**
- **Don't improvise finishers at this gate.** Finishers ran during the per-task pipeline (when applicable). If a cross-surface concern surfaces here, the impeccable audit + polish is the tool — not a finisher.

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:design` | Invokes this skill at the end of the design half. Hand-off point: all design-task commits landed, branch checked out, `.arsenal/tasks/phase-N/` populated. |
| `/arsenal-build:close-feature-phase` | Runs **after** this skill (after the feature half). Reads `.arsenal/tasks/phase-N/design-summary.md` to compose the PR body. |
| `/arsenal-build:run-task-design` | Per-task pipeline that ran during the design half. Audit fix dispatches at Gate 1B reuse its design-implementer prompt. |
| `impeccable` (specifically `:audit` and `:polish`) | Gate 1B dispatch, ONLY if the user opts in. Never automatic. |
| `coderabbit:review` | **Not invoked here.** Runs once at `close-feature-phase` Gate 4 across the entire phase. |

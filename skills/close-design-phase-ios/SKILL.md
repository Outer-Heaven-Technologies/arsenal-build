---
name: close-design-phase-ios
description: Closes the DESIGN half of a development phase for a native iOS project by running two gates in order — visual phase audit (optional impeccable `audit` + `polish` per surface shipped, gateable; never automatic) → docs update (if scope drifted). Writes a brief design-pipeline summary to `.tasks/phase-N/design-summary.md` for the feature pipeline's PR body. Does NOT push, does NOT open a PR, does NOT trim TASKS.md, does NOT run CodeRabbit, does NOT run snapshot or Periphery gates (all of that happens at `close-feature-phase-ios`, the phase terminus). Invoked by `design-ios` at the end of the design half, or directly to wrap up a design half whose per-task work already landed.
---

# Close Design Phase — iOS

Run the design-pipeline wrap for a completed design half of a phase. **Two gates**, both invoked in order; each may no-op if conditions don't apply. This skill returns the phase branch to the orchestrator for the feature pipeline to pick up — it does NOT push, PR, trim, or run snapshot/Periphery/CodeRabbit.

This skill is normally invoked by `design-ios` after every design-domain task in the phase is committed and `[x]`-marked. Can also be invoked directly to re-run the wrap.

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

## Where this fits in the phase pipeline

```
design-ios   →   close-design-phase-ios  ← YOU ARE HERE   →   features-ios   →   close-feature-phase-ios
   (design tasks)         (design-pipeline wrap)                        (feature tasks)            (phase-final wrap + PR)
```

This skill's terminus is "design-summary.md written and the branch is ready for the feature pipeline." Nothing is pushed; no PR is opened.

## Why no CodeRabbit, no Periphery, no snapshot verification here

All three are regression / quality gates that should run once per phase, at the phase terminus (`close-feature-phase-ios`):

- **CodeRabbit** wants to see the full PR diff including the feature pipeline's commits — single pass per PR.
- **Periphery** wants to scan the whole project after both halves land — running it twice would double the triage work for no benefit.
- **Snapshot test verification** confirms primitives didn't drift across the full phase — if a feature-pipeline change accidentally breaks a primitive, you want it caught at the terminus, not before the feature pipeline runs.

The phase isn't done until both halves land. Single CodeRabbit pass per PR.

## Why impeccable is OPTIONAL here (never automatic)

Finishers run at the per-task level inside `run-task-design-ios`'s pipeline (Step 8 of that skill — per the task's `finishers: [list]` tag), so per-surface polish is already in place by the time this skill runs.

Audit + polish at the phase level are useful as a cross-surface review, but only when the project actually wants them. This skill **prompts the user** before invoking; never automatic.

## Prerequisites

| File / state | Used for |
|---|---|
| `docs/TASKS.md` | Phase block with `### Design tasks` all `[x]`. This skill does not modify the file. |
| `.tasks/phase-N/` | Working dir; this skill writes `.tasks/phase-N/design-summary.md`. |
| Phase branch checked out (`phase-N/short-description`) | All design-pipeline commits already on it. |
| `docs/DESIGN_SYSTEM.md` | Read for the project's view-layer module path and token convention. |
| `mcp__xcode__*` MCP | Build for the optional audit (if user opts in). |
| `mcp__ios-simulator__*` MCP | A11y check via `ui_describe_all` for the optional audit. |
| `impeccable` skill installed (OPTIONAL) | Required only if the user opts in to Gate 1's impeccable audit. Soft-prompt; never block on absence. |

## Inputs (for direct invocation)

When invoked directly, the caller passes the phase number. All other state is read from disk:

- `docs/TASKS.md` (read at Gate 1 to enumerate surfaces shipped)
- `.tasks/phase-N/task-N-design.md` files (read at Gate 1 to aggregate variant coverage and navigation scripts)
- Phase branch must already be checked out

## The two gates

### Gate 1 of 2 — Visual phase audit (optional, user-opted)

**Step A — Aggregate the design output.**

For each `domain: design` task in this phase (`### Design tasks` subsection, all `[x]`):

- Read `.tasks/phase-N/task-N-design.md` to identify the surface implemented, the mockup pointer, navigation script, and variant coverage.
- Note the implemented SwiftUI view path.

Build a phase-level summary: which surfaces shipped, which mockup regions translated, which states rendered, which finishers ran during per-task pipelines.

**Step B — Optional impeccable audit + polish (prompt the user).**

Prompt the user:

> "Phase N design half is complete. Would you like to run an optional impeccable audit + polish pass across all surfaces shipped this phase? Audit catches a11y / token / perf concerns across the phase; polish does final spacing/alignment micro-detail. [Y / N — default N]
>
> Surfaces shipped this phase:
> - [list of SwiftUI view paths]
>
> If yes, this skill will dispatch `impeccable:audit <surface>` (covers a11y via `mcp__ios-simulator__ui_describe_all`, Dynamic Type, Reduced Motion, token consistency, perf — P0–P3 scored report) and `impeccable:polish <surface>` (final spacing/alignment) for each. The audit produces findings; the user triages fix/dismiss/discuss before this skill exits. Audit fixes commit as `fix(phase-N): <description>` on the same branch."

- **If N (default):** skip. Note in `design-summary.md`: `Impeccable audit: skipped at user request.`
- **If Y:** dispatch `impeccable:audit <surface>` per shipped view. Aggregate findings. For fix-list items, dispatch a fix subagent (using the design-implementer template at `run-task-design-ios/references/design-implementer-prompt.md`) and commit as `fix(phase-N): <description>`. After fixes, dispatch `impeccable:polish <surface>` per shipped surface. Loop until findings fixed or explicitly dismissed.

**If `impeccable` is not installed:**

Tell the user: "impeccable skill not detected. The audit step is optional anyway — skipping. If you'd like to run it, install impeccable and re-invoke this skill with `--force`." Do not block.

**Multi-surface audits:**

If the phase shipped 2+ surfaces, audit each separately and aggregate findings into a single triage list.

### Gate 2 of 2 — Update docs (if scope drifted)

**Only update docs if scope changed.** If the phase shipped exactly what was planned, this gate is a no-op pass-through.

When warranted:
- Note any design decisions in the Key Decisions Log.
- If a mockup ↔ DESIGN_SYSTEM gap was resolved this half (e.g., DESIGN_SYSTEM updated to match a mockup value), confirm the update committed and note it in the summary.
- If a feature spec's design-relevant section drifted from what was built, flag for follow-up — don't edit specs inline.

### After both gates: write the design-summary.md and report

Write `.tasks/phase-N/design-summary.md` for the feature pipeline's PR body at `close-feature-phase-ios` Gate 7:

```markdown
# Phase N — Design half summary

**Surfaces shipped:** [comma-separated list of SwiftUI view paths]
**States rendered (aggregated):** [list — default, loading, empty, error, dynamic-type-XL, reduced-motion, etc.]
**Finishers run during per-task pipeline:** harden: M tasks, animate: N tasks, clarify: P tasks (aggregated from per-task reports)
**Impeccable audit (phase-level):** [run — N findings, M fixed | skipped at user request | skill not installed]
**Cross-surface findings:** [notable cross-surface concerns, or "none"]
**Mockup ↔ DESIGN_SYSTEM gaps resolved this half:** [list, or "none"]
**Snapshot baselines added this half:** [list of theme primitives that got new baselines, for Gate 3 of `close-feature-phase-ios` to verify]
**TODOs left for the feature pipeline:** [list of `// TODO(feature-pipeline):` comments aggregated from design-pipeline commits]
```

Return to the caller:

- **Status:** DONE | NEEDS_CONTEXT (cite which artifact is missing)
- **Phase:** N
- **Surfaces audited (if any):** count
- **Audit findings:** Critical: A, Important: B, Minor: C — or "skipped"
- **Audit fixes committed:** count
- **Docs updated:** yes | no
- **`design-summary.md`:** written to `.tasks/phase-N/design-summary.md`
- **Branch state:** clean and ready for the feature pipeline

## Anti-patterns — never do these

- **Don't push, don't open a PR.** That's `close-feature-phase-ios`'s terminus.
- **Don't trim TASKS.md.** Phase isn't complete until the feature half lands.
- **Don't run CodeRabbit.** Single CodeRabbit pass per PR, at the feature close.
- **Don't run Periphery.** Whole-project dead-code scan runs at the feature close.
- **Don't verify snapshot baselines.** Runs at the feature close after both halves landed.
- **Don't run the full test suite.** Feature-pipeline concern (data-layer correctness).
- **Don't auto-dispatch impeccable.** Always prompt the user first.
- **Don't fail when impeccable is missing.** Optional here.
- **Don't read archived task lists speculatively.**
- **Don't improvise finishers at this gate.** Per-task finishers ran during `run-task-design-ios` (Step 8 of that skill). Phase-level audit + polish is the tool here — not finishers.
- **Don't archive PNG screenshots.** That's `close-feature-phase-ios`'s Gate 6 Step B.

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:design-ios` | Invokes this skill at the end of the design half. |
| `/arsenal-build:close-feature-phase-ios` | Runs **after** this skill (after the feature half). Reads `.tasks/phase-N/design-summary.md` for the PR body. |
| `/arsenal-build:run-task-design-ios` | Per-task pipeline that ran during the design half. Audit fix dispatches at Gate 1B reuse its design-implementer prompt. |
| `impeccable` (specifically `:audit` and `:polish`) | Gate 1B dispatch, ONLY if the user opts in. Never automatic. Per-task finishers (`harden`, `animate`, `clarify`, etc.) ran during the per-task pipeline, not here. |
| `coderabbit:review` | **Not invoked here.** Runs once at `close-feature-phase-ios` Gate 5. |
| `mcp__xcode__*` MCP | Used at Gate 1 only when the user opts in to the audit (for build verification). |
| `mcp__ios-simulator__*` MCP | Used at Gate 1 only when the user opts in (for a11y tree inspection). |
| `periphery` CLI | **Not invoked here.** Runs at `close-feature-phase-ios` Gate 2. |
| `swift-snapshot-testing` | **Not invoked here.** Verification runs at `close-feature-phase-ios` Gate 3. |

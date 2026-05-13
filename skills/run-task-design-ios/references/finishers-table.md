# Finishers Table

Used by `run-task-design-ios` for the finisher pass step. Maps the `finishers: [...]` task tags set during `expand-phase` (the expansion sub-skill, design-domain tasks only) to the impeccable sub-skill that will run as a post-implementation polish pass on the just-modified files. Finishers are scoped to the design pipeline — they never run in the feature pipeline.

Each finisher subagent gets the matching impeccable skill as its system prompt, scoped to the files the implementer modified for this task. Output is either a PR-ready patch (committed as `polish(phase-N): [skill] pass on T-N`) or "no changes needed."

## Mapping

| Tag | Impeccable skill | Auto-flag rule (set by `expand-phase`) | What it catches |
|---|---|---|---|
| `harden` | impeccable harden | Feature spec § Important mentions force-quit, scenePhase, write-then-animate ordering, error-path correctness | Force-quit resistance, error states, edge cases (force-quit during animation, mid-write interruption), scenePhase invariants, network-loss handling |
| `animate` | impeccable animate | Design brief proposes motion (cross-fade, paced animation, deceleration curve, transition) | Reduced Motion gating, animation curves match mockup, transition timing, haptic alignment with motion |
| `clarify` | impeccable clarify | Task introduces net-new copy strings (not locked verbatim in the feature spec) | UX copy, microcopy, voice/tone match (per project's `CLAUDE.md` § Voice + tone), error messages |
| `typeset` | impeccable typeset | Task touches typography hierarchy on a high-traffic screen | Font/size/weight choices within the project's theme typography tokens; hierarchy reads at glance |
| `arrange` | impeccable arrange | Layout drift suspected — usually after first-pass implementation of a new screen | Spacing rhythm using the project's theme spacing tokens; alignment; visual hierarchy |
| `normalize` | impeccable normalize | Mid-phase realization that DESIGN_SYSTEM drift is happening across multiple tasks | Realigns drifted UI back to system standards; flags new patterns that should be extracted into DESIGN_SYSTEM |
| `distill` | impeccable distill | Surface feels cluttered — manual flag only, never auto | Removes unnecessary complexity; reduces cognitive load |
| `polish` | impeccable polish | Phase-completion only (handled by `close-design-phase-ios`, not run-task-design-ios) | Final spacing/alignment/micro-detail pass before PR |
| `audit` | impeccable audit | Phase-completion only (handled by `close-design-phase-ios`, not run-task-design-ios) | A11y, perf, token consistency, anti-patterns — produces a P0–P3 scored report |

## Skipped by default for iOS

These impeccable skills are not auto-flagged. They can still be invoked manually if the user decides a specific surface needs them, but `expand-phase` never adds them to a task's `finishers: [...]` list automatically.

| Tag | Why skipped |
|---|---|
| `adapt` | Single form factor (iPhone). No responsive breakpoints to manage. |
| `colorize` | Color decisions are locked at the brand level (DESIGN.md) and at the surface level (mockups). Adding color is a brand-level action, not a per-task polish. |
| `bolder` | Same — visual intensity is a brand decision. |
| `quieter` | Same. |
| `delight` | Surface-by-surface decision. The skill won't auto-add delight; the user flags specific surfaces (e.g., milestone celebrations) for manual delight passes. |
| `overdrive` | Reserved for surfaces where the user explicitly wants ambitious visual effects (custom shaders, spring physics, scroll-driven reveals). Hand-flag only. |
| `onboard` | The onboarding flow is its own multi-task workstream — the entire onboarding feature spec drives it, not a finisher pass. |
| `optimize` | Performance is a phase-completion concern (audit pass), not a per-task one. Audit covers it. |
| `extract` | Run only when the phase touched 3+ surfaces and produced repeated patterns. Hand-flag at phase completion, not auto. |

## Finisher dispatch template

When dispatching a finisher subagent, `run-task-design-ios` substitutes:

```
## Your Role
You are running the [SKILL_NAME] pass on a just-implemented iOS task. The implementer has reported DONE and the visual fidelity, spec, and code quality reviews have all passed. Your job is the final polish on the [SKILL_NAME] axis.

## Task
[Task title and one-line summary from TASKS.md]

## Scope (files only)
[List of *.swift paths the implementer modified — do not touch anything else]

## Inputs
- The implementer's modified files
- Design brief: .tasks/phase-N/task-N-design.md (if design:yes)
- Feature spec excerpt: [the relevant section of planning/features/<feature>.md]

## Constraints
- Token discipline: zero hex literals outside the project's theme module (path declared in DESIGN_SYSTEM.md). Use theme tokens.
- Anti-pattern list from the design brief is binding.
- Voice/tone rules from the project's CLAUDE.md (§ Voice + tone) are binding when modifying user-facing copy.
- Don't restructure code outside the [SKILL_NAME] axis. If you see issues outside scope, report them as observations — don't fix.

## Workflow
[Inline the relevant section of the impeccable sub-skill — e.g., for harden, the harden checklist; for animate, the animate review patterns]

## Output
Either:
- A PR-ready diff covering only [SKILL_NAME] axis improvements
- "No changes needed" if the implementation already meets the bar

If you produce changes, commit them as: `polish(phase-N): [skill-name] pass on T-N`

## Do NOT
- Modify files outside the scoped list.
- Touch code unrelated to the [SKILL_NAME] axis.
- Auto-update snapshot baselines (if your changes affect a theme-module primitive, report it; user approves baseline updates).
```

## Notes

- **Finisher passes are sequential, not parallel.** They modify the same files; running them in parallel creates conflicts. Dispatch one at a time in the order listed in the task's `finishers: [...]` array.
- **Default order (when multiple finishers apply to the same task):** harden → animate → typeset → arrange → clarify. Harden first because correctness invariants matter more than aesthetics; clarify last because copy is the easiest to iterate on.
- **A finisher reporting "no changes needed" is a valid result and should be commit-skipped** — don't create empty commits.
- **If a finisher proposes changes that would touch a primitive in the project's theme module,** stop and ask the user. Theme changes need broader thought (snapshot test impact, brand spec alignment) than a per-task finisher should make alone.

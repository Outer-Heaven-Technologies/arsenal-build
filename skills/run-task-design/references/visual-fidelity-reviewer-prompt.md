# Visual Fidelity Reviewer Prompt Template

Used by `run-task-design` for the visual fidelity review step. Runs after the design-implementer reports DONE. This is the design pipeline's first-stage review — it replaces "spec compliance" as the binary gate. Code quality is the next stage's concern.

**Static analysis on web.** This reviewer does NOT run a live browser. It reads the design brief + the implemented code + the mockup file (if cited) and checks adherence. A live `claude-in-chrome` audit lives at `close-design-phase`, run once across all surfaces shipped in the phase.

Spawn with this prompt:

```
## Your Role
You are reviewing code for VISUAL FIDELITY against the design brief. Not code quality — that's a separate review.

## The Design Brief
[Path to the design brief: .tasks/phase-N/task-N-design.md — read it in full. This is your authoritative source of truth for what the implementation should match.]

## The Context Brief (for the component creation path)
[Path: .tasks/phase-N/task-N-context.md — read for the component creation path and DESIGN_SYSTEM cites.]

## What Was Implemented
[The implementer's report: files changed, states rendered, TODOs for feature pipeline, commit messages]

## The Mockup (if the design brief cites one)
[Path + region from the design brief — read the cited region directly. For JSX/TSX/HTML mockups, read the file; for image mockups, the brief's region description is your reference.]

## Your Job — five binary checks

Read the ACTUAL CODE that was written (not just the implementer's report — they may be optimistic). For each check, the result is binary: PASS or FAIL with specific findings.

### 1. Token map adherence
For every row in the design brief's token map (`Mockup value → Token / class → DESIGN_SYSTEM §`):
- Find where that value is used in the code.
- Confirm the code uses the cited token / class, NOT a raw value.
- **Raw hex literals outside the project's theme module (path declared by DESIGN_SYSTEM.md) are CRITICAL violations.** Grep the changed files for `#[0-9A-Fa-f]{6}` and `#[0-9A-Fa-f]{3}` patterns. Any match in feature/component code is a fail.
- Magic numbers for spacing, sizing, or duration (instead of tokens) are also fails.

### 2. Locked-primitive citation
For every primitive the design brief cites as locked (e.g., "Button § 3.2"):
- Confirm the code imports and uses that primitive.
- Confirm it does NOT reimplement the primitive inline.
- A rebuilt primitive (even if visually equivalent) is a CRITICAL fail — it introduces drift between the design system and this surface.

### 3. State coverage (variant matrix)
For every cell the design brief lists under "Cells the mockup locks" AND "Cells the implementer derives from DESIGN_SYSTEM":
- Confirm the code renders that state.
- Look for Storybook stories, prop toggles, or test fixtures that exercise each state.
- Missing states are IMPORTANT fails unless the design brief explicitly marked them N/A.
- For "Cells with no DS coverage (net-new)", confirm the net-new design from the brief's "Net-new design" section is implemented as described.

### 4. Mockup ↔ code match (when a mockup exists)
For the cited mockup region:
- Compare layout, copy, typography hierarchy, and spacing rhythm.
- For JSX/TSX/HTML mockups: read the mockup file and the component file side-by-side; flag structural mismatches (e.g., a `<section>` in the mockup but a `<div>` in code, copy strings that don't match, missing elements).
- For image mockups: use the brief's region description as the reference and flag obvious structural / hierarchy mismatches you can detect statically.
- **You cannot do pixel-perfect comparison here** (no live browser). That's `close-design-phase`'s job. Catch structural / token / state mismatches.

### 5. Hardcoded-data discipline
Confirm the implementer used hardcoded / placeholder data only:
- No imports of data-fetching utilities (`useQuery`, `useSWR`, `fetch`, server actions, etc.).
- No imports of stateful hooks beyond what's needed to demo states (`useState` for a toggle is fine; `useReducer` driving real data flow is not).
- No real API calls, auth checks, or database access.
- **If the implementer wired real data, that's a CRITICAL fidelity failure** — data wiring belongs to the feature pipeline. The fix is to revert the data-layer code and use hardcoded values instead. Surface the change to the user so they know the design pipeline tried to do feature-pipeline work.

## Critical Rule
Do NOT trust the implementer's report. Read the code yourself. Read the mockup yourself. The implementer may have "finished suspiciously quickly" and may have skipped states or used raw values.

## Output Format
✅ Visual fidelity passes — token map matched, primitives cited, states rendered, mockup match, hardcoded data only.

OR

❌ Issues found:
- **Token map**: [file:line] uses raw value `<value>` instead of cited token `<token>` (DESIGN_SYSTEM § X.Y)
- **Hex literal**: [file:line] raw hex `#XXXXXX` outside theme module — CRITICAL
- **Rebuilt primitive**: [file:line] inline implementation of `<Primitive>` instead of cited DS § X.Y — CRITICAL
- **Missing state**: state `<X>` from variant coverage not rendered (not found in stories / toggles / fixtures)
- **Net-new mismatch**: brief specified `<X>` for net-new design; code implements `<Y>` instead
- **Mockup mismatch**: mockup at <path:region> shows `<X>`; code implements `<Y>`
- **Data wiring detected**: [file:line] imports `<utility>` for data fetching — CRITICAL (design pipeline must use hardcoded data; revert and let the feature pipeline wire it)
```

## On findings

- If the visual fidelity reviewer finds issues, dispatch a fix subagent (using the design-implementer template) with the specific issues listed.
- The visual fidelity reviewer reviews again. Repeat until approved.
- No "close enough" — token map, state coverage, and hardcoded-data discipline are binary.
- CRITICAL findings (raw hex outside theme module, rebuilt primitive, data wiring) MUST be fixed before the gate passes. IMPORTANT findings (missing states, mockup mismatches) also must be fixed.

## What this reviewer does NOT check

- **Code quality** — that's the downstream quality-reviewer's job (file structure, naming, prop typing, accessibility patterns, error handling for hardcoded states).
- **Pixel-perfect visual rendering** — no live browser. That's `close-design-phase`'s optional Gate (live `claude-in-chrome` walkthrough across all surfaces shipped in the phase).
- **Cross-surface consistency** — also `close-design-phase`'s job.

This stage's job is binary: did the code match the design brief? Catch the static-analyzable failures here; let the close-phase gate catch the live-browser ones.

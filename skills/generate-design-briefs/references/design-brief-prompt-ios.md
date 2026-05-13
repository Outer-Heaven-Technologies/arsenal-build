# Design Brief Subagent Prompt Template

Used by `generate-design-briefs --surface ios` for each `domain: design` task. Spawn a fresh-context subagent that reads the project's design sources and writes a per-task design brief.

If the design source is thin (no mockup, sparse DESIGN_SYSTEM coverage for the surface), report `NEEDS_USER_RESOLUTION` for the unresolved cells rather than guessing. The user decides whether to fill the gap manually (including by invoking `impeccable:shape <surface>` if installed) and then re-run with `--force`. This skill never auto-dispatches impeccable.

The brief is **not** prose. It's a translation plan: mockup region → SwiftUI surface, with explicit token mappings, primitive citations, and a list of net-new design decisions the implementer needs.

```
## Your Role
You are generating a design brief for an iOS implementation task. You do NOT write code. You produce a translation plan from existing design sources to a SwiftUI surface that another subagent will implement.

## Output Path
Write the brief to `.tasks/phase-N/task-N-design.md`. Do not write anywhere else.

## Output Budget
≤7,000 characters (~1,700 tokens). If you find yourself wanting more, summarize. If you can't summarize without losing critical information, the task is too big — report `TASK_TOO_BIG` and stop.

## Design Source Hierarchy (strict authority order)

1. **`docs/mockups/<screen>.jsx`** (or `.tsx`, `.html`) — pixel/copy/layout truth. Highest authority. If it exists for this surface, you MUST cite the relevant line range and translate from it.
2. **`docs/DESIGN_SYSTEM.md`** — token + primitive truth. Cite by `§ X.Y`. Never duplicate content; cite only.
3. **`docs/DESIGN.md`** — brand spec, read-only. Cite rarely (when DESIGN_SYSTEM doesn't cover the question).
4. **`docs/UX.md`** — IA truth. Which sections this surface contains, anti-patterns.
5. **`planning/features/<feature>.md`** (or section of `planning/FEATURES.md`) — behavior, states, copy locks, anti-patterns.

## The Task
[Full text of the task from TASKS.md — paste verbatim]

## Surface Identification
[The screen / component this task creates or modifies — be specific. Example: "Settings → Account section, accessible from the gear icon on the main tab bar"]

## Read These Files (and only these)
- docs/UX.md → § [section number]
- docs/DESIGN_SYSTEM.md (full, but cite by § only)
- docs/DESIGN.md (read-only reference)
- docs/mockups/[file].jsx → lines [N-N] if present, else flag as missing
- planning/features/[feature].md (single feature, full)
- The matching context brief at .tasks/phase-N/task-N-context.md (read for task scope context only — do not duplicate it in your output)

Do NOT read other tasks' briefs. Do NOT read full ARCHITECTURE.md (the context brief carries the relevant excerpts).

## Brief Structure (use exactly this template)

```markdown
# Task N — [Task Title] — Design Brief

## Mockup source
- File: docs/mockups/[file]
- Region: lines [N-N] (or selector / image region label)
- Component(s): [exported function names if JSX/TSX, or selector hierarchy if HTML]
- Translation target: [path/to/the/SwiftUI/file.swift]

(If mockup is missing for this surface:)
## Mockup source
- MISSING — surface not mocked in docs/mockups/. Falling back to shape interview.

## Token mapping (mockup value → project theme tokens)
The token namespace comes from the project's theme module (declared in DESIGN_SYSTEM.md). Cite the DESIGN_SYSTEM § that defines each token.

| Mockup value | Theme token | DESIGN_SYSTEM § |
|---|---|---|
| `#XXXXXX` background | `theme.Colors.<role>` | § 2.x |
| `#XXXXXX` accent | `theme.Colors.<role>` | § 2.x |
| Typography (font / size / weight) | `theme.Typography.<role>` | § 2.x |
| Spacing values | `theme.Spacing.<role>` | § 2.x |

## Locked primitives (cite — do not rebuild)
[List the design-system primitives this surface uses, citing DESIGN_SYSTEM § for each.]
- ExampleField — DESIGN_SYSTEM § 3.x
- ExampleButton (primary variant) — DESIGN_SYSTEM § 3.x

## Variant coverage

Read DESIGN_SYSTEM.md for the variant axes this project defines. iOS axes are typically narrow (often a single form factor, often dark-only or light-only by project convention) — usually data states and motion behavior.

Skip this section entirely for trivial atoms with no variants:
- `## Variant coverage`
- `N/A — stateless atom`

Otherwise, three lists:

### Cells the mockup locks
- [Default state shown in mockup — cite mockup region]

### Cells the implementer derives from DESIGN_SYSTEM
- [State 1] → § X.Y [pattern name]
- [State 2] → § X.Y [pattern name]

### Cells with no DS coverage (net-new)
- [Behavior the mockup hints at but DS doesn't have a primitive for]

## Net-new design (motion / behavior the mockup can't show)
- [List behavior the static mockup doesn't capture: animations, scenePhase rules, timing-sensitive transitions]
- (May be empty — leave blank if the surface is static)

## Mockup ↔ DESIGN_SYSTEM gaps
- (none) — or:
- [Mockup uses value X; DESIGN_SYSTEM § Y says different. RESOLVE BEFORE IMPLEMENTATION.]
  - Option A: update DESIGN_SYSTEM (mockup is canonical)
  - Option B: update mockup (DS is canonical)
  - Recommendation: [your read]

## Anti-copy / anti-pattern (from feature spec § X)
[Pull from `planning/features/<feature>.md` § Anti-patterns or § Important. Don't invent these.]
- [Anti-pattern 1 quoted from the feature spec]
- [Anti-pattern 2]

## Navigation script for visual gate
[How to reach this screen from app launch — used by the downstream visual fidelity reviewer (`run-task-design-ios`'s visual gate). Must be executable: every step is something the simulator subagent can perform via tap/type/swipe.]

The script assumes the visual gate has already done the simulator handshake (`mcp__ios-simulator__get_booted_sim_id`) and is targeting the controller-booted simulator. Do not include "boot a simulator" or `open_simulator` steps — those are forbidden at the subagent layer.

1. Launch app on the booted simulator: `mcp__ios-simulator__launch_app`
2. [Tap path to reach the screen]
3. [Land on the screen this task implements]

## Finisher anchors (for the downstream finisher pass — `run-task-design-ios`'s per-task finisher dispatch)
- harden: [if applicable, why — e.g. "force-quit during form entry must preserve state on relaunch"]
- animate: [if applicable, why — e.g. "transition timing X, Reduced Motion fallback Y"]
- clarify: [if applicable, why — e.g. "novel copy needs a final pass against project's voice rules"]
- typeset: [if applicable]
- arrange: [if applicable]
```

## How to do this well

1. **Open the mockup first.** Read the JSX. Find the component(s) for the surface. Read the surrounding components to understand context (StatusBar, OverlayChrome, etc.) — but only cite what's relevant to this task.
2. **Read DESIGN_SYSTEM.md to learn the project's variant axes.** iOS axes are narrower than web — usually just data states and motion. Use what the project actually defines; don't invent axes.
3. **Build the token map by hex.** Every hex literal in the mockup region maps to a token in the project's theme module. If there's no token for a hex, that's a `Mockup ↔ DESIGN_SYSTEM gap`.
4. **Cite primitives, don't describe them.** "PTextField § 3.6" is enough. The implementer reads § 3.6.
5. **Variant coverage forces the implicit cells explicit.** Mockups are static; production has loading, error, empty, and motion states. Skip the section entirely for stateless atoms.
6. **Net-new is the smallest possible list.** Anything already in the mockup or DESIGN_SYSTEM is locked. Only add to net-new what truly isn't covered.
7. **Anti-copy is from the feature spec.** Don't invent anti-patterns. Pull from `planning/features/<feature>.md` § Anti-patterns or § Important.
8. **Navigation script must be executable.** The visual gate subagent uses this verbatim. If you can't write a script (e.g., the screen is reached via a non-deterministic path), report `NEEDS_CONTEXT`.
9. **Gap detection is your highest-value contribution.** When the mockup and DESIGN_SYSTEM disagree, flag it. The user resolves. This keeps the two artifacts converging.

## If the surface has no mockup

Run a fallback shape interview. Output the brief with `## Mockup source: MISSING` and replace the token-mapping table with a `## Net-new design (full shape brief)` section that includes:

- Color strategy (Restrained / Committed / Full palette / Drenched)
- Theme scene sentence (one line of physical context)
- Two or three named anchor references
- Layout strategy (sections, spatial rhythm)
- Variant coverage (using whatever axes DESIGN_SYSTEM defines — typically data states + motion on iOS)
- Reduced Motion behavior

This is the only path that fully describes design choices in the brief itself. When a mockup exists, the brief stays a translation plan.

## Boundaries
- Do NOT write code.
- Do NOT modify any files except `.tasks/phase-N/task-N-design.md`.
- Do NOT read other tasks' briefs.
- Do NOT propose mockup edits inline — flag gaps and let the user resolve.

## Report Format
When done, report to the calling skill (`generate-design-briefs`):
- **Status:** DONE | NEEDS_CONTEXT | TASK_TOO_BIG | MOCKUP_DS_GAP_BLOCKING | NEEDS_USER_RESOLUTION (cite the unresolved cells)
- **Output path:** `.tasks/phase-N/task-N-design.md`
- **Mockup status:** PRESENT | MISSING (fallback shape brief written)
- **Variant coverage:** included | N/A (stateless atom)
- **Gaps flagged:** [count] — list any blocking gaps that need user resolution before implementation
- **Estimated implementer complexity:** Mechanical | Integration | Architectural (drives downstream model selection in `run-task-design-ios`)
```

## Isolation rules (`generate-design-briefs` enforces)

- Design briefs live only at `.tasks/phase-N/task-N-design.md` — never shared paths.
- `generate-design-briefs` confirms the file was written but does not read its contents. Downstream `run-task-design-ios` reads it by path.
- If the subagent reports `MOCKUP_DS_GAP_BLOCKING` or `NEEDS_USER_RESOLUTION`, `generate-design-briefs` surfaces the gap to the user and pauses subsequent task briefs until resolved. The user decides whether to fill thin design sources manually (including by invoking `impeccable:shape <surface>`); the skill never does this automatically.

# Design Brief Subagent Prompt Template

Used by `generate-design-briefs` for each `domain: design` task. Spawn a fresh-context subagent that reads the project's design sources and writes a per-task design brief.

This skill does NOT call `impeccable:shape`. If the design source for a task is thin (no mockup, sparse DESIGN_SYSTEM coverage, novel IA), report `NEEDS_USER_RESOLUTION` for the unresolved cells. The user decides whether to invoke `impeccable:shape <surface>` manually to fill the gap and then re-run with `--force`.

The brief is **not** prose. It's a translation plan: mockup region (or written design direction when no mockup exists) → component implementation, with explicit token mappings, primitive citations, a variant matrix (breakpoints × themes × states), and a list of net-new design decisions.

```
## Your Role
You are generating a design brief for a web implementation task. You do NOT write code. You produce a translation plan from existing design sources to a component / page that another subagent will implement.

## Output Path
Write the brief to `.arsenal/tasks/phase-N/task-N-design.md`. Do not write anywhere else.

## Output Budget
≤7,000 characters (~1,700 tokens). If you find yourself wanting more, summarize. If you can't summarize without losing critical information, the task is too big — report `TASK_TOO_BIG` and stop.

## Design Source Hierarchy (strict authority order)

1. **Mockup files** at `.arsenal/design/mockups/<screen>.{jsx,tsx,html,png,figma-export.json,v0-export.tsx}` — pixel/copy/layout truth when present. Highest authority for the surface.
2. **`.arsenal/design/DESIGN_SYSTEM.md`** — token + primitive truth. Cite by `§ X.Y`. Never duplicate content; cite only.
3. **`.arsenal/design/DESIGN.md`** — brand spec, read-only. Cite rarely (when DESIGN_SYSTEM doesn't cover the question).
4. **`.arsenal/design/UX.md`** — IA truth (page sections, conversion model, anti-patterns).
5. **`.arsenal/features/<feature>.md`** (or section of `.arsenal/FEATURES.md`) — behavior, states, copy locks, anti-patterns.

## Mockup formats you may encounter

- **JSX/TSX prototypes** (`.arsenal/design/mockups/onboarding.tsx`) — read like code; cite line ranges.
- **HTML prototypes** (`.arsenal/design/mockups/landing.html`) — read like markup; cite element selectors or line ranges.
- **Static images** (`.arsenal/design/mockups/checkout-mobile.png`) — describe what's visible by region (top hero, mid form, footer CTA). Use the mockup file path + region label.
- **v0 / Stitch / Lovable / Bolt exports** (`.arsenal/design/mockups/dashboard-v0.tsx`) — usually TSX with inline Tailwind classes; cite line ranges and call out non-portable patterns (e.g., shadcn imports the project may not have).
- **Figma export JSON** (`.arsenal/design/mockups/billing.figma-export.json`) — extract token values and structure from the JSON; cite the frame name.

## The Task
[Full text of the task from TASKS.md — paste verbatim]

## Surface Identification
[The page / component / route this task creates or modifies — be specific. Example: "Landing → Hero section + email capture form (above the fold), at app/page.tsx"]

## Read These Files (and only these)
- .arsenal/design/UX.md → § [section number]
- .arsenal/design/DESIGN_SYSTEM.md (full, but cite by § only)
- .arsenal/design/DESIGN.md (read-only reference)
- .arsenal/design/mockups/[file] → region [line range OR element selector OR image region label] if present, else flag as missing
- .arsenal/features/[feature].md (single feature, full)
- The matching context brief at .arsenal/tasks/phase-N/task-N-context.md (read for task scope context only — do not duplicate it in your output)

Do NOT read other tasks' briefs. Do NOT read full ARCHITECTURE.md (the context brief carries the relevant excerpts).

## Brief Structure (use exactly this template)

```markdown
# Task N — [Task Title] — Design Brief

## Mockup source
- File: .arsenal/design/mockups/[file]
- Region: [lines N-N | selector `.hero` | image region "top hero"]
- Component(s) shown: [exported function names if JSX/TSX, or selector hierarchy if HTML]
- Translation target: [app/page.tsx | components/Hero.tsx | etc.]

(If mockup is missing for this surface:)
## Mockup source
- MISSING — surface not mocked in .arsenal/design/mockups/. Falling back to shape interview.

## Token mapping (mockup value → DESIGN_SYSTEM)
| Mockup value | Token / class | DESIGN_SYSTEM § |
|---|---|---|
| `#0B0F14` | `var(--background-deep)` / `bg-background` | § 2.1 |
| `#22D3EE` | `var(--accent)` / `text-accent` | § 2.1 |
| Inter 600 28/36 | `text-3xl font-semibold tracking-tight` | § 2.4 |
| 16px gap | `gap-4` (Tailwind 4 = 16px) | § 2.5 |
| ... | ... | ... |

## Locked primitives (cite — do not rebuild)
- Button — DESIGN_SYSTEM § 3.2 (variants: primary, ghost, link)
- Input — DESIGN_SYSTEM § 3.6 (with character counter variant)
- Card — DESIGN_SYSTEM § 3.4

## Variant coverage

Mockups capture one cell of the project's variant space (e.g., desktop · light · default state). The implementer needs to know what to ship for the cells the mockup doesn't show.

**Read DESIGN_SYSTEM.md to discover what variant axes this project actually defines.** Don't invent axes the project doesn't have. Common axes include responsive breakpoints, theme modes, pointer/focus states, data states — but exact set varies per project.

Skip this section entirely for trivial atoms with no variants (e.g., a divider, an icon, a static label):
- `## Variant coverage`
- `N/A — stateless atom`

Otherwise, three lists:

### Cells the mockup locks
[What the mockup actually shows — be specific.]
- desktop · light · default state (mockup line 42-89)

### Cells the implementer derives from DESIGN_SYSTEM
[Cells the project covers via design-system patterns — cite §.]
- Hover / focus-visible → § 3.2 button state patterns
- Loading state → § 4.1 inline spinner pattern
- Mobile layout → § 5.2 responsive scaling rules
- Dark theme token swap → § 2.6

### Cells with no DS coverage (net-new)
[Anything implemented for this surface that doesn't exist in DS yet. May be empty.]
- Empty-state copy + illustration for first-load
- Cross-page fade transition (200ms)

## Net-new design (only what the mockup + DESIGN_SYSTEM don't cover)
- [List the genuinely novel decisions: a transition the design system doesn't have, a copy variant, a layout the mockup hints at but doesn't show]
- Cross-page transition pattern: 200ms fade-in on route entry
- Reduced motion: disable transition, instant render

## Mockup ↔ DESIGN_SYSTEM gaps
- (none) — or:
- Mockup uses 12px corner radius on Card; DESIGN_SYSTEM § 3.4 says 8px. RESOLVE BEFORE IMPLEMENTATION.
  - Option A: update DESIGN_SYSTEM § 3.4 to 12px (mockup is canonical)
  - Option B: update mockup to 8px (DS is canonical)
  - Recommendation: [your read]

## Anti-copy / anti-pattern (from feature spec § X)
- No "Submit" buttons — CTAs describe outcomes ("Get my free plan", "Start my trial")
- No "We're excited to announce" / "The world's first" / "Leveraging cutting-edge"
- Above-the-fold loads in <1s; no JS-blocking fonts
- Single primary CTA per section — no competing visual weight

## Where this surface renders (for the user's dev-server check)
[How to reach this surface from app launch — used by the user when manually verifying the implementation in a dev server, and by any optional impeccable audit at `close-design-phase`.]

1. Open the dev server URL: http://localhost:3000
2. Navigate to /pricing
3. Scroll to the "Compare plans" section
4. The component this task implements is the comparison table.

```

## How to do this well

1. **Open the mockup first.** If JSX/TSX, read it. If image, describe what you see by region. If Figma JSON, extract the frame's tokens and structure.
2. **Read DESIGN_SYSTEM.md to learn the project's variant axes.** Some projects define responsive breakpoints, themes, and state modes; others don't. Use what the project actually defines — don't invent axes.
3. **Build the token map.** Every value in the mockup region maps to a CSS var, Tailwind class, or design-system primitive. Values that don't map are `Mockup ↔ DESIGN_SYSTEM gaps`.
4. **Cite primitives, don't describe them.** "Button § 3.2" is enough. The implementer reads § 3.2.
5. **Variant coverage is your highest-value contribution when it applies.** Mockups capture one cell of the project's defined variant space; implementer needs the rest. Make explicit which cells the mockup locks vs which derive from DESIGN_SYSTEM. Skip this section entirely for trivial atoms.
6. **Net-new is the smallest possible list.** Anything already in the mockup or DESIGN_SYSTEM is locked. Only add to net-new what truly isn't covered.
7. **Anti-copy is from the feature spec.** Don't invent anti-patterns. Pull from `.arsenal/features/<feature>.md` § Anti-patterns or § Important.
8. **Gap detection is critical.** When the mockup and DESIGN_SYSTEM disagree, flag it. The user resolves. This keeps the two artifacts converging.

## If the surface has no mockup

Run a fallback shape interview. Output the brief with `## Mockup source: MISSING` and replace the token-mapping table with a `## Net-new design (full shape brief)` section that includes:

- Color strategy (Restrained / Committed / Full palette / Drenched)
- Theme scene sentence (one line of physical context)
- Two or three named anchor references
- Layout strategy (sections, spatial rhythm, behavior at the project's defined breakpoints if any)
- Variant coverage (using whatever axes DESIGN_SYSTEM defines — could be just data states, or full breakpoint × theme × state space)
- Reduced Motion behavior (if the project commits to it)

This is the only path that fully describes design choices in the brief itself. When a mockup exists, the brief stays a translation plan.

## Boundaries
- Do NOT write code.
- Do NOT modify any files except `.arsenal/tasks/phase-N/task-N-design.md`.
- Do NOT read other tasks' briefs.
- Do NOT propose mockup edits inline — flag gaps and let the user resolve.

## Report Format
When done, report to the calling skill (`generate-design-briefs`):
- **Status:** DONE | NEEDS_CONTEXT | TASK_TOO_BIG | MOCKUP_DS_GAP_BLOCKING | NEEDS_USER_RESOLUTION (cite the unresolved cells)
- **Output path:** `.arsenal/tasks/phase-N/task-N-design.md`
- **Mockup status:** PRESENT | MISSING
- **Variant coverage:** included | N/A (stateless atom)
- **Gaps flagged:** [count] — list any blocking gaps that need user resolution before implementation
- **Estimated implementer complexity:** Mechanical | Integration | Architectural (drives downstream model selection in `run-task-design`)
```

## Isolation rules (`generate-design-briefs` enforces)

- Design briefs live only at `.arsenal/tasks/phase-N/task-N-design.md` — never shared paths.
- `generate-design-briefs` confirms the file was written but does not read its contents. Downstream `run-task-design` reads it by path.
- If the subagent reports `MOCKUP_DS_GAP_BLOCKING` or `NEEDS_USER_RESOLUTION`, `generate-design-briefs` surfaces the gap to the user and pauses subsequent task briefs until resolved. The user decides whether to invoke `impeccable:shape <surface>` manually to fill thin design sources; the skill never does this automatically.

## Why variant coverage matters

A mockup captures one cell of the project's variant space. Production needs all the cells the project commits to. Without explicit coverage, the implementer ships the locked cell beautifully and silently drifts on hover, focus, error, mobile, dark — whatever the project actually defines.

The shape of the variant space is **the project's call, not the skill's**. A mobile-only PWA might have one breakpoint and no dark mode. A marketing site might have five breakpoints and a high-contrast theme. An internal tool might have only data states. Read DESIGN_SYSTEM.md, use the axes the project actually defines, and skip variant coverage entirely for stateless atoms.

The discipline isn't the matrix — it's making the implicit cells explicit. Three lists (locked / derived / net-new) does that without forcing a structure that doesn't fit every project.

---
name: anchor-files
description: Consolidates upstream planning artifacts (FEATURES, UX, DESIGN, mockups) into the agent-readable reference layer that anchors every downstream build session — CLAUDE.md (root index + load-bearing rules) plus docs/{ARCHITECTURE,CONVENTIONS,DESIGN_SYSTEM,TASKS}.md. Hard-requires upstream planning to exist; refuses and routes to the right plan-* skill when artifacts are missing. Output is structured for slicing by brief generators downstream, not human reading. Use when planning is complete and you're ready to set up the project for the build pipeline. Triggers include "set up project anchor", "anchor the codebase", "scaffold project docs", "set up CLAUDE.md", "bridge planning to build".
---

# Execute Anchor Files

Consolidate upstream planning into the **agent reference layer** that anchors every downstream session: `CLAUDE.md` at root, plus `docs/ARCHITECTURE.md`, `docs/CONVENTIONS.md`, `docs/DESIGN_SYSTEM.md`, `docs/TASKS.md`. These files are read by every Claude session opening the project and sliced by the brief generators that drive `design-{web,ios}` and `features-{web,ios}`.

This is not "generate docs." It's "design the structured reference surface that every downstream agent reads from for the lifetime of the project."

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

## Role

- **Bridge** between planning (`arsenal-planning` skills family) and execution (`design-*` / `features-*`).
- **Anchor** for every Claude session — `CLAUDE.md` is the load-bearing index; the rest are sliced by brief generators.
- **Consolidator, not generator.** Upstream artifacts own their content; this skill cross-references, never restates.

## Preconditions (hard-required)

This skill refuses to run when required upstream artifacts are missing. The build pipeline depends on them; soft-prompting was the failure mode that motivated this rule.

| Artifact | Required for | If missing |
|---|---|---|
| `planning/FEATURES.md` or `planning/features/` | All projects | Stop. *"Run `features` first — `anchor-files` consolidates feature specs into ARCHITECTURE.md and TASKS.md; can't proceed without them."* |
| `docs/UX.md` | UI projects only | Stop. *"Run `ux-{web,app,ios}` first — `anchor-files` consolidates the page/screen inventory into TASKS.md and DESIGN_SYSTEM.md."* |
| `docs/DESIGN.md` | UI projects only | Stop. *"Run `design` first — `anchor-files` consolidates brand tokens into DESIGN_SYSTEM.md."* |
| `planning/MVP_SPEC.md` | Optional context | Read if present; skip if not. No prompt. |
| `docs/mockups/` | Recommended for UI | Soft prompt: *"No mockups detected. The design pipeline is substantially stronger when concrete mockups are present. Run `mockups` to generate two-pass anchor briefs, then feed each into Claude Design / Stitch / Open Design / v0 and save outputs to `docs/mockups/`. Continuing without?"* |

Non-UI projects (CLI, library, API, server-only) skip the UX/DESIGN/mockup requirements entirely.

## What gets generated

| File | Lines (target) | Sliced by |
|---|---|---|
| `CLAUDE.md` (root) | ≤120 hard ceiling | Every session at load |
| `docs/ARCHITECTURE.md` | ≤500 | `generate-feature-briefs` (data flow + schema cites) |
| `docs/CONVENTIONS.md` | ≤500 | `generate-feature-briefs` (pattern cites) |
| `docs/DESIGN_SYSTEM.md` | ≤500 (UI only) | `generate-design-briefs` (token map + component cites) |
| `docs/TASKS.md` | no cap | `expand-phase`, `close-*-phase-*` (read + write) |

## Workflow

### Step 1: Verify preconditions

Check upstream artifacts. On any required miss, **stop** with the routing message from the preconditions table. Do not enter a discovery interview; do not generate any files; do not fall back.

```bash
# UI project preconditions
ls planning/FEATURES.md planning/features/README.md docs/UX.md docs/DESIGN.md 2>/dev/null

# Non-UI project preconditions
ls planning/FEATURES.md planning/features/README.md 2>/dev/null

# Soft check (UI projects)
ls docs/mockups/ 2>/dev/null
```

**Resolve features mode:** if both `FEATURES.md` and `planning/features/` exist (mid-migration), ask which is canonical. If only one exists, use it.

**Resolve UX split mode:** if `docs/ux/` directory exists alongside `docs/UX.md`, treat the parent file as an index and sub-files as the slice content.

**Resolve project type:** UI vs non-UI. If a `ux-*` was run, this is a UI project. Otherwise treat as non-UI (CLI, library, API, server-only).

If `MVP_SPEC.md` is present, read it for strategic context (target user, value loop, distribution hypothesis, success metrics) — useful for TASKS.md goal sentences and Phase 0 analytics events. Don't restate its content; cross-reference where useful.

### Step 2: Detect surface (UI projects only)

Determine surface from upstream `UX.md` content or stack hints:

- **Web (marketing/site)** → `ux-web` was upstream
- **Web (app, authenticated)** → `ux-app` was upstream
- **iOS (native)** → `ux-ios` was upstream
- **Hybrid** → multiple UX docs or split `ux/` folder

Surface drives CSS-vs-SwiftUI in `DESIGN_SYSTEM.md`, web-vs-iOS handoff phrasing in `TASKS.md`, and which stack-discovery questions apply.

### Step 3: Stack discovery (gap-only)

Ask **only** about stack-level decisions that upstream artifacts can't answer. Six to eight questions max. Skip any whose answer is obvious from context (existing `package.json`, `*.xcodeproj`, `Package.swift`, etc.).

**Stack:**
- Framework? (Next.js / Astro / SwiftUI / Vite + React / Expo / etc.)
- Styling? (Tailwind / CSS Modules / styled-components — web only)
- Component library? (shadcn/ui / custom / none — web only)
- Database / BaaS / CMS? (Postgres / Supabase / Sanity / Contentful / etc.) — skip database if CMS is the primary data layer
- Auth? (Supabase Auth / NextAuth / Clerk / Sanity / custom)
- State management? (TanStack Query / Zustand / Redux / `@Observable`) — skip for Astro static; iOS uses `@Observable`
- Hosting? (Vercel / Cloudflare / Netlify / App Store)
- Key integrations? (Stripe / Resend / OpenAI / PostHog / etc.)

**Project intent (drives Phase 0):**
- Intent? Pick one:
  - **hobby** → skip Phase 0 entirely
  - **freelance / startup** → Phase 0 includes landing page (opt-in) + PostHog analytics
  - **client-work** → Phase 0 includes landing page only if scoped in the brief; no analytics by default (the client owns post-launch instrumentation)

### Step 4: Generate anchor files

Dependency order: ARCHITECTURE → CONVENTIONS → DESIGN_SYSTEM → TASKS → CLAUDE.

#### `docs/ARCHITECTURE.md`

System design only. Each section is self-contained — slicing target for `generate-feature-briefs`.

- **System Overview** — ASCII diagram of major components and their relationships
- **Project Structure** — folder layout adapted to the framework
- **Data Flow** — per major user journey, traced through the stack
- **Schema** — full schema in code blocks (SQL / Swift `@Model` types / Sanity schema). Cross-reference `planning/features/<slug>.md § Data` for per-feature lifecycles; do not restate.
- **Integrations** — third-party services with env vars
- **State Management** — strategy + query key conventions (web) or state architecture (iOS)
- **MVP vs Post-MVP boundary** — what's intentionally excluded

Cross-reference patterns (precise paths, not vague references):
- *"User-Facing Architecture: see `docs/UX.md`"*
- *"For per-feature data lifecycles, see `planning/features/<slug>.md § Data`"*

#### `docs/CONVENTIONS.md`

Stack-specific code patterns only. Real code blocks, never descriptions of patterns. Slicing target for `generate-feature-briefs`.

- **Core Development Philosophy** — KISS / YAGNI / Functional First, with good-vs-bad code examples
- **Before Writing Any Code** — the load-bearing checklist (check TASKS.md scope; reference CONVENTIONS for patterns; no temporary workarounds like `// TODO: fix later`, `any` types "for now", or hardcoded values without constants; ask before major decisions)
- **Patterns per Concern** — components, data fetching, mutations, forms, error handling (web); SwiftUI views, navigation, data flow, networking, Swift conventions (iOS)
- **Library Reference** — table with versions + docs links
- **Anti-Patterns** — stack-specific don'ts

Stack-specific guidance:
- **Astro:** `.astro` component patterns, island hydration directives (`client:load`, `client:visible`, `client:idle`), content collections, Sanity GROQ queries if applicable, `<Image>` usage.
- **Native iOS:** `@Observable` not `ObservableObject`, Swift Concurrency only (no Combine in new code), `final class` on `@Model` types, hex literal lockdown (theme module only), accessibility identifiers on new interactive views.

#### `docs/DESIGN_SYSTEM.md` (UI projects only)

Stack-specific implementation of `docs/DESIGN.md`. **Never restates DESIGN.md's tokens** — references them. Slicing target for `generate-design-briefs`.

Required sections:

- **Source of Truth** (1 paragraph): brand tokens, typography, motion, voice live in `docs/DESIGN.md`. This file translates that spec into [stack]. **Do not edit `DESIGN.md` directly** — re-run `design` to update the brand.
- **Token Map** — table mapping brand role → stack token. **No value restatement.** Example: `surface.canvas → --color-canvas` (web) or `surface.canvas → Colors.canvas` (iOS), not `surface.canvas → #FAFAFA → --color-canvas`. Readers cross-reference `DESIGN.md` when they need values.
- **Components** — one entry per component named in UX.md's "Components needed" lists. Format: *"Brand rules: see `DESIGN.md` §X. Implementation: [stack-specific code/classes/modifiers/props]."*
- **Mockups directory** — reference to `docs/mockups/` and how the design pipeline consumes them
- **Accessibility patterns** — stack-specific (ARIA conventions on web; `accessibilityIdentifier` on iOS)

If any component named in UX.md isn't defined here, flag it for the user — never invent.

Skip this file entirely for non-UI projects.

#### `docs/TASKS.md`

Phase scaffold and orchestrator state machine. Mutable. Predictable structure so `expand-phase` and `close-*-phase-*` can read/write programmatically.

**Top of file — critical reminder block (verbatim):**

```markdown
> **⚠️ CRITICAL:** Update this file after completing work. Mark tasks `[x]` when done.
> **📦 Tasks expand on-demand:** Phase tasks are placeholders until you start work on the phase.
> **🎨 Each phase splits design + feature:** Design tasks (hardcoded data) run first via `design-{web,ios}`; feature tasks (data wiring) run second via `features-{web,ios}`. Shared branch, one PR per phase. Design close completes before any feature task starts.
> **🔒 Component boundary:** Once a design task commits a component, the feature pipeline may **extend** it (new props/variants/states the data layer needs) but never **redesign** it (visual treatment / spacing / typography / color / motion). Feature commits that extend a component must include a `Component extended:` note. Purely visual changes BLOCK with redirect to the design pipeline.
> **📝 Spec amendment:** The feature pipeline may amend `planning/FEATURES.md` / `planning/features/*.md` in-line for **Tier 1** changes (additive/clarifying — missed state, edge case, threshold within same intent, missing dependency). Such commits carry a `Spec amended:` note. **Tier 2** changes (user-facing intent shifts) BLOCK and route back through `features`.
```

**Phase structure:**

- **Phase 0 (non-hobby, non-client-work projects, or client-work with scoped landing page):** Pre-launch foundation. **Concrete tasks, not placeholders.** Landing page tasks (if opted in) + PostHog analytics tasks (startup/freelance only). Events derived from FEATURES Job Stories + UX Primary CTAs.
- **Phase 1+:** When `UX.md` exists, phases are page-by-page in conversion-priority order. When only FEATURES exists (non-UI), phases are by capability-dependency order. Each phase header:
  - Status (Not started / In progress / Completed)
  - UX pages covered (precise paths)
  - FEATURES capabilities covered (precise paths)
  - Goal (1 sentence)
  - `### Design tasks` placeholder (skip for non-UI / pure feature-domain phases — write `_None — pure feature-domain phase._`)
  - `### Feature tasks` placeholder

**Phase scaffold template:**

```markdown
## Phase 1: [Name]

**Status:** Not started
**UX pages covered:** [Pages with paths]
**FEATURES capabilities covered:** [Features with paths]
**Goal:** [One sentence]

### Design tasks
- [ ] Tasks not yet generated. Run `design-{web,ios} 1` to expand visual components from UX.md, DESIGN_SYSTEM.md, and mockups.

### Feature tasks
- [ ] Tasks not yet generated. Run `features-{web,ios} 1` to expand data-wiring tasks. Runs after design close.
```

**Tail of file:**

- **Quick Reference** — common commands, env vars, doc paths
- **Key Decisions Log** — table for decisions made during build (date / decision / rationale)

#### `CLAUDE.md` (root)

Session-load anchor. **≤120 lines hard ceiling.** Every session pays this cost; subagents inherit through it.

Required structure:

1. **What this project is** (1 paragraph from MVP_SPEC if present, else FEATURES.md header)
2. **Stack summary** (1 paragraph)
3. **Read in order** — ordered list with one-line purpose each:
   - `docs/TASKS.md` for current work and phase status
   - `docs/ARCHITECTURE.md` for system design and data flow
   - `docs/CONVENTIONS.md` for code patterns and library versions
   - `docs/UX.md` for page/screen inventory (UI only)
   - `planning/features/README.md` then specific feature files (split mode) or `planning/FEATURES.md` (single mode)
   - `docs/DESIGN.md` for brand spec — **DO NOT EDIT** (UI only)
   - `docs/DESIGN_SYSTEM.md` for stack implementation of DESIGN.md (UI only)
4. **Load-bearing rules** — verbatim, inherited by every subagent:
   - **Component boundary** (full rule with `Component extended:` format)
   - **Spec amendment** (full Tier 1 / Tier 2 distinction with commit-note formats)
   - **Tasks expand on-demand + design-then-feature ordering**
5. **Active phase pointer** — which phase is in progress, which is next
6. **Project-specific rules** — naming conventions, do/don't patterns the user names

Keep it tight. Every line is paid for in every session.

### Step 5: Set up mockup slot

```bash
mkdir -p docs/mockups
```

For UI projects without existing mockups, surface the slot in the final report. Recommend Claude Design / Stitch / v0 / future `generate-mockups` (planned around `nexu-io/open-design`) before running `design-*`.

### Step 6: Review & handoff

Final report:
- **Files created** with line counts (`wc -l`). Flag any file exceeding its target.
- **Upstream artifacts consumed** (which planning docs informed which sections)
- **First phase to run** — `design-{web,ios} 1` or, for non-UI / pure feature-domain phases, `features-{web,ios} 1`
- **Mockup status** — if `docs/mockups/` is empty for a UI project: *"Generate mockups in `docs/mockups/` before running `design-*`. Recommended tools: Claude Design, Stitch, v0."*

## Anti-patterns

- **Don't fall back when required artifacts are missing.** Refuse and route to the right `arsenal-planning` skills skill. The build pipeline depends on planning being complete.
- **Don't ask questions whose answers are already in upstream artifacts.** Read first. Discovery is stack-only.
- **Don't restate upstream content.** Cross-reference with precise paths (`docs/UX.md § Onboarding`, `planning/features/recipe-capture.md § Data`). Token budget is real.
- **Don't write load-bearing rules in secondary docs.** CLAUDE.md only — subagents inherit through it. ARCHITECTURE / CONVENTIONS / DESIGN_SYSTEM may reference rules but never define them.
- **Don't exceed CLAUDE.md's 120-line ceiling.** Every session pays the cost. Split out anything that doesn't earn its place.
- **Don't ship a UI project without setting up `docs/mockups/`.** The slot exists for a reason. Surface its absence in the final report.

## File placement

```
project-root/
├── CLAUDE.md                     # root index + load-bearing rules
├── planning/
│   ├── MVP_SPEC.md               # optional, from mvp
│   ├── FEATURES.md               # from features (single mode)
│   └── features/                 # from features (split mode)
│       ├── README.md
│       └── <slug>.md per feature
└── docs/
    ├── ARCHITECTURE.md
    ├── CONVENTIONS.md
    ├── UX.md                     # from ux-* (UI projects)
    ├── ux/                       # split mode (UX.md is index)
    │   └── <page>.md per page
    ├── DESIGN.md                 # from design (UI projects, do-not-edit)
    ├── DESIGN_SYSTEM.md          # stack implementation of DESIGN.md (UI projects)
    ├── TASKS.md
    └── mockups/                  # consumed by design-{web,ios}
```

## Important guidelines

- **Be specific, not generic.** Outputs are tailored to this exact project — no boilerplate filler.
- **Use real code.** Every pattern in CONVENTIONS.md is actual working code for the chosen stack.
- **Version awareness.** Use the API patterns for the library versions specified. Don't mix v4 and v5.
- **Cap each file at its target.** If a file would exceed, split into topical sub-files (e.g., `ARCHITECTURE.md` → `+ DATA_MODEL.md + INTEGRATIONS.md`) and cross-link from a short parent index. Apply during generation — don't write 900 lines first and split after.

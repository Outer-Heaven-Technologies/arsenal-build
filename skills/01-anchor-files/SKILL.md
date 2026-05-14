---
name: anchor-files
description: Consolidates upstream planning artifacts (FEATURES, UX, DESIGN, mockups) into the agent-readable reference layer that anchors every downstream build session — CLAUDE.md (root index + load-bearing rules) plus .arsenal/{ARCHITECTURE,CONVENTIONS,TASKS}.md and .arsenal/design/DESIGN_SYSTEM.md. Hard-requires upstream planning to exist; refuses and routes to the right plan-* skill when artifacts are missing. Output is structured for slicing by brief generators downstream, not human reading. Use when planning is complete and you're ready to set up the project for the build pipeline. Triggers include "set up project anchor", "anchor the codebase", "scaffold project docs", "set up CLAUDE.md", "bridge planning to build".
---

# Execute Anchor Files

Consolidate upstream planning into the **agent reference layer** that anchors every downstream session: `CLAUDE.md` at root, plus `.arsenal/ARCHITECTURE.md`, `.arsenal/CONVENTIONS.md`, `.arsenal/design/DESIGN_SYSTEM.md`, `.arsenal/TASKS.md`. These files are read by every Claude session opening the project and sliced by the brief generators that drive `design` and `features`.

This is not "generate docs." It's "design the structured reference surface that every downstream agent reads from for the lifetime of the project."

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

## Role

- **Bridge** between planning (`arsenal-planning` skills family) and execution (`design` / `features`).
- **Anchor** for every Claude session — `CLAUDE.md` is the load-bearing index; the rest are sliced by brief generators.
- **Consolidator, not generator.** Upstream artifacts own their content; this skill cross-references, never restates.

## Preconditions (hard-required)

This skill refuses to run when required upstream artifacts are missing. The build pipeline depends on them; soft-prompting was the failure mode that motivated this rule.

| Artifact | Required for | If missing |
|---|---|---|
| `.arsenal/FEATURES.md` or `.arsenal/features/` | All projects | Stop. *"Run `features` first — `anchor-files` consolidates feature specs into ARCHITECTURE.md and TASKS.md; can't proceed without them."* |
| `.arsenal/design/UX.md` | UI projects only | Stop. *"Run `ux-{web,app}` first — `anchor-files` consolidates the page/screen inventory into TASKS.md and DESIGN_SYSTEM.md."* |
| `.arsenal/design/DESIGN.md` | UI projects only | Stop. *"Run `design` first — `anchor-files` consolidates brand tokens into DESIGN_SYSTEM.md."* |
| `.arsenal/strategy/MVP_SPEC.md` | Optional context | Read if present **AND accessible** (not denied by `.claude/settings.json`). Skip silently if absent or if strategy is denied (post-execution-start state). No prompt either way. |
| `.arsenal/design/mockups/` | Recommended for UI | Soft prompt: *"No mockups detected. The design pipeline is substantially stronger when concrete mockups are present. Run `mockups` to generate two-pass anchor briefs, then feed each into Claude Design / Stitch / Open Design / v0 and save outputs to `.arsenal/design/mockups/`. Continuing without?"* |

Non-UI projects (CLI, library, API, server-only) skip the UX/DESIGN/mockup requirements entirely.

## What gets generated

| File | Lines (target) | Sliced by |
|---|---|---|
| `CLAUDE.md` (root) | ≤120 hard ceiling | Every session at load |
| `.arsenal/ARCHITECTURE.md` | ≤500 | `generate-feature-briefs` (data flow + schema cites) |
| `.arsenal/CONVENTIONS.md` | ≤500 | `generate-feature-briefs` (pattern cites) |
| `.arsenal/design/DESIGN_SYSTEM.md` | ≤500 (UI only) | `generate-design-briefs` (token map + component cites) |
| `.arsenal/TASKS.md` | no cap | `expand-phase`, `close-*-phase-*` (read + write) |

## Workflow

### Step 1: Verify preconditions

Check upstream artifacts. On any required miss, **stop** with the routing message from the preconditions table. Do not enter a discovery interview; do not generate any files; do not fall back.

```bash
# UI project preconditions
ls .arsenal/FEATURES.md .arsenal/features/README.md .arsenal/design/UX.md .arsenal/design/DESIGN.md 2>/dev/null

# Non-UI project preconditions
ls .arsenal/FEATURES.md .arsenal/features/README.md 2>/dev/null

# Soft check (UI projects)
ls .arsenal/design/mockups/ 2>/dev/null
```

**Resolve features mode:** if both `FEATURES.md` and `.arsenal/features/` exist (mid-migration), ask which is canonical. If only one exists, use it.

**Resolve UX split mode:** if `.arsenal/design/ux/` directory exists alongside `.arsenal/design/UX.md`, treat the parent file as an index and sub-files as the slice content.

**Resolve project type:** UI vs non-UI. If a `ux-*` was run, this is a UI project. Otherwise treat as non-UI (CLI, library, API, server-only).

If `MVP_SPEC.md` is present **and accessible** (no `.claude/settings.json` deny on `.arsenal/strategy/**`), read it for strategic context (target user, value loop, distribution hypothesis, success metrics) — useful for TASKS.md goal sentences and Phase 0 analytics events. Don't restate its content; cross-reference where useful. If denied (post-execution-start re-anchor scenario), skip silently — anchor-files works without it.

### Step 2: Detect surface (UI projects only)

Determine surface from upstream `UX.md` content or stack hints:

- **Web (marketing/site)** → `arsenal-planning:ux-web` was upstream
- **Web (app, authenticated)** → `arsenal-planning:ux-app` was upstream
- **Hybrid** → multiple UX docs or split `ux/` folder

Surface drives which stack-discovery questions apply.

### Step 3: Stack discovery (gap-only)

Ask **only** about stack-level decisions that upstream artifacts can't answer. Six to eight questions max. Skip any whose answer is obvious from context (existing `package.json`, etc.).

**Stack:**
- Framework? (Next.js / Astro / Vite + React / etc.)
- Styling? (Tailwind / CSS Modules / styled-components)
- Component library? (shadcn/ui / custom / none)
- Database / BaaS / CMS? (Postgres / Supabase / Sanity / Contentful / etc.) — skip database if CMS is the primary data layer
- Auth? (Supabase Auth / NextAuth / Clerk / Sanity / custom)
- State management? (TanStack Query / Zustand / Redux) — skip for Astro static
- Hosting? (Vercel / Cloudflare / Netlify)
- Key integrations? (Stripe / Resend / OpenAI / PostHog / etc.)

**Project intent (drives Phase 0):**
- Intent? Pick one:
  - **hobby** → skip Phase 0 entirely
  - **freelance / startup** → Phase 0 includes landing page (opt-in) + PostHog analytics
  - **client-work** → Phase 0 includes landing page only if scoped in the brief; no analytics by default (the client owns post-launch instrumentation)

### Step 4: Generate anchor files

Dependency order: ARCHITECTURE → CONVENTIONS → DESIGN_SYSTEM → TASKS → CLAUDE.

#### `.arsenal/ARCHITECTURE.md`

System design only. Each section is self-contained — slicing target for `generate-feature-briefs`.

- **System Overview** — ASCII diagram of major components and their relationships
- **Project Structure** — folder layout adapted to the framework
- **Data Flow** — per major user journey, traced through the stack
- **Schema** — full schema in code blocks (SQL / Sanity schema). Cross-reference `.arsenal/features/<slug>.md § Data` for per-feature lifecycles; do not restate.
- **Integrations** — third-party services with env vars
- **State Management** — strategy + query key conventions
- **MVP vs Post-MVP boundary** — what's intentionally excluded

Cross-reference patterns (precise paths, not vague references):
- *"User-Facing Architecture: see `.arsenal/design/UX.md`"*
- *"For per-feature data lifecycles, see `.arsenal/features/<slug>.md § Data`"*

#### `.arsenal/CONVENTIONS.md`

Stack-specific code patterns only. Real code blocks, never descriptions of patterns. Slicing target for `generate-feature-briefs`.

- **Core Development Philosophy** — KISS / YAGNI / Functional First, with good-vs-bad code examples
- **Before Writing Any Code** — the load-bearing checklist (check TASKS.md scope; reference CONVENTIONS for patterns; no temporary workarounds like `// TODO: fix later`, `any` types "for now", or hardcoded values without constants; ask before major decisions)
- **Patterns per Concern** — components, data fetching, mutations, forms, error handling
- **Library Reference** — table with versions + docs links
- **Anti-Patterns** — stack-specific don'ts

Stack-specific guidance:
- **Astro:** `.astro` component patterns, island hydration directives (`client:load`, `client:visible`, `client:idle`), content collections, Sanity GROQ queries if applicable, `<Image>` usage.

#### `.arsenal/design/DESIGN_SYSTEM.md` (UI projects only)

Stack-specific implementation of `.arsenal/design/DESIGN.md`. **Never restates DESIGN.md's tokens** — references them. Slicing target for `generate-design-briefs`.

Required sections:

- **Source of Truth** (1 paragraph): brand tokens, typography, motion, voice live in `.arsenal/design/DESIGN.md`. This file translates that spec into [stack]. **Do not edit `DESIGN.md` directly** — re-run `design` to update the brand.
- **Token Map** — table mapping brand role → stack token. **No value restatement.** Example: `surface.canvas → --color-canvas`, not `surface.canvas → #FAFAFA → --color-canvas`. Readers cross-reference `DESIGN.md` when they need values.
- **Components** — one entry per component named in UX.md's "Components needed" lists. Format: *"Brand rules: see `DESIGN.md` §X. Implementation: [stack-specific code/classes/modifiers/props]."*
- **Mockups directory** — reference to `.arsenal/design/mockups/` and how the design pipeline consumes them
- **Accessibility patterns** — ARIA conventions

If any component named in UX.md isn't defined here, flag it for the user — never invent.

Skip this file entirely for non-UI projects.

#### `.arsenal/TASKS.md`

Phase scaffold and orchestrator state machine. Mutable. Predictable structure so `expand-phase` and `close-*-phase-*` can read/write programmatically.

**Top of file — critical reminder block (verbatim):**

```markdown
> **⚠️ CRITICAL:** Update this file after completing work. Mark tasks `[x]` when done.
> **📦 Tasks expand on-demand:** Phase tasks are placeholders until you start work on the phase.
> **🎨 Each phase splits design + feature:** Design tasks (hardcoded data) run first via `design`; feature tasks (data wiring) run second via `features`. Shared branch, one PR per phase. Design close completes before any feature task starts.
> **🔒 Component boundary:** Once a design task commits a component, the feature pipeline may **extend** it (new props/variants/states the data layer needs) but never **redesign** it (visual treatment / spacing / typography / color / motion). Feature commits that extend a component must include a `Component extended:` note. Purely visual changes BLOCK with redirect to the design pipeline.
> **📝 Spec amendment:** The feature pipeline may amend `.arsenal/FEATURES.md` / `.arsenal/features/*.md` in-line for **Tier 1** changes (additive/clarifying — missed state, edge case, threshold within same intent, missing dependency). Such commits carry a `Spec amended:` note. **Tier 2** changes (user-facing intent shifts) BLOCK and route back through `features`.
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
- [ ] Tasks not yet generated. Run `design 1` to expand visual components from UX.md, DESIGN_SYSTEM.md, and mockups.

### Feature tasks
- [ ] Tasks not yet generated. Run `features 1` to expand data-wiring tasks. Runs after design close.
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
   - `.arsenal/TASKS.md` for current work and phase status
   - `.arsenal/ARCHITECTURE.md` for system design and data flow
   - `.arsenal/CONVENTIONS.md` for code patterns and library versions
   - `.arsenal/design/UX.md` for page/screen inventory (UI only)
   - `.arsenal/features/README.md` then specific feature files (split mode) or `.arsenal/FEATURES.md` (single mode)
   - `.arsenal/design/DESIGN.md` for brand spec — **DO NOT EDIT** (UI only)
   - `.arsenal/design/DESIGN_SYSTEM.md` for stack implementation of DESIGN.md (UI only)
4. **Load-bearing rules** — verbatim, inherited by every subagent:
   - **Component boundary** (full rule with `Component extended:` format)
   - **Spec amendment** (full Tier 1 / Tier 2 distinction with commit-note formats)
   - **Tasks expand on-demand + design-then-feature ordering**
5. **Active phase pointer** — which phase is in progress, which is next
6. **Project-specific rules** — naming conventions, do/don't patterns the user names

Keep it tight. Every line is paid for in every session.

### Step 5: Set up mockup slot

```bash
mkdir -p .arsenal/design/mockups
```

For UI projects without existing mockups, surface the slot in the final report. Recommend Claude Design / Stitch / v0 / future `generate-mockups` (planned around `nexu-io/open-design`) before running `design`.

### Step 6: Review & handoff

Final report:
- **Files created** with line counts (`wc -l`). Flag any file exceeding its target.
- **Upstream artifacts consumed** (which planning docs informed which sections)
- **First phase to run** — `design 1` or, for non-UI / pure feature-domain phases, `features 1`
- **Mockup status** — if `.arsenal/design/mockups/` is empty for a UI project: *"Generate mockups in `.arsenal/design/mockups/` before running `design`. Recommended tools: Claude Design, Stitch, v0."*

## Anti-patterns

- **Don't fall back when required artifacts are missing.** Refuse and route to the right `arsenal-planning` skills skill. The build pipeline depends on planning being complete.
- **Don't ask questions whose answers are already in upstream artifacts.** Read first. Discovery is stack-only.
- **Don't restate upstream content.** Cross-reference with precise paths (`.arsenal/design/UX.md § Onboarding`, `.arsenal/features/recipe-capture.md § Data`). Token budget is real.
- **Don't write load-bearing rules in secondary docs.** CLAUDE.md only — subagents inherit through it. ARCHITECTURE / CONVENTIONS / DESIGN_SYSTEM may reference rules but never define them.
- **Don't exceed CLAUDE.md's 120-line ceiling.** Every session pays the cost. Split out anything that doesn't earn its place.
- **Don't ship a UI project without setting up `.arsenal/design/mockups/`.** The slot exists for a reason. Surface its absence in the final report.

## File placement

```
project-root/
├── CLAUDE.md                          # root index + load-bearing rules
└── .arsenal/
    ├── ARCHITECTURE.md
    ├── CONVENTIONS.md
    ├── TASKS.md
    ├── FEATURES.md                    # from features (single mode)
    ├── features/                      # from features (split mode)
    │   ├── README.md
    │   └── <slug>.md per feature
    ├── design/
    │   ├── UX.md                      # from ux-{web,app} (UI projects)
    │   ├── ux/                        # split mode (UX.md is index)
    │   │   └── <page>.md per page
    │   ├── DESIGN.md                  # from design (UI projects, do-not-edit)
    │   ├── DESIGN_SYSTEM.md           # stack implementation of DESIGN.md
    │   └── mockups/                   # consumed by design
    ├── tasks/                         # ephemera (gitignored)
    │   ├── phase-N/
    │   ├── parallel/
    │   └── archive/
    └── strategy/                      # user archive; denied during build
        ├── MVP_SPEC.md                # optional, from mvp
        ├── MARKET_RESEARCH.md
        ├── RESEARCH_PLAN.md
        ├── GTM_STRATEGY.md
        ├── REVENUE_MODEL.md
        └── mockup-briefs/
```

## Important guidelines

- **Be specific, not generic.** Outputs are tailored to this exact project — no boilerplate filler.
- **Use real code.** Every pattern in CONVENTIONS.md is actual working code for the chosen stack.
- **Version awareness.** Use the API patterns for the library versions specified. Don't mix v4 and v5.
- **Cap each file at its target.** If a file would exceed, split into topical sub-files (e.g., `ARCHITECTURE.md` → `+ DATA_MODEL.md + INTEGRATIONS.md`) and cross-link from a short parent index. Apply during generation — don't write 900 lines first and split after.

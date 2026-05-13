```
   ▄▀█ █▀█ █▀ █▀▀ █▄ █ ▄▀█ █     █▄▄ █ █ █ █   █▀▄
   █▀█ █▀▄ ▄█ ██▄ █ ▀█ █▀█ █▄▄   █▄█ █▄█ █ █▄▄ █▄▀

   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
       O U T E R   H E A V E N   T E C H
   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
```

A Claude Code plugin. The execution half of the arsenal pipeline — takes planning artifacts and ships real code:

```
planning artifacts (FEATURES, UX, DESIGN, mockups, MVP_SPEC) → anchor-files → per-phase design + feature pipelines → landing page
```

Each phase splits into **design tasks** (visual components, hardcoded data) and **feature tasks** (wire to real data). The two halves run sequentially on the same branch, share a single PR, and enforce a strict component boundary: the feature pipeline may extend components but never redesign them.

Variants by surface:

- **`design-web`** + **`features-web`** for web/frontend (Next.js, Astro, Vite, Node, Python, etc.).
- **`design-ios`** + **`features-ios`** for native iOS (SwiftUI + Xcode + simulator-driven visual fidelity gate on the design pipeline).

Pick the variant that matches your stack.

## Install

In Claude Code:

```
/plugin marketplace add Outer-Heaven-Technologies/arsenal-build
/plugin install arsenal-build@arsenal-build
```

Then restart Claude Code. The skills register under the `arsenal-build:` namespace.

> **Where do planning artifacts come from?** Most users pair this plugin with [arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning), which produces `MVP_SPEC.md`, `FEATURES.md`, `UX.md`, `DESIGN.md`, and `planning/mockup-briefs/` at the canonical paths arsenal-build consumes. You can also produce those artifacts by hand or via any other system — arsenal-build will accept them as long as the file names and locations match (or override via `.arsenal/config.yaml`; see Configuration).

### For plugin developers

Fork the repo and edit `SKILL.md` files directly. To get a live-edit loop without reinstalling on every change, symlink your local `skills/` into Claude Code's plugin cache:

```bash
git clone https://github.com/Outer-Heaven-Technologies/arsenal-build.git ~/Dev/arsenal-build
ln -sfn ~/Dev/arsenal-build/skills ~/.claude/plugins/cache/arsenal-build/arsenal-build/0.1.0/skills
```

## How to invoke a skill

Two ways:

1. **Slash command** — `/arsenal-build:anchor-files`, `/arsenal-build:design-web 1`, `/arsenal-build:features-ios 2`, etc.
2. **Natural language** — Claude reads each skill's `description` and auto-fires:
   - "Set up the project anchor" → `arsenal-build:anchor-files`
   - "Begin work on phase 1" → `arsenal-build:design-web` (design half first), then `arsenal-build:features-web` (feature half)

The orchestrators (`anchor-files`, `design-{web,ios}`, `features-{web,ios}`, `landing`) are the user-facing entry points. The sub-skills (`expand-phase`, `generate-design-briefs`, `generate-feature-briefs`, `run-task-*`, `close-*-phase-*`) are dispatched by the orchestrators but are also independently invokable for surgical work when upstream specs change mid-phase.

## The pipeline

```
planning artifacts (from arsenal-planning or any source)
                │
                ▼
        anchor-files  ──►  CLAUDE.md + docs/{ARCHITECTURE,CONVENTIONS,DESIGN_SYSTEM,TASKS}.md
                │
                ▼
   per-phase design half  ──►  per-phase feature half  ──►  single PR per phase
        │                          │
        ├─ design-{web,ios}         ├─ features-{web,ios}
        ├─ expand-phase             ├─ generate-feature-briefs
        ├─ generate-design-briefs   ├─ run-task-feature-{web,ios}
        ├─ run-task-design-{web,ios}  └─ close-feature-phase-{web,ios}  ──►  push + PR
        └─ close-design-phase-{web,ios}  (NO push, NO PR)

                ▼
        landing  ──►  high-converting landing page (separate flow)
```

> **Per-phase split.** Each `TASKS.md` phase splits into **`### Design tasks`** (visual components with hardcoded data) and **`### Feature tasks`** (wire components to real data). The two pipelines run sequentially on the same branch, sharing a single PR. The design close completes **before** any feature task starts.

> **Why split?** Design judgement and feature wiring fail differently. Letting feature implementers drift into visual changes was the failure mode that motivated this architecture. The feature pipeline may **extend** components (new props, variants, or states the data layer needs) but never **redesign** them. Commits modifying a component file include a `Component extended: <path> — <why>` note. Visual changes BLOCK with redirect to the design pipeline.

> **Impeccable boundaries.** Not dispatched from any feature-pipeline skill. Not auto-dispatched from `run-task-design-*` for shape or craft (the user invokes manually if blocked on an under-specified design). Auto-dispatched on iOS as per-task finishers (`harden`, `animate`, `clarify`, `typeset`, `arrange`) per the task's `finishers: [list]` tag. May be invoked from `close-design-phase-*` as a phase-level audit, gateable (the user must opt in).

> **Cross-plugin handoff.** Upstream of `anchor-files`, you need `planning/FEATURES.md` (or `planning/features/`), `docs/UX.md`, and `docs/DESIGN.md`. These are produced by [arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning) (skills `features`, `ux-{web,app,ios}`, `design`). If artifacts are missing, `anchor-files` stops and routes to the right arsenal-planning skill — or you can produce them by hand at the canonical paths.

## Skill table

| # | Skill | Stage | What it does |
|---|-------|-------|--------------|
| 1 | [`anchor-files`](#anchor-files--consolidate-planning-into-the-agent-reference-layer) | Bridge | Consolidate upstream planning into the agent reference layer — `CLAUDE.md`, `ARCHITECTURE.md`, `CONVENTIONS.md`, `DESIGN_SYSTEM.md`, `TASKS.md`. Hard-requires upstream planning. |
| 2a | [`design-web`](#design-web--design-ios--build-visual-components) / [`design-ios`](#design-web--design-ios--build-visual-components) | Orchestrator | **Design-half entry point.** Loops design tasks for a phase (visual components with hardcoded data). Runs FIRST in every phase that has design tasks. Dispatches the four sub-skills below in order. **Does NOT push or PR.** |
|  | ↳ `expand-phase` *(shared sub-skill)* | sub-skill | Turns phase placeholders into a concrete tagged task list grouped under `### Design tasks` / `### Feature tasks`. Tags: `domain: design \| feature`, `research:`, plus `finishers:` / `snapshot:` (iOS design-domain only). Idempotent. Independently invokable for surgical re-expansion. Args: `--phase`, `--surface`, `--scope`, `--force`. |
|  | ↳ `generate-design-briefs` | sub-skill | Writes per-task context briefs (≤3k tokens) + design briefs (≤1.7k tokens) for `domain: design` tasks. Design source is read directly from mockups + `DESIGN_SYSTEM.md`. Idempotent; `--force` regenerates. |
|  | ↳ `run-task-design-web` / `run-task-design-ios` | per task | Per-task design pipeline. **Web:** researcher → design-implementer (hardcoded data) → visual fidelity review (static analysis) → quality review → atomic commit. **iOS:** adds simulator-based visual fidelity gate + finisher pass. |
|  | ↳ `close-design-phase-web` / `close-design-phase-ios` | phase wrap | Two gates: optional impeccable audit + docs update. Writes `.tasks/phase-N/design-summary.md`. **Does NOT push or PR.** |
| 2b | [`features-web`](#features-web--features-ios--wire-components-to-real-data) / [`features-ios`](#features-web--features-ios--wire-components-to-real-data) | Orchestrator | **Feature-half entry point.** Runs AFTER 2a completes for the same phase. Loops feature tasks (wire components to real data). Dispatches the three sub-skills below. **Opens the single PR per phase**, covering both halves' commits. |
|  | ↳ `generate-feature-briefs` | sub-skill | Writes per-task context briefs for `domain: feature` tasks, with an `## Available components` manifest sourced from design-pipeline-committed paths. Idempotent; `--force` regenerates. |
|  | ↳ `run-task-feature-web` / `run-task-feature-ios` | per task | Per-task feature pipeline. Researcher → feature-implementer (component-boundary rule; `Component extended: …` commit note) → spec compliance review → quality review → atomic commit. No impeccable, no visual fidelity gate. |
|  | ↳ `close-feature-phase-web` / `close-feature-phase-ios` | phase wrap | Phase terminus. **Web:** 6 gates — tests → Playwright → docs → CodeRabbit → trim+archive → push+PR. **iOS:** 7 gates — tests → Periphery → snapshot verify → docs → CodeRabbit → trim+archive (with PNG cleanup) → push+PR. |
| 3 | [`landing`](#landing--high-converting-landing-pages) | Standalone | Build a high-converting landing page — researched, structured, deployed. |
| 4 | [`dispatch-parallel`](#dispatch-parallel--fan-out-independent-investigations) | Utility (off-pipeline) | Fan out 2–5 read-only investigations to parallel investigator subagents; reconcile results into one `SUMMARY.md` with cross-investigation overlap detection and severity-tagged recommendations. Use for audits, debug sessions, code analysis. Also shipped in arsenal-planning — both copies are identical; install whichever plugin(s) you have, the skill works the same. |

> **Step 2 reading guide.** Only `2a` and `2b` are user-facing orchestrators. The `↳` rows are sub-skills the orchestrator dispatches in sequence; they share the orchestrator's branch and resolved scope. Sub-skills are also independently invokable (e.g., `/arsenal-build:expand-phase --phase 2 --force`).

See [`PIPELINE.md`](PIPELINE.md) for the full artifact dependency graph and entry-point matrix.

## Load-bearing rules

Both rules live in the project's `CLAUDE.md` (written by `anchor-files`) so every subagent inherits them. They're enforced in the feature-implementer prompt and feature-pipeline spec reviewer.

- **Component boundary.** The feature pipeline may extend a component (new prop, variant, or state the data layer needs) but never redesign it (visual treatment, spacing, typography, color, motion). Commits that touch a component file include a `Component extended: <path> — <why>` note. Purely visual changes BLOCK with redirect to the design pipeline.

- **Spec amendment (Tier 1 / Tier 2).** **Tier 1** in-line amendments to `FEATURES.md` (additive/clarifying — missed state, edge-case alt-path, threshold within same intent, missing dependency) ship in the same atomic commit as the code change with a `Spec amended: <feature-slug> — <why>` note. **Tier 2** intent changes (user-facing behavior shifts, data-model breakage, AC outcome rewritten) BLOCK with `reason: spec change required` and route back through `arsenal-planning:features` (or wherever the spec was authored). The phase wrap collates all Tier 1 amendments into the PR body.

## Configuration

Tracked artifacts live in `planning/` and `docs/` by default. Override these locations by creating `.arsenal/config.yaml` at the project root:

```yaml
# .arsenal/config.yaml
paths:
  planning: planning/                     # default
  docs: docs/                             # default
  mockups: docs/mockups/                  # default
  mockup_briefs: planning/mockup-briefs/  # default
```

Every skill that reads or writes a tracked artifact performs a preflight check: if `.arsenal/config.yaml` exists, it uses the configured `paths.*` values; if absent, it uses defaults silently — no prompting just to confirm defaults.

| Variable | Default | Holds |
|---|---|---|
| `paths.planning` | `planning/` | MARKET_RESEARCH.md, MVP_SPEC.md, FEATURES.md (or features/*.md), GTM_STRATEGY.md, REVENUE_MODEL.md |
| `paths.docs` | `docs/` | UX.md, DESIGN.md, DESIGN_SYSTEM.md, ARCHITECTURE.md, CONVENTIONS.md, TASKS.md |
| `paths.mockups` | `docs/mockups/` | Mockup files (consumed by design pipeline) |
| `paths.mockup_briefs` | `planning/mockup-briefs/` | Mockup briefs |

**File names are not configurable** — only their wrapping directory is. `TASKS.md` is `TASKS.md` whether it lives in `docs/` or `arsenal-docs/`.

**Consuming an artifact from arsenal-planning:** if config (or defaults) point to a location where the expected artifact is missing, build skills will ask you where to find it instead of failing. You can also point at a non-canonical location via the config file.

## Skill details

Each section follows the same shape: a one-line purpose, the steps it runs, how to invoke it, and what it reads and writes.

---

### `anchor-files` — consolidate planning into the agent reference layer

Bridge between planning (arsenal-planning, or hand-authored equivalents) and execution (`design-*` / `features-*`). Consolidates upstream planning into the **agent reference layer**: `CLAUDE.md` at root plus `docs/{ARCHITECTURE,CONVENTIONS,DESIGN_SYSTEM,TASKS}.md`. These files are read by every Claude session opening the project and sliced by the brief generators that drive the build pipeline.

This is not "generate docs." It's "design the structured reference surface that every downstream agent reads from for the lifetime of the project."

**Hard-required upstream (no fallback)**

The build pipeline depends on planning being complete. Missing artifacts → skill stops and routes to the corresponding arsenal-planning skill (or asks you to point at the missing artifact via config or prompt) rather than papering over the gap.

| Artifact | Required | If missing |
|---|---|---|
| `planning/FEATURES.md` or `planning/features/` | Yes | Stop. Route to `arsenal-planning:features` (or ask for path). |
| `docs/UX.md` | UI projects only | Stop. Route to `arsenal-planning:ux-{web,app,ios}`. |
| `docs/DESIGN.md` | UI projects only | Stop. Route to `arsenal-planning:design`. |
| `planning/MVP_SPEC.md` | Optional | Read if present; skip if not. |
| `docs/mockups/` | Recommended for UI | Soft prompt: run `arsenal-planning:mockups` to generate briefs, then feed each into Claude Design / Stitch / Open Design / v0. |

**How it works**

1. **Verify preconditions.** Hard-check the artifacts above (after resolving any `.arsenal/config.yaml` overrides). No discovery interview if anything required is missing.
2. **Detect surface.** Web marketing / web app / native iOS / non-UI. Drives stack questions and which files generate.
3. **Stack-only discovery.** ~6–8 questions max — framework, styling, database/CMS, auth, state management, hosting, key integrations, project intent. Skip anything answered by existing artifacts (e.g., existing `package.json` or `*.xcodeproj`).
4. **Generate anchor files.** Dependency order — ARCHITECTURE → CONVENTIONS → DESIGN_SYSTEM → TASKS → CLAUDE.
5. **Set up mockup slot.** `mkdir -p docs/mockups`. Surface its absence in the final report if empty.
6. **Review + handoff.** Files created, sizes flagged, next step (`design-{web,ios} 1`).

**File sizing — load-bearing**

| File | Target | Read by |
|---|---|---|
| `CLAUDE.md` | **≤120 lines hard ceiling** | Every session at load |
| `ARCHITECTURE.md` | ≤500 | `generate-feature-briefs` (slices data flow + schema sections) |
| `CONVENTIONS.md` | ≤500 | `generate-feature-briefs` (slices pattern sections) |
| `DESIGN_SYSTEM.md` | ≤500 (UI only) | `generate-design-briefs` (slices token map + component sections) |
| `TASKS.md` | No cap | `expand-phase`, `close-*-phase-*` (state machine — read + write) |

Each output section is self-contained so brief generators can excerpt cleanly. Cross-references are precise (`docs/ARCHITECTURE.md § Data Flow / Recipe Capture`), not vague.

**Load-bearing rules in CLAUDE.md verbatim**

CLAUDE.md is the only place where the build pipeline's load-bearing rules appear in full — subagents inherit them through CLAUDE.md and can't load secondary docs to fetch them:

- **Component boundary** — extend not redesign; `Component extended: <path> — <why>` commit note required
- **Spec amendment (Tier 1 / Tier 2)** — see Load-bearing rules section above
- **Tasks expand on-demand + design-then-feature ordering**

**A few load-bearing decisions**

- **`TASKS.md` is a phase scaffold, not a complete task list.** Phase entries are placeholders that `expand-phase` expands on demand against fresh context.
- **`DESIGN_SYSTEM.md` is stack-specific implementation** (CSS vars, SwiftUI tokens). `DESIGN.md` stays canonical and untouched.
- **`CONVENTIONS.md` mandates concrete sections** — KISS / YAGNI / Functional First, "Before Writing Any Code", anti-patterns — with **real working code** for the chosen stack, not pseudocode.
- **In split-features mode**, `anchor-files` reads only `planning/features/README.md`, never individual feature files.

**How to use it**

- **Slash command:** `/arsenal-build:anchor-files`
- **Or trigger with:** "set up project anchor", "anchor the codebase", "scaffold project docs", "set up CLAUDE.md", "bridge planning to build"

- **Inputs (hard-required):** `planning/FEATURES.md` or `planning/features/`, `docs/UX.md` (UI), `docs/DESIGN.md` (UI). `MVP_SPEC.md` optional.
- **Outputs:** `CLAUDE.md` at root; `docs/ARCHITECTURE.md`, `docs/CONVENTIONS.md`, `docs/DESIGN_SYSTEM.md` (UI only), `docs/TASKS.md`. Creates `docs/mockups/` directory.

---

### `design-web` / `design-ios` — build visual components

The design half of each phase. Loops design tasks — each task builds a component / view with hardcoded data, exercises every state from the design brief's variant coverage, and commits atomically. **Does NOT push or PR.** Returns the branch for the feature pipeline.

**Per-task pipeline (inside `run-task-design-{web,ios}`):**

1. Optional researcher (when `research: yes`).
2. Design-implementer: reads design brief, mockup at cited region, token map, variant coverage. Builds the component at the path declared in DESIGN_SYSTEM.md (or the context brief). Hardcoded data only — no `@Query`, no server actions.
3. Visual fidelity review.
   - **Web:** static analysis — token map adherence, locked-primitive citation, state coverage, mockup ↔ code match, hardcoded-data discipline. No live browser.
   - **iOS:** full simulator-mediated gate — build → snapshot test → token-discipline scan → a11y identifier check → screenshot → mockup compare → Reduced Motion verification.
4. Spec compliance review (iOS only; web design pipeline's two stages are visual fidelity + quality).
5. Code quality review.
6. Finisher pass (iOS only; per `finishers: [list]` tag — `harden`, `animate`, `clarify`, `typeset`, `arrange`).
7. Atomic commit + `[x]` in `TASKS.md` under `### Design tasks`.

**No automatic impeccable dispatch** for shape / craft. If the design brief is thin, the implementer BLOCKS — the user invokes `impeccable:shape <surface>` manually if they want. **Finishers (iOS) ARE auto-dispatched** per the task's `finishers: [list]` tag.

**Phase wrap (`close-design-phase-{web,ios}`):** two gates — (1) optional impeccable audit + polish, gateable; (2) docs update if scope drifted. Writes `.tasks/phase-N/design-summary.md` for the feature-pipeline PR body. **No push, no PR, no CodeRabbit, no trim** — that's all the feature close's job.

**How to use:**

- **Slash command:** `/arsenal-build:design-web N` or `/arsenal-build:design-ios N`
- **Or trigger with:** "design phase 1", "build the visual components for phase 2", "design the onboarding screens"

**Inputs:** `TASKS.md` (required), `UX.md`, `DESIGN_SYSTEM.md`, `DESIGN.md`, feature specs (design-relevant parts), `docs/mockups/` (recommended).
**Outputs:** atomic commits of design-pipeline components, `[x]` flips in `### Design tasks`, `.tasks/phase-N/design-summary.md`.

---

### `features-web` / `features-ios` — wire components to real data

The feature half of each phase. Loops feature tasks — each task wires existing components (committed by the design pipeline earlier in the same phase) to real data: queries, mutations, services, server actions, state management. **Opens the single PR per phase at the end.**

**Per-task pipeline (inside `run-task-feature-{web,ios}`):**

1. Optional researcher (when `research: yes`).
2. Feature-implementer: reads context brief with `## Available components` manifest enumerating design-pipeline-committed paths. Wires real data. Treats components as read-only design surfaces; may extend (new props/variants/states) but not redesign (visual treatment / spacing / typography / color). Every commit touching a component file includes a `Component extended: <path> — <why>` note.
3. Spec compliance review: checks AC compliance AND enforces the `Component extended:` rule. Purely visual changes disguised as extensions are CRITICAL failures.
4. Code quality review.
5. Atomic commit + `[x]` in `TASKS.md` under `### Feature tasks`.

**No impeccable, no visual fidelity gate, no finishers** — all are design-pipeline territory. If a feature task seems to need design judgement, it was misclassified — the implementer BLOCKS with `reason: purely visual change required — redirect to design pipeline`.

**Phase wrap (`close-feature-phase-{web,ios}`):**

- **Web (6 gates):** final integration test → Playwright (if configured) → docs update (if drift) → **CodeRabbit (hard gate, covers full phase)** → trim `TASKS.md` + archive `.tasks/phase-N/` → push branch + open PR.
- **iOS (7 gates):** `RunAllTests` → Periphery dead-code scan → snapshot test verification → docs update → **CodeRabbit** → trim+archive (with `*.png` cleanup) → push + open PR.

CodeRabbit covers the full phase (design + feature commits together) — single pass per PR.

**How to use:**

- **Slash command:** `/arsenal-build:features-web N` or `/arsenal-build:features-ios N`
- **Or trigger with:** "begin work on phase 1" (after design half complete), "wire the components to data", "ship the phase"

Run AFTER `design-{web,ios}` + `close-design-phase-{web,ios}` for the same phase. If invoked before the design half completes, this orchestrator stops and tells the user to run the design half first.

Scope flexibly — *"phase 1, just the <feature-name> feature"* or *"the <story-name> story within <feature>"* both work. Pick model per task: simple → Sonnet/Haiku, complex → Opus.

**Inputs:** `TASKS.md`, `ARCHITECTURE.md`, `CONVENTIONS.md`, `DESIGN_SYSTEM.md` (required); feature specs; phase branch with design-pipeline commits already present.
**Outputs:** atomic commits of feature-pipeline data wiring, `[x]` flips in `### Feature tasks`, trimmed `TASKS.md`, archived briefs, single PR for the phase.

**Pure feature-domain phases** (no design tasks — pure backend, schema migrations, infrastructure): `features-{web,ios}` runs the whole phase end-to-end, no design half. The `### Design tasks` subsection is the `_None — pure feature-domain phase._` placeholder.

**iOS-specific:**

| Element | iOS variant |
|---|---|
| Tooling | Apple Xcode MCP, iOS Simulator MCP, Periphery, swift-snapshot-testing |
| Design source | `docs/mockups/*.{jsx,tsx,html,png,figma-export.json}` — consumed by `design-ios` |
| Implementer rules | Hex literal lockdown (theme module only), accessibility identifiers, `@Observable` not `ObservableObject`, Swift Concurrency only |
| Finisher set | Auto-dispatched in `run-task-design-ios` — `harden`, `animate`, `clarify`, `typeset`, `arrange` per the task's `finishers: [list]` tag |

Both `design-ios` and `features-ios` prompt to confirm Xcode is open and a simulator is booted before tasks dispatch.

Pick the iOS variants when the project root has `*.xcodeproj` or `Package.swift` for an app target. Pick the web variants for everything else (web/frontend, Node/Python servers, cross-platform projects).

---

### `landing` — high-converting landing pages

Builds a landing page designed to convert — researched first, structured second, made beautiful third. Pairs with the `frontend-design` / `impeccable` skills for the visual layer.

**How it works**

1. **Research.** Reads existing planning docs (`MVP_SPEC.md`, `MARKET_RESEARCH.md`, `DESIGN_SYSTEM.md`) if they exist — `MARKET_RESEARCH.md` from arsenal-planning is the unified dossier that contains competitive analysis in §3. Otherwise asks for a one-liner / audience / primary action / 2–3 competitors. Crawls 2–3 competitor landing pages before writing copy.
2. **Structure.** Imposes a proven section sequence: above-fold → social proof → mechanism → objection FAQ → final CTA → minimal footer. Headlines answer "what's in it for me?" in <10 words; **loss-aversion framing** preferred — *~2x stronger than gain framing*.
3. **Copy guidelines.** CTAs describe outcomes ("Get my free plan" beats "Submit"). Anti-startup-speak list bans "We are excited to announce", "The world's first", "Leveraging cutting-edge".
4. **Build.** Astro preferred (zero JS); plain HTML+Tailwind for single page; Next.js only if dynamic.
5. **Review.** Against checklists.

**Defaults the skill picks for you (swappable):**

- **Email capture:** Kit (ConvertKit) — free up to 10k subs, project-specific tags.
- **Analytics:** PostHog — re-uses Phase 0 instance if `anchor-files` already set one up.
- **Repo:** lives in a separate repo from the core product, independent deploys.

**Hard rules the skill enforces (non-negotiable):**

- **Mobile-first.** Design the mobile layout first, then expand. >50% of traffic is mobile.
- **44×44 touch targets.** Minimum tap-target size on every interactive element.
- **Single primary CTA.** One main call-to-action, visually dominant.
- **<2s load time.** Beyond that, *every second costs ~7% in conversions*.
- **Performance non-negotiables.** Compressed images (WebP preferred), minimal JS, no render-blocking third-party scripts above the fold.
- **Email forms ≤2 fields.** Email only, or email + name.

Out of scope: multi-page marketing sites, paid-ad strategy, ongoing A/B testing.

**How to use it**

- **Slash command:** `/arsenal-build:landing`
- **Or trigger with:** "build a landing page", "create a waitlist", "coming soon page", "build the marketing page"

Also auto-fires from a TASKS.md task that mentions a landing page.

- **Inputs:** any of `planning/MVP_SPEC.md`, `planning/MARKET_RESEARCH.md` (unified dossier), `docs/DESIGN_SYSTEM.md` (optional). Otherwise the skill asks.
- **Outputs:** a deployable landing page repo, or a `/coming-soon` route inside an existing app — actual code, not specs.

---

### `dispatch-parallel` — fan out independent investigations

A utility skill, **off the linear pipeline**. Dispatches 2–5 read-only investigations to parallel investigator subagents and reconciles their results into a single `SUMMARY.md` with cross-investigation overlap detection and severity-tagged recommendations. Use for audits, debug sessions, code analysis — any case where the work is genuinely disjoint and parallelization actually pays for itself.

Also shipped in arsenal-planning (it's required there by `market-analysis`). The two copies are identical — Claude Code's plugin namespacing keeps them addressable separately as `/arsenal-build:dispatch-parallel` and `/arsenal-planning:dispatch-parallel`. Install whichever plugin(s) you have; the skill behaves the same in both.

**The independence gate is the whole skill.** Before any dispatch, it checks three criteria — disjoint scope, no shared mutations, result-independence — and refuses to fan out if they don't hold. Failing the gate is the success case for dependent work; the skill recommends sequential execution.

**Does NOT compose with `run-task-{design,feature}-{web,ios}`.** Investigations are read-only by design — investigator subagents have zero write capability. When findings recommend code changes, the SUMMARY's "Next steps" block points the user at the per-task pipelines (`run-task-design-*` or `run-task-feature-*` for sequential, one-fix-at-a-time work) or the orchestrators (`design-*` / `features-*` for pattern-spanning work as a TASKS.md phase). The two skills connect through filesystem and user judgment, not direct invocation.

**Locked contracts:**
- **Count bounds:** N = 1 refuses with suggestion; 2 ≤ N ≤ 5 normal; N ≥ 6 hard-refuses (recommend phase modeling).
- **Idempotence:** default skip per-investigation if `investigation-{N}-result.md` exists; `--force` regenerates.
- **Conflict handling:** when investigations contradict each other, SUMMARY flags conflicts and the skill exits cleanly.
- **Investigator tools:** broad read (Read / Glob / Grep / read-only Bash / WebSearch / Firecrawl / claude-in-chrome), zero write.

**How to use it**

- **Slash command:** `/arsenal-build:dispatch-parallel`
- **Or trigger with:** "investigate these in parallel", "fan out on these issues", "run these checks concurrently", "audit X, Y, and Z separately"

- **Inputs:** 2–5 investigation descriptions via `--investigation` (repeated) or `--from-file <path>`, optional `--surface web|ios`, optional `--force`, optional `--max <N>` (capped at 5).
- **Outputs (`.tasks/parallel/<run-id>/`):** `investigation-N-result.md` per investigation (≤3k tokens), `SUMMARY.md` (aggregated with overlap detection + conflict flags + next-step recommendations).

## File layout

```
arsenal-build/
├── .claude-plugin/
│   └── plugin.json              # plugin manifest (name: arsenal-build)
├── skills/
│   ├── anchor-files/SKILL.md
│   │
│   ├── # Design + feature pipeline orchestrators (thin):
│   ├── design-web/SKILL.md
│   ├── design-ios/SKILL.md
│   ├── features-web/SKILL.md
│   ├── features-ios/SKILL.md
│   │
│   ├── # Shared sub-skills:
│   ├── expand-phase/SKILL.md
│   ├── generate-design-briefs/
│   │   ├── SKILL.md
│   │   └── references/{design-brief-prompt-web,design-brief-prompt-ios}.md
│   ├── generate-feature-briefs/SKILL.md
│   │
│   ├── # Per-task pipelines:
│   ├── run-task-design-web/
│   │   ├── SKILL.md
│   │   └── references/{researcher,design-implementer,visual-fidelity-reviewer,quality-reviewer}-prompt.md
│   ├── run-task-design-ios/
│   │   ├── SKILL.md
│   │   ├── references/{researcher,design-implementer,visual-fidelity-reviewer,spec-reviewer,quality-reviewer}-prompt.md
│   │   └── references/finishers-table.md
│   ├── run-task-feature-web/
│   │   ├── SKILL.md
│   │   └── references/{researcher,feature-implementer,spec-reviewer,quality-reviewer}-prompt.md
│   ├── run-task-feature-ios/
│   │   ├── SKILL.md
│   │   └── references/{researcher,feature-implementer,spec-reviewer,quality-reviewer}-prompt.md
│   │
│   ├── # Phase close skills:
│   ├── close-design-phase-web/SKILL.md     # 2 gates; no push, no PR
│   ├── close-design-phase-ios/SKILL.md     # 2 gates; no push, no PR
│   ├── close-feature-phase-web/SKILL.md    # 6 gates; opens single PR per phase
│   ├── close-feature-phase-ios/SKILL.md    # 7 gates; opens single PR per phase
│   │
│   ├── landing/SKILL.md
│   └── dispatch-parallel/                  # utility skill, off-pipeline (also in arsenal-planning)
│       ├── SKILL.md
│       └── references/investigator-prompt.md
├── PIPELINE.md
├── LICENSE
└── README.md
```

## Philosophy

- **Atomic commits, fresh context.** Every per-task subagent dispatches in fresh context with a brief generated from the surrounding planning artifacts. A blocked task is never retried with the same model and the same prompt.
- **Briefs hand off via filesystem.** Subagents never inherit session history. Per-task briefs live at `.tasks/phase-N/task-N-context.md` and `task-N-design.md`; SUMMARYs sit in the phase folder. Easy to inspect, easy to regenerate.
- **Two pipelines, one PR.** Design judgement and feature wiring fail differently. Keep them in separate prompts, separate review stages, separate commits. Same branch, same PR — the boundary is enforced by the `Component extended:` rule and Tier 1 / Tier 2 spec amendment routing.
- **Match depth to stakes.** Pick model per task. Simple → Sonnet/Haiku. Complex → Opus. Don't waste tokens.

## Pairs with

- **[arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning)** — the planning half. Produces the `MVP_SPEC.md`, `FEATURES.md`, `UX.md`, `DESIGN.md`, and `planning/mockup-briefs/` artifacts that arsenal-build consumes. Optional but recommended; you can also produce planning artifacts by hand at canonical paths and arsenal-build will accept them.

## License

[AGPL-3.0-or-later](LICENSE). Use arsenal commercially in your workflow — the artifacts it produces are yours. You can't repackage *the tool itself* as a closed-source product.

<details>
<summary>Full license summary</summary>

**You can freely:**

- Use arsenal in Claude Code as part of your workflow, including for paid client work or building your own commercial product. The artifacts arsenal produces (planning docs, code, designs) are yours — they aren't covered by the license.
- Modify the skills for your own use.
- Share modifications publicly under AGPL.

**The AGPL kicks in when you redistribute the skills themselves:**

- Wrapping arsenal's prompts/skills in a UI and selling it as a SaaS or product → you must release your source under AGPL.
- Forking arsenal and shipping a paid plugin built from it → same.
- Embedding the `SKILL.md` content (modified or not) into a commercial product → same.

For a commercial license that allows proprietary derivatives, contact Outer Heaven Technologies.

</details>

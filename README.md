```
   ▄▀█ █▀█ █▀ █▀▀ █▄ █ ▄▀█ █     █▄▄ █ █ █ █   █▀▄
   █▀█ █▀▄ ▄█ ██▄ █ ▀█ █▀█ █▄▄   █▄█ █▄█ █ █▄▄ █▄▀

   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
       O U T E R   H E A V E N   T E C H
   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
```

A Claude Code plugin. The execution half of the arsenal pipeline for **web/frontend projects** (Next.js, Astro, Vite, Node, Python, etc.). Takes planning artifacts and ships real code:

```
planning artifacts (FEATURES, UX, DESIGN, mockups, MVP_SPEC) → setup → per-phase design + feature pipelines → landing page
```

Each phase splits into **design tasks** (visual components with hardcoded data) and **feature tasks** (wire components to real data). The two halves run sequentially on the same branch, share a single PR, and enforce a strict component boundary: the feature pipeline may extend components but never redesign them.

For native iOS projects (SwiftUI + Xcode + simulator-driven visual fidelity gate), see [arsenal-build-io](https://github.com/Outer-Heaven-Technologies/arsenal-build-io).

## Install

In Claude Code:

```
/plugin marketplace add Outer-Heaven-Technologies/arsenal-build
/plugin install arsenal-build@arsenal-build
```

Then restart Claude Code. The skills register under the `arsenal-build:` namespace.

> **Where do planning artifacts come from?** Most users pair this plugin with [arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning), which produces `MVP_SPEC.md`, `FEATURES.md`, `UX.md`, `DESIGN.md`, and `.arsenal/strategy/mockup-briefs/` at the canonical paths arsenal-build consumes. You can also produce those artifacts by hand or via any other system — arsenal-build will accept them as long as the file names and locations match (or override via `.arsenal/config.yaml`; see Configuration).

### For plugin developers

Fork the repo and edit `SKILL.md` files directly. To get a live-edit loop without reinstalling on every change, symlink your local `skills/` into Claude Code's plugin cache:

```bash
git clone https://github.com/Outer-Heaven-Technologies/arsenal-build.git ~/Dev/arsenal-build
ln -sfn ~/Dev/arsenal-build/skills ~/.claude/plugins/cache/arsenal-build/arsenal-build/0.2.0/skills
```

## How to invoke a skill

Two ways:

1. **Slash command** — `/arsenal-build:setup`, `/arsenal-build:design 1`, `/arsenal-build:features 2`, etc.
2. **Natural language** — Claude reads each skill's `description` and auto-fires:
   - "Set up the project" / "Bootstrap the build pipeline" → `arsenal-build:setup`
   - "Begin work on phase 1" → `arsenal-build:design` (design half first), then `arsenal-build:features` (feature half)

The orchestrators (`setup`, `design`, `features`, `landing`) are the user-facing entry points. The sub-skills (`expand-phase`, `generate-design-briefs`, `generate-feature-briefs`, `run-task-design`, `run-task-feature`, `close-design-phase`, `close-feature-phase`) are dispatched by the orchestrators but are also independently invokable for surgical work when upstream specs change mid-phase.

## The pipeline

```
planning artifacts (from arsenal-planning or any source)
                │
                ▼
        setup  ──►  CLAUDE.md + .arsenal/{ARCHITECTURE,CONVENTIONS,TASKS}.md + .arsenal/design/DESIGN_SYSTEM.md
                │
                ▼
   per-phase design half  ──►  per-phase feature half  ──►  single PR per phase
        │                          │
        ├─ design                   ├─ features
        ├─ expand-phase             ├─ generate-feature-briefs
        ├─ generate-design-briefs   ├─ run-task-feature
        ├─ run-task-design          └─ close-feature-phase  ──►  push + PR
        └─ close-design-phase  (NO push, NO PR)

                ▼
        landing  ──►  high-converting landing page (separate flow)
```

> **Per-phase split.** Each `TASKS.md` phase splits into **`### Design tasks`** (visual components with hardcoded data) and **`### Feature tasks`** (wire components to real data). The two pipelines run sequentially on the same branch, sharing a single PR. The design close completes **before** any feature task starts.

> **Why split?** Design judgement and feature wiring fail differently. Letting feature implementers drift into visual changes was the failure mode that motivated this architecture. The feature pipeline may **extend** components (new props, variants, or states the data layer needs) but never **redesign** them. Commits modifying a component file include a `Component extended: <path> — <why>` note. Visual changes BLOCK with redirect to the design pipeline.

> **Impeccable boundaries.** Not dispatched from any feature-pipeline skill. Not auto-dispatched from `run-task-design` (the user invokes manually if blocked on an under-specified design). May be invoked from `close-design-phase` as a phase-level audit, gateable (the user must opt in).

> **Cross-plugin handoff.** Upstream of `setup`, you need `.arsenal/FEATURES.md` (or `.arsenal/features/`), `.arsenal/design/UX.md`, and `.arsenal/design/DESIGN.md`. These are produced by [arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning) (skills `features`, `ux-web` or `ux-app`, `design`). If artifacts are missing, `setup` stops and routes to the right arsenal-planning skill — or you can produce them by hand at the canonical paths.

## Skill table

| # | Skill | Stage | What it does |
|---|-------|-------|--------------|
| 1 | [`setup`](#setup--bootstrap-the-project-for-the-build-pipeline) | Bridge | Bootstrap a project for the build pipeline — `CLAUDE.md`, `ARCHITECTURE.md`, `CONVENTIONS.md`, `DESIGN_SYSTEM.md`, `TASKS.md`, and (when missing) lightweight `FEATURES.md` / `UX.md` / `DESIGN.md`. Three starting points: consume existing planning artifacts (A), ingest user-pointed source docs (B), or interview from scratch (C). arsenal-planning is the recommended deep path. |
| 2a | [`design`](#design--build-visual-components) | Orchestrator | **Design-half entry point.** Loops design tasks for a phase (visual components with hardcoded data). Runs FIRST in every phase that has design tasks. Dispatches the four sub-skills below in order. **Does NOT push or PR.** |
|  | ↳ `expand-phase` *(shared sub-skill)* | sub-skill | Turns phase placeholders into a concrete tagged task list grouped under `### Design tasks` / `### Feature tasks`. Tags: `domain: design \| feature`, `research:`. Idempotent. Independently invokable for surgical re-expansion. Args: `--phase`, `--scope`, `--force`. |
|  | ↳ `generate-design-briefs` | sub-skill | Writes per-task context briefs (≤3k tokens) + design briefs (≤1.7k tokens) for `domain: design` tasks. Design source is read directly from mockups + `DESIGN_SYSTEM.md`. Idempotent; `--force` regenerates. |
|  | ↳ `run-task-design` | per task | Per-task design pipeline. Researcher → design-implementer (hardcoded data) → visual fidelity review (static analysis) → quality review → atomic commit. |
|  | ↳ `close-design-phase` | phase wrap | Two gates: optional impeccable audit + docs update. Writes `.arsenal/tasks/phase-N/design-summary.md`. **Does NOT push or PR.** |
| 2b | [`features`](#features--wire-components-to-real-data) | Orchestrator | **Feature-half entry point.** Runs AFTER 2a completes for the same phase. Loops feature tasks (wire components to real data). Dispatches the three sub-skills below. **Opens the single PR per phase**, covering both halves' commits. |
|  | ↳ `generate-feature-briefs` | sub-skill | Writes per-task context briefs for `domain: feature` tasks, with an `## Available components` manifest sourced from design-pipeline-committed paths. Idempotent; `--force` regenerates. |
|  | ↳ `run-task-feature` | per task | Per-task feature pipeline. Researcher → feature-implementer (component-boundary rule; `Component extended: …` commit note) → spec compliance review → quality review → atomic commit. No impeccable, no visual fidelity gate. |
|  | ↳ `close-feature-phase` | phase wrap | Phase terminus. 6 gates: tests → Playwright (if configured) → docs → CodeRabbit → trim+archive → push+PR. Opens the single PR per phase covering both design + feature commits. |
| 3 | [`landing`](#landing--high-converting-landing-pages) | Standalone | Build a high-converting landing page — researched, structured, deployed. |
| 4 | [`dispatch-parallel`](#dispatch-parallel--fan-out-independent-investigations) | Utility (off-pipeline) | Fan out 2–5 read-only investigations to parallel investigator subagents; reconcile results into one `SUMMARY.md` with cross-investigation overlap detection and severity-tagged recommendations. Use for audits, debug sessions, code analysis. Also shipped in arsenal-planning and arsenal-build-io — all three copies are identical. |

> **Step 2 reading guide.** Only `2a` and `2b` are user-facing orchestrators. The `↳` rows are sub-skills the orchestrator dispatches in sequence; they share the orchestrator's branch and resolved scope. Sub-skills are also independently invokable (e.g., `/arsenal-build:expand-phase --phase 2 --force`).

See [`PIPELINE.md`](PIPELINE.md) for the full artifact dependency graph and entry-point matrix.

## Load-bearing rules

Both rules live in the project's `CLAUDE.md` (written by `setup`) so every subagent inherits them. They're enforced in the feature-implementer prompt and feature-pipeline spec reviewer.

- **Component boundary.** The feature pipeline may extend a component (new prop, variant, or state the data layer needs) but never redesign it (visual treatment, spacing, typography, color, motion). Commits that touch a component file include a `Component extended: <path> — <why>` note. Purely visual changes BLOCK with redirect to the design pipeline.

- **Spec amendment (Tier 1 / Tier 2).** **Tier 1** in-line amendments to `FEATURES.md` (additive/clarifying — missed state, edge-case alt-path, threshold within same intent, missing dependency) ship in the same atomic commit as the code change with a `Spec amended: <feature-slug> — <why>` note. **Tier 2** intent changes (user-facing behavior shifts, data-model breakage, AC outcome rewritten) BLOCK with `reason: spec change required` and route back through `arsenal-planning:features` (or wherever the spec was authored). The phase wrap collates all Tier 1 amendments into the PR body.

## Configuration

All arsenal artifacts live under `.arsenal/` at the project root. No configuration is required for typical projects:

| Path | Holds |
|---|---|
| `.arsenal/strategy/` | User archive (MARKET_RESEARCH, RESEARCH_PLAN, MVP_SPEC, mockup-briefs, GTM_STRATEGY, REVENUE_MODEL). **Denied during build execution.** |
| `.arsenal/FEATURES.md` or `.arsenal/features/` | Feature specs. Gated per phase. |
| `.arsenal/{ARCHITECTURE,CONVENTIONS,TASKS}.md` | Anchor docs. Always readable during build. |
| `.arsenal/design/{UX,DESIGN,DESIGN_SYSTEM}.md` + `.arsenal/design/mockups/` | Design reference set. Always readable during build. |
| `.arsenal/tasks/phase-N/`, `.arsenal/tasks/parallel/`, `.arsenal/tasks/archive/` | Per-task briefs + ephemera. Gitignored. Phase-N gated per active phase. |

**File names are not configurable.** Only the `.arsenal/` root location may be overridden via `.arsenal/config.yaml` in unusual cases.

## Gating

The build pipeline gates agent reads via `.claude/settings.json`. The first invocation of `expand-phase` writes the baseline (just the strategy lockdown), then computes per-phase denies on every invocation. Example for phase 2 in split mode, with in-scope features `auth-flow` and `dashboard`, and existing phase-1 and phase-3 folders:

```json
{
  "permissions": {
    "deny": [
      "Read(.arsenal/strategy/**)",
      "Read(.arsenal/features/billing.md)",
      "Read(.arsenal/features/admin.md)",
      "Read(.arsenal/tasks/phase-1/**)",
      "Read(.arsenal/tasks/phase-3/**)"
    ]
  }
}
```

On every `expand-phase` invocation, the per-phase entries are recomputed:
- Baseline: `Read(.arsenal/strategy/**)` is always present (strategy archive denied during build).
- Out-of-scope features are denied individually (split mode): `Read(.arsenal/features/<other-slug>.md)`. `.arsenal/features/README.md` stays readable.
- Other phases' task folders are denied: `Read(.arsenal/tasks/phase-X/**)`.
- The current phase's task folder and in-scope features are intentionally NOT denied — they're readable.

`close-feature-phase` removes the per-phase entries at phase end, restoring the broad baseline.

**`landing` self-lifts the strategy deny** on invocation to read MARKET_RESEARCH and MVP_SPEC, then restores. The user can manually remove `.arsenal/`-prefixed entries from `.claude/settings.json` to revisit planning.

## Skill details

Each section follows the same shape: a one-line purpose, the steps it runs, how to invoke it, and what it reads and writes.

---

### `setup` — bootstrap the project for the build pipeline

Bootstrap a project for the arsenal-build pipeline. Produces the **agent reference layer**: `CLAUDE.md` at root plus `.arsenal/{ARCHITECTURE,CONVENTIONS,TASKS}.md` and `.arsenal/design/DESIGN_SYSTEM.md`. When upstream planning artifacts are missing, also produces lightweight versions of `FEATURES.md`, `UX.md`, and `DESIGN.md` at canonical paths so the build pipeline can run regardless of starting state.

This is not "generate docs." It's "design the structured reference surface that every downstream agent reads from for the lifetime of the project."

**Three starting points**

Setup runs against whatever state the project is in. Per missing planning artifact, the user picks the path. Different artifacts can take different paths in the same run.

| Path | Trigger | What setup does |
|---|---|---|
| **A — Consume** | `FEATURES`, `UX` (UI), `DESIGN` (UI) already exist | Reads them as-is. **Recommended path** — arsenal-planning produces the deepest specs. |
| **B — Ingest** | An artifact is missing but the user has related docs (PRD, Notion brief, hand-written spec, Slack thread) | Dispatches a subagent to read user-pointed docs and write a **lightweight** version with a `Generated by setup` banner. |
| **C — Interview** | An artifact is missing and there's no doc to point at | Runs a short intake interview, then dispatches a subagent to write a **lightweight** version from the transcript, with the banner. |

Generated planning artifacts always carry a `Generated by setup — lightweight bootstrap` banner naming the source. The downstream build pipeline reads them as-is; the banner is the user's signal that running the corresponding `arsenal-planning:<skill>` later is a one-step deepening operation.

| Artifact | Required | If missing |
|---|---|---|
| `.arsenal/FEATURES.md` or `.arsenal/features/` | Yes | User picks A (run arsenal-planning first), B (ingest docs), or C (interview). |
| `.arsenal/design/UX.md` | UI projects only | Same A / B / C choice. |
| `.arsenal/design/DESIGN.md` | UI projects only | Same A / B / C choice. |
| `.arsenal/strategy/MVP_SPEC.md` | Optional | Read if present; skip if not. Setup never generates this — arsenal-planning's `mvp` owns it. |
| `.arsenal/design/mockups/` | Recommended for UI | Soft prompt: run `arsenal-planning:mockups` to generate briefs, then feed each into Claude Design / Stitch / Open Design / v0. |

**How it works**

1. **Probe state.** Check upstream artifacts at canonical paths. Build a per-artifact state map.
2. **Decide path per artifact.** For each required-but-missing artifact, ask the user A / B / C.
3. **Execute paths.** Path A reads existing files directly. Path B dispatches `references/ingest-from-docs-prompt.md` per artifact. Path C runs an intake interview in the main session, then dispatches `references/bootstrap-from-intake-prompt.md` per artifact. Each subagent writes one lightweight planning artifact with the banner.
4. **Stack-only discovery.** ~6–8 questions max — framework, styling, database/CMS, auth, state management, hosting, key integrations, project intent. Skip anything obvious from context.
5. **Generate build anchors.** Dependency order — ARCHITECTURE → CONVENTIONS → DESIGN_SYSTEM → TASKS → CLAUDE.
6. **Set up mockup slot.** `mkdir -p .arsenal/design/mockups`. Surface its absence in the final report if empty.
7. **Review + handoff.** Files created, sizes flagged, per-file path provenance (A / B / C), deepening invite for any B / C artifact, next step (`/arsenal-build:design 1`).

**File sizing — load-bearing**

| File | Target | Read by |
|---|---|---|
| `CLAUDE.md` | **≤120 lines hard ceiling** | Every session at load |
| `ARCHITECTURE.md` | ≤500 | `generate-feature-briefs` (slices data flow + schema sections) |
| `CONVENTIONS.md` | ≤500 | `generate-feature-briefs` (slices pattern sections) |
| `DESIGN_SYSTEM.md` | ≤500 (UI only) | `generate-design-briefs` (slices token map + component sections) |
| `TASKS.md` | No cap | `expand-phase`, `close-*-phase` (state machine — read + write) |

Each output section is self-contained so brief generators can excerpt cleanly. Cross-references are precise (`.arsenal/ARCHITECTURE.md § Data Flow / Recipe Capture`), not vague.

**Load-bearing rules in CLAUDE.md verbatim**

CLAUDE.md is the only place where the build pipeline's load-bearing rules appear in full — subagents inherit them through CLAUDE.md and can't load secondary docs to fetch them:

- **Component boundary** — extend not redesign; `Component extended: <path> — <why>` commit note required
- **Spec amendment (Tier 1 / Tier 2)** — see Load-bearing rules section above
- **Tasks expand on-demand + design-then-feature ordering**

**A few load-bearing decisions**

- **`TASKS.md` is a phase scaffold, not a complete task list.** Phase entries are placeholders that `expand-phase` expands on demand against fresh context.
- **`DESIGN_SYSTEM.md` is stack-specific implementation** (CSS vars, theme tokens). `DESIGN.md` stays canonical and untouched.
- **`CONVENTIONS.md` mandates concrete sections** — KISS / YAGNI / Functional First, "Before Writing Any Code", anti-patterns — with **real working code** for the chosen stack, not pseudocode.
- **In split-features mode**, `setup` reads only `.arsenal/features/README.md`, never individual feature files.

**How to use it**

- **Slash command:** `/arsenal-build:setup`
- **Or trigger with:** "set up the project", "set up the build pipeline", "bootstrap the project", "scaffold project docs", "set up CLAUDE.md", "ready to start building", "bridge planning to build"

- **Inputs:** any combination of `.arsenal/FEATURES.md` (or `.arsenal/features/`), `.arsenal/design/UX.md` (UI), `.arsenal/design/DESIGN.md` (UI). Missing inputs trigger Path B (user-pointed docs) or Path C (interview). `MVP_SPEC.md` optional context.
- **Outputs (always):** `CLAUDE.md` at root; `.arsenal/ARCHITECTURE.md`, `.arsenal/CONVENTIONS.md`, `.arsenal/design/DESIGN_SYSTEM.md` (UI only), `.arsenal/TASKS.md`. Creates `.arsenal/design/mockups/` directory.
- **Outputs (Paths B / C only):** lightweight `.arsenal/FEATURES.md`, `.arsenal/design/UX.md`, `.arsenal/design/DESIGN.md` at canonical paths, each carrying the `Generated by setup` banner.

---

### `design` — build visual components

The design half of each phase. Loops design tasks — each task builds a component with hardcoded data, exercises every state from the design brief's variant coverage, and commits atomically. **Does NOT push or PR.** Returns the branch for the feature pipeline.

**Per-task pipeline (inside `run-task-design`):**

1. Optional researcher (when `research: yes`).
2. Design-implementer: reads design brief, mockup at cited region, token map, variant coverage. Builds the component at the path declared in DESIGN_SYSTEM.md (or the context brief). Hardcoded data only — no server actions, no live queries.
3. Visual fidelity review: static analysis — token map adherence, locked-primitive citation, state coverage, mockup ↔ code match, hardcoded-data discipline.
4. Code quality review.
5. Atomic commit + `[x]` in `TASKS.md` under `### Design tasks`.

**No automatic impeccable dispatch** for shape / craft. If the design brief is thin, the implementer BLOCKS — the user invokes `impeccable:shape <surface>` manually if they want.

**Phase wrap (`close-design-phase`):** two gates — (1) optional impeccable audit + polish, gateable; (2) docs update if scope drifted. Writes `.arsenal/tasks/phase-N/design-summary.md` for the feature-pipeline PR body. **No push, no PR, no CodeRabbit, no trim** — that's all the feature close's job.

**How to use:**

- **Slash command:** `/arsenal-build:design N`
- **Or trigger with:** "design phase 1", "build the visual components for phase 2"

**Inputs:** `TASKS.md` (required), `UX.md`, `DESIGN_SYSTEM.md`, `DESIGN.md`, feature specs (design-relevant parts), `.arsenal/design/mockups/` (recommended).
**Outputs:** atomic commits of design-pipeline components, `[x]` flips in `### Design tasks`, `.arsenal/tasks/phase-N/design-summary.md`.

---

### `features` — wire components to real data

The feature half of each phase. Loops feature tasks — each task wires existing components (committed by the design pipeline earlier in the same phase) to real data: queries, mutations, services, server actions, state management. **Opens the single PR per phase at the end.**

**Per-task pipeline (inside `run-task-feature`):**

1. Optional researcher (when `research: yes`).
2. Feature-implementer: reads context brief with `## Available components` manifest enumerating design-pipeline-committed paths. Wires real data. Treats components as read-only design surfaces; may extend (new props/variants/states) but not redesign (visual treatment / spacing / typography / color). Every commit touching a component file includes a `Component extended: <path> — <why>` note.
3. Spec compliance review: checks AC compliance AND enforces the `Component extended:` rule. Purely visual changes disguised as extensions are CRITICAL failures.
4. Code quality review.
5. Atomic commit + `[x]` in `TASKS.md` under `### Feature tasks`.

**No impeccable, no visual fidelity gate** — both are design-pipeline territory. If a feature task seems to need design judgement, it was misclassified — the implementer BLOCKS with `reason: purely visual change required — redirect to design pipeline`.

**Phase wrap (`close-feature-phase`):** 6 gates — final integration test → Playwright (if configured) → docs update (if drift) → **CodeRabbit (hard gate, covers full phase)** → trim `TASKS.md` + archive `.arsenal/tasks/phase-N/` → push branch + open PR. CodeRabbit covers the full phase (design + feature commits together) — single pass per PR.

**How to use:**

- **Slash command:** `/arsenal-build:features N`
- **Or trigger with:** "begin work on phase 1" (after design half complete), "wire the components to data", "ship the phase"

Run AFTER `design` + `close-design-phase` for the same phase. If invoked before the design half completes, this orchestrator stops and tells the user to run the design half first.

Scope flexibly — *"phase 1, just the <feature-name> feature"* or *"the <story-name> story within <feature>"* both work. Pick model per task: simple → Sonnet/Haiku, complex → Opus.

**Inputs:** `TASKS.md`, `ARCHITECTURE.md`, `CONVENTIONS.md`, `DESIGN_SYSTEM.md` (required); feature specs; phase branch with design-pipeline commits already present.
**Outputs:** atomic commits of feature-pipeline data wiring, `[x]` flips in `### Feature tasks`, trimmed `TASKS.md`, archived briefs, single PR for the phase.

**Pure feature-domain phases** (no design tasks — pure backend, schema migrations, infrastructure): `features` runs the whole phase end-to-end, no design half. The `### Design tasks` subsection is the `_None — pure feature-domain phase._` placeholder.

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
- **Analytics:** PostHog — re-uses Phase 0 instance if `setup` already set one up.
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

- **Inputs:** any of `.arsenal/strategy/MVP_SPEC.md`, `.arsenal/strategy/research/MARKET_RESEARCH.md` (unified dossier), `.arsenal/design/DESIGN_SYSTEM.md` (optional). Otherwise the skill asks.
- **Outputs:** a deployable landing page repo, or a `/coming-soon` route inside an existing app — actual code, not specs.

---

### `dispatch-parallel` — fan out independent investigations

A utility skill, **off the linear pipeline**. Dispatches 2–5 read-only investigations to parallel investigator subagents and reconciles their results into a single `SUMMARY.md` with cross-investigation overlap detection and severity-tagged recommendations. Use for audits, debug sessions, code analysis — any case where the work is genuinely disjoint and parallelization actually pays for itself.

Also shipped in arsenal-planning (where `market-analysis` requires it) and arsenal-build-io. All three copies are identical — Claude Code's plugin namespacing addresses them separately. Install whichever plugin(s) you have; the skill behaves the same in each.

**The independence gate is the whole skill.** Before any dispatch, it checks three criteria — disjoint scope, no shared mutations, result-independence — and refuses to fan out if they don't hold. Failing the gate is the success case for dependent work; the skill recommends sequential execution.

**Does NOT compose with `run-task-{design,feature}`.** Investigations are read-only by design — investigator subagents have zero write capability. When findings recommend code changes, the SUMMARY's "Next steps" block points the user at the per-task pipelines (`run-task-design` / `run-task-feature` for sequential, one-fix-at-a-time work) or the orchestrators (`design` / `features` for pattern-spanning work as a TASKS.md phase). The two skills connect through filesystem and user judgment, not direct invocation.

**Locked contracts:**
- **Count bounds:** N = 1 refuses with suggestion; 2 ≤ N ≤ 5 normal; N ≥ 6 hard-refuses (recommend phase modeling).
- **Idempotence:** default skip per-investigation if `investigation-{N}-result.md` exists; `--force` regenerates.
- **Conflict handling:** when investigations contradict each other, SUMMARY flags conflicts and the skill exits cleanly.
- **Investigator tools:** broad read (Read / Glob / Grep / read-only Bash / WebSearch / Firecrawl / claude-in-chrome), zero write.

**How to use it**

- **Slash command:** `/arsenal-build:dispatch-parallel`
- **Or trigger with:** "investigate these in parallel", "fan out on these issues", "run these checks concurrently", "audit X, Y, and Z separately"

- **Inputs:** 2–5 investigation descriptions via `--investigation` (repeated) or `--from-file <path>`, optional `--force`, optional `--max <N>` (capped at 5).
- **Outputs (`.arsenal/tasks/parallel/<run-id>/`):** `investigation-N-result.md` per investigation (≤3k tokens), `SUMMARY.md` (aggregated with overlap detection + conflict flags + next-step recommendations).

## File layout

Project-side artifact layout (what arsenal-build reads and writes in a project):

```
project-root/
├── CLAUDE.md                              ← stays at root
└── .arsenal/
    ├── config.yaml                        ← optional override
    ├── ARCHITECTURE.md                    ← was docs/
    ├── CONVENTIONS.md                     ← was docs/
    ├── TASKS.md                           ← was docs/
    ├── FEATURES.md                        ← was planning/ (single-mode)
    ├── features/                          ← was planning/ (split-mode)
    │   ├── README.md
    │   └── <slug>.md
    ├── design/                            ← grouped; read as a unit by design pipeline
    │   ├── UX.md                          ← was docs/
    │   ├── DESIGN.md                      ← was docs/
    │   ├── DESIGN_SYSTEM.md               ← was docs/
    │   └── mockups/                       ← was docs/
    ├── tasks/                             ← was .tasks/ at project root
    │   ├── phase-N/
    │   ├── parallel/
    │   └── archive/
    └── strategy/                          ← user archive; DENIED during build
        ├── MARKET_RESEARCH.md             ← was planning/
        ├── RESEARCH_PLAN.md
        ├── MVP_SPEC.md
        ├── mockup-briefs/
        ├── GTM_STRATEGY.md
        └── REVENUE_MODEL.md
```

Plugin-side skill layout (what's inside this repo):

```
arsenal-build/
├── .claude-plugin/
│   └── plugin.json                        # plugin manifest (name: arsenal-build)
├── skills/                                # folder names are numbered by pipeline order; frontmatter `name:` is the bare slug
│   ├── 01-setup/SKILL.md           # name: setup (bridge — runs once before phases)
│   │
│   ├── # Design half — 02 family:
│   ├── 02-design/SKILL.md                 # name: design (orchestrator)
│   ├── 02a-expand-phase/SKILL.md          # name: expand-phase (first sub dispatched)
│   ├── 02b-generate-design-briefs/
│   │   ├── SKILL.md                       # name: generate-design-briefs
│   │   └── references/design-brief-prompt.md
│   ├── 02c-run-task-design/
│   │   ├── SKILL.md                       # name: run-task-design
│   │   └── references/{researcher,design-implementer,visual-fidelity-reviewer,quality-reviewer}-prompt.md
│   ├── 02d-close-design-phase/SKILL.md    # name: close-design-phase (2 gates; no push, no PR)
│   │
│   ├── # Feature half — 03 family:
│   ├── 03-features/SKILL.md               # name: features (orchestrator)
│   ├── 03a-generate-feature-briefs/SKILL.md  # name: generate-feature-briefs
│   ├── 03b-run-task-feature/
│   │   ├── SKILL.md                       # name: run-task-feature
│   │   └── references/{researcher,feature-implementer,spec-reviewer,quality-reviewer}-prompt.md
│   ├── 03c-close-feature-phase/SKILL.md   # name: close-feature-phase (6 gates; opens single PR per phase)
│   │
│   ├── 04-landing/SKILL.md                # name: landing (standalone)
│   │
│   └── dispatch-parallel/                 # utility, off-pipeline (no number prefix) — also in arsenal-planning + arsenal-build-io
│       ├── SKILL.md                       # name: dispatch-parallel
│       └── references/investigator-prompt.md
├── PIPELINE.md
├── LICENSE
└── README.md
```

> **Folder numbering convention.** Pipeline skills carry numeric prefixes (`01-`, `02-`, `02a-`, …) reflecting execution order so a reader can see the flow when listing the directory. Utility skills (`dispatch-parallel`) sit unnumbered to signal "off-pipeline." Claude Code uses the frontmatter `name:` to register the skill, so slash commands stay clean — `/arsenal-build:design`, not `/arsenal-build:02-design`.

## Philosophy

- **Atomic commits, fresh context.** Every per-task subagent dispatches in fresh context with a brief generated from the surrounding planning artifacts. A blocked task is never retried with the same model and the same prompt.
- **Briefs hand off via filesystem.** Subagents never inherit session history. Per-task briefs live at `.arsenal/tasks/phase-N/task-N-context.md` and `task-N-design.md`; SUMMARYs sit in the phase folder. Easy to inspect, easy to regenerate.
- **Two pipelines, one PR.** Design judgement and feature wiring fail differently. Keep them in separate prompts, separate review stages, separate commits. Same branch, same PR — the boundary is enforced by the `Component extended:` rule and Tier 1 / Tier 2 spec amendment routing.
- **Match depth to stakes.** Pick model per task. Simple → Sonnet/Haiku. Complex → Opus. Don't waste tokens.

## Pairs with

- **[arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning)** — the planning half. Produces the `MVP_SPEC.md`, `FEATURES.md`, `UX.md`, `DESIGN.md`, and `.arsenal/strategy/mockup-briefs/` artifacts that arsenal-build consumes. Optional but recommended; you can also produce planning artifacts by hand at canonical paths.
- **[arsenal-build-io](https://github.com/Outer-Heaven-Technologies/arsenal-build-io)** — the iOS variant. Same pipeline shape, SwiftUI + Xcode + simulator-driven visual fidelity gate on the design pipeline. Install alongside arsenal-build if you ship both web and iOS surfaces from the same workspace.

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

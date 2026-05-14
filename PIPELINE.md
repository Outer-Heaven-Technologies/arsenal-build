# Pipeline — arsenal-build

The web/frontend execution half of the arsenal pipeline. Takes planning artifacts (typically from [arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning), but any equivalent source works) and ships real code.

For native iOS projects, see [arsenal-build-io](https://github.com/Outer-Heaven-Technologies/arsenal-build-io) — same pipeline shape with SwiftUI + Xcode + simulator-driven visual fidelity gate.

## Linear flow

```
   .arsenal/FEATURES.md  +  .arsenal/strategy/{MVP_SPEC,MARKET_RESEARCH}.md  +  .arsenal/design/{UX,DESIGN}.md  +  .arsenal/design/mockups/
            (typically produced by arsenal-planning)
                              │
                              ▼
                       setup  ──►  CLAUDE.md + .arsenal/{ARCHITECTURE,CONVENTIONS,TASKS}.md + .arsenal/design/DESIGN_SYSTEM.md
                              │
                              ▼
   ┌─────── per-phase ─────────────────────────────────────────────────────────┐
   │                                                                            │
   │   design pipeline                            feature pipeline               │
   │   ───────────────                            ────────────────               │
   │   design               ──►  close-design-   ──►  features            ──►  close-feature-phase   ──►  PR
   │   (loop design tasks)       phase                (loop feature tasks)      (tests, CodeRabbit, push, PR)
   │                             (visual audit,
   │                              optional;
   │                              NO push, NO PR)
   └────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                          landing  (standalone, after MVP is functional)
```

**Strict per-phase sequence:** `design → close-design-phase → features → close-feature-phase`. Design close must complete before the feature half starts; the feature pipeline's available-components manifest sources from design-pipeline commits already on the branch.

**Shared branch, single PR:** both halves commit to `phase-N/short-description`. `close-design-phase` does NOT push or PR — it returns the branch. `close-feature-phase` opens the single PR at the end.

## Per-phase pipeline detail

```
                 design pipeline                          feature pipeline
                 ───────────────                          ────────────────
phase branch ──► design          ──►   close-design-  ──►   features              ──►   close-feature-phase   ──► PR
                 (loop design          phase                (loop feature tasks)        (tests, CodeRabbit, push, PR)
                  tasks)               (visual audit,
                                        optional)
                       │                       │                        │                                │
                       ├─► expand-phase        └─► design-summary.md    ├─► generate-feature-briefs      └─► single PR per phase
                       ├─► generate-design-                             ├─► run-task-feature
                       │   briefs                                       │   (researcher → impl →
                       └─► run-task-design                              │    spec → quality →
                           (researcher → impl                           │    commit)
                            → visual fidelity                           │
                            → quality
                            → commit)
```

**Component boundary** (load-bearing): the feature pipeline may **extend** components from the design pipeline (new props, variants, or states the data layer needs) but never **redesign** them (visual treatment, spacing, typography, color). Feature commits that touch component files include a `Component extended: <path> — <why>` note. Visual changes BLOCK with redirect to the design pipeline. Enforced in `run-task-feature`'s feature-implementer prompt and spec reviewer.

**Spec amendment** (load-bearing): Tier 1 in-line amendments to `FEATURES.md` (additive/clarifying) ship in the same atomic commit as the code change with a `Spec amended: <feature-slug> — <why>` note. Tier 2 intent changes BLOCK and route back to spec authoring (arsenal-planning:features, or wherever the spec was authored). The phase wrap collates Tier 1 amendments into the PR body.

**Impeccable boundaries:**

- NOT dispatched from any feature-pipeline skill.
- NOT auto-dispatched from `run-task-design` (user invokes manually when blocked on under-specified design).
- MAY be invoked from `close-design-phase` as a phase-level audit, gateable (user must opt in).

## Artifact dependency graph

Which file each skill **reads** (`R`) and **writes** (`W`):

| Skill | Reads | Writes |
|---|---|---|
| `setup` | `.arsenal/FEATURES.md` / `.arsenal/features/`†, `.arsenal/design/UX.md`† (UI only), `.arsenal/design/DESIGN.md`† (UI only), `.arsenal/strategy/MVP_SPEC.md` (optional context); soft prompts for `.arsenal/design/mockups/` on UI projects | `CLAUDE.md`, `.arsenal/ARCHITECTURE.md`, `.arsenal/CONVENTIONS.md`, `.arsenal/design/DESIGN_SYSTEM.md` (UI only), `.arsenal/TASKS.md`; AND when missing, lightweight `.arsenal/FEATURES.md` / `.arsenal/design/UX.md` / `.arsenal/design/DESIGN.md` with `Generated by setup` banner |
| `design` | `.arsenal/TASKS.md`*, `.arsenal/design/UX.md`, feature specs, `.arsenal/design/DESIGN_SYSTEM.md`, `.arsenal/design/DESIGN.md`, mockups (optional); args: `--phase`, `--scope` | dispatches `expand-phase`, `generate-design-briefs`, `run-task-design` per design task, then `close-design-phase` |
| `features` | `.arsenal/TASKS.md`*, `.arsenal/ARCHITECTURE.md`*, `.arsenal/CONVENTIONS.md`*, `.arsenal/design/DESIGN_SYSTEM.md`, feature specs, phase branch with design-pipeline commits already present; args: `--phase`, `--scope` | dispatches `generate-feature-briefs`, `run-task-feature` per feature task, then `close-feature-phase` |
| `expand-phase` (shared) | `.arsenal/TASKS.md`*, `.arsenal/design/UX.md`, feature specs, `.arsenal/ARCHITECTURE.md` skim, `.arsenal/CONVENTIONS.md` skim; args: `--phase`, `--scope`, `--force` | `.arsenal/TASKS.md` phase block rewritten with concrete tagged tasks grouped under `### Design tasks` and `### Feature tasks`; tags include `domain:` / `research:` |
| `generate-design-briefs` | `.arsenal/TASKS.md`* (re-read for tags), per-task UX / feature spec (design parts) / DESIGN_SYSTEM cites / mockups; args: `--phase`, `--task`, `--force` | `.arsenal/tasks/phase-N/task-N-context.md` + `.arsenal/tasks/phase-N/task-N-design.md` per `domain: design` task (≤3k / ≤1.7k tokens) |
| `generate-feature-briefs` | `.arsenal/TASKS.md`* (re-read for tags), per-task feature spec / ARCHITECTURE / CONVENTIONS cites, design-pipeline-committed component paths via `git log`; args: `--phase`, `--task`, `--force` | `.arsenal/tasks/phase-N/task-N-context.md` per `domain: feature` task (≤3k tokens), with `## Available components` manifest of design-pipeline commits |
| `run-task-design` | per-task briefs at `.arsenal/tasks/phase-N/task-N-{context,design,research}.md`*, mockup files; args: `--phase`, `--task`, `--force` | code commits (semantic, hardcoded data), `[x]` flip in `.arsenal/TASKS.md` under `### Design tasks`, optional research file |
| `run-task-feature` | per-task brief at `.arsenal/tasks/phase-N/task-N-context.md`*, available-components manifest from the brief; args: `--phase`, `--task`, `--force` | code commits (semantic, with `Component extended:` notes when touching component files), `[x]` flip in `.arsenal/TASKS.md` under `### Feature tasks` |
| `close-design-phase` | aggregated design briefs in `.arsenal/tasks/phase-N/`, phase branch, `impeccable` (optional, gateable at Gate 1) | `.arsenal/tasks/phase-N/design-summary.md` for the PR body; optional audit-fix commits; **does NOT push or PR** |
| `close-feature-phase` | `.arsenal/TASKS.md`, full phase block, `.arsenal/tasks/phase-N/` (including `design-summary.md`), phase branch | fix commits, trimmed `.arsenal/TASKS.md`, archived `.arsenal/tasks/phase-N/`, pushed branch + single PR for the phase |
| `landing` (standalone) | `.arsenal/strategy/MVP_SPEC.md`, `.arsenal/strategy/research/MARKET_RESEARCH.md` (unified — contains competitive analysis in §3), `.arsenal/design/DESIGN_SYSTEM.md` | landing page repo or route |
| `dispatch-parallel` (utility, off-pipeline) | 2–5 investigation descriptions (CLI args or `--from-file`); args: `--investigation`, `--from-file`, `--force`, `--max` | `.arsenal/tasks/parallel/<run-id>/investigation-N-result.md` per investigation (≤3k tokens), `SUMMARY.md` (aggregated, with file-overlap detection and severity-tagged recommendations) |

`*` = required (must exist at invocation time).
`†` = required for the build pipeline overall; if absent at setup time, setup offers Path B (ingest user-pointed source docs) or Path C (intake interview) to generate a lightweight version with a `Generated by setup` banner. Path A (consume existing artifacts, typically from arsenal-planning) is recommended for depth.
All others soft — skill prompts to run upstream or proceeds with reduced fidelity.

> **`dispatch-parallel` is shipped in three plugins** — arsenal-planning (where `market-analysis` requires it), arsenal-build (web), and arsenal-build-io (iOS). All copies are identical and Claude Code's plugin namespacing addresses them separately (`/arsenal-planning:dispatch-parallel`, `/arsenal-build:dispatch-parallel`, `/arsenal-build-io:dispatch-parallel`).

## Cross-plugin handoff

Upstream of `setup`, you may have planning artifacts already produced by [arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning). This is the recommended path because arsenal-planning produces the deepest specs. Setup also works without them: when an artifact is missing, setup offers Path B (ingest user-pointed source docs) or Path C (intake interview) to generate a lightweight version with a `Generated by setup` banner. Hand-authoring at canonical paths is also fine.

| Artifact arsenal-build expects | Where arsenal-planning produces it | If missing at setup time |
|---|---|---|
| `.arsenal/strategy/MVP_SPEC.md` | `arsenal-planning:mvp` | `setup` reads if present; skips if not. Setup never generates MVP_SPEC. |
| `.arsenal/FEATURES.md` or `.arsenal/features/` | `arsenal-planning:features` | `setup` offers Path A (run arsenal-planning:features first; recommended), B (ingest user-pointed docs), or C (intake interview) |
| `.arsenal/design/UX.md` | `arsenal-planning:ux-web` or `arsenal-planning:ux-app` | UI projects only; same A / B / C choice |
| `.arsenal/design/DESIGN.md` | `arsenal-planning:design` | UI projects only; same A / B / C choice |
| `.arsenal/design/mockups/` (populated) | User generates via Claude Design / Stitch / v0 from `arsenal-planning:mockups` briefs | Soft prompt — visual fidelity output less consistent without |
| `.arsenal/strategy/research/MARKET_RESEARCH.md` | `arsenal-planning:market-analysis` | `landing` falls back to asking for competitor info inline; setup reads as optional context but never generates |

**Recommended workflow with arsenal-planning installed:** run `arsenal-planning` skills end-to-end through `mockups`, generate mockups into `.arsenal/design/mockups/`, then run `arsenal-build:setup` and proceed phase-by-phase.

**Recommended workflow without arsenal-planning:** produce the artifacts above by hand at the canonical paths and arsenal-build will accept them. The `.arsenal/config.yaml` file can remap directories if your project uses a different layout (see Configuration in README).

**iOS variant:** for native iOS (SwiftUI + Xcode) projects, use [arsenal-build-io](https://github.com/Outer-Heaven-Technologies/arsenal-build-io) instead. Same upstream planning artifacts, same overall pipeline shape, but the design pipeline gets a simulator-driven visual fidelity gate and per-task finishers (`harden`, `animate`, `clarify`, `typeset`, `arrange`).

## Entry-point matrix

Where to start given what you already have:

| You have | Start with | Skip |
|---|---|---|
| All planning docs + mockups generated, no code | `setup` | — |
| Project docs already written (CLAUDE.md, ARCHITECTURE.md, etc.) | `design` (start of each phase); `features` runs after | `setup` |
| Phase branch with design commits already present | `features` | `design` (already done) |
| TASKS.md phase needs re-expansion (specs changed) | `expand-phase --phase N --force` | — |
| Need only a landing page | `landing` | everything |

When you enter mid-pipeline, the active skill detects upstream artifacts and uses them as context. Missing required artifacts trigger a hard prompt (route to arsenal-planning or supply path); missing soft artifacts trigger a soft note.

## Branching paths

Skip stages that don't fit:

| Stage | Skip when | Cost of skipping |
|---|---|---|
| `setup` | Project docs already exist and you trust them | Some downstream skills may break if doc structure doesn't match expectations |
| `design` (design half) | Pure feature-domain phase (`### Design tasks` is `_None — pure feature-domain phase._`) | None — `features` handles the whole phase |
| `landing` | Internal tool; no public surface | None |

## Auto-trigger vs explicit invocation

These require explicit invocation (no auto-trigger via natural language):

- `setup` — scaffolds five doc files; deliberate
- `design` — writes code and commits; very deliberate
- `features` — writes code and commits; very deliberate

Auto-fire on natural language matching their description:

- `landing` — fires on "build a landing page", "create a waitlist", "coming soon page"
- `dispatch-parallel` — fires on "investigate these in parallel", "fan out on these issues", "run these checks concurrently", "audit X, Y, and Z separately"

Sub-skills (`expand-phase`, `generate-*-briefs`, `run-task-*`, `close-*-phase`) don't auto-fire — they're dispatched by their orchestrator. You can invoke them explicitly for surgical work when upstream artifacts change mid-phase.

## File layout (build artifacts)

```
project-root/
├── CLAUDE.md                              # setup (root index + load-bearing rules)
└── .arsenal/
    ├── config.yaml                        # optional override (rarely needed)
    ├── ARCHITECTURE.md                    # written by setup
    ├── CONVENTIONS.md                     # written by setup
    ├── TASKS.md                           # phase scaffold (read+write state machine)
    ├── FEATURES.md                        # single-mode (or features/ for split-mode)
    ├── features/                          # split-mode
    │   ├── README.md
    │   └── <slug>.md
    ├── design/                            # grouped; read as a unit by design pipeline
    │   ├── UX.md
    │   ├── DESIGN.md
    │   ├── DESIGN_SYSTEM.md               # stack-specific impl; UI only
    │   └── mockups/                       # populated by user from arsenal-planning:mockups briefs
    ├── tasks/                             # per-task briefs + phase artifacts (gitignored)
    │   ├── phase-1/
    │   │   ├── task-1-context.md
    │   │   ├── task-1-design.md           # if domain: design
    │   │   ├── task-1-research.md         # if research: yes
    │   │   ├── design-summary.md          # close-design-phase writes this
    │   │   └── spec-amendments.md         # close-feature-phase aggregates Tier 1 amendments
    │   ├── parallel/                      # dispatch-parallel outputs
    │   └── archive/                       # close-feature-phase moves completed phases here
    │       └── phase-1/
    │           └── tasks.md
    └── strategy/                          # user archive; DENIED during build execution
        ├── MARKET_RESEARCH.md
        ├── RESEARCH_PLAN.md
        ├── MVP_SPEC.md
        ├── mockup-briefs/
        ├── GTM_STRATEGY.md
        └── REVENUE_MODEL.md
```

All artifacts live under `.arsenal/` at the project root. File names are not configurable; only the `.arsenal/` root location may be overridden via `.arsenal/config.yaml` in unusual cases.

## Pairs with

- **[arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning)** — produces upstream planning artifacts. Optional but recommended.
- **[arsenal-build-io](https://github.com/Outer-Heaven-Technologies/arsenal-build-io)** — the iOS variant. Run alongside arsenal-build if your workspace ships both web and iOS surfaces.

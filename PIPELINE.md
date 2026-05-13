# Pipeline — arsenal-build

The execution half of the arsenal pipeline. Takes planning artifacts (typically from [arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning), but any equivalent source works) and ships real code:

```
planning artifacts ──► anchor-files ──► per-phase design + feature pipelines ──► single PR per phase
                                                                                              │
                                                                                              ▼
                                                                                          landing
                                                                                          (standalone)
```

Each phase splits into **design tasks** (visual components with hardcoded data) and **feature tasks** (wire components to real data). The two pipelines run sequentially on the same branch, sharing a single PR.

## Linear flow

```
   planning/{FEATURES,MVP_SPEC,MARKET_RESEARCH}.md  +  docs/{UX,DESIGN}.md  +  docs/mockups/
            (typically produced by arsenal-planning)
                              │
                              ▼
                       anchor-files  ──►  CLAUDE.md + docs/{ARCHITECTURE,CONVENTIONS,DESIGN_SYSTEM,TASKS}.md
                              │
                              ▼
   ┌─────── per-phase ─────────────────────────────────────────────────────────┐
   │                                                                            │
   │   design pipeline                            feature pipeline               │
   │   ───────────────                            ────────────────               │
   │   design-{web,ios}    ──►  close-design-     ──►  features-{web,ios}  ──►  close-feature-phase-{web,ios}  ──►  PR
   │   (loop design tasks)      phase-{web,ios}        (loop feature tasks)     (tests, CodeRabbit, push, PR)
   │                            (visual audit,
   │                             optional;
   │                             NO push, NO PR)
   └────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                          landing  (standalone, after MVP is functional)
```

**Strict per-phase sequence:** `design-* → close-design-phase-* → features-* → close-feature-phase-*`. Design close must complete before the feature half starts; the feature pipeline's available-components manifest sources from design-pipeline commits already on the branch.

**Shared branch, single PR:** both halves commit to `phase-N/short-description`. `close-design-phase-*` does NOT push or PR — it returns the branch. `close-feature-phase-*` opens the single PR at the end.

## Per-phase pipeline detail

```
                 design pipeline                                feature pipeline
                 ───────────────                                ────────────────
phase branch ──► design-{web,ios}   ──►   close-design-      ──►   features-{web,ios}     ──►   close-feature-phase-{web,ios}   ──► PR
                 (loop design             phase-{web,ios}          (loop feature tasks)         (tests, CodeRabbit, push, PR)
                  tasks)                  (visual audit,
                                           optional)
                       │                         │                        │                                │
                       ├─► expand-phase          └─► design-summary.md    ├─► generate-feature-briefs      └─► single PR per phase
                       ├─► generate-design-                               ├─► run-task-feature-
                       │   briefs                                         │   {web,ios}
                       └─► run-task-design-                               │   (researcher → impl →
                           {web,ios}                                      │    spec → quality →
                           (researcher → impl                             │    commit)
                            → visual fidelity                             │
                            → quality →
                            finishers (iOS)
                            → commit)
```

**Component boundary** (load-bearing): the feature pipeline may **extend** components from the design pipeline (new props, variants, or states the data layer needs) but never **redesign** them (visual treatment, spacing, typography, color). Feature commits that touch component files include a `Component extended: <path> — <why>` note. Visual changes BLOCK with redirect to the design pipeline. Enforced in `run-task-feature-*`'s feature-implementer prompt and spec reviewer.

**Spec amendment** (load-bearing): Tier 1 in-line amendments to `FEATURES.md` (additive/clarifying) ship in the same atomic commit as the code change with a `Spec amended: <feature-slug> — <why>` note. Tier 2 intent changes BLOCK and route back to spec authoring (arsenal-planning:features, or wherever the spec was authored). The phase wrap collates Tier 1 amendments into the PR body.

**Impeccable boundaries:**

- NOT dispatched from any feature-pipeline skill.
- NOT auto-dispatched from `run-task-design-*` for `shape` or `craft` (user invokes manually when blocked on under-specified design).
- AUTO-dispatched from `run-task-design-ios` as per-task finishers (`harden`, `animate`, `clarify`, `typeset`, `arrange`) per the task's `finishers: [list]` tag.
- MAY be invoked from `close-design-phase-*` as a phase-level audit, gateable (user must opt in).

## Artifact dependency graph

Which file each skill **reads** (`R`) and **writes** (`W`):

| Skill | Reads | Writes |
|---|---|---|
| `anchor-files` | `FEATURES.md` / `features/`*, `docs/UX.md`* (UI only), `docs/DESIGN.md`* (UI only), `MVP_SPEC.md` (optional context); soft prompts for `docs/mockups/` on UI projects | `CLAUDE.md`, `docs/ARCHITECTURE.md`, `docs/CONVENTIONS.md`, `docs/DESIGN_SYSTEM.md` (UI only), `docs/TASKS.md` |
| `design-web` | `TASKS.md`*, `UX.md`, feature specs, `DESIGN_SYSTEM.md`, `DESIGN.md`, mockups (optional); args: `--phase`, `--scope` | dispatches `expand-phase`, `generate-design-briefs`, `run-task-design-web` per design task, then `close-design-phase-web` |
| `features-web` | `TASKS.md`*, `ARCHITECTURE.md`*, `CONVENTIONS.md`*, `DESIGN_SYSTEM.md`, feature specs, phase branch with design-pipeline commits already present; args: `--phase`, `--scope` | dispatches `generate-feature-briefs`, `run-task-feature-web` per feature task, then `close-feature-phase-web` |
| `design-ios` | same as `design-web` + `docs/mockups/`, `mcp__xcode__*`, `mcp__ios-simulator__*` | dispatches `expand-phase`, `generate-design-briefs`, `run-task-design-ios` per design task, then `close-design-phase-ios` |
| `features-ios` | same as `features-web` + `mcp__xcode__*`, `mcp__ios-simulator__*` | dispatches `generate-feature-briefs`, `run-task-feature-ios` per feature task, then `close-feature-phase-ios` |
| `expand-phase` (shared) | `TASKS.md`*, `UX.md`, feature specs, `ARCHITECTURE.md` skim, `CONVENTIONS.md` skim (iOS also: `docs/mockups/`); args: `--phase`, `--surface`, `--scope`, `--force` | `TASKS.md` phase block rewritten with concrete tagged tasks grouped under `### Design tasks` and `### Feature tasks`; tags include `domain:` / `research:` (web + iOS) and `finishers:` / `snapshot:` (iOS, design-domain only) |
| `generate-design-briefs` | `TASKS.md`* (re-read for tags), per-task UX / feature spec (design parts) / DESIGN_SYSTEM cites / mockups; args: `--phase`, `--surface`, `--task`, `--force` | `.tasks/phase-N/task-N-context.md` + `.tasks/phase-N/task-N-design.md` per `domain: design` task (≤3k / ≤1.7k tokens) |
| `generate-feature-briefs` | `TASKS.md`* (re-read for tags), per-task feature spec / ARCHITECTURE / CONVENTIONS cites, design-pipeline-committed component paths via `git log`; args: `--phase`, `--surface`, `--task`, `--force` | `.tasks/phase-N/task-N-context.md` per `domain: feature` task (≤3k tokens), with `## Available components` manifest of design-pipeline commits |
| `run-task-design-web` | per-task briefs at `.tasks/phase-N/task-N-{context,design,research}.md`*, mockup files; args: `--phase`, `--task`, `--force` | code commits (semantic, hardcoded data), `[x]` flip in `TASKS.md` under `### Design tasks`, optional research file |
| `run-task-design-ios` | same + `mcp__xcode__*`, `mcp__ios-simulator__*`, `impeccable` (for finishers per `finishers:` tag) | code + finisher commits, `[x]` flip, `.tasks/phase-N/task-N-{before,after}.png` |
| `run-task-feature-web` | per-task brief at `.tasks/phase-N/task-N-context.md`*, available-components manifest from the brief; args: `--phase`, `--task`, `--force` | code commits (semantic, with `Component extended:` notes when touching component files), `[x]` flip in `TASKS.md` under `### Feature tasks` |
| `run-task-feature-ios` | same + `mcp__xcode__*`, `mcp__ios-simulator__*` (for tests on booted device) | code commits (with `Component extended:` notes), `[x]` flip |
| `close-design-phase-web` | aggregated design briefs in `.tasks/phase-N/`, phase branch, `impeccable` (optional, gateable at Gate 1) | `.tasks/phase-N/design-summary.md` for the PR body; optional audit-fix commits; **does NOT push or PR** |
| `close-design-phase-ios` | same + `mcp__xcode__*`, `mcp__ios-simulator__*`, `impeccable` (optional, gateable) | `.tasks/phase-N/design-summary.md`; optional audit-fix commits; **does NOT push or PR** |
| `close-feature-phase-web` | `TASKS.md`, full phase block, `.tasks/phase-N/` (including `design-summary.md`), phase branch | fix commits, trimmed `TASKS.md`, archived `.tasks/phase-N/`, pushed branch + single PR for the phase |
| `close-feature-phase-ios` | same + `mcp__xcode__*` (RunAllTests, GetBuildLog), `periphery` CLI, `swift-snapshot-testing` | fix commits, trimmed `TASKS.md`, archived `.tasks/phase-N/` (with `*.png` deleted), pushed branch + single PR |
| `landing` (standalone) | `MVP_SPEC.md`, `MARKET_RESEARCH.md` (unified — contains competitive analysis in §3), `DESIGN_SYSTEM.md` | landing page repo or route |
| `dispatch-parallel` (utility, off-pipeline) | 2–5 investigation descriptions (CLI args or `--from-file`), optional `--surface web\|ios`; args: `--investigation`, `--from-file`, `--surface`, `--force`, `--max` | `.tasks/parallel/<run-id>/investigation-N-result.md` per investigation (≤3k tokens), `SUMMARY.md` (aggregated, with file-overlap detection and severity-tagged recommendations) |

`*` = required. All others soft — skill prompts to run upstream or proceeds with reduced fidelity.

> **`dispatch-parallel` is shipped in both plugins** — arsenal-build for code audits / debug sessions / parallel reviews during execution, and arsenal-planning where `market-analysis` requires it. The two copies are identical and Claude Code's plugin namespacing addresses them separately (`/arsenal-build:dispatch-parallel` vs `/arsenal-planning:dispatch-parallel`).

## Cross-plugin handoff

Upstream of `anchor-files`, you need planning artifacts. Most users produce them via [arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning), but any source works — hand-authored, imported from elsewhere — as long as file names and locations match (or are overridden via `.arsenal/config.yaml`).

| Artifact arsenal-build expects | Where arsenal-planning produces it | If missing |
|---|---|---|
| `planning/MVP_SPEC.md` | `arsenal-planning:mvp` | `anchor-files` reads if present; skips if not |
| `planning/FEATURES.md` or `planning/features/` | `arsenal-planning:features` | `anchor-files` stops; route to `arsenal-planning:features` (or supply path via config / prompt) |
| `docs/UX.md` | `arsenal-planning:ux-{web,app,ios}` | UI projects only; `anchor-files` stops, route to the right `ux-*` |
| `docs/DESIGN.md` | `arsenal-planning:design` | UI projects only; `anchor-files` stops, route to `design` |
| `docs/mockups/` (populated) | User generates via Claude Design / Stitch / v0 from `arsenal-planning:mockups` briefs | Soft prompt — visual fidelity output less consistent without |
| `planning/MARKET_RESEARCH.md` | `arsenal-planning:market-analysis` | `landing` falls back to asking for competitor info inline |

**Recommended workflow with arsenal-planning installed:** run `arsenal-planning` skills end-to-end through `mockups`, generate mockups into `docs/mockups/`, then run `arsenal-build:anchor-files` and proceed phase-by-phase.

**Recommended workflow without arsenal-planning:** produce the artifacts above by hand at the canonical paths and arsenal-build will accept them. The `.arsenal/config.yaml` file can remap directories if your project uses a different layout (see Configuration in README).

## Entry-point matrix

Where to start given what you already have:

| You have | Start with | Skip |
|---|---|---|
| All planning docs + mockups generated, no code | `anchor-files` | — |
| Project docs already written (CLAUDE.md, ARCHITECTURE.md, etc.) | `design-{web,ios}` (start of each phase); `features-{web,ios}` runs after | `anchor-files` |
| Phase branch with design commits already present | `features-{web,ios}` | `design-{web,ios}` (already done) |
| TASKS.md phase needs re-expansion (specs changed) | `expand-phase --phase N --force` | — |
| Need only a landing page | `landing` | everything |

When you enter mid-pipeline, the active skill detects upstream artifacts and uses them as context. Missing required artifacts trigger a hard prompt (route to arsenal-planning or supply path); missing soft artifacts trigger a soft note.

## Branching paths

Skip stages that don't fit:

| Stage | Skip when | Cost of skipping |
|---|---|---|
| `anchor-files` | Project docs already exist and you trust them | Some downstream skills may break if doc structure doesn't match expectations |
| `design-{web,ios}` (design half) | Pure feature-domain phase (`### Design tasks` is `_None — pure feature-domain phase._`) | None — `features-{web,ios}` handles the whole phase |
| Web vs iOS variants | Pick by project root: `*.xcodeproj` / `Package.swift` → iOS; `package.json` / `Cargo.toml` / `go.mod` → web | — |
| `landing` | Internal tool; no public surface | None |

## Auto-trigger vs explicit invocation

These require explicit invocation (no auto-trigger via natural language):

- `anchor-files` — scaffolds five doc files; deliberate
- `design-web` / `design-ios` — writes code and commits; very deliberate
- `features-web` / `features-ios` — writes code and commits; very deliberate

Auto-fire on natural language matching their description:

- `landing` — fires on "build a landing page", "create a waitlist", "coming soon page"

- `dispatch-parallel` — fires on "investigate these in parallel", "fan out on these issues", "run these checks concurrently", "audit X, Y, and Z separately"

Sub-skills (`expand-phase`, `generate-*-briefs`, `run-task-*`, `close-*-phase-*`) don't auto-fire — they're dispatched by their orchestrator. You can invoke them explicitly for surgical work when upstream artifacts change mid-phase.

## File layout (build artifacts)

```
project-root/
├── CLAUDE.md                          # anchor-files (root index + load-bearing rules)
├── docs/                              # written by anchor-files
│   ├── ARCHITECTURE.md
│   ├── CONVENTIONS.md
│   ├── DESIGN_SYSTEM.md               # stack-specific impl; UI only
│   ├── TASKS.md                       # phase scaffold (read+write state machine)
│   └── mockups/                       # populated by user (or future generate-mockups skill)
├── .arsenal/
│   └── config.yaml                    # optional path overrides (see Configuration in README)
├── .tasks/                            # per-task briefs + phase artifacts (gitignored)
│   ├── phase-1/
│   │   ├── task-1-context.md
│   │   ├── task-1-design.md           # if domain: design
│   │   ├── task-1-research.md         # if research: yes
│   │   ├── task-1-before.png          # iOS only
│   │   ├── task-1-after.png           # iOS only
│   │   ├── design-summary.md          # close-design-phase-* writes this
│   │   └── spec-amendments.md         # close-feature-phase-* aggregates Tier 1 amendments
│   └── archive/                       # close-feature-phase-* moves completed phases here
│       └── phase-1/
│           └── tasks.md
```

The `docs/` and `planning/` wrapping directories are configurable via `.arsenal/config.yaml`. File names are not.

## Pairs with

- **[arsenal-planning](https://github.com/Outer-Heaven-Technologies/arsenal-planning)** — produces upstream planning artifacts. Optional but recommended.

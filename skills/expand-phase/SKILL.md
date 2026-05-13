---
name: expand-phase
description: Expands a `TASKS.md` phase's placeholder tasks into a concrete, tagged task list grouped into `### Design tasks` and `### Feature tasks` subsections. Reads the phase header, relevant `docs/UX.md` sections, feature specs (single or split mode), and a high-level skim of `ARCHITECTURE.md` / `CONVENTIONS.md` (plus `docs/mockups/` on iOS) — then generates atomic tasks tagged with `domain:` / `research:` (web + iOS) and `finishers:` / `snapshot:` (iOS, design-domain tasks only), and writes them back to `TASKS.md`. Invoked by `features-web` / `design-web` (and iOS counterparts) after Step 1 preflight; can be invoked directly to re-expand a phase whose feature spec or UX changed. Does NOT auto-fire on "begin work on phase N" phrasing — that's the orchestrators' territory. Use slash command for surgical re-expansion.
---

# Expand Phase

Take a phase whose `TASKS.md` block has placeholder tasks (e.g., `- [ ] Tasks not yet generated...`) and rewrite it as a concrete, tagged task list ready for brief generation and per-task execution.

This skill is normally invoked by `features-web` or `features-ios` after Step 1 preflight (workspace setup, scope resolution). It can also be invoked directly to re-expand a phase — useful when the upstream feature spec or `UX.md` changed after the original expansion and the task list is now stale.

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

## When this skill expands vs. no-ops

| State of phase in `TASKS.md` | Behavior |
|---|---|
| Placeholder tasks (`- [ ] Tasks not yet generated...`) | Expand: read artifacts, generate tasks, tag, write back. |
| Concrete tasks already in place (e.g., Phase 0 typically does, or a previously expanded phase being resumed) | No-op pass-through. Report "phase N already has concrete tasks; nothing to expand." |
| User explicitly requests re-expansion (`--force` flag) | Replace the existing task block. Warn the user that any unchecked tasks will be regenerated and existing checkmarks will be re-evaluated against the new task list. |

The orchestrators won't invoke this skill on a phase that doesn't need expansion — but direct invocation must handle these cases gracefully.

## Inputs

Caller passes:

- `--phase <N>` — phase number (required)
- `--surface web|ios` — surface flag, drives the tag set and auto-flag rules (required)
- `--scope <shape>` — sub-scope filter (optional, default `full`):
  - `--scope full` — expand the entire phase as-defined in `TASKS.md` (default). Maps to natural-language scoping like *"expand phase 1"* or *"expand the whole phase"*.
  - `--scope feature=<slug>` — generate tasks only for the named feature within the phase. `<slug>` matches a `planning/features/<slug>.md` filename (split mode) or a feature name from `planning/FEATURES.md` (single mode). Maps to natural-language scoping like *"expand phase 1, just the <feature-name> feature"*.
  - `--scope user-story=<id>` — generate tasks only for the named user story within a feature. `<id>` is a story identifier from the feature spec (e.g., a `## User story` section heading or an explicit story slug). The orchestrator's Step 1 resolves the natural-language story name to the feature it belongs to plus the story id; expand-phase reads only the relevant story within the feature spec. Maps to natural-language scoping like *"just the <story-name> story within <feature>"*.
  - `--scope ux-section=<name>` — generate tasks only for the named `UX.md` section/page within the phase. `<name>` is the section heading (or a slug derived from it). Maps to natural-language scoping like *"build the <section-name>"*.
- `--force` — optional. When set, replace existing concrete tasks instead of no-op.

Files read from disk (read-only):

- `docs/TASKS.md` — the phase header and metadata (which UX pages + features it covers, current task state)
- `docs/UX.md` — the relevant sections for this phase / scope (page sections, conversion or engagement model, components needed)
- `planning/FEATURES.md` (single mode) **or** specific `planning/features/<slug>.md` files (split mode, by exact path)
- `docs/ARCHITECTURE.md`, `docs/CONVENTIONS.md` — high-level skim for cross-cutting constraints
- (iOS only) `docs/mockups/` — surface awareness during tagging (which tasks touch a screen with a mockup). Mockup contents themselves are read in depth by `generate-design-briefs` later, not here.

Files written:

- `docs/TASKS.md` — phase block updated in place: the placeholder line replaced with the concrete tagged task list.

## Workflow

### Step 1: Read scope-relevant artifacts

Read only what this phase × scope needs — don't read everything:

- The phase header in `TASKS.md` (which UX pages and features it covers, the goal sentence)
- Relevant sections of `docs/UX.md`:
  - `--scope full`: every section/page named in the phase header
  - `--scope feature=<slug>`: only sections that the named feature touches
  - `--scope user-story=<id>`: only the section(s) the named story touches (typically a subset of the feature's sections)
  - `--scope ux-section=<name>`: only that one section
- **Feature specs** — depends on mode and scope:
  - **Single mode** (`planning/FEATURES.md` exists): read the file, extract only the features this phase × scope covers, drop the rest from context. For `--scope user-story=<id>`, extract only the matching story within the matching feature.
  - **Split mode** (`planning/features/` exists): read `planning/features/README.md` first to confirm which feature files this phase × scope needs, then read **only those files** (`planning/features/<slug>.md` per feature, by exact path). For `--scope user-story=<id>`, read only the parent feature file and within it focus on the matching story section. Never glob the directory.
- High-level skim of `ARCHITECTURE.md` and `CONVENTIONS.md` for cross-cutting constraints (data flow, schema, style rules)
- (iOS only) Skim `docs/mockups/` to know which surfaces in scope have a mockup file present (used in tagging, not read in depth)

### Step 2: Generate concrete tasks

For each feature × page combination within the scope, generate atomic tasks. Default granularity:

- One task per acceptance criterion in the feature spec, OR
- One task per `UX.md` section/component if no feature specs exist, OR
- Use the phase header's stated goals if neither upstream artifact exists for this scope

Granularity heuristic: a task should fit within the per-task context-brief budget (≤3,000 tokens / ~12,000 characters) that `generate-design-briefs` / `generate-feature-briefs` will produce downstream. If a single AC would clearly exceed that budget, split it into 2+ tasks.

### Step 3: Tag each task

Every task is tagged inline. Tag set depends on `--surface`.

#### Tag set — `--surface web`

```
- domain:   design | feature   # which pipeline executes this task
- research: yes/no              # platform API, library drift, unfamiliar pattern
```

`domain:` is the primary axis. It routes the task to either the design pipeline (`design-web` → `close-design-phase-web`) or the feature pipeline (`features-web` → `close-feature-phase-web`). The two run in strict sequence per phase: all design tasks complete (including phase close) before any feature task starts. The tag also determines task placement under `### Design tasks` vs `### Feature tasks` in the output.

**`domain: design`** auto-flags for:
- Any task that creates or modifies a route / page / component a user sees
- Any task that introduces new UI states (empty, loading, error, success, hover/focus/active)
- Any task that changes layout, typography, or color usage on an existing surface
- Component scaffolding with hardcoded / placeholder data

**`domain: feature`** auto-flags for:
- Pure API routes, server-side handlers, schema migrations, auth middleware, scheduled jobs, scripts, infra config
- Wiring an existing component to real data (queries, mutations, state management, server actions)
- Extensions to an existing component required by the data layer — new props, variants, or states the data needs. Visual changes are not allowed in feature tasks; those are design-domain.

If a task could plausibly belong to either domain (e.g. "build a search input that talks to the backend"), split it: one design task that builds the input with hardcoded results, one feature task that wires it to the search API. That split is the whole point — design produces visual surfaces with hardcoded data, feature wires them to real data.

**`research: yes`** auto-flags for (orthogonal to domain):
- Platform API integrations (Stripe webhooks, OAuth flows, third-party SDKs)
- Recently changed library APIs (Next.js App Router migrations, React 19 idioms, etc.)
- Unfamiliar patterns the project hasn't encountered before
- Anything explicitly flagged in a feature spec's "Important" section as needing care

For straightforward backend or wiring tasks, set `research: no`.

#### Tag set — `--surface ios`

```
- domain:    design | feature   # which pipeline executes this task
- research:  yes/no              # Apple platform API, library drift, unfamiliar pattern
- finishers: [list]              # impeccable skills to apply post-impl — design-domain tasks ONLY
- snapshot:  yes/no              # task touches a theme-module primitive — design-domain tasks ONLY
```

`domain:` is the primary axis. `finishers:` and `snapshot:` apply **only** to `domain: design` tasks — they describe visual-layer polish and theme-regression discipline. Feature-domain tasks may carry `research:` but never `finishers:` or `snapshot:`. The two pipelines run in strict sequence per phase: design tasks (executed via `design-ios` → `close-design-phase-ios`) complete before any feature task starts (`features-ios` → `close-feature-phase-ios`).

**`domain: design`** auto-flags for:
- Any task that creates or modifies a user-facing SwiftUI view in the project's feature folder
- Any task that introduces new UI states (empty, loading, error, success)
- Any task touching theme-module primitives or token values
- Component scaffolding with hardcoded / placeholder data

**`domain: feature`** auto-flags for:
- Pure data-layer (`@Model` types, services, engines, schema migrations)
- Wiring an existing view to real data (`@Query`, `@Environment`, services)
- View extensions required by the data layer (new props or state — but no visual treatment changes)

**`research: yes`** auto-flags for (orthogonal to domain):
- Family Controls, CloudKit schema, StoreKit 2, WidgetKit, NotificationCenter, scenePhase semantics, SwiftData migration, Apple Push, HealthKit — anywhere convention drift between iOS versions is likely.

Set `research: no` for pattern-extension tasks mirroring an existing screen's structure.

**`finishers:` auto-flags** (design-domain only):

- `[harden]` if the feature spec § Important mentions force-quit resistance, scenePhase, write-then-animate ordering, or error-path correctness.
- `[animate]` if the design brief proposes motion (cross-fade, paced animation, transition).
- `[clarify]` if the task introduces net-new copy strings (not locked verbatim in the feature spec).
- `[typeset]` if the task touches typography hierarchy on a high-traffic screen.
- `[arrange]` only when layout drift is suspected.

**`snapshot: yes`** (design-domain only) if the task touches a theme-module primitive (per `DESIGN_SYSTEM.md`'s declared theme path) or theme token values. The design implementer adds a snapshot test; verification runs later in the phase at `close-feature-phase-ios` (snapshot tests are regression discipline, not design judgement).

### Step 4: Write the expanded tasks back to `TASKS.md`

Replace the placeholder line in the phase with the concrete task list. Use the surface-appropriate output format.

#### Output format — `--surface web`

Tasks are written into two subsections per phase: **Design tasks** first, **Feature tasks** second. Design tasks (including `close-design-phase-web`) complete before any feature task starts.

```markdown
## Phase 1: Core Loop (Landing + Onboarding)

**Status:** In progress
**UX pages covered:** Landing, Onboarding
**FEATURES capabilities covered:** Email Capture, Stripe Payments
**Goal:** Prove the activation event works end-to-end.

### Design tasks
- [ ] Build Landing → Hero section component *(see UX.md → Landing § 1; domain: design, research: no)*
- [ ] Build Landing → Email capture form component *(see `planning/features/email-capture.md` § AC#1; domain: design, research: no)*

### Feature tasks
- [ ] Wire Stripe checkout webhook handler *(see `planning/features/payments.md` § Webhook; domain: feature, research: yes — Stripe API)*
- [ ] Add Postgres migration for Subscription table *(see `planning/features/payments.md` § Data; domain: feature, research: no)*
- [ ] Wire email capture form to Kit API *(see `planning/features/email-capture.md` § Data; domain: feature, research: no)*
```

Use the actual path that exists for the project — split-mode references use specific file paths, single-mode references use `FEATURES.md → <feature>` syntax. Don't include both in a real TASKS.md, just pick what matches the project structure.

If a phase has no design tasks (pure backend / data / infra phase), still emit the `### Design tasks` subsection with a single line: `_None — pure feature-domain phase._`. Same shape for phases with no feature tasks (pure component-library phase). Subsection headers always appear so downstream orchestrators can locate their domain.

#### Output format — `--surface ios`

Tasks are written into two subsections per phase: **Design tasks** first, **Feature tasks** second. Design tasks (including `close-design-phase-ios`) complete before any feature task starts.

```markdown
## Phase N: [Name]

**Status:** In progress
**UX pages covered:** [...]
**FEATURES capabilities covered:** [...]
**Goal:** [one sentence]

### Design tasks
- [ ] **T1** — Build NoteCard view *(see `planning/features/notes.md`; domain: design, research: no, finishers: [clarify], snapshot: no)*
- [ ] **T2** — Build NoteSyncIndicator with motion *(see `planning/features/notes.md`; domain: design, research: yes — scenePhase, finishers: [animate, harden], snapshot: yes)*

### Feature tasks
- [ ] **T3** — Wire Note @Query into NoteCard *(see `planning/features/notes.md`; domain: feature, research: no)*
- [ ] **T4** — Add weekly digest aggregation service *(see `planning/features/notes.md`; domain: feature, research: no)*
```

If a phase has no design tasks (pure data / service phase), still emit the `### Design tasks` subsection with a single line: `_None — pure feature-domain phase._`. Same shape for phases with no feature tasks. Subsection headers always appear so downstream orchestrators can locate their domain.

### Step 5: Confirm with user, then drop context

Confirm the expanded task list with the user before reporting back to the caller. **At this point, drop feature specs and `UX.md` from your own context** — once the task list is written and confirmed, the briefs (`generate-design-briefs` and `generate-feature-briefs` write them next, per domain) and `TASKS.md` are the working artifacts. In split mode, this means closing the specific feature files that were opened during this expansion; in single mode, releasing `FEATURES.md`. The `planning/features/README.md` index can stay accessible (it's tiny) for cross-phase reference.

### Step 6: Return to caller

Report to the invoking skill (or to the user if invoked directly):

- **Status:** DONE | NO_OP (already-concrete phase, no `--force`) | NEEDS_CONTEXT
- **Phase:** N
- **Scope:** full | feature=<slug> | user-story=<id> | ux-section=<name>
- **Surface:** web | ios
- **Tasks generated:** count
- **Tag summary:**
  - domain: design — `<count>` tasks
  - domain: feature — `<count>` tasks
  - research: yes — `<count>` tasks (across both domains)
  - (iOS, design-domain only) snapshot: yes — `<count>` tasks
  - (iOS, design-domain only) finishers: aggregated list with counts (e.g., `harden: 2, animate: 1, clarify: 3`)
- **TASKS.md updated:** path to the file, line range of the rewritten phase block

The caller's next step is to generate briefs for every task; this skill does not write briefs itself.

## Anti-patterns — never do these

- **Don't bypass the deny rules.** If `.claude/settings.json` denies `Read(planning/features/*)`, read only the specific feature files for this phase × scope by exact path — the legitimate way to opt in. Listing the whole directory or globbing it defeats the isolation that split mode is designed for.
- **Don't ship the wrong tag set for the surface or domain.** Web tasks get `domain:` + `research:`. iOS tasks get those plus `finishers:` + `snapshot:` — but only on `domain: design` tasks. Cross-pollination (`finishers:` on a feature-domain task, `snapshot:` on a web task) is a defect.
- **Don't emit tasks outside the subsection structure.** Every task line lives under either `### Design tasks` or `### Feature tasks` for its phase. Mixed-order task lines under the bare phase header are a defect — downstream orchestrators look tasks up by subsection.
- **Don't mix domains in a single task.** A task that says "build the component AND wire it to the API" must be split into two tasks. Design produces visual surfaces with hardcoded data; feature wires them to real data. Mixing them collapses the pipeline separation.
- **Don't re-expand a phase that already has concrete tasks unless `--force` is set.** No-op pass-through with a clear report. Re-expanding silently would clobber checkmark state from in-progress work.

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:features-web` | Invokes this skill at Step 2 with `--surface web` after its own Step 1 preflight. Hand-off point: phase number + resolved scope arg. |
| `/arsenal-build:features-ios` | Same, with `--surface ios`. |
| `/arsenal-build:generate-design-briefs` | Runs *after* this skill, during the design half. Reads the `### Design tasks` subsection from `TASKS.md` and writes per-task context briefs + design briefs (`.tasks/phase-N/task-N-{context,design}.md`) for `domain: design` tasks. |
| `/arsenal-build:generate-feature-briefs` | Runs *after* this skill, during the feature half (after the design pipeline closes). Reads the `### Feature tasks` subsection and writes per-task context briefs (`.tasks/phase-N/task-N-context.md`) with an `## Available components` manifest sourced from design-pipeline commits. |
| `/arsenal-planning:features` | Upstream — produces the feature specs this skill reads. Updates to feature specs after a phase has been expanded may warrant a `--force` re-expansion. |
| `/arsenal-planning:ux-web` / `/arsenal-planning:ux-app` / `/arsenal-planning:ux-ios` | Upstream — produce the `UX.md` this skill reads. Updates to `UX.md` after a phase has been expanded may warrant a `--force` re-expansion. |
| `/arsenal-build:anchor-files` | Upstream — produces the `TASKS.md` placeholder phase scaffold this skill expands. |

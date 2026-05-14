---
name: expand-phase
description: Expands a `TASKS.md` phase's placeholder tasks into a concrete, tagged task list grouped into `### Design tasks` and `### Feature tasks` subsections. Reads the phase header, relevant `.arsenal/design/UX.md` sections, feature specs (single or split mode), and a high-level skim of `ARCHITECTURE.md` / `CONVENTIONS.md` — then generates atomic tasks tagged with `domain:` / `research:` and writes them back to `TASKS.md`. Invoked by `features` / `design` after Step 1 preflight; can be invoked directly to re-expand a phase whose feature spec or UX changed. Does NOT auto-fire on "begin work on phase N" phrasing — that's the orchestrators' territory. Use slash command for surgical re-expansion.
---

# Expand Phase

Take a phase whose `TASKS.md` block has placeholder tasks (e.g., `- [ ] Tasks not yet generated...`) and rewrite it as a concrete, tagged task list ready for brief generation and per-task execution.

This skill is normally invoked by `features` after Step 1 preflight (workspace setup, scope resolution). It can also be invoked directly to re-expand a phase — useful when the upstream feature spec or `UX.md` changed after the original expansion and the task list is now stale.

## Paths

All arsenal artifacts live under `.arsenal/` at the project root.

| What | Path | Notes |
|---|---|---|
| Strategy archive (denied during build) | `.arsenal/strategy/` | MVP_SPEC.md, mockup-briefs/, GTM_STRATEGY.md, REVENUE_MODEL.md, research/{MARKET_RESEARCH,RESEARCH_PLAN}.md |
| Feature specs | `.arsenal/FEATURES.md` (single-mode) or `.arsenal/features/<slug>.md` (split-mode) | Gated per phase via `.claude/settings.json` |
| Project anchor docs | `.arsenal/{ARCHITECTURE,CONVENTIONS,TASKS}.md` | Always readable during build |
| Design reference set | `.arsenal/design/{UX,DESIGN,DESIGN_SYSTEM}.md` + `.arsenal/design/mockups/` | Always readable during build |
| Per-task briefs + ephemera | `.arsenal/tasks/phase-N/`, `.arsenal/tasks/parallel/`, `.arsenal/tasks/archive/` | Gitignored; phase-N gated per active phase |

**Configuration:** `.arsenal/config.yaml` may override the root location, but defaults work for nearly all projects. File names are not configurable.

**Gating:** `expand-phase` writes baseline denies and per-phase allow rules to `.claude/settings.json`. `close-feature-phase` reverts at phase end. Strategy stays fully denied throughout build.

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
- `--scope <shape>` — sub-scope filter (optional, default `full`):
  - `--scope full` — expand the entire phase as-defined in `TASKS.md` (default). Maps to natural-language scoping like *"expand phase 1"* or *"expand the whole phase"*.
  - `--scope feature=<slug>` — generate tasks only for the named feature within the phase. `<slug>` matches a `.arsenal/features/<slug>.md` filename (split mode) or a feature name from `.arsenal/FEATURES.md` (single mode). Maps to natural-language scoping like *"expand phase 1, just the <feature-name> feature"*.
  - `--scope user-story=<id>` — generate tasks only for the named user story within a feature. `<id>` is a story identifier from the feature spec (e.g., a `## User story` section heading or an explicit story slug). The orchestrator's Step 1 resolves the natural-language story name to the feature it belongs to plus the story id; expand-phase reads only the relevant story within the feature spec. Maps to natural-language scoping like *"just the <story-name> story within <feature>"*.
  - `--scope ux-section=<name>` — generate tasks only for the named `UX.md` section/page within the phase. `<name>` is the section heading (or a slug derived from it). Maps to natural-language scoping like *"build the <section-name>"*.
- `--force` — optional. When set, replace existing concrete tasks instead of no-op.

Files read from disk (read-only):

- `.arsenal/TASKS.md` — the phase header and metadata (which UX pages + features it covers, current task state)
- `.arsenal/design/UX.md` — the relevant sections for this phase / scope (page sections, conversion or engagement model, components needed)
- `.arsenal/FEATURES.md` (single mode) **or** specific `.arsenal/features/<slug>.md` files (split mode, by exact path)
- `.arsenal/ARCHITECTURE.md`, `.arsenal/CONVENTIONS.md` — high-level skim for cross-cutting constraints

Files written:

- `.arsenal/TASKS.md` — phase block updated in place: the placeholder line replaced with the concrete tagged task list.

## Workflow

### Step 0: Configure phase-scoped permissions

Before reading artifacts, `expand-phase` writes the gating rules to `.claude/settings.json` so the brief generators and per-task implementers (downstream) see only the in-scope features and the current phase's task folder.

**Read `.arsenal/TASKS.md` first** to identify phase N's in-scope features (look at the `**FEATURES capabilities covered:**` line or equivalent in the phase header). Then resolve their paths:

- **Split mode** (`.arsenal/features/` directory exists): `.arsenal/features/<slug>.md` per named feature
- **Single mode** (`.arsenal/FEATURES.md` file exists): `.arsenal/FEATURES.md` (one file — no per-feature gating possible)

**Detect existing settings.json state:**

| Condition | Meaning |
|---|---|
| `.claude/settings.json` doesn't exist OR has no `permissions.deny` containing `Read(.arsenal/strategy/**)` | First execution phase — apply baseline lockdown (the project is transitioning from planning to build) |
| Baseline already present | Subsequent phase — refresh the per-phase entries only, keep baseline intact |

**Apply the permission set.** Read `.claude/settings.json` if it exists (preserving any non-arsenal entries the user added — arsenal owns only entries whose path begins with `.arsenal/`). Compute and write the deny list:

1. **Always present (baseline):**
   - `Read(.arsenal/strategy/**)` — strategy archive denied during build
2. **Out-of-scope features** (split mode only): for every `.arsenal/features/*.md` file that exists in the project and is NOT in phase N's in-scope set, add a per-file deny: `Read(.arsenal/features/<other-slug>.md)`. Exclude `.arsenal/features/README.md` (the index — always readable). In single mode, skip this step (FEATURES.md is implicitly allowed).
3. **Other phases:** for every `.arsenal/tasks/phase-*/` directory that exists and is NOT phase N, add a per-phase deny: `Read(.arsenal/tasks/phase-X/**)`.

**The current phase's task folder (`.arsenal/tasks/phase-N/`) and all in-scope features are intentionally NOT denied — they're readable.**

**Example for phase 2 in split mode, with in-scope features `auth-flow` and `dashboard`, and existing phase-1 and phase-3 folders:**

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

**Tell the user.** On first-ever lockdown, surface: *"First execution phase — locking down strategy archive and gating feature/task access. Edits to `.arsenal/strategy/` won't be readable by build subagents. Remove `.arsenal/`-prefixed entries from `.claude/settings.json` to revisit planning."* On subsequent phases, no announcement — just proceed.

**Idempotency on re-expansion (`--force`):** the user may have changed the phase's feature scope or new phases may have appeared. Recompute the deny list from scratch and overwrite. This is the canonical refresh moment.

**Gitignore upkeep:** ensure `.gitignore` at the project root excludes `.arsenal/tasks/` (per-task briefs are ephemeral, not version-controlled). Append the line if missing; create the file if absent.

### Step 1: Read scope-relevant artifacts

Read only what this phase × scope needs — don't read everything:

- The phase header in `TASKS.md` (which UX pages and features it covers, the goal sentence)
- Relevant sections of `.arsenal/design/UX.md`:
  - `--scope full`: every section/page named in the phase header
  - `--scope feature=<slug>`: only sections that the named feature touches
  - `--scope user-story=<id>`: only the section(s) the named story touches (typically a subset of the feature's sections)
  - `--scope ux-section=<name>`: only that one section
- **Feature specs** — depends on mode and scope:
  - **Single mode** (`.arsenal/FEATURES.md` exists): read the file, extract only the features this phase × scope covers, drop the rest from context. For `--scope user-story=<id>`, extract only the matching story within the matching feature.
  - **Split mode** (`.arsenal/features/` exists): read `.arsenal/features/README.md` first to confirm which feature files this phase × scope needs, then read **only those files** (`.arsenal/features/<slug>.md` per feature, by exact path). For `--scope user-story=<id>`, read only the parent feature file and within it focus on the matching story section. Never glob the directory.
- High-level skim of `ARCHITECTURE.md` and `CONVENTIONS.md` for cross-cutting constraints (data flow, schema, style rules)

### Step 2: Generate concrete tasks

For each feature × page combination within the scope, generate atomic tasks. Default granularity:

- One task per acceptance criterion in the feature spec, OR
- One task per `UX.md` section/component if no feature specs exist, OR
- Use the phase header's stated goals if neither upstream artifact exists for this scope

Granularity heuristic: a task should fit within the per-task context-brief budget (≤3,000 tokens / ~12,000 characters) that `generate-design-briefs` / `generate-feature-briefs` will produce downstream. If a single AC would clearly exceed that budget, split it into 2+ tasks.

### Step 3: Tag each task

Every task is tagged inline.

```
- domain:   design | feature   # which pipeline executes this task
- research: yes/no              # platform API, library drift, unfamiliar pattern
```

`domain:` is the primary axis. It routes the task to either the design pipeline (`design` → `close-design-phase`) or the feature pipeline (`features` → `close-feature-phase`). The two run in strict sequence per phase: all design tasks complete (including phase close) before any feature task starts. The tag also determines task placement under `### Design tasks` vs `### Feature tasks` in the output.

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

### Step 4: Write the expanded tasks back to `TASKS.md`

Replace the placeholder line in the phase with the concrete task list.

Tasks are written into two subsections per phase: **Design tasks** first, **Feature tasks** second. Design tasks (including `close-design-phase`) complete before any feature task starts.

```markdown
## Phase 1: Core Loop (Landing + Onboarding)

**Status:** In progress
**UX pages covered:** Landing, Onboarding
**FEATURES capabilities covered:** Email Capture, Stripe Payments
**Goal:** Prove the activation event works end-to-end.

### Design tasks
- [ ] Build Landing → Hero section component *(see UX.md → Landing § 1; domain: design, research: no)*
- [ ] Build Landing → Email capture form component *(see `.arsenal/features/email-capture.md` § AC#1; domain: design, research: no)*

### Feature tasks
- [ ] Wire Stripe checkout webhook handler *(see `.arsenal/features/payments.md` § Webhook; domain: feature, research: yes — Stripe API)*
- [ ] Add Postgres migration for Subscription table *(see `.arsenal/features/payments.md` § Data; domain: feature, research: no)*
- [ ] Wire email capture form to Kit API *(see `.arsenal/features/email-capture.md` § Data; domain: feature, research: no)*
```

Use the actual path that exists for the project — split-mode references use specific file paths, single-mode references use `FEATURES.md → <feature>` syntax. Don't include both in a real TASKS.md, just pick what matches the project structure.

If a phase has no design tasks (pure backend / data / infra phase), still emit the `### Design tasks` subsection with a single line: `_None — pure feature-domain phase._`. Same shape for phases with no feature tasks (pure component-library phase). Subsection headers always appear so downstream orchestrators can locate their domain.

### Step 5: Confirm with user, then drop context

Confirm the expanded task list with the user before reporting back to the caller. **At this point, drop feature specs and `UX.md` from your own context** — once the task list is written and confirmed, the briefs (`generate-design-briefs` and `generate-feature-briefs` write them next, per domain) and `TASKS.md` are the working artifacts. In split mode, this means closing the specific feature files that were opened during this expansion; in single mode, releasing `FEATURES.md`. The `.arsenal/features/README.md` index can stay accessible (it's tiny) for cross-phase reference.

### Step 6: Return to caller

Report to the invoking skill (or to the user if invoked directly):

- **Status:** DONE | NO_OP (already-concrete phase, no `--force`) | NEEDS_CONTEXT
- **Phase:** N
- **Scope:** full | feature=<slug> | user-story=<id> | ux-section=<name>
- **Tasks generated:** count
- **Tag summary:**
  - domain: design — `<count>` tasks
  - domain: feature — `<count>` tasks
  - research: yes — `<count>` tasks (across both domains)
- **TASKS.md updated:** path to the file, line range of the rewritten phase block

The caller's next step is to generate briefs for every task; this skill does not write briefs itself.

## Anti-patterns — never do these

- **Don't bypass the deny rules.** If `.claude/settings.json` denies `Read(.arsenal/features/*)`, read only the specific feature files for this phase × scope by exact path — the legitimate way to opt in. Listing the whole directory or globbing it defeats the isolation that split mode is designed for.
- **Don't ship the wrong tag set for the domain.** Tasks get `domain:` + `research:`. Stray tags (`finishers:`, `snapshot:`) are a defect.
- **Don't emit tasks outside the subsection structure.** Every task line lives under either `### Design tasks` or `### Feature tasks` for its phase. Mixed-order task lines under the bare phase header are a defect — downstream orchestrators look tasks up by subsection.
- **Don't mix domains in a single task.** A task that says "build the component AND wire it to the API" must be split into two tasks. Design produces visual surfaces with hardcoded data; feature wires them to real data. Mixing them collapses the pipeline separation.
- **Don't re-expand a phase that already has concrete tasks unless `--force` is set.** No-op pass-through with a clear report. Re-expanding silently would clobber checkmark state from in-progress work.

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:features` | Invokes this skill at Step 2 after its own Step 1 preflight. Hand-off point: phase number + resolved scope arg. |
| `/arsenal-build:generate-design-briefs` | Runs *after* this skill, during the design half. Reads the `### Design tasks` subsection from `TASKS.md` and writes per-task context briefs + design briefs (`.arsenal/tasks/phase-N/task-N-{context,design}.md`) for `domain: design` tasks. |
| `/arsenal-build:generate-feature-briefs` | Runs *after* this skill, during the feature half (after the design pipeline closes). Reads the `### Feature tasks` subsection and writes per-task context briefs (`.arsenal/tasks/phase-N/task-N-context.md`) with an `## Available components` manifest sourced from design-pipeline commits. |
| `/arsenal-planning:features` | Upstream — produces the feature specs this skill reads. Updates to feature specs after a phase has been expanded may warrant a `--force` re-expansion. |
| `/arsenal-planning:ux-web` / `/arsenal-planning:ux-app` / `/arsenal-planning:ux-ios` | Upstream — produce the `UX.md` this skill reads. Updates to `UX.md` after a phase has been expanded may warrant a `--force` re-expansion. |
| `/arsenal-build:setup` | Upstream — produces the `TASKS.md` placeholder phase scaffold this skill expands. |

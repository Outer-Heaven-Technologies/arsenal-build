---
name: close-feature-phase-ios
description: Closes a completed development phase for a native iOS project (SwiftUI + Xcode) by running seven gates in order — `RunAllTests` integration check → Periphery dead-code scan → snapshot test verification → docs update (if scope drifted) → CodeRabbit review (hard gate, runs across the full phase including design-pipeline commits) → trim TASKS.md and archive `.tasks/phase-N/` (deleting `*.png` debug artifacts first) → push branch and open PR. This is the phase's terminus — opens the single PR per phase covering both design and feature commits. Does NOT call impeccable (visual audits live at `close-design-phase-ios`, optionally and earlier in the phase). Invoked by `features-ios` at the end of the feature half, or directly to wrap up a phase whose per-task work already landed. Use when the user wants to "wrap up", "close out", "finish", or "ship" an iOS phase.
---

# Close Feature Phase — iOS

Run the phase-final gate sequence for a native iOS project. **Seven gates in order**, none optional. Cleanup (Gate 6) runs **before** push + PR (Gate 7) — once the PR URL is generated, the wrap is over.

This skill is normally invoked by `features-ios` after every feature-domain task in the phase is committed and `[x]`-marked. It can also be invoked directly to re-run the wrap on a branch whose per-task commits already landed.

> **Don't pattern-match on "push" as the punchline.** Read every gate. **Skipping any of these seven gates is a defect.**

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

## Where this fits in the phase pipeline

```
design-ios   →   close-design-phase-ios   →   features-ios   →   close-feature-phase-ios  ← YOU ARE HERE
   (design tasks)         (design-pipeline wrap)        (feature tasks)            (phase-final wrap + PR)
```

This is the phase's terminus. Opens the single PR per phase covering design + feature commits.

## Why no visual phase audit gate here

Visual audit (impeccable `audit` + `polish`) lives at `close-design-phase-ios` and is OPTIONAL there (user-opted). The feature half doesn't introduce visual changes (the component-boundary rule prohibits them), so a phase-wrap visual audit at this stage catches nothing new.

## Why Periphery and snapshot verification STAY here

Both are regression-discipline gates, not design judgement:

- **Periphery** scans the whole project for dead code introduced anywhere across the phase (design + feature). Catches orphaned helpers, unused imports, dead `@State` declarations.
- **Snapshot test verification** confirms no theme primitive drifted from its frozen baseline. If a design-pipeline task touched a primitive and updated the baseline (with user approval at the per-task level), this gate confirms the baseline holds; if a feature-pipeline task accidentally broke a primitive, this gate catches it.

Both run once per phase, at the phase terminus.

## Prerequisites

| File / state | Used for |
|---|---|
| `docs/TASKS.md` | Phase block with `### Design tasks` (all `[x]`), `### Feature tasks` (all `[x]`) (Gate 6 extracts the full phase block) |
| `.tasks/phase-N/` | Working dir — context briefs, design briefs, research files, `*.png` screenshots (Gate 6 archives + deletes PNGs) |
| Phase branch checked out (`phase-N/short-description`) | Every gate runs against this branch's HEAD |
| `<Project>.xcodeproj` (or app-target `Package.swift`) | Gate 2 (Periphery) scans this |
| `.periphery.yml` at project root | Gate 2 reads if exists; first-run bootstrap prompts the user to generate one |
| `mcp__xcode__*` MCP tools | Gates 1, 3 (tests, build) |
| `coderabbit` plugin (Gate 5 hard gate) | Prompts to install or waive if absent |
| `swift-snapshot-testing` linked to test target | Gate 3 snapshot verification |

## Inputs (for direct invocation)

When invoked directly, the caller passes the phase number. All other state is read from disk:

- `docs/TASKS.md` (Gate 6 extracts the phase block)
- `.tasks/phase-N/` contents (Gate 6 archives, after deleting `task-N-{before,after}.png` files)
- Phase branch must already be checked out

## The seven gates

### Gate 1 of 7 — Final integration check

- `mcp__xcode__RunAllTests` — full suite must pass.
- Cross-task scan for integration regressions: `git diff phase-N/start..HEAD --stat`.

Do not advance to Gate 2 with a red test suite.

**If the test suite fails (J3 contract — locked):**

This skill does NOT dispatch a debug subagent unilaterally. The contract:

1. **Report the failure** with the build log captured via `mcp__xcode__GetBuildLog` and the failing test names.
2. **Run the cross-task scan** to localize the regression.
3. **Recommend one of three responses:**
   - **Single-task regression** → recommend rolling back that task's commit (`git revert <task-commit>`) and re-running that task via `/arsenal-build:run-task-feature-ios --phase N --task N --force` (or the design counterpart if a design task's diff caused it). Re-enter Gate 1.
   - **Multi-task integration regression** → recommend adding a single fix task to the phase via the per-task pipeline, then re-enter Gate 1.
   - **Flaky test** → recommend explicit waiver with justification logged for the PR body at Gate 7.
4. **Wait for user direction.** Do not mutate code, do not dispatch a fix, do not advance to Gate 2.

### Gate 2 of 7 — Dead-code scan (Periphery)

Run `periphery scan --project <Project>.xcodeproj --schemes <Scheme> --targets <Target>`.

**First-run bootstrap.** If no `.periphery.yml` config exists at the project root, prompt the user to generate one with iOS-aware defaults:

```yaml
project: <Project>.xcodeproj
schemes: [<Scheme>]
targets: [<Target>]
retain_objc_accessible: true
retain_unused_protocol_func_params: true
retain_swift_ui_previews: true
retain_codable_properties: true
external_encodable_protocols: ["Encodable", "Decodable"]
disable_redundant_public_analysis: true
retain_assigned_values: true
```

Commit `.periphery.yml`. Edit when Periphery flags reflection-accessed types as unused.

**Output goes to user triage — never auto-delete.** Periphery has known false positives on:
- SwiftData `@Model` types accessed only via `FetchDescriptor` / key paths
- `@objc` methods called from Storyboards / `#selector` / KVO
- SwiftUI `_PreviewProvider` content
- Property wrappers with framework auto-import semantics
- Methods satisfying protocol conformance with no in-project callers

Present findings grouped by category. For each: ask **delete** (truly dead), **keep** (false positive — add to `.periphery.yml`), or **wire up** (intended but never connected). For **keep** findings, update `.periphery.yml` and commit.

### Gate 3 of 7 — Snapshot test verification

If any theme-module primitive was modified during the phase (typically via a design-pipeline task with `snapshot: yes`), all snapshot tests must pass against current baselines. Baseline updates require explicit user approval — never auto-accept.

If no primitives were modified this phase, this gate is a no-op pass-through.

Run filtered: `mcp__xcode__RunSomeTests` targeting snapshot tests only.

If a snapshot test fails:
- Capture the diff (if PointFree's swift-snapshot-testing wrote one).
- Prompt: "Snapshot test `<Test>` failed. Was the change intentional? If yes, update the baseline (commit as `test(phase-N): update snapshot baseline for [primitive]`). If no, dispatch a fix subagent."
- **Never silently update baselines.**

### Gate 4 of 7 — Update docs + spec-amendment summary

Two sub-steps:

**A. Update docs (if architecture/conventions drifted).** If the phase shipped exactly what was planned at the architecture level, this sub-step is a no-op pass-through. When warranted:
- Update `TASKS.md` progress tracking (final `[x]` flips for any tasks still unchecked).
- Note any decisions in the Key Decisions Log.
- If architecture changed meaningfully, flag what needs updating in `ARCHITECTURE.md`.

Note: feature spec drift is handled by the spec-amendment rule in `run-task-feature-ios`, not here. Tier 1 amendments are committed in-line alongside code with a `Spec amended:` note; Tier 2 changes BLOCK and route back through `features`. By the time this gate runs, FEATURES.md is already current — this sub-step is for architecture-level drift only.

**B. Collate spec-amendment summary (always runs).** Enumerate Tier 1 amendments committed during the phase:

```bash
git log --grep='Spec amended:' --format='%H%n%s%n%b%n---END---' phase-N/start..HEAD
```

If any amendments are found:
- Write a consolidated summary to `.tasks/phase-N/spec-amendments.md`. One entry per amendment:
  ```markdown
  ## <feature-slug>

  **Commit:** <sha>
  **Summary:** <one-line from the `Spec amended:` note>
  **Detail:** <the rest of the note's body, if any>
  ```
- This file is included in the PR body at Gate 7, alongside `design-summary.md` and CodeRabbit dismissal reasoning.

If no amendments found, this sub-step is a no-op.

The amendment summary exists so the PR reviewer (human) gets a single-page view of "what FEATURES.md said it would do vs what was built and why" without having to grep git history themselves.

### Gate 5 of 7 — CodeRabbit review (HARD GATE)

Run `/coderabbit:review --base main` to review **all committed changes in this phase — design-pipeline commits and feature-pipeline commits together**. Single CodeRabbit pass per PR.

Present findings as Critical / Important / Suggestions. User triages fix/dismiss/discuss:

- **fix**: dispatch fix subagents (via `run-task-feature-ios` or `run-task-design-ios` depending on the file domain), commit fixes with `fix(phase-N): [description]`. Run `/coderabbit:review uncommitted` to verify.
- **dismiss**: note the reasoning for the PR body.
- Repeat until clean.

**Hard gate.** If CodeRabbit is not installed:
- Prompt the user: install now, or explicitly waive for this phase?
- If waived, log in the PR body.
- **Do not silently skip.**

Do not advance to Gate 6 until CodeRabbit is clean.

### Gate 6 of 7 — Trim `TASKS.md` + archive `.tasks/phase-N/`

Cleanup runs **before** push + PR.

**Step A — Trim TASKS.md:**

Extract the entire phase block (the `## Phase N` heading, all metadata, `### Design tasks`, `### Feature tasks`, all `[x]` lines) to `.tasks/phase-N/tasks.md` for archival. Replace with a trimmed completion stub:

```markdown
## Phase N: [Name] ✅

**Status:** Completed [YYYY-MM-DD]
**UX pages covered:** [...]
**FEATURES capabilities covered:** [...]
**Goal:** [one sentence]
**Archived tasks:** `.tasks/archive/phase-N/tasks.md`
**PR:** #TBD
```

Use `**PR:** #TBD` as a placeholder — Gate 7 fills in the real number.

If any tasks are still unchecked, do not auto-trim. Leave as a `### Deferred` subsection. Ask the user before discarding any unchecked task.

**Step B — Delete debug screenshots BEFORE archive (iOS-specific):**

Delete `.tasks/phase-N/task-N-before.png` and `.tasks/phase-N/task-N-after.png` files for every task in the phase. These were captured during per-task visual fidelity gates upstream — they're debug artifacts, not historical record.

```bash
rm -f .tasks/phase-N/task-*-before.png .tasks/phase-N/task-*-after.png
```

**Step C — Cleanup `.tasks/phase-N/`:**

Prompt the user:

> "Phase N has passed all review gates and TASKS.md has been trimmed. The working files in `.tasks/phase-N/` (context briefs, design briefs, research files, design-summary.md, archived task list) are no longer needed for active work. Choose:
> - **Archive** (default) — move to `.tasks/archive/phase-N/`
> - **Delete** — remove entirely
> - **Keep** — leave in place"

Default to archive. Run the chosen action.

### Gate 7 of 7 — Push + open PR (terminus)

- `git push -u origin phase-N/short-description`
- `gh pr create` with:
  - Title: `Phase N: [phase description]`
  - Body: auto-generated summary of tasks completed (design + feature), files changed, deferred concerns, CodeRabbit dismissal reasoning (or waiver from Gate 5), Periphery findings summary, snapshot baseline updates (if any), and the archive pointer (`Phase tasks archived to .tasks/archive/phase-N/tasks.md`). Include the design-pipeline summary if `close-design-phase-ios` recorded one (`.tasks/phase-N/design-summary.md` — read at this gate before the archive move, OR pulled from archive if already moved). Include the spec-amendment summary if Gate 4 wrote one (`.tasks/phase-N/spec-amendments.md`) — surface it as a `## Spec amendments` section so PR reviewers see at a glance what FEATURES.md was wrong about and how the build patched it.
- Once the PR number is returned, update the `**PR:** #TBD` placeholder in `TASKS.md`, commit as `chore(phase-N): record PR number in TASKS.md`, push.
- **Don't start the next phase until the PR is merged.**

**Final report to user (after Gate 7):**
- Summary of what was built (design + feature)
- Number of tasks completed (by domain)
- Any concerns flagged during implementation
- Any deferred "Minor" issues from per-task code review
- Build / test results
- Periphery findings summary
- PR URL

## Anti-patterns — never do these

- **Don't skip any of the seven phase-completion gates.**
- **Don't treat push + PR as the terminus while leaving cleanup for later.** Trim and archive (Gate 6) run *before* push + PR (Gate 7).
- **Don't archive `*.png` screenshots.** Delete them at Step B of Gate 6.
- **Don't downgrade CodeRabbit to optional.** Hard gate.
- **Don't dispatch a debug subagent unilaterally on Gate 1 failure.** J3 contract.
- **Don't auto-accept snapshot test baselines.** User reviews and approves explicitly.
- **Don't auto-delete Periphery findings.** User triages every finding.
- **Don't run a visual phase audit here.** Visual audits live at `close-design-phase-ios` (optional, gateable, earlier in the phase).
- **Don't invoke impeccable.** This skill never calls impeccable.

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:features-ios` | Invokes this skill at the end of the feature half. |
| `/arsenal-build:close-design-phase-ios` | Runs **before** this skill (earlier in the phase). May leave `.tasks/phase-N/design-summary.md` for this skill's Gate 7 to include in the PR body. |
| `/arsenal-build:run-task-feature-ios` / `/arsenal-build:run-task-design-ios` | Per-task pipelines that ran during the phase. CodeRabbit fix dispatches at Gate 5 may reuse one of these. |
| `/coderabbit:review` | Gate 5 dispatch. Hard gate. |
| `mcp__xcode__*` MCP | Gates 1, 3 (RunAllTests, RunSomeTests, GetBuildLog). |
| `periphery` CLI | Gate 2. |
| `swift-snapshot-testing` | Gate 3. |
| `impeccable` | **Not invoked from this skill, ever.** Visual audits live at `close-design-phase-ios`. |

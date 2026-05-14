---
name: close-feature-phase
description: Closes a completed development phase for web/frontend projects by running six gates in order — final integration test → Playwright test coverage (if configured) → docs update (if scope drifted) → CodeRabbit review (hard gate, runs across the full phase including design-pipeline commits) → trim TASKS.md and archive `.arsenal/tasks/phase-N/` → push branch and open PR. This is the phase's terminus — it opens the single PR per phase covering both design and feature commits. Does NOT call impeccable (visual audits live at `close-design-phase`, optionally and earlier in the phase). Invoked by `features` at the end of the feature half, or directly to wrap up a phase whose per-task work already landed. Use when the user wants to "wrap up", "close out", "finish", or "ship" a phase.
---

# Close Feature Phase — Web

Run the phase-final gate sequence for a web/frontend project. **Six gates in order**, none optional. Cleanup (Gate 5) runs **before** push + PR (Gate 6) — once the PR URL is generated, the wrap is over.

This skill is normally invoked by `features` after every feature-domain task in the phase is committed and `[x]`-marked. It can also be invoked directly to re-run the wrap on a branch whose task commits already landed (e.g., debugging a wrap that aborted mid-gate).

> **Don't pattern-match on "push" as the punchline.** The push command is the celebratory sentence at the end, not the actual finish line. Read every gate. **Skipping any of these six gates is a defect.**

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

## Where this fits in the phase pipeline

```
design   →   close-design-phase   →   features   →   close-feature-phase  ← YOU ARE HERE
   (design tasks)         (design-pipeline wrap)        (feature tasks)            (phase-final wrap + PR)
```

This is the phase's terminus. It opens the single PR per phase covering both design and feature commits. The design close (`close-design-phase`) ran earlier and did NOT push or PR — that's deliberate, so CodeRabbit runs once across the entire branch at this gate.

## Why no visual phase audit gate here

Visual audit (impeccable, claude-in-chrome, etc.) lives at `close-design-phase` and is OPTIONAL there (gateable, user-opted). The feature half doesn't introduce visual changes (the component-boundary rule prohibits them), so a phase-wrap visual audit at this stage would catch nothing new. If a feature task somehow slipped a visual change past the spec reviewer, that's a defect in the per-task pipeline — surface it directly, don't re-audit.

## Prerequisites

| File / state | Used for |
|---|---|
| `.arsenal/TASKS.md` | Phase block with the `## Phase N` heading, metadata, `### Design tasks` (all `[x]`), `### Feature tasks` (all `[x]`) (Gate 5 extracts the full phase block) |
| `.arsenal/tasks/phase-N/` | Working dir from this phase — context briefs, design briefs, research files (Gate 5 archives) |
| Phase branch checked out (`phase-N/short-description`) | Every gate runs against this branch's HEAD |
| `coderabbit` plugin (Gate 4 hard gate) | Prompts to install or waive if absent (do not silently skip) |

## Inputs (for direct invocation)

When invoked directly (not from `features`), the caller passes the phase number (e.g., `N=1`). All other state is read from disk:

- `.arsenal/TASKS.md` (read at Gate 5 to extract the full phase block — design + feature tasks)
- `.arsenal/tasks/phase-N/` contents (read at Gate 5 to archive)
- Phase branch must already be checked out

## The six gates

### Gate 1 of 6 — Final integration check

- Run the full test suite — everything should pass.
- Do a quick scan of all files changed in this phase (`git diff phase-N/start..HEAD --stat`) for cross-task integration issues.

Do not advance to Gate 2 with a red test suite.

**If the test suite fails (J3 contract — locked):**

This skill does NOT dispatch a debug subagent unilaterally and does NOT pick a recovery path. Until a dedicated `arsenal-build:debug` skill ships, the contract is:

1. **Report the failure** with the full test output captured to the user.
2. **Run the cross-task scan** to localize the regression — which task's commit(s) introduced files that touch the failing area.
3. **Recommend one of three responses** based on the scan:
   - **Single-task regression** — one task's diff clearly introduced the failure → recommend rolling back that task's commit (`git revert <task-commit>`) and re-running just that task via `/arsenal-build:run-task-feature --phase N --task N --force` (or the design counterpart if a design task's diff caused it). Re-enter Gate 1 after.
   - **Multi-task integration regression** — failure spans multiple tasks' diffs or sits in shared infrastructure → recommend adding a single fix task to the phase via the per-task pipeline, then re-enter Gate 1.
   - **Flaky test** — failure can't be reproduced on rerun or is in a known-unstable test → recommend explicit waiver with justification logged for the PR body at Gate 6.
4. **Wait for user direction.** Do not mutate code, do not dispatch a fix, do not advance to Gate 2.

### Gate 2 of 6 — Test coverage for new features (if Playwright agents are available)

If this phase added user-facing functionality and the project has Playwright agents configured (check for `.claude/agents/` planner/generator definitions), suggest: "This phase added new functionality. Want me to run the Playwright planner to generate test coverage?"

If the user agrees, run the planner agent with context about what was built, then the generator to produce test files, and the healer to fix any that don't pass. The resulting test files become part of the project.

If no Playwright agents are present, this gate is a no-op pass-through (not a waive — genuinely nothing to run).

### Gate 3 of 6 — Update docs + spec-amendment summary

Two sub-steps:

**A. Update docs (if architecture/conventions drifted).** If the phase shipped exactly what was planned at the architecture level, this sub-step is a no-op pass-through. When warranted:
- Update TASKS.md progress tracking (final `[x]` flips for any tasks still unchecked).
- Note any decisions made during implementation in the Key Decisions Log.
- If the architecture changed meaningfully, note what needs updating in ARCHITECTURE.md (don't rewrite it during execution — that's a separate step).

Note: feature spec drift is handled by the spec-amendment rule in the per-task pipeline (`run-task-feature`), not here. Tier 1 amendments are committed in-line alongside code with a `Spec amended:` note; Tier 2 changes BLOCK and route back through `features`. By the time this gate runs, FEATURES.md is already current — this sub-step is for architecture-level drift only.

**B. Collate spec-amendment summary (always runs).** Enumerate Tier 1 amendments committed during the phase:

```bash
git log --grep='Spec amended:' --format='%H%n%s%n%b%n---END---' phase-N/start..HEAD
```

If any amendments are found:
- Write a consolidated summary to `.arsenal/tasks/phase-N/spec-amendments.md`. One entry per amendment:
  ```markdown
  ## <feature-slug>

  **Commit:** <sha>
  **Summary:** <one-line from the `Spec amended:` note>
  **Detail:** <the rest of the note's body, if any>
  ```
- This file is included in the PR body at Gate 6, alongside `design-summary.md` and CodeRabbit dismissal reasoning.

If no amendments found, this sub-step is a no-op.

The amendment summary exists so the PR reviewer (human) gets a single-page view of "what FEATURES.md said it would do vs what was built and why" without having to grep git history themselves.

### Gate 4 of 6 — CodeRabbit review (HARD GATE)

Run `/coderabbit:review --base main` (or the project's default branch) to review **all committed changes in this phase — design-pipeline commits and feature-pipeline commits together**. This is the single CodeRabbit pass per PR.

Present the findings to the user, organized as:
- **Critical / bugs** — would break functionality
- **Important / improvements** — real quality issues
- **Suggestions / nitpicks** — stylistic or minor

For each group, ask the user: "Which should we fix, which should we dismiss, and which need discussion?"

The user triages:
- For items marked "fix": dispatch fix subagents on the same branch, commit fixes with `fix(phase-N): [description]`. Use `run-task-feature` or `run-task-design` as the fix-task harness depending on the file domain.
- For items marked "dismiss": note the reasoning. Goes into the PR body in Gate 6.
- After fixes are committed, run `/coderabbit:review uncommitted` to verify no new issues.
- Repeat until clean.

**This is a hard gate.** If the CodeRabbit plugin is not installed:
- Prompt the user: "CodeRabbit plugin not detected. Install it now (recommended), or explicitly waive this gate for this phase?"
- If waived, log the waiver in the PR body so the reviewer on the GitHub side knows local CodeRabbit was skipped.
- **Do not silently skip.**

**Do not advance to Gate 5 until CodeRabbit is clean** — every finding either fixed or explicitly dismissed with reasoning logged for the PR body.

### Gate 5 of 6 — Trim `TASKS.md` + archive `.arsenal/tasks/phase-N/`

Cleanup runs **before** push + PR, deliberately, so the PR body in Gate 6 can reference the trimmed `TASKS.md` and the archive path directly.

**Step A — Trim TASKS.md:**

Extract the entire phase block from TASKS.md (the `## Phase N` heading, all metadata, `### Design tasks` subsection, `### Feature tasks` subsection, all `[x]` task lines beneath them) and write it to `.arsenal/tasks/phase-N/tasks.md` for archival. Then replace the phase block in TASKS.md with a trimmed completion stub:

```markdown
## Phase 1: Core Loop (Landing + Onboarding) ✅

**Status:** Completed [YYYY-MM-DD]
**UX pages covered:** Landing, Onboarding
**FEATURES capabilities covered:** Email Capture, Stripe Payments
**Goal:** Prove the activation event works end-to-end.
**Archived tasks:** `.arsenal/tasks/archive/phase-1/tasks.md`
**PR:** #TBD
```

Use `**PR:** #TBD` as a placeholder — Gate 6 fills in the real number after the PR is opened.

The stub keeps the same shape as the original placeholder scaffold (header + metadata) plus a `Status: Completed` flip, an archive pointer, and the PR number. Future agents reading TASKS.md see "this phase is done, here's where the detail lived" without loading the actual checklist into context.

**If any tasks are still unchecked** (rare — usually only if the user explicitly deferred), do not auto-trim those. Leave them in place as a `### Deferred` subsection under the completion stub, or move them to a future phase. Ask the user before discarding any unchecked task.

**Step B — Cleanup `.arsenal/tasks/phase-N/`:**

Prompt the user:

> "Phase N has passed all review gates and TASKS.md has been trimmed. The working files in `.arsenal/tasks/phase-N/` (context briefs, design briefs, research files, archived task list) are no longer needed for active work. Choose:
> - **Archive** (default) — move to `.arsenal/tasks/archive/phase-N/`
> - **Delete** — remove entirely
> - **Keep** — leave in place"

Default to archive. Run the chosen action:
- Archive: `mkdir -p .arsenal/tasks/archive && mv .arsenal/tasks/phase-N .arsenal/tasks/archive/phase-N`
- Delete: `rm -rf .arsenal/tasks/phase-N`
- Keep: no-op

### Gate 6 of 6 — Push + open PR (terminus)

- Push the phase branch to remote: `git push -u origin phase-N/short-description`
- Create a pull request (via GitHub MCP if available):
  - Title: `Phase N: [phase description]`
  - Body: auto-generated summary of tasks completed (design + feature), files changed, deferred concerns, CodeRabbit dismissal reasoning (or waiver from Gate 4), and the archive pointer (`Phase tasks archived to .arsenal/tasks/archive/phase-N/tasks.md`). Include the design-pipeline summary if `close-design-phase` recorded one (`.arsenal/tasks/phase-N/design-summary.md`). Include the spec-amendment summary if Gate 3 wrote one (`.arsenal/tasks/phase-N/spec-amendments.md`) — surface it as a `## Spec amendments` section so PR reviewers see at a glance what FEATURES.md was wrong about and how the build patched it.
  - Base branch: main (or the user's default)
- If GitHub tools aren't available, tell the user: "Phase branch pushed. Ready to create a PR manually."
- Once the PR number is returned, update the `**PR:** #TBD` placeholder in `TASKS.md` (set during Gate 5), commit as `chore(phase-N): record PR number in TASKS.md`, and push.
- **Final action: revert phase-scoped permissions in `.claude/settings.json`.** Read the current settings, strip out the per-phase entries that `expand-phase` added at Step 0:
  - **Remove** any `Read(.arsenal/features/<slug>.md)` entries (per-feature out-of-scope denies for this phase). In split mode, this returns features to the "all denied by broad rule" state. In single mode, no-op.
  - **Remove** any `Read(.arsenal/tasks/phase-X/**)` entries (per-phase other-phase denies).
  - **Add back the broad baseline denies:** `Read(.arsenal/features/**)` and `Read(.arsenal/tasks/**)`. The strategy deny (`Read(.arsenal/strategy/**)`) stays in place — it's never touched at phase close.
  - **Preserve any non-arsenal entries** in the deny/allow lists — arsenal owns only entries with `.arsenal/` path prefix.
  - Result is the "between phases" baseline: strategy + features + tasks all broadly denied, ready for the next `expand-phase` invocation to narrow.
- **Do not start the next phase until the current phase's PR is merged.**

**This gate is the terminus.** Once the PR URL is generated, the PR-number commit is pushed, and the permission revert has landed, the phase wrap is complete.

**Final report to user (after Gate 6):**
- Summary of what was built (design + feature)
- Number of tasks completed (by domain)
- Any concerns flagged during implementation
- Any deferred "Minor" issues from per-task code review
- Test results
- PR URL

## Anti-patterns — never do these

- **Don't skip any of the six phase-completion gates.** The phase isn't done until Gate 6 is open with a clean diff.
- **Don't treat push + PR as the terminus while leaving cleanup for later.** Trim and archive (Gate 5) run *before* push + PR (Gate 6).
- **Don't downgrade CodeRabbit to optional.** Gate 4 is a hard gate. Silent skip is a defect.
- **Don't dispatch a debug subagent unilaterally on Gate 1 failure.** Follow the J3 contract: report, scan, recommend, wait for user.
- **Don't read archived task lists speculatively.** Completed phases' detail is reference material, not active context.
- **Don't run a visual phase audit here.** Visual audits live at `close-design-phase` (optional, gateable, earlier in the phase). If a visual concern surfaced after the design close, that's a follow-up phase, not this gate.
- **Don't invoke impeccable.** This skill never calls impeccable. The feature pipeline doesn't use it.

## Integration with other skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:features` | Invokes this skill at the end of the feature half. Hand-off point: all feature-task commits landed, branch checked out, `.arsenal/tasks/phase-N/` populated. |
| `/arsenal-build:close-design-phase` | Runs **before** this skill (earlier in the phase). May leave an annotation at `.arsenal/tasks/phase-N/design-summary.md` that this skill picks up for the PR body. |
| `/arsenal-build:run-task-feature` / `/arsenal-build:run-task-design` | Per-task pipelines that ran during the phase. CodeRabbit fix dispatches at Gate 4 may reuse one of these to apply fixes. |
| `/coderabbit:review` | Gate 4 dispatch. Hard gate; prompts install/waive if absent. |
| `impeccable` | **Not invoked from this skill, ever.** Visual audits live at `close-design-phase`. |

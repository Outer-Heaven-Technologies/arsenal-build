---
name: dispatch-parallel
description: Fans out 2–5 genuinely independent investigations to parallel investigator subagents, then reconciles their results into a single SUMMARY.md. For read-only fan-outs only — audits, debug sessions, code analysis, exploratory checks where each investigation targets disjoint files/subsystems and results don't depend on each other. Does NOT call `run-task-{web,ios}` (those handle code changes sequentially); when investigations recommend follow-up code work, the SUMMARY points the user at run-task. Triggers on "investigate these in parallel", "fan out on these issues", "run these checks concurrently", "audit X, Y, and Z separately". Does NOT auto-fire on "build phase N" or "fix this bug" (single-bug → arsenal-build:debug or a regular task). Hard refuse at 6+; refuse-with-suggestion at 1. Idempotent by default; `--force` regenerates already-completed investigations.
---

# Dispatch Parallel

Run 2–5 independent investigations in parallel, then write one reconciled `SUMMARY.md`. Each investigation gets its own fresh-context investigator subagent dispatched from `references/investigator-prompt.md`. Investigations are read-only by design — no code changes, no commits, no `TASKS.md` state.

The skill's value is the **independence gate** (Step 2) and the **reconciliation** (Step 6). The parallel dispatch itself is mechanical; what makes parallel-vs-sequential the right call is the upfront check that the work is genuinely disjoint.

## When this skill fires vs. no-ops

| Investigation count N | Behavior |
|---|---|
| N = 0 | Reject input parsing. Tell the caller no investigations were provided. |
| N = 1 | **Refuse with suggestion.** One investigation isn't parallel — recommend `/arsenal-build:run-task-{web,ios}` for code-change work, or dispatch a single investigator inline outside this skill. dispatch-parallel is for N ≥ 2. |
| 2 ≤ N ≤ 5 | **Normal case.** Run the independence gate, then dispatch. |
| N ≥ 6 | **Hard refuse.** Reconciliation cost on >5 results exceeds parallel benefit, and the user is likely modeling the wrong problem — recommend `/arsenal-build:features-{web,ios}` with a TASKS.md phase instead. |

Per-investigation idempotence: if `.tasks/parallel/<run-id>/investigation-{N}-result.md` already exists at the canonical path, that investigation is skipped with status `SKIPPED — result exists` unless `--force` is set.

## Inputs

Caller passes:

- `--investigation "<one-line description>"` — repeated 2–5 times for inline use, OR
- `--from-file <path>` — file with one investigation per line. Use when each investigation needs more than ~80 chars of description.
- `--surface web|ios` — optional. Passed to investigator subagents as project-context flag (helps them know the codebase shape). Omit when investigations are surface-agnostic.
- `--force` — optional. Regenerate investigations whose result file already exists. Default is idempotent (L1-style skip).
- `--max <N>` — optional. Overrides the default upper bound (5). Setting `--max 6` or higher fails fast with the design-rationale message.

Files this skill reads (read-only):

- `references/investigator-prompt.md` — the subagent template, read on dispatch
- Reconciliation reads only the `## Files touched` line and `## Recommendations` section from each result file (minimum needed to build SUMMARY)

Files written:

- `.tasks/parallel/<run-id>/investigation-{N}-result.md` per investigation (≤3,000 tokens / ~12,000 chars — same budget as context briefs)
- `.tasks/parallel/<run-id>/SUMMARY.md` — aggregated report

**Run-id format:** `YYYY-MM-DD-HHMMSS` (timestamp). Naturally orderable and unique per dispatch.

## Independence gate

The gate runs at Step 2 before any dispatch. It is the whole point of this skill. If the gate fails, the skill refuses to fan out and recommends sequential execution.

**Triadic test** (all three must hold):

1. **Disjoint scope.** Each investigation targets a different file / directory / subsystem / test target. If two investigations both target `payments/`, that's overlap — refuse, or surface the overlap to the user and ask whether one investigation should be the parent.
2. **No shared state mutations.** Investigations are pure-read by design (the investigator subagent has zero write capability). The skill enforces this by dispatching investigators, not implementers. If a user describes an investigation in mutating terms ("delete the dead code in X"), surface that mismatch and ask: do you want investigation (find the dead code, recommend deletions) or sequential task execution (actually delete it, via run-task)?
3. **Result-independence.** Investigation N's output does not affect investigation M's prompt. If the user describes a dependency ("look at the auth flow, THEN check whether payments uses it"), refuse — that's sequential by definition.

**Failure modes:**

- **Hard refuse** when (1) clearly fails (investigations target the same file/subsystem). Print the overlapping target. Recommend: merge into one investigation, or run sequentially.
- **Soft confirm** when (2) or (3) is ambiguous. Show the parsed list and ask the user to confirm independence or refine.
- **Hard refuse with count rationale** when count > 5. Recommend modeling as a `TASKS.md` phase.

## Composition (and explicit non-composition with run-task)

This skill does NOT call `run-task-{web,ios}`. Investigations are read-only; run-task requires a TASKS.md phase block, pre-generated briefs, a phase branch, and ends with an atomic commit + `[x]` flip. Forcing dispatch-parallel to compose with run-task would mean synthesizing all of that for every read-only audit — heavy machinery without value.

For fixing things investigations find: the SUMMARY's "Next steps" block points the user at `/arsenal-build:run-task-{web,ios}` (sequential, one fix at a time) or `/arsenal-build:features-{web,ios}` (model as a phase). The connection is filesystem + user judgment, not direct invocation.

The investigator subagent CAN call other read-only tools — including `impeccable:audit`, `impeccable:critique`, `claude-in-chrome`, `firecrawl`, `WebSearch`, etc. — when the investigation description warrants them. The investigator-prompt template lists these as available without forcing their use.

## Reconciliation logic

After all N investigators report back:

1. Confirm each `investigation-{N}-result.md` exists on disk. Per filesystem-handoff discipline, this skill does NOT read the full content.
2. Read only the `## Files touched` line and `## Recommendations` section from each result file. Other sections (Method, Findings, Confidence) stay subagent-territory.
3. Compute cross-investigation overlap: intersect the `Files touched` lists. Flag any path that appears in 2+ investigations as an informational note (not a blocker — overlap on shared files like `package.json` is normal).
4. Aggregate recommendations by severity (Critical / Important / Suggested). Dedupe identical strings. Cite source investigation per recommendation.
5. Flag CONFLICTs: when two investigations produce contradictory recommendations (e.g., "remove X" vs "X is load-bearing"), the SUMMARY surfaces them as `## Conflicts requiring user resolution`. **The skill does NOT pause for resolution.** It writes the SUMMARY with the CONFLICT section populated, reports back to the caller, and exits cleanly. The user invokes whatever follow-up makes sense after reading the SUMMARY. (Same J3-pattern as close-phase: surface + recommend + exit; never act unilaterally.)
6. Write `SUMMARY.md` to disk. Report to caller.

## Workflow

### Step 1: Validate inputs and idempotence

- Parse the investigation list from `--investigation` repeats or `--from-file`.
- Count check: reject 0; refuse-with-suggestion 1; normal 2–5; hard-refuse 6+.
- Compute `<run-id>` = current timestamp `YYYY-MM-DD-HHMMSS`.
- If a `.tasks/parallel/<run-id>/` directory already exists (extremely rare collision OR user re-running same run-id): require `--force` to overwrite.
- Per-investigation idempotence: when re-dispatching (e.g., a prior run failed partially), check each `investigation-{N}-result.md` — skip those that exist unless `--force`.
- `mkdir -p .tasks/parallel/<run-id>/`.

### Step 2: Run the independence gate

- Parse each investigation's described scope (file paths, subsystems, test targets).
- For each pair, check the three independence criteria (disjoint scope, no shared mutations, result-independence).
- Present the parsed list to the user with a structured confirmation:

  > "I'll dispatch N investigators in parallel. Each targets:
  > - Investigation 1: [parsed scope] — [original description]
  > - Investigation 2: ...
  > Confirm independent? (Yes / Refine / Cancel)"

- On **Yes**: proceed to Step 3.
- On **Refine**: ask which investigations need to be merged, split, or removed. Re-run the gate after refinement.
- On **Cancel**: stop. Recommend sequential execution if appropriate.

### Step 3: Dispatch N investigator subagents in parallel

- For each investigation, spawn a fresh-context investigator subagent using `references/investigator-prompt.md`. Substitute the bracketed placeholders: investigation text, surface flag (or "none"), result file path, run-id.
- Dispatch all N simultaneously. This is the whole point — subagents are isolated by design and won't collide.
- Subagents read project files broadly but write only their own result file. Other subagents' result files do not exist yet at dispatch time, and even on `--force` regeneration, each subagent ignores siblings.

### Step 4: Wait for all subagents to return

- Each subagent reports back when its investigation is complete with a status: DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED.
- This skill does NOT impose a per-subagent timeout. Investigators control their own scope; a hanging subagent is the user's signal to cancel and re-dispatch with a narrower investigation.
- DO NOT begin reconciliation until all N have reported. Partial reconciliation breaks the "one SUMMARY per run" contract.

### Step 5: Handle each subagent's response

| Status | Action |
|---|---|
| **DONE** | Confirm `investigation-{N}-result.md` exists on disk. Continue to reconciliation. |
| **DONE_WITH_CONCERNS** | Confirm file exists. Concerns appear in the result file's `## Confidence` section; SUMMARY surfaces them. |
| **NEEDS_CONTEXT** | The result file may be partial or empty. Note in SUMMARY which investigations need additional context. User can re-dispatch the specific investigation with `--force` after providing what was missing. |
| **BLOCKED** | The result file says only "BLOCKED — <reason>". SUMMARY flags this clearly. **This skill does NOT itself retry or dispatch fix subagents.** User decides whether to retry with different framing. |

### Step 6: Reconcile into SUMMARY.md

Follow the algorithm in the "Reconciliation logic" section above. Write `.tasks/parallel/<run-id>/SUMMARY.md`.

### Step 7: Report to caller

Report:

- **Status:** DONE | DONE_WITH_FAILURES (M of N succeeded) | NEEDS_CONTEXT (count of partial investigations)
- **Run-id:** `<run-id>`
- **Investigations dispatched:** N
- **Results written:** M (≤ N if any failed)
- **Cross-investigation file overlap:** count, with paths cited (informational)
- **Aggregated recommendations:** counts by severity (Critical: A, Important: B, Suggested: C)
- **Conflicts requiring user resolution:** count
- **SUMMARY path:** `.tasks/parallel/<run-id>/SUMMARY.md`
- **Next-step recommendation:** based on findings — run-task / features-* / debug / discard

## Reference files

| File | Used in | Purpose |
|---|---|---|
| `references/investigator-prompt.md` | Step 3 | Template for the investigator subagent. Defines the brief structure (Investigation, Method, Findings, Files touched, Recommendations, Confidence), the broad-read / zero-write tool allowlist, and the `impeccable:audit` / `:critique` availability note. |

The template has bracketed placeholders. Substitute every one before dispatch — never ship a template with stubs intact.

## Anti-patterns — never do these

- **Don't dispatch parallel when the work isn't genuinely independent.** The triadic test is the whole skill's value. Failing the gate and refusing is the success case for the dependent-work scenario, not a defect.
- **Don't exceed 5 parallel investigations.** Hard cap. Beyond 5, reconciliation cost outpaces parallel benefit AND the user is probably modeling the wrong problem — they likely want a TASKS.md phase via `/arsenal-build:features-{web,ios}`.
- **Don't accumulate investigator context in this skill's own context.** Reconciliation reads only the `Files touched` line and `Recommendations` section. Full content stays in `.tasks/parallel/<run-id>/`. Reading everything defeats the filesystem hand-off pattern (same discipline as `generate-design-briefs` / `generate-feature-briefs`).
- **Don't dispatch fix subagents from this skill.** Investigations report. Fixes are sequential — that's `/arsenal-build:run-task-{web,ios}`. The SUMMARY recommends fix follow-ups; the user invokes them explicitly.
- **Don't pause inside the skill for conflict resolution.** When investigations contradict each other, the SUMMARY flags the conflict in a `## Conflicts requiring user resolution` section and the skill exits cleanly. Same J3 pattern as close-phase: surface + recommend + exit; never act unilaterally.
- **Don't re-run an investigation without `--force`.** Mirrors L1 (briefs) and M1 (tasks). Default skip if `investigation-{N}-result.md` already exists at the canonical path.

## Integration with other arsenal skills

| Skill | Relationship |
|---|---|
| `/arsenal-build:run-task-{web,ios}` | **Downstream, not direct dispatch.** When investigations recommend code changes, the SUMMARY's "Next steps" points the user at run-task for sequential fix execution. No direct invocation from this skill. |
| `/arsenal-build:features-{web,ios}` | **Downstream.** When investigations reveal a pattern-spanning need (e.g., audit reveals 6 files need the same refactor), the SUMMARY recommends modeling as a TASKS.md phase via the orchestrator. |
| `/arsenal-build:expand-phase` | **Indirect.** Investigations that produce a list of remediation tasks may motivate a `--force` re-expansion of a TASKS.md phase. The user makes that call after reading SUMMARY. |
| `/arsenal-build:close-phase-{web,ios}` Gate 1 | **Pattern parallel.** Both skills follow the "surface + recommend + exit" pattern (J3 contract) when faced with a decision the skill shouldn't make unilaterally. |
| `arsenal-build:debug` (future, not yet shipped) | **Future composition candidate.** When `arsenal-build:debug` exists, dispatch-parallel could call it as the investigator for single-bug investigations. Until then, the bespoke `investigator-prompt.md` handles all investigation types. |
| `impeccable` (specifically `:audit` and `:critique`) | **Available to investigator subagents.** When the investigation description says "audit X" or "critique Y", the investigator subagent can invoke `impeccable:audit <surface>` or `impeccable:critique <surface>` against that target. The prompt template lists these as available; the subagent decides when to invoke them based on the investigation framing. |
| `mcp__claude-in-chrome`, `mcp__firecrawl`, `WebSearch` | **Available to investigators.** Investigations that need live-browser inspection or web research get these through the investigator subagent's tool allowlist. |

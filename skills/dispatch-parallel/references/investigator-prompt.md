# Investigator Subagent Prompt Template

Used by `dispatch-parallel` for each parallel investigation. Spawn a fresh-context investigator subagent with this prompt, substituting the bracketed placeholders. Investigators are **read-broad, write-zero** — they can read any project file, follow imports, run read-only shell commands, and invoke read-only sibling skills (`impeccable:audit`, `impeccable:critique`, `claude-in-chrome`, `firecrawl`, `WebSearch`). They cannot modify any file except their own result file.

Investigations may target either pipeline (design or feature) — the project's `TASKS.md` splits each phase into `### Design tasks` and `### Feature tasks` subsections, and either domain may be under investigation. The investigator's recommendations should cite which pipeline a follow-up task belongs to when relevant.

```
## Your Role
You are running ONE investigation in parallel with sibling investigators you cannot see and must not coordinate with. Your scope is fixed. Produce a single result file at the path specified below.

## Output Path
Write your findings to `.arsenal/tasks/parallel/[run-id]/investigation-[N]-result.md`. Do not write anywhere else.

## Output Budget
≤3,000 tokens (~12,000 characters). If you cannot fit your findings within budget, summarize down rather than paste full file contents. Cite by `path:line` instead of duplicating code blocks. If the investigation genuinely needs more depth than 3k tokens, report `TASK_TOO_BIG` and recommend the user split the investigation.

## The Investigation
[The investigation description — paste verbatim from the dispatch arg]

## Project surface
[web | ios | none]
[If web: codebase is likely Next.js / Astro / Vite / Node / Python — look for React patterns, package.json conventions, etc.]
[If ios: codebase is Swift/SwiftUI — look for Apple platform patterns, theme module, SwiftData/CloudKit usage.]
[If none: no surface assumptions; let the codebase reveal itself.]

## Tools available to you
- **Read, Glob, Grep** — broad project file access; follow imports, read related files
- **Bash (read-only commands only)** — `git log`, `git diff`, `git blame`, `cat`, `grep`, `find`, `ls`, `wc`, etc. Do NOT run anything that mutates state (no `git checkout`, no `rm`, no `npm install`, etc.)
- **WebSearch, Firecrawl** — for documentation / library API research when needed
- **claude-in-chrome MCP** — for live-browser inspection if available and the investigation involves a running web surface
- **impeccable sub-skills** — when the investigation explicitly asks for an audit or critique (`audit X`, `critique Y` framing), you may invoke `impeccable:audit <surface>` or `impeccable:critique <surface>` against the target. These are read-only and report findings without modifying files.

## Tools you do NOT have access to
- **Write, Edit, NotebookEdit** — investigators never modify project files
- Any mutating Bash command (state changes, installs, builds, file moves/deletes)
- Any subagent dispatch (you don't fan out further; that's dispatch-parallel's job, and recursive fan-out defeats the independence model)

## Boundaries
- Read ONLY files relevant to your investigation. Do not glob the whole project unless the investigation explicitly says "audit the whole codebase for X."
- Do not read other investigations' result files. They don't exist yet, and even on `--force` regeneration each investigator works in isolation.
- Do not coordinate with sibling investigators. If you discover findings that overlap with another investigation's scope, note the overlap in `## Files touched` but do not adjust your investigation's focus.
- Do not propose follow-up investigations. Recommend code changes if warranted (that goes to the user via SUMMARY → run-task), but don't recursively scope new investigations.

## Brief Structure (use exactly this template)

```markdown
# Investigation [N] — [Short Title]

## Investigation
[The investigation description — paste verbatim from the dispatch arg]

## Method
[1-3 paragraphs on what you did: which files you read, which tools you used, what searches you ran, whether you invoked impeccable. Be specific so the user can audit your approach.]

## Findings
[The substantive section. Use `file:line` cites for every claim. Group by theme if the investigation is broad. Use prose; tables when grouping is helpful.]

- Finding 1: [description with `path/to/file.ts:42` cite]
- Finding 2: ...
- ...

## Files touched
[Comma-separated list of every file path you read substantively. This is the SUMMARY's overlap-detection input — keep it accurate. Include only files where you actually drew findings from, not every file you glanced at.]

Example: `src/auth/login.ts, src/auth/session.ts, tests/auth.test.ts`

## Recommendations
[Severity-tagged action items. Critical = must fix / would break functionality. Important = real quality issue worth addressing. Suggested = stylistic or minor improvement.]

### Critical
- [Recommendation with rationale and the file:line that motivates it]

### Important
- ...

### Suggested
- ...

(If a section is empty, omit it. If all sections are empty, write "No actionable recommendations" — sometimes "this is fine" is the correct finding.)

## Confidence
[High / Medium / Low + reasoning. Cite anything that limits your confidence: files you couldn't fully read, areas the investigation touched that needed deeper analysis than the 3k budget allowed, library versions you couldn't verify.]
```

## How to do this well

1. **Start narrow.** Read what the investigation description names directly before chasing imports outward. The investigation's scope is your authority — don't drift into adjacent concerns.
2. **Cite by `path:line`, don't paste.** Code blocks eat your token budget. A `src/auth/login.ts:42-58` cite is enough; the user can open the file.
3. **Distinguish findings from recommendations.** Findings are "this exists in the code." Recommendations are "this should change." Don't conflate.
4. **Be honest about confidence.** Low confidence with a clear reason is more useful than high confidence on shaky evidence. The Confidence section is for the user; mark it down when the investigation hit limits.
5. **Use impeccable when it fits.** If the investigation says "audit the dashboard" and `impeccable:audit dashboard` would do meaningful work, run it. Don't manually re-implement what an existing skill does. Cite impeccable's output in your Findings section if you invoked it.
6. **`Files touched` is the SUMMARY's input.** Keep it accurate — it drives cross-investigation overlap detection. If you read a file but drew no findings from it, omit it.

## Report Format

When done, report to the calling skill (`dispatch-parallel`):

- **Status:** DONE | DONE_WITH_CONCERNS | NEEDS_CONTEXT | BLOCKED | TASK_TOO_BIG
- **Investigation:** [N]
- **Output path:** `.arsenal/tasks/parallel/[run-id]/investigation-[N]-result.md`
- **Findings count:** [number of items in the Findings section]
- **Recommendations:** Critical=A, Important=B, Suggested=C
- **Confidence:** High | Medium | Low
- **Tools invoked:** [comma-list — e.g., "Grep, Bash (git log), impeccable:audit"]

If `BLOCKED`: write the result file with only the BLOCKED reason in the `## Investigation` section and leave subsequent sections empty. The SUMMARY will surface this for the user.

If `TASK_TOO_BIG`: write a partial result file with what you have, and recommend a narrower follow-up investigation in the Confidence section.
```

## Isolation rules (`dispatch-parallel` enforces)

- Result files live only at `.arsenal/tasks/parallel/<run-id>/investigation-{N}-result.md` — never shared paths.
- `dispatch-parallel` confirms the file exists after the investigator returns. It reads only the `## Files touched` line and `## Recommendations` section for reconciliation; it does NOT read Findings, Method, or Confidence.
- Each investigator is fresh-context — no conversation history, no carryover, no awareness of sibling investigators.
- Investigators have zero write capability outside their own result file. Write/Edit/NotebookEdit are denied at the system-prompt level.

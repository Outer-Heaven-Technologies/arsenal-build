# Researcher Subagent Prompt Template

Used by `run-task-design-web` for the researcher dispatch step when a task is flagged `research: yes`. Spawn a fresh-context researcher subagent with this prompt, substituting the bracketed placeholders with task-specific values.

```
## Research Task
[The specific question to research — e.g., "Current best practices for Family Controls authorization flow in iOS 18+"]

## Scope
[What the relevant feature spec / UX.md / ARCHITECTURE.md says about this task — paste the minimum relevant excerpt]

## Your Job
1. Use available web search and documentation tools to gather current, authoritative best practices.
2. Focus on: API patterns, common pitfalls, version-specific concerns, security/privacy considerations.
3. Output a single markdown file at `.tasks/phase-N/task-N-research.md` with:
   - Summary (3-5 bullet points)
   - Recommended pattern (with code if relevant)
   - Pitfalls to avoid
   - Sources (authoritative ones only)
4. Keep the file under 2k tokens. Lean is better than exhaustive.

## Boundaries
- Do not read other tasks' research files or context briefs.
- Do not modify code.
- Do not read full canonical project docs — your scope above is sufficient.
```

## Isolation rules (controller enforces)

- Research files live only at `.tasks/phase-N/task-N-research.md` — never shared paths.
- The controller confirms the file was written but does not read its contents.
- Each task gets a fresh research pass — no caching across tasks even when the same API recurs.

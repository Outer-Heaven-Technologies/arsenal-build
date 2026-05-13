# Code Quality Reviewer Prompt Template

Used by `run-task-design` for the code quality review step. Runs only after visual fidelity review has passed. Spawn with this prompt:

```
## Your Role
You are reviewing code for QUALITY. Spec compliance has already been verified.

## Conventions for This Project
[Relevant excerpts from CONVENTIONS.md — the standards this code should meet]

## What to Review
[List of files changed by the implementer]

## Review Criteria
- Does the code follow project conventions and patterns?
- Proper error handling, type safety, defensive programming?
- Clear naming that describes what things do (not how they work)?
- Each file has one clear responsibility?
- Tests verify behavior (not just mock behavior)?
- No premature abstraction, no over-engineering?
- Security concerns? Performance issues?
- SOLID principles where appropriate?

## Output Format
**Strengths:** [what was done well]

**Issues** (if any):
- Critical: [must fix before proceeding]
- Important: [should fix]
- Minor: [suggestions, can be deferred]

**Assessment:** ✅ Approved | ❌ Needs fixes
```

## On findings

- Dispatch a fix subagent (using the implementer template) with the specific issues.
- Quality reviewer reviews again.
- **Critical** and **Important** issues must be fixed. **Minor** issues can be noted and deferred (record them in the phase summary so they're not lost).

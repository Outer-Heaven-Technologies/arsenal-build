# Researcher Subagent Prompt Template — iOS

Used by `run-task-design-ios` for the researcher dispatch step when a task is flagged `research: yes`. Spawn a fresh-context researcher subagent with this prompt, substituting the bracketed placeholders.

iOS research most often touches: Apple platform APIs (Family Controls, CloudKit, StoreKit 2, WidgetKit, NotificationCenter, HealthKit), SwiftUI 18.x idioms (`@Observable`, `.task`, `.fullScreenCover` semantics, `scenePhase`), or library API drift across recent iOS versions.

```
## Research Task
[The specific question to research — be narrow. Examples:
- "Current best practices for `.fullScreenCover` + `scenePhase` abandonment cleanup in SwiftUI 18.x"
- "CloudKit `NSPersistentCloudKitContainer` + SwiftData first-import event detection — `eventChangedNotification` vs polling fetch"
- "StoreKit 2 — handling restore purchases when no entitlement exists vs entitlement expired"
- "Reduced Motion gating for paced animations in SwiftUI — when does Apple recommend collapsing entirely vs shortening?"]

## Scope (your reading boundaries)
[Relevant excerpt from the feature spec / UX / ARCHITECTURE — paste the minimum needed for context. Usually 200-500 words.]

## Your Job

1. Search for current, authoritative best practices. Authority hierarchy:
   a. Apple Developer Documentation (developer.apple.com)
   b. WWDC session transcripts and sample code (last 2 years)
   c. Apple-staff or core-team Swift forum posts
   d. PointFree, Swift By Sundell, Donny Wals (when topic is in their wheelhouse)
   e. GitHub issue threads on Apple-published sample code
   f. Stack Overflow only when the answer is verifiable against Apple docs

2. Focus on:
   - **API patterns** — the canonical way Apple expects this to be used
   - **Common pitfalls** — gotchas that cause production bugs (memory cycles, race conditions, off-thread mutations)
   - **Version-specific concerns** — what changed in iOS 17 → 18, what's deprecated, what's the new idiom
   - **Privacy / entitlement implications** — does this need a usage description, an entitlement, App Store review attention?

3. Output a single markdown file at `.tasks/phase-N/task-N-research.md` with this structure:

```markdown
# Research — [Task Title]

## Summary (3-5 bullets)
- [most important finding 1]
- [most important finding 2]
- ...

## Recommended pattern
[Code snippet in Swift if relevant — keep under 30 lines]

## Pitfalls to avoid
- [pitfall 1 — what goes wrong, why, citation]
- [pitfall 2]

## Version notes
- iOS 17: [behavior]
- iOS 18: [behavior, what changed]
- iOS 18.x: [if applicable]

## Sources
- Apple: [URL]
- WWDC: [year, session number, title]
- [other authoritative sources]
```

4. **Keep the file under 2,000 tokens.** Lean is better than exhaustive. The implementer reads this verbatim — every sentence costs context.

## Boundaries
- Do NOT read other tasks' research files or context briefs.
- Do NOT modify code.
- Do NOT read full canonical project docs — your scope above is sufficient.
- Do NOT speculate. If sources disagree or the topic is genuinely unsettled, say so explicitly — better to flag uncertainty than to ship a confident wrong answer.

## Common iOS research targets — quick reference
- **Family Controls:** distinguish `ManagedSettings`, `DeviceActivity`, `FamilyActivityPicker`, `AuthorizationCenter`. Note: most consumer apps don't qualify for the Family Controls entitlement.
- **CloudKit + SwiftData:** all `@Model` attributes must be optional or have default values; no `@Attribute(.unique)`; the mirror lags behind local writes. `NSPersistentCloudKitContainer.eventChangedNotification` is the modern signal.
- **StoreKit 2:** `Transaction.currentEntitlements`, `Transaction.updates`, `AppStore.sync()` for restores. Sandbox vs production differences matter.
- **scenePhase:** `.background` fires before suspension, `.inactive` during transitions. Don't rely on `.onDisappear` for `.fullScreenCover` cleanup — it doesn't fire on backgrounding.
- **SwiftUI animation:** `.transition(.opacity)` cross-fade is most reliable. `.matchedGeometryEffect` is powerful but easy to break. Reduced Motion check via `@Environment(\.accessibilityReduceMotion)`.
- **Concurrency:** `MainActor` annotation > manual hop. `Task` cancellation is cooperative — long loops must check `Task.isCancelled` or `try Task.checkCancellation()`.
- **WidgetKit:** App Groups for shared SwiftData; `WidgetCenter.shared.reloadAllTimelines()` to push updates; widgets cannot do async-throwing in `getTimeline`.

If your task touches one of these, prioritize Apple docs over third-party blogs.
```

## Isolation rules (controller enforces)

- Research files live only at `.tasks/phase-N/task-N-research.md` — never shared paths like `docs/research/`.
- The controller confirms the file was written but does not read its contents.
- Each task gets a fresh research pass — no caching across tasks even when the same API recurs (Task 2 and Task 5 both touching CloudKit get separate, scoped research).
- Researcher subagents are fresh-context; no conversation history, no carryover.

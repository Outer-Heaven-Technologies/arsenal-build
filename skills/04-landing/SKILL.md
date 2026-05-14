---
name: landing
description: Builds high-converting landing pages — researches the audience, structures for conversion (relevance → mechanism → confidence → action), then builds. Default stack: Astro for performance. Pairs with frontend-design for visuals. Auto-fires on "build a landing page", "create a waitlist", "coming soon page", "pre-launch site", "build the marketing page". Use when the user wants a landing page, waitlist, coming-soon page, or pre-launch site.
---

# Execute Landing

Build landing pages that convert, grounded in research about the specific project and modern conversion best practices. This skill handles the full flow: research what the page needs to say, structure it for conversion, and build it.

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

## Philosophy

- **Research the project before designing the page.** A landing page for a developer tool is fundamentally different from one for a consumer app. Research the target audience, competitors' landing pages, and the specific value proposition before writing a single line of code.
- **One page, one goal, one action.** Every element on the page should drive toward a single conversion action. If it doesn't support the goal, cut it.
- **Structure for psychology, not aesthetics.** The page sequence matters: relevance → mechanism → confidence → action. Get the structure right first, then make it beautiful.
- **Performance is conversion.** Every second of load time costs ~7% in conversions. A gorgeous page that loads in 5 seconds loses to a clean page that loads in 1.5.

## Workflow

### Step 0: Lift strategy lockdown (temporarily)

`landing` legitimately reads `MVP_SPEC.md` and `MARKET_RESEARCH.md` from `.arsenal/strategy/`, but during build phases that directory is denied by `.claude/settings.json` (set up by `expand-phase` on first execution-phase entry). Before doing research, lift the strategy deny:

1. Read `.claude/settings.json`. If `permissions.deny` contains `Read(.arsenal/strategy/**)`, **remove that entry temporarily**.
2. Preserve any non-arsenal entries the user added.
3. Write back.
4. Surface to user: *"Lifting `.arsenal/strategy/` lockdown for landing-page research. Will restore at the end."*

After all landing-page work is done (Step 5 review complete), **restore the strategy deny** by re-adding `Read(.arsenal/strategy/**)` to `permissions.deny` and writing settings back. Tell the user: *"Strategy lockdown restored. Build state preserved."*

If `.claude/settings.json` doesn't exist or doesn't have a strategy deny, this is a pre-execution-phase invocation (planning hasn't transitioned to build yet) — proceed without any mutation. The strategy directory is freely readable in that state.

### Step 1: Research Phase

Before building anything, gather context. This research shapes every decision about copy, structure, and design.

**If project planning docs exist (from mvp / setup):**
- Read `.arsenal/strategy/MVP_SPEC.md` for the core value proposition, target user, and distribution hypothesis
- Read `.arsenal/strategy/research/MARKET_RESEARCH.md` (unified executive dossier from `market-analysis`) for audience pain points (§2), demand signals (§1.3), competitor positioning (§3), and SWOT (§5)
- The `MARKET_RESEARCH.md` above is the unified executive dossier from `mvp` — it contains the competitive analysis in §3 (Industry Structure & Competition), so competitor positioning + per-competitor breakdowns + feature/pricing matrices + the positioning map are all there
- Read `.arsenal/design/DESIGN_SYSTEM.md` if it exists for brand/visual direction

**If no planning docs exist, ask the user:**
- What does this product do in one sentence?
- Who is it for? (specific audience, not "everyone")
- What's the single action you want visitors to take? (sign up, join waitlist, buy, download)
- What are 2-3 competitors? (so you can research their landing pages)

**Then research:**
- Crawl 2-3 competitor landing pages to understand how they position, what sections they include, and what their CTAs say
- Note what works and what's generic/weak in competitor pages
- Identify the primary objections the target audience would have (price? trust? complexity? switching cost?)

### Step 2: Page Structure

Build the page structure before writing copy or code. The sequence follows the proven conversion flow:

**Above the fold (visible without scrolling):**
- **Headline:** Answer "what's in it for me?" in under 10 words. Lead with the benefit, not the product name. Frame around what the visitor gains or stops losing — loss aversion is ~2x stronger than gain framing.
- **Subheadline:** Clarify how you deliver the benefit. 1-2 sentences max.
- **Primary CTA:** The one action. Make it specific — "Start free trial" converts better than "Sign up" (implies temporary vs. permanent commitment). Button should be visually dominant.
- **Supporting visual:** Screenshot, demo, hero image, or short video thumbnail. Not decorative — it should demonstrate the product or reinforce the value prop.

**Social proof section:**
- Testimonials with full names, photos, and specific outcomes ("Reduced manual work by 15 hours/week" not "Great product!")
- Logos of recognizable customers/users if available
- Numbers that build credibility (users, downloads, rating, uptime)
- Position proof near the claims it validates and near the CTA it supports

**How it works / mechanism section:**
- 3-step or 3-feature explanation of how the product delivers value
- Keep it scannable — icon + heading + one sentence per step
- This answers "okay, but how does it actually work?"

**Objection handling:**
- FAQ section addressing the 3-5 most likely objections identified in research
- Use structured data (FAQPage schema) for AI search visibility
- Each answer should be 2-3 sentences max — concise, direct, confidence-building

**Final CTA:**
- Repeat the primary CTA with a slightly different angle
- This catches visitors who needed more information before committing
- Can include a secondary/softer CTA for visitors not ready (e.g., "or subscribe to updates")

**Footer:**
- Minimal: links to about/team, terms, privacy
- Don't distract — no full site navigation

### Step 3: Copy Guidelines

- **Be specific, not clever.** "Save 4 hours every week on invoice management" beats "Revolutionize your workflow."
- **Use the visitor's language.** Mirror the words they use to describe their problem (from research/community posts), not internal product jargon.
- **One idea per section.** Don't stack multiple value props in one block.
- **CTA text should describe the outcome, not the action.** "Get my free plan" > "Submit." "See it in action" > "Watch demo."
- **Avoid:** "We are excited to announce...", "Welcome to...", "The world's first...", "Leveraging cutting-edge...", and any other startup-speak that says nothing. Lead with the user's problem, not your story.

### Step 4: Technical Implementation

**Project structure:**
Landing pages should live in their own repository, separate from the core product codebase. This keeps deployment independent — you can ship landing page changes without touching the product, and vice versa.

Recommended stack for standalone landing pages:
- **Astro** (preferred) — ships zero JS by default, blazing fast static output, supports React/Vue/Svelte components if needed, built-in image optimization. Perfect for landing pages because performance is conversion.
- **Plain HTML + Tailwind** — even simpler for a single page. One `index.html`, Tailwind build, deploy as static. No framework overhead.
- **Next.js** — only if the landing page needs dynamic features (auth-gated content, server-side personalization, API routes for form submission). For a standard landing page or waitlist, it's more than you need.

The landing page repo should be self-contained: its own `package.json`, its own deploy config, its own domain/subdomain. If the core product is at `app.product.com`, the landing page lives at `product.com` or `www.product.com`.

Ask the user which stack they prefer. If they don't have a preference, default to Astro.

**Performance non-negotiables:**
- Target < 2 second load time. This is the critical threshold — every second beyond costs ~7% in conversions.
- Compress all images (WebP format preferred, fallback to optimized JPEG/PNG)
- Minimal JavaScript — the page should work without JS where possible
- No render-blocking third-party scripts above the fold
- Use a CDN if deploying independently

**Mobile-first:**
- Design for mobile first, then expand to desktop. Over 50% of traffic is mobile.
- Touch targets minimum 44x44px
- Stack elements vertically on mobile — no side-by-side CTAs
- Test on an actual device, not just browser resize
- Replace autoplay videos with thumbnail + play button on mobile

**Email capture (if waitlist/pre-launch):**
- Maximum 1-2 fields (email only, or email + name). Every additional field reduces conversion.
- Inline validation — show errors as they type, not after submit
- Clear confirmation state — tell them what happens next ("Check your inbox" not just "Thanks!")
- **Kit (formerly ConvertKit)** is the recommended email platform. Free up to 10,000 subscribers with tagging, basic automation, and landing page forms. Use Kit's form embed or API to capture emails directly from the landing page.
  - Create a tag per project (e.g., "myproject-waitlist", "myproject-lead") to segment subscribers across projects in one Kit account
  - Set up a welcome sequence: confirmation email → value email (what to expect) → launch notification when ready
  - Kit's API (`https://api.convertkit.com/v3/`) supports programmatic subscriber creation if you prefer a custom form over their embed
- For Outer Heaven client projects: create separate Kit tags (or forms) per client. One Kit account, segmented by project.

**Analytics:**
- **PostHog** is the recommended analytics platform. Free up to 1M events/month. Provides web analytics, session replay, funnels, and feature flags in one tool.
- Install the PostHog JS snippet in the landing page `<head>` tag
- Track: page views (autocaptured), CTA clicks, form submissions, scroll depth
- Set up a conversion event for the primary action (e.g., `posthog.capture('waitlist_signup')`)
- If the project already has PostHog from Phase 0 of setup, use the same project — don't create a second instance. If the landing page is a separate repo/domain, create a separate PostHog project to keep data clean.
- PostHog has an MCP server — Claude Code can query your analytics data directly for insights during GTM execution

**SEO & AI search basics:**
- Semantic HTML (proper heading hierarchy, meaningful alt text)
- Meta title and description that match the headline/value prop
- FAQPage schema markup on the FAQ section
- Open Graph tags for social sharing (title, description, image)
- Fast load + mobile-friendly = good Core Web Vitals = better ranking

### Step 5: Build & Review

**If this is a new standalone landing page:**
- Initialize a new repo (`git init`, add remote)
- Scaffold the project with the chosen stack (Astro: `npm create astro@latest`, or plain HTML + Tailwind)
- Build the page following the structure from Step 2
- Deploy to a static hosting provider (Vercel, Netlify, Cloudflare Pages)

**If this is a task within an existing project's TASKS.md:**
- Check whether the task specifies a separate repo or a route within the existing app
- If separate repo: scaffold as above
- If within the existing app (e.g., a `/coming-soon` route in Next.js): build using the project's existing stack and conventions

Then review against this checklist:

**Conversion checklist:**
- [ ] Headline answers "what's in it for me?" within 5 seconds
- [ ] Single primary CTA is visually dominant and above the fold
- [ ] Social proof includes specific outcomes, not just praise
- [ ] FAQ addresses top 3 objections from research
- [ ] CTA text describes the outcome, not the action
- [ ] Page has one goal — no competing CTAs or distracting navigation

**Technical checklist:**
- [ ] Loads in under 2 seconds (test with Lighthouse or PageSpeed Insights)
- [ ] Mobile layout works on a real device
- [ ] Touch targets are 44x44px minimum
- [ ] Images are compressed and in modern formats
- [ ] Email capture works end-to-end (submit → stored → confirmation shown)
- [ ] Analytics events fire for page view, CTA click, and form submission
- [ ] Open Graph tags render correctly (test with social preview tools)
- [ ] FAQPage schema is valid (test with Google's Rich Results tool)

## What This Skill Does NOT Cover

- **Full marketing site with multiple pages** — this is for a single landing page. A full site is a different scope.
- **Ad campaign strategy** — that's gtm territory. This skill builds the page; GTM decides how to drive traffic to it.
- **Ongoing A/B testing** — the skill builds a strong v1. Iteration based on real data is a post-launch activity.

## Integration with Other Skills

| Skill | Relationship |
|-------|-------------|
| `/arsenal-build:setup` | Phase 0 may include a landing page task. This skill handles the execution of that task. |
| `/arsenal-build:features` | If a task in TASKS.md involves building a landing page, the implementer subagent should use this skill's structure and guidelines. |
| `/arsenal-planning:gtm` | GTM defines the positioning and channel strategy. This skill turns that positioning into an actual page. |
| `frontend-design` / Impeccable | Handles the visual execution. This skill handles the conversion structure and copy strategy. They're complementary — this skill says *what* to build, Impeccable says *how it should look*. |

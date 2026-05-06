# Brief: Amazon-style press release for cross-tool latent-graph traversal product

## Context
- Slice: phase-1-narrative
- Repo: https://github.com/BinaryBourbon/the-product (clone this; work in branch `phase-1-narrative/growth-marketer`)
- Prior artifacts you **must** read before writing a single word:
  - `decisions/0003-product-framing.md` — the human-approved product definition; this is your source of truth. Do not deviate from it.
  - `decisions/0002-g0-pending.md` — records the G0 decision and the reframe that was made; explains what framing was explicitly rejected.
  - `discovery/phase-0-framing.md` — background on the two original candidates; useful for context but **not** authoritative on product definition.

## Task
Write `narrative/phase-1-press-release.md`: an Amazon-style "working backwards" press release for this product, written as if it has already launched and is being announced to the world. The press release should be grounded in the corrected product framing from ADR-0003 — the product is a **cross-tool latent-graph traversal interface**, not a dashboard. The standard Amazon PR structure: headline, sub-headline, dateline opening paragraph (who benefits and how), customer quote, what the product does (in plain language), second customer quote, and a closing call-to-action. Keep it under 600 words total.

**The framing you must use (quoted directly from the human's G0 decision):**

> "The pain is **manual cross-tool detective work on siloed data**. The user already knows that PostHog signal X is probably related to a recent GitHub change Y — they just have to manually extract the relevant identifier from one tool, translate it into the right query in another tool, then do that two or three more hops to get a complete picture. The info IS LINKED in reality (a user, an event, a commit, a deploy, an error, a ticket all share identifiers and timestamps) but no tool exposes those links. The product's job is to reveal and traverse that latent graph, not to put four dashboards on one screen."

**Initial tool scope (from ADR-0003):** GitHub + PostHog + Honeycomb only. Do not mention Linear, Sentry, Datadog, PagerDuty, or any other tool.

**The user:** a software engineer, tech lead, or engineering manager mid-investigation — they have a partial signal and a hypothesis, and the bottleneck is the mechanical work of connecting the dots across tools.

## Acceptance
- [ ] `narrative/phase-1-press-release.md` exists and follows Amazon PR structure (headline, sub-headline, opening paragraph, customer quote, product description, second customer quote, call-to-action)
- [ ] Product is described as a traversal / joined-index tool, NOT as a dashboard or unified view
- [ ] Mentions GitHub, PostHog, and Honeycomb as the initial connected tools
- [ ] Does not invent a product name — refer to it as "the product" or use a placeholder like `[PRODUCT NAME]`
- [ ] Does not reference Candidate A (messaging/bots) at all
- [ ] PR opened against `main` with title `phase-1-narrative: growth-marketer output`

## Out of scope
- Do not pick a product name
- Do not design any UI or write any code
- Do not create any files other than `narrative/phase-1-press-release.md`
- Do not edit `ROADMAP.md` (tech-lead owns that)
- Do not invoke any further agents
- Do not reframe the pain as tab-switching or cognitive load — the human explicitly rejected that framing

## ADR commit to reference
The canonical product definition was committed to `main` at commit `6bf9289` (ADR-0003 did not exist at that commit — it will be in the commit immediately after; use `git log --oneline decisions/0003-product-framing.md` to find the exact SHA once you clone).

## Git workflow
```
git clone https://github.com/BinaryBourbon/the-product.git
cd the-product
git checkout -b phase-1-narrative/growth-marketer
# read decisions/0003-product-framing.md and decisions/0002-g0-pending.md
# write narrative/phase-1-press-release.md
git add narrative/phase-1-press-release.md
git commit -m "phase-1-narrative: growth-marketer output"
git push -u origin phase-1-narrative/growth-marketer
# open PR against main via gh or GitHub MCP
```

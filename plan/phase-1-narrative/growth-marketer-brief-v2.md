# Brief: Amazon-style press release for the workspace that replaces the IDE

## Context
- Slice: phase-1-narrative (v2 — previous attempt PR #4 was closed; see note below)
- Repo: https://github.com/BinaryBourbon/the-product (clone this; work in branch `phase-1-narrative-v2/growth-marketer`)
- Prior artifacts you **must** read before writing a single word:
  - `decisions/0004-product-framing-v2.md` — **your source of truth.** The human-approved product definition. Do not deviate from it.
  - `decisions/0003-product-framing.md` — superseded; shows what framing was explicitly **rejected** (cross-tool graph as *the product*). Read it so you know what to avoid.
  - `decisions/0002-g0-pending.md` — G0 record; background only.

## Why the previous attempt was rejected

PR #4 described a cross-tool investigation tool. That framing was **too narrow**. The human's correction (verbatim, from ADR-0004):

> "we're replacing the IDE. we're the new piece of software that engineers and product people spend their time in to plan / execute / follow up on work"

The cross-tool latent graph (GitHub + PostHog + Honeycomb connected via shared identifiers) is **one capability** inside the product — it serves the execute and follow-up phases. It is not the product. Do not lead with it, do not define the product by it.

## Task

Write `narrative/phase-1-press-release.md`: an Amazon-style "working backwards" press release for this product, written as if it has already launched and is being announced to the world.

**The product framing you must use (from ADR-0004, quoting the human verbatim):**

> "we're replacing the IDE. we're the new piece of software that engineers and product people spend their time in to plan / execute / follow up on work"

The product is a **workspace and working environment** — the primary surface where engineering and product work happens, covering three phases:
1. **Plan** — scope work, understand context, align on what to build and why
2. **Execute** — do the work: write code, move tickets, configure systems, ship changes
3. **Follow up** — close the loop: did it ship? did it work? what do the metrics say?

**Users:** both **engineers** (who build and ship) and **product managers** (who plan and measure outcomes). Both are first-class. Do not write a tool for engineers that PMs happen to also use, or vice versa.

**What the product is NOT** (these are explicitly foreclosed — do not use this language):
- Not a dashboard
- Not a viewer
- Not a "unified pane of glass"
- Not an investigation-only tool
- Not a Slack replacement

**On the cross-tool graph:** You may mention that the product connects GitHub, PostHog, and Honeycomb data — but only as an example of how it closes the plan/execute/follow-up loop, not as the headline capability.

**Naming:** Do not invent a product name. Use `[PRODUCT NAME]` as a placeholder throughout.

**Amazon PR format:** headline, sub-headline, dateline opening paragraph (who benefits and how), customer quote (engineer or PM), what the product does in plain language (one to two paragraphs), second customer quote (the other persona), closing call-to-action. Under 600 words total.

## Acceptance
- [ ] `narrative/phase-1-press-release.md` exists and follows Amazon PR structure
- [ ] Product is described as a workspace/working environment replacing the IDE — spanning plan, execute, follow-up
- [ ] Both engineers AND product managers are represented (ideally one quote each)
- [ ] Cross-tool graph (GitHub + PostHog + Honeycomb) appears only as a supporting capability, not the headline
- [ ] No dashboard / viewer / unified-pane language anywhere
- [ ] No product name invented — uses `[PRODUCT NAME]` placeholder
- [ ] PR opened against `main` with title `phase-1-narrative: growth-marketer output (v2)`

## Out of scope
- Do not pick a product name
- Do not design any UI or write any code
- Do not create any files other than `narrative/phase-1-press-release.md`
- Do not edit `ROADMAP.md` (tech-lead owns that)
- Do not invoke any further agents
- Do not reuse or reference the press release from the previous attempt

## ADR commit reference
The canonical product definition is `decisions/0004-product-framing-v2.md`. After cloning, run:
```
git log --oneline decisions/0004-product-framing-v2.md
```
to confirm the exact commit SHA.

## Git workflow
```
git clone https://github.com/BinaryBourbon/the-product.git
cd the-product
git checkout -b phase-1-narrative-v2/growth-marketer
# read decisions/0004-product-framing-v2.md and decisions/0003-product-framing.md
# write narrative/phase-1-press-release.md
git add narrative/phase-1-press-release.md
git commit -m "phase-1-narrative: growth-marketer output (v2)"
git push -u origin phase-1-narrative-v2/growth-marketer
# open PR against main via gh or GitHub MCP
```

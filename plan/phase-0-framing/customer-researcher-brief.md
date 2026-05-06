# Brief: side-by-side framing of two product candidates

## Context
- Slice: phase-0-framing
- Repo: https://github.com/BinaryBourbon/the-product (clone this; work in branch `phase-0-framing/customer-researcher`)
- Prior artifacts you should read:
  - `ROADMAP.md` — describes the two candidates
  - `OPERATING_MODEL.md` — how the team works; explains your role and deliverable format

## Task
Produce `discovery/phase-0-framing.md`: a side-by-side framing of the two product candidates below. For each candidate write three things — (1) **user profile**: who is the primary user, their role, context, and sophistication; (2) **strongest pain**: the sharpest problem this product addresses, i.e. the moment the user feels maximum friction today; (3) **smallest validatable wedge**: the tightest possible first slice that lets us learn whether the pain is real and the approach works, before building anything else. Label **every factual claim** with one of: `[observation]` (directly verifiable fact), `[inference]` (reasoned from observations), or `[hypothesis]` (assumption that needs testing). Do NOT pick a winner — the human makes that call (G0). Keep it under 600 words.

**Candidate A — Slack replacement where bots are first-class citizens**
A modern team-messaging product in which automated agents (bots, integrations, pipelines) share channels as peers rather than as second-class webhook emitters. The bet: async human+bot collaboration is underserved because Slack's bot model is bolted on rather than native.

**Candidate B — Single pane of glass for engineering and product**
A unified dashboard / command surface that replaces the daily tab-switching between Jira, GitHub, and PostHog (or equivalent). The bet: context-switching across tools is a meaningful enough tax that teams will pay for a consolidated view.

## Acceptance
- [ ] `discovery/phase-0-framing.md` exists and contains clearly headed sections for each candidate covering user profile, strongest pain, and smallest validatable wedge, with all claims tagged `[observation]`, `[inference]`, or `[hypothesis]`
- [ ] No winner is picked; the document ends with a neutral "Questions for G0" section listing 3–5 decision criteria the human should weigh
- [ ] PR opened against `main` with title `phase-0-framing: customer-researcher output`

## Out of scope
- Do not recommend one candidate over the other
- Do not design any UI, write any code, or propose any architecture
- Do not create any files other than `discovery/phase-0-framing.md`
- Do not edit `ROADMAP.md` (tech-lead owns that file)
- Do not invoke any further specialist agents

## Git workflow
```
git clone https://github.com/BinaryBourbon/the-product.git
cd the-product
git checkout -b phase-0-framing/customer-researcher
# write discovery/phase-0-framing.md
git add discovery/phase-0-framing.md
git commit -m "phase-0-framing: customer-researcher output"
git push -u origin phase-0-framing/customer-researcher
# open PR against main via gh or GitHub MCP
```

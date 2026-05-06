# ADR-0002: G0 — Project framing (pending human decision)

- **Status:** proposed
- **Date:** 2026-05-06
- **Decider:** human (awaiting G0 input)
- **Slice:** phase-0-framing

## Context

`discovery/phase-0-framing.md` now exists with a full side-by-side comparison of the two candidates. The core trade-off is this: **Candidate A** (bot-first messaging) bets that the Slack bot model is structurally broken and that a greenfield messaging tool with bots as peers can displace it in bot-heavy async teams — a high-upside, high-switching-cost, network-effects-dependent play that likely requires a new data model from day one. **Candidate B** (unified eng/product dashboard) bets that the re-orientation cost of switching between Jira, GitHub, and PostHog is painful enough that teams will pay for an opinionated aggregation layer on top of tools they already own — a lower switching cost, single-user-adoption play that can be prototyped as a read-only integration before any real commitment. Neither pain has been validated by primary research yet; both rest on inferences and hypotheses.

## Options considered

- **A — Bot-first messaging replacement.** Pros: novel data model, LLM-wave timing, high defensibility once adopted. Cons: displaces entrenched tool with network effects; needs critical mass to be useful; higher technical risk.
- **B — Single pane of glass for eng/product.** Pros: additive (no migration), single-user-adoptable on day one, lower technical risk, faster to prototype. Cons: integration-layer products commoditize quickly; may be dismissed as "just a dashboard."

## Decision

None yet — awaiting G0 human input.

## Three questions for the human to weigh

1. **Behavior change appetite.** Candidate A asks early users to move their whole team to a new messaging tool. Candidate B asks one person to open a new tab. How much migration friction can early adopters absorb, and how much are you willing to help them through it?

2. **Go-to-market model.** Candidate A's value scales with the number of teammates on it (network effects); selling it means selling teams, not individuals. Candidate B can be adopted virally by a single eng manager and expand from there. Which distribution motion fits the resources and relationships available right now?

3. **Technical risk tolerance before G1.** Candidate A's wedge (bots as conversational peers) likely requires a purpose-built messaging backend from the start. Candidate B's wedge (read-only unified view) can be assembled from public APIs in a weekend prototype. How much pre-G1 build risk is acceptable before you know the pain is real?

## Consequences

Once the human picks a candidate (or reframes), the tech-lead will close this ADR as accepted, dispatch `growth-marketer` for G1 (narrative), and open the next slice.

## Revisit when

Human provides G0 input.

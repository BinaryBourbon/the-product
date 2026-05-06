# ADR-0002: G0 — Project framing

- **Status:** accepted
- **Date:** 2026-05-06
- **Decider:** human
- **Slice:** phase-0-framing

## Context

`discovery/phase-0-framing.md` now exists with a full side-by-side comparison of the two candidates. The core trade-off is this: **Candidate A** (bot-first messaging) bets that the Slack bot model is structurally broken and that a greenfield messaging tool with bots as peers can displace it in bot-heavy async teams — a high-upside, high-switching-cost, network-effects-dependent play that likely requires a new data model from day one. **Candidate B** (unified eng/product dashboard) bets that the re-orientation cost of switching between Jira, GitHub, and PostHog is painful enough that teams will pay for an opinionated aggregation layer on top of tools they already own — a lower switching cost, single-user-adoption play that can be prototyped as a read-only integration before any real commitment. Neither pain has been validated by primary research yet; both rest on inferences and hypotheses.

## Options considered

- **A — Bot-first messaging replacement.** Pros: novel data model, LLM-wave timing, high defensibility once adopted. Cons: displaces entrenched tool with network effects; needs critical mass to be useful; higher technical risk.
- **B — Single pane of glass for eng/product.** Pros: additive (no migration), single-user-adoptable on day one, lower technical risk, faster to prototype. Cons: integration-layer products commoditize quickly; may be dismissed as "just a dashboard."

## Decision

Human selected **Candidate B**, with a critical reframe of the pain. The discovery doc described the pain as "re-orientation cost when tab-switching between tools." The human corrected this: the actual pain is **manual cross-tool detective work on siloed data** — the user knows objects are linked across tools but must manually extract identifiers, translate them into the right query in another tool, and repeat for each hop. The product is not a unified dashboard; it is a cross-tool latent-graph traversal interface. See ADR-0003 for the full durable product definition.

## Consequences

G0 closed. ADR-0003 records the corrected product framing. Next: dispatch `growth-marketer` for G1 (press-release narrative).

## Revisit when

ADR-0003 is superseded.

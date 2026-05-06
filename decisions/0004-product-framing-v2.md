# ADR-0004: Product framing v2 — workspace that replaces the IDE

- **Status:** accepted
- **Date:** 2026-05-06
- **Decider:** human (G1 reframe)
- **Slice:** phase-1-narrative
- **Supersedes:** ADR-0003

## Context

The growth-marketer's press release (PR #4) described a compelling cross-tool investigation product but was rejected at G1. The human's correction:

> "we're replacing the IDE. we're the new piece of software that engineers and product people spend their time in to plan / execute / follow up on work"

This is a category-level reframe. The product is not an investigation tool, not a dashboard, not a viewer. It is a **working environment** — the primary surface where engineering and product work happens, spanning the full work cycle. Cross-tool latent-graph traversal (ADR-0003) is one capability in service of the execute and follow-up phases; it is not the product itself.

ADR-0003 is superseded. Its content is preserved as a record of a useful capability that is now correctly positioned as a feature, not the product definition.

## Who the user is

**Engineers and product managers, jointly.** This is a meaningful expansion beyond ADR-0003, which implied an eng-only audience (tech leads, SREs, staff engineers mid-investigation). The product must work for both:

- **Engineers** — plan work, write and ship code, debug, follow up on what shipped
- **Product managers** — plan features, track execution, follow up on outcomes (metrics, user feedback)

Both spend their workday across the same underlying data graph (commits, deploys, tickets, events, errors) but approach it from different vantage points. The product must serve both without forcing either to adopt the other's mental model.

## What the product is

A **workspace and working environment** that replaces the IDE as the primary surface where engineering and product work happens. The human's exact words:

> "we're the new piece of software that engineers and product people spend their time in to plan / execute / follow up on work"

The job-to-be-done spans three phases:

1. **Plan** — scope work, understand context, align on what to build and why
2. **Execute** — do the work: write code, move tickets, configure systems, ship changes
3. **Follow up** — close the loop: did it ship? did it work? what do the metrics say?

The product owns all three phases. It is not a tool that users visit occasionally for a specific task — it is where they live.

## What the product is NOT

- **Not a dashboard.** Dashboards are read-only views you visit and leave. This is a working environment you stay in.
- **Not a viewer.** A unified viewer puts four tool UIs on one screen. This replaces those tools.
- **Not an investigation-only tool.** Investigation (cross-tool detective work) is one moment in the execute/follow-up cycle, not the whole product.
- **Not a Slack replacement.** Communication is a means to coordinating work, not the work itself. (Candidate A from G0 is fully off the table.)

## How prior capabilities map in

The cross-tool latent-graph traversal from ADR-0003 is **one feature** in service of the execute and follow-up phases:

- A user mid-execution who notices a PostHog anomaly can traverse the graph (PostHog event → GitHub commit → Honeycomb trace) to understand what changed and why — without leaving the workspace.
- A PM following up on a shipped feature can traverse the graph (deploy → feature flag → funnel metric → error rate) to answer "did it work?"

The graph traversal capability is preserved and correctly scoped. It is not the product; it is a power tool inside the product.

## Open architectural questions (for G2)

- **Code editing in-product vs IDE-as-launchable-tool.** Does the product include a code editor, or does it launch the user's existing IDE (VS Code, Neovim, etc.) for the write-code step? This is load-bearing for the "replaces the IDE" claim and must be decided at G2 before any engineering slice begins.
- **Naming.** Product name is TBD. Agents should refer to it as "the product" or `[PRODUCT NAME]` until naming is decided.

## Initial tool scope

Inherited from ADR-0003. Start with GitHub + PostHog + Honeycomb (existing MCP integrations). Linear, Sentry, Datadog, PagerDuty, and others are out of scope until G3 or later.

## Decision

We are building a **workspace that replaces the IDE** for engineers and PMs — the primary surface for planning, executing, and following up on work. Cross-tool latent-graph traversal (GitHub + PostHog + Honeycomb) is a core capability in service of execute and follow-up, not the product definition. The code-editing model and naming are open questions for G2.

## Consequences

- **Enables.** A much larger market and a much more ambitious product. Replaces a tool category (IDEs + project trackers + analytics) rather than adding a layer on top.
- **Forecloses.** Positioning as an investigation tool, viewer, or dashboard. Any design that doesn't own the full plan/execute/follow-up loop is out of scope.
- **Accepted downside.** The surface area is significantly larger. G2 must tightly bound the first buildable slice to prevent scope sprawl. The code-editing question is high-stakes and must be resolved before engineering begins.

## Revisit when

- G2 resolves the code-editing architectural question and the first engineering slice is scoped.
- User research reveals that "replacing the IDE" is too ambitious as a wedge and a narrower entry point is needed.

# ADR-0003: Product framing — cross-tool latent-graph traversal

- **Status:** superseded by ADR-0004
- **Date:** 2026-05-06
- **Decider:** human (G0 decision)
- **Slice:** phase-0-framing

## Context

At G0 the human selected Candidate B but rejected the discovery doc's framing of the pain. The framing in `discovery/phase-0-framing.md` described the problem as "re-orientation cost / cognitive shift when moving between Jira/GitHub/PostHog tabs." The human corrected this:

> "Wrong framing: re-orientation cost / cognitive shift when moving between tabs.
> Correct framing: the pain is **manual cross-tool detective work on siloed data**. The user already knows that PostHog signal X is probably related to a recent GitHub change Y — they just have to manually extract the relevant identifier from one tool, translate it into the right query in another tool, then do that two or three more hops to get a complete picture. The info IS LINKED in reality (a user, an event, a commit, a deploy, an error, a ticket all share identifiers and timestamps) but no tool exposes those links. The product's job is to reveal and traverse that latent graph, not to put four dashboards on one screen."

This ADR records that framing as the durable product definition. Future agents must treat this as the source of truth. Do not revert to "unified dashboard" or "single pane of glass" language.

## Who the user is

A software engineer, tech lead, or engineering manager at a team that ships software instrumented across multiple surfaces: a code host (GitHub), a product analytics tool (PostHog), and an observability tool (Honeycomb). They are mid-investigation — they have a partial signal (a spike in errors, a drop in a funnel, a regression on a metric) and they already have a hypothesis about what caused it. The bottleneck is not understanding; it is **the mechanical work of connecting the dots across tools**: copying an identifier out of one query result, reformulating it as a search in another tool, and repeating until the full chain of evidence is assembled.

## What the pain actually is

Not tab-switching. Not cognitive load from context shifts. The pain is **manual cross-tool detective work on siloed data**: the user knows the objects are related (a user ID appears in PostHog, GitHub, and Honeycomb; a commit SHA relates to a deploy, a feature flag rollout, and a set of errors) but no tool exposes those relations. Each hop is a manual extraction + translation + re-query cycle. A complete picture of "what changed, for whom, when, and with what effect" requires three to five of these hops, every time, on every investigation.

## What the product is

A **cross-tool joined index** — given any object (a user ID, an event name, a commit SHA, a deploy timestamp, an error fingerprint), surface everything in any connected tool that refers to that same object or is linked to it by a shared identifier or timestamp proximity. Let the user traverse that graph with one action per hop rather than one tool-switch + reformulation per hop.

This is **not** a dashboard. It is **not** a unified view of four tools side by side. It is closer to a universal search / graph traversal interface where the corpus is the union of all connected tool data and the edges are shared identifiers.

## Initial tool scope

Start with the three tools for which MCP integrations already exist:

- **GitHub** — commits, PRs, deploys, issues
- **PostHog** — events, funnels, user properties, feature flags
- **Honeycomb** — traces, spans, derived columns, BubbleUp results

These three tools alone cover the most common investigation chain: *product signal (PostHog) → recent change (GitHub) → system behaviour (Honeycomb)*. Do not design for Linear, Sentry, Datadog, PagerDuty, or any other tool in phase 1. They are explicitly out of scope until G3 or later.

## Decision

We are building a cross-tool latent-graph traversal product. The unit of navigation is any object (identifier or timestamp) that appears in more than one tool. The initial scope is GitHub + PostHog + Honeycomb. The interface is search / traversal, not a dashboard.

## Consequences

- **Enables.** A product that compounds in value as more tools are connected; each new integration multiplies the number of traversable edges, not just the number of data sources visible.
- **Forecloses.** Building a traditional multi-source dashboard. Any design that puts tool views side by side without exposing the shared-identifier graph is explicitly out of scope.
- **Accepted downside.** The latent-graph model requires data ingestion and a cross-tool index, not just proxied API calls. This is a higher-effort foundation than a read-only aggregation layer, but it is necessary to deliver the core value (traversal, not display).

## Revisit when

- A user research session reveals that the manual-hop tax is not the primary pain and that display/summarisation is what users actually pay for.
- The graph traversal model proves technically unworkable within the initial tool APIs (e.g., shared identifiers are not consistently available across GitHub + PostHog + Honeycomb).

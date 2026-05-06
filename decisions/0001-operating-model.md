# ADR-0001: Orchestrator + slices operating model

- **Status:** accepted
- **Date:** 2026-05-05
- **Decider:** human (Jake) + tech-lead
- **Slice:** meta

## Context

This product is being built by a team of AI agents (defined in [`jhgaylor/aod-specs`](https://github.com/jhgaylor/aod-specs)) with no shared memory across conversations and finite per-conversation token budgets. Two structural constraints follow:

1. Specialists cannot see each other's chats. Coordination must happen through written artifacts.
2. Any single conversation that runs too long loses the thread. Work must be sliced into chunks small enough to finish in one chat.

Without a deliberate operating model, a fleet of specialists produces fast, confident, mutually inconsistent output.

## Options considered

- **A — Decentralized "repo as bus."** Each agent reads the repo state and picks up the next thing. No central coordinator. Pros: resilient, no single point of failure. Cons: requires agents to make scope decisions, which they're bad at; risk of multiple agents grabbing the same task or none grabbing the right one.
- **B — Pure orchestrator.** One long-running conversation drives everything; specialists are called only as tools. Pros: maximum coherence. Cons: orchestrator's context fills up fast; one bad turn cascades.
- **C — Hybrid: orchestrator + repo-as-state.** A `tech-lead` agent owns the roadmap and dispatches specialists per task; the repo is the durable state across conversations. The orchestrator's chat is short-lived and disposable; the repo is what persists. Pros: combines coherence with statelessness; gates give the human a real seat at the table. Cons: orchestrator quality is load-bearing; bad briefs produce bad work.

## Decision

We chose **C**. The `tech-lead` agent (definition in `jhgaylor/aod-specs/agents/tech-lead.yml`) owns the roadmap and dispatches specialists. This repo is the durable state. Human gates at G0 (framing), G1 (narrative), G2 (architecture), and G3 (each slice). See [OPERATING_MODEL.md](../OPERATING_MODEL.md) for the full description.

## Consequences

- **Enables.** Long-running coordinated work without any single conversation being long-running. Clear handoffs. A human-readable record of what happened and why.
- **Forecloses.** Pure peer-to-peer specialist collaboration. Specialists don't talk to each other; they talk through the repo.
- **Accepted downside.** The tech-lead is a bottleneck and a single point of quality. A bad brief sends a specialist down the wrong path. Mitigated by tight slice scoping and gates.

## Revisit when

- The roadmap consistently has > 2 slices in flight (signal: orchestrator is overloaded; either the model is wrong or the slices are too big).
- A specialist returns work that materially contradicts another specialist's output more than twice (signal: repo-as-bus is leaking; we may need explicit cross-specialist sync).
- We try to onboard a second human to the loop (signal: the model is too tech-lead-centric for multi-stakeholder work).

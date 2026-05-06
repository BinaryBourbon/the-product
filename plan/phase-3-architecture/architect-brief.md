# Brief: Tidepool v0 — architecture proposal

## Context
- Slice: phase-3-architecture
- Repo: https://github.com/BinaryBourbon/the-product (clone this; work in branch `phase-3-architecture/architect`)
- Prior artifacts you **must** read before proposing anything:
  - `decisions/0004-product-framing-v2.md` — product is Tidepool: a workspace replacing the IDE for engineers + PMs; plan/execute/follow-up loop
  - `decisions/0005-product-name.md` — canonical spelling: **Tidepool**
  - `decisions/0006-execute-model.md` — **critical**: Execute phase is prompt-driven via coding agents. No IDE integration. Manual editing is leak-through. The specific agent runtime is the most important open question you must answer.
  - `specs/v0/spec.md` — full engineering spec: scenario, screen inventory, acceptance criteria, out-of-scope list
  - `design/v0/flow-plan-execute-followup.md` — Mermaid flow of the full loop
  - `design/v0/wireframes.md` — ASCII wireframes for all key screens

## Task

Produce `specs/v0/architecture.md` — a v0 architecture proposal that gives the first engineering slice a concrete, buildable foundation. Cover every section below. Be opinionated: make a call on each open question and state your rationale. Flag disagreements with the spec explicitly; do not silently deviate.

### Required sections

**1. Tech stack**
Propose the v0 stack (frontend framework, backend language/framework, database, real-time transport). Justify each choice in one sentence. Default to boring/proven over clever — v0 needs to ship, not scale.

**2. Deployment**
Tidepool v0 must be deployable on **Render** (constraint from the team — do not propose Vercel, Railway, Fly, AWS, or any other platform). Describe how the app deploys: what Render service types are used, what the release process looks like.

**3. Coding agent runtime — pick one**
This is the most load-bearing decision in the architecture. ADR-0006 settles *what* (prompt-driven; no IDE), but leaves *which runtime* open for you to decide. Options to evaluate:

- **Claude Code as a subprocess** — spawn `claude` CLI in a sandboxed environment; stream stdout back to Tidepool; agent opens PR via GitHub CLI. Pros: high quality on multi-file changes, we dog-food the product's own infrastructure. Cons: subprocess management, sandboxing cost, Anthropic API pricing at scale.
- **AoD (Agent on Demand)** — the same AoD infrastructure this team already uses to run specialist agents. Pros: already provisioned, no new vendor, we dog-food our own platform. Cons: AoD is tuned for single-turn specialist work, not multi-turn interactive coding loops; may need extension.
- **OpenAI Codex / Responses API** — cloud API, no subprocess. Pros: simple API surface, file-system tools built in. Cons: another vendor, quality on complex repo-aware changes unclear.
- **Custom thin wrapper** — wrap any model with a simple tool loop (read file, write file, run tests, open PR). Pros: full control, portable across models. Cons: non-trivial to build reliably.

Pick one. State why. If your choice has a meaningful dependency (e.g., Claude Code CLI must be installed in the sandbox), name it.

**4. Data model**
Define the core entities for v0. At minimum: `WorkItem`, `AgentRun`, `PullRequest` (mirrored from GitHub), `FollowUpCheck`. For each: key fields, relationships, and where it persists (DB table, external API, or both). The cross-tool correlation model matters here — how does Tidepool know that a GitHub PR, a PostHog flag, and a Honeycomb trace are all "about" the same work item?

**5. Tool integrations**
For each of the three tools (GitHub, PostHog, Honeycomb), specify:
- Auth model for v0 (API token, OAuth — which?)
- Key API calls Tidepool makes (list them; don't over-specify)
- Webhook or polling strategy for real-time state (e.g., PR merge detection)

**6. PR review surface**
The spec calls for an in-Tidepool PR review surface (diff view, merge/discard CTAs, follow-up prompt panel). Propose how this is built: pull diff via GitHub API and render it server-side? iframe the GitHub PR? local diff renderer? Justify your choice.

**7. Streaming agent output**
The wireframes show live-streamed agent progress. Propose the transport: SSE, WebSocket, long-poll? Where does the stream originate (agent subprocess → backend → client)? How does the frontend reconnect on drop?

**8. State persistence**
Where does Tidepool state live between sessions? What persists in your database vs. is re-fetched from external APIs on load? Specifically address: work item state machine, agent run log, PR review state.

**9. Open questions for the engineering slice**
List any decisions you could not resolve from the spec and design alone — things that require the engineer to spike or prototype before committing to an approach. Keep this list short (≤5 items); if it's longer, the architecture is under-specified.

## Constraints (non-negotiable)
- v0 deploys on **Render** — no exceptions
- No IDE integration of any kind — ADR-0006 is closed
- GitHub + PostHog + Honeycomb only — no other integrations in v0
- API-token auth for v0 integrations (no OAuth flows required)
- Single-user for v0 (no real-time multi-user sync required)

## Acceptance
- [ ] `specs/v0/architecture.md` exists with all 9 required sections
- [ ] A specific coding agent runtime is chosen with rationale (not "it depends")
- [ ] Render deployment is addressed concretely (service types named)
- [ ] Data model covers at minimum the 4 entities listed above
- [ ] No IDE integration appears anywhere in the proposal
- [ ] PR opened against `main` with title `phase-3-architecture: architect output`

## Out of scope
- Do not write any code
- Do not create files other than `specs/v0/architecture.md`
- Do not design UI or modify design files
- Do not re-open the IDE question — ADR-0006 is settled
- Do not edit `ROADMAP.md`
- Do not invoke any further agents

## Git workflow
```
git clone https://github.com/BinaryBourbon/the-product.git
cd the-product
git checkout -b phase-3-architecture/architect
# read decisions/0004-product-framing-v2.md, decisions/0006-execute-model.md,
#      specs/v0/spec.md, design/v0/*.md
# write specs/v0/architecture.md
git add specs/v0/architecture.md
git commit -m "phase-3-architecture: architect output"
git push -u origin phase-3-architecture/architect
# open PR against main via gh or GitHub MCP
```

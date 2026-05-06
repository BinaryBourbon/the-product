# Brief: Tidepool v0 — smallest IDE-replacing workspace slice

## Context
- Slice: phase-2-design
- Repo: https://github.com/BinaryBourbon/the-product (clone this; work in branch `phase-2-design/designer`)
- Prior artifacts you **must** read before designing anything:
  - `decisions/0004-product-framing-v2.md` — **primary source of truth.** Product is a workspace replacing the IDE for engineers + PMs; plan/execute/follow-up loop; cross-tool latent graph (GitHub + PostHog + Honeycomb) is a capability in service of execute/follow-up.
  - `decisions/0005-product-name.md` — product is named **Tidepool** (one word, capital T, lowercase p).
  - `narrative/phase-1-press-release.md` — approved press release; the tone and user stories here are canonical.
  - `decisions/0003-product-framing.md` — superseded; read it to understand what *not* to design (investigation-only tool, dashboard, viewer).

## Task

Design the **smallest workspace experience that is recognizable as Tidepool** — a v0 slice that covers **one complete Plan → Execute → Follow-up loop** on a real, concrete piece of work, integrating at least one of GitHub, PostHog, or Honeycomb natively.

This is **not** a full app design. It is the minimum surface that:
1. Lets a user (engineer or PM) start from a plan/intent ("we're shipping feature X")
2. Executes on it (code ships, or a ticket moves, or a flag flips — pick one concrete path)
3. Follows up on it (did it work? one connected signal from PostHog, GitHub, or Honeycomb closes the loop)

The goal is to give the engineer at G2 something concrete enough to challenge, spec, and estimate — not a complete design system.

## Open architectural question you must take a position on

**Does code editing happen inside Tidepool, or does Tidepool launch an external editor with deep-linking back?**

Two options:
- **A — Embedded editor.** Tidepool includes a code editing surface (e.g., Monaco editor). The user never leaves. Fully integrated execute phase. Higher build cost; stronger "replaces the IDE" claim.
- **B — Launched IDE with deep-linking.** Tidepool opens the user's existing editor (VS Code, Neovim, etc.) for the write-code step, with a link back to Tidepool when done. Lower build cost for v0; weaker integration but faster to ship. Code editing is delegated; Tidepool owns planning, context, and follow-up.

**Pick a default for v0 and explain your reasoning** in `specs/v0/spec.md`. The engineer at G2 may challenge your choice — that's expected and welcome.

## Required outputs

### `design/v0/` — flows and wireframes
Produce at least:
- `design/v0/flow-plan-execute-followup.md` — a Mermaid flow diagram showing the one concrete loop end-to-end (what screens/states exist, how the user moves between them, where the tool integrations surface)
- `design/v0/wireframes.md` — ASCII or Mermaid-based wireframes (or simple HTML if you prefer) for each key state in the loop: the plan view, the execute view, the follow-up view. Do not require Figma.

### `specs/v0/spec.md` — engineering spec
Must contain:
- **One-sentence product definition** (using Tidepool, consistent with ADR-0004/ADR-0005)
- **The concrete scenario** you designed for (e.g., "an engineer ships a feature flag rollout and confirms the PostHog funnel metric moved")
- **Your code-editing position** (option A or B, with rationale)
- **Screen inventory** — list every distinct screen/state in the v0 loop with a one-line description
- **Acceptance criteria** — bulleted, testable criteria the first engineering slice must satisfy to be considered "done"
- **Out of scope for v0** — explicit list of things that are real Tidepool features but not in this slice

## Acceptance
- [ ] `design/v0/flow-plan-execute-followup.md` exists with a complete Mermaid flow diagram
- [ ] `design/v0/wireframes.md` exists with wireframes for each key state
- [ ] `specs/v0/spec.md` exists with all required sections (product definition, scenario, code-editor position, screen inventory, acceptance criteria, out of scope)
- [ ] All documents use the name `Tidepool` (never `[PRODUCT NAME]`)
- [ ] Design is for the Plan→Execute→Follow-up loop, not an investigation-only tool
- [ ] PR opened against `main` with title `phase-2-design: designer output`

## Out of scope
- Do not design a full design system, color palette, or typography (v0 only)
- Do not design onboarding, auth, billing, or settings
- Do not design for more than one complete loop scenario
- Do not write any code
- Do not create any files outside `design/v0/` and `specs/v0/spec.md`
- Do not edit `ROADMAP.md`
- Do not invoke any further agents

## Git workflow
```
git clone https://github.com/BinaryBourbon/the-product.git
cd the-product
git checkout -b phase-2-design/designer
# read decisions/0004-product-framing-v2.md and decisions/0005-product-name.md
# write design/v0/flow-plan-execute-followup.md
# write design/v0/wireframes.md
# write specs/v0/spec.md
git add design/v0/ specs/v0/spec.md
git commit -m "phase-2-design: designer output"
git push -u origin phase-2-design/designer
# open PR against main via gh or GitHub MCP
```

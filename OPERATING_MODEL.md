# Operating Model

How a team of specialist AI agents builds one product together.

## The shape

One **tech-lead** agent owns the roadmap and dispatches specialists. Specialists never see each other's chats. All handoffs are files in this repo.

```
                    ┌────────────────┐
                    │   tech-lead    │
                    │  (orchestrator)│
                    └───────┬────────┘
                            │ dispatches via the bundled `aod` skill
            ┌───────────────┼───────────────┬──────────────┬─────────────┐
            ▼               ▼               ▼              ▼             ▼
   customer-researcher   designer       engineer      release-validator  ...
            │               │               │              │
            └───────────────┴──────┬────────┴──────────────┘
                                   ▼
                       writes artifacts to this repo
```

The tech-lead does not code, design, or research. Its job is decompose → dispatch → integrate → gate.

## The slice loop

Work is organized into **slices**. A slice is one user-visible capability that one engineer can ship in one conversation. Anything bigger is multiple slices.

Each slice goes through this sequence (some steps skip when not applicable):

1. **Discovery** — `customer-researcher` validates the pain. Output: `discovery/<slice>.md`.
2. **Narrative** — `growth-marketer` writes the press-release-style framing. Output: `narrative/<slice>.md`.
3. **Design** — `designer` produces flows + an initial spec. Output: `design/<slice>/` and `specs/<slice>/spec.md`.
4. **Build** — `general-purpose-engineer` implements against the spec. Output: a PR.
5. **Review** — `pr-reviewer` reviews the PR.
6. **Validate** — `release-validator` verifies post-merge / post-deploy.
7. **Measure** — `product-analyst` reads PostHog/Honeycomb after the slice has been live for ≥1 week. Output: `specs/<slice>/measurement.md`.

Steps 1–3 collapse into a single discovery cycle for the project's first slice (phase 0). Steps 6–7 don't apply until something is shipped.

## Gates

The tech-lead **must stop and ask the human** at these points. No exceptions.

| Gate | When | Question to the human |
| --- | --- | --- |
| **G0 — project framing** | Once, at the start | "Are we solving the right problem for the right user?" |
| **G1 — narrative** | After the press release is drafted | "Does this product, described this way, sound worth building?" |
| **G2 — architecture** | Before the first slice is built | "Are we aligned on the tech direction?" |
| **G3 — slice acceptance** | After every slice is validated | "Ship this slice live to users? Move to the next?" |

Between gates, the tech-lead operates without check-in. At a gate, it summarizes state in a comment on a PR (or in ROADMAP.md) and waits.

## How the tech-lead dispatches

When dispatching a specialist, the tech-lead writes a **brief** — a small markdown doc — and passes it as the prompt. The brief always has these sections:

```markdown
# Brief: <one-line task>

## Context
- Slice: <slice-name>
- Repo: https://github.com/BinaryBourbon/the-product (clone this; work in branch `<slice-name>/<your-role>`)
- Prior artifacts you should read: <links to repo files>

## Task
<one paragraph: what to do, scoped tight>

## Acceptance
- [ ] <output file path 1> exists and contains <what>
- [ ] <output file path 2> exists and contains <what>
- [ ] PR opened against `main` with title `<slice-name>: <role> output`

## Out of scope
<things the specialist should explicitly NOT do>
```

The specialist clones the repo, does the work in their branch, opens a PR, and ends their conversation. The tech-lead reads the PR (via the `github` MCP), integrates the output (merges, requests changes, or re-dispatches with revised brief), and updates ROADMAP.md.

One conversation = one brief. Specialists do not handle multiple briefs in one chat.

## The roadmap discipline

[ROADMAP.md](./ROADMAP.md) is the single source of truth for what's open. It must fit on one screen. If it doesn't, the tech-lead is managing too many slices in flight — kill or defer some.

Format:

```markdown
## Now (≤2 slices in flight)
- <slice-name> — <status>: <one-line>

## Next (≤3 queued)
- <slice-name> — <one-line description, why this is next>

## Gates pending human input
- <gate-id>: <question, link to artifact>

## Done (last 5)
- <slice-name> — shipped <date>
```

## Where decisions live

Anything that affects future slices is an **ADR** in `decisions/` — numbered, immutable once merged, superseded by later ADRs rather than edited. The tech-lead writes ADRs for: tech stack, deployment target, design system choice, naming, license, anything that closes off an option. Specialists can propose ADRs but the tech-lead merges them.

See [`decisions/TEMPLATE.md`](./decisions/TEMPLATE.md).

## Anti-patterns

These have wrecked autonomous-team experiments before. Do not do them.

- **Letting a specialist pick the next thing.** Specialists scope-creep. Only the tech-lead opens new slices.
- **Skipping the brief.** "Just go work on it" produces seven mutually inconsistent things. The brief forces specificity.
- **Long conversations.** A specialist conversation that runs > ~50 turns has lost the thread. Start fresh.
- **Skipping gates.** "I'm sure the human would approve" is how you wake up to fifty PRs of incoherent work.
- **Inventing customers.** If `discovery/` doesn't have evidence for a claim, the claim is a hypothesis.
- **Code before architecture.** No engineer slice runs before G2.

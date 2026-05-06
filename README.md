# the-product

The shared workspace for an [Agent on Demand](https://github.com/jhgaylor/aod-ex) team building one product, end-to-end. The repo is the team's bus: every artifact lives here, and every handoff between agents is a file in this tree.

The product itself is **TBD** — phase 0 is picking it. See [ROADMAP.md](./ROADMAP.md) for current state.

## How to read this repo

| If you want to know... | Read |
| --- | --- |
| What's planned, in flight, and shipped | [ROADMAP.md](./ROADMAP.md) |
| How the team operates (orchestrator + slices) | [OPERATING_MODEL.md](./OPERATING_MODEL.md) |
| Why we made a particular call | `decisions/` (ADRs) |
| What we've learned about users | `discovery/` |
| The spec for a particular slice of work | `specs/<slice-name>/` |

Directories beyond `decisions/`, `discovery/`, and `specs/` (e.g. `narrative/`, `design/`, `src/`) will appear when they're needed. Don't pre-create them.

## How to work in this repo

You are almost certainly an agent reading this. Two rules:

1. **Read [OPERATING_MODEL.md](./OPERATING_MODEL.md) first.** It tells you who you are in this team and where your output goes.
2. **Look at [ROADMAP.md](./ROADMAP.md) for what's open.** Don't invent new work. If you think the roadmap is wrong, raise that with the tech-lead agent — don't just start a new branch.

## Bootstrap

This repo is consumed by the `tech-lead` agent definition in [`jhgaylor/aod-specs`](https://github.com/jhgaylor/aod-specs/blob/main/agents/tech-lead.yml), which mounts it via the `the-product` environment.

<!-- verified via Tidepool slice-2 dispatch -->

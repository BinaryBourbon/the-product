# ADR-0008: v0 resolved open questions — multi-repo, AoD agent choice, PostHog spike

- **Status:** accepted
- **Date:** 2026-05-06
- **Decider:** human
- **Slice:** phase-4-build-slice-1

## Context

Three open questions from §9 of `specs/v0/architecture.md` required human decisions before the first engineering slice could begin. This ADR records all three resolutions.

---

## Resolution 1: GitHub repository scope — multi-repo from v0

**Decision:** The data model supports multiple GitHub repos from day one. `work_items.github_repo` (and `pull_requests.github_repo`) are present in the schema as specified in `specs/v0/architecture.md §4`.

**What is NOT in v0:** A repo-selector UI. The v0 dogfood deployment uses a single default repo. The `github_repo` field on the Create Work Item form is a free-text input that defaults to the operator's configured repo. Multi-repo UI surfaces in a later slice when needed.

**Implication for slice-1:** The schema must include `work_items.github_repo`. The create-work-item form must include a `github_repo` field (free-text, pre-filled from a default config). No repo picker, no GitHub API call to list repos.

---

## Resolution 2: AoD agent for v0 — reuse general-purpose-engineer

**Decision:** For v0, Tidepool dispatches the existing `general-purpose-engineer` agent (agent ID sourced from config, not hardcoded as a literal). The human's exact words:

> "users will ultimately need to define their own agents. for now use general-purpose-engineer."

**Long-term direction (record verbatim):** Users will ultimately define their own agents in Tidepool. This is a future product surface — the user-defines-agents UI is explicitly NOT a v0 feature. However, architecture decisions in v0 must not foreclose it.

**Implication for slice-1 and beyond:** The `agent_id` used in `POST $AOD_BASE_URL/api/conversations` must come from a configurable record — an environment variable (`AOD_AGENT_ID`) or a DB-backed config row — not hardcoded as a string literal in application code. When the user-defines-agents surface arrives, Tidepool replaces the env var / config value with a user-managed record; the dispatch call itself does not change.

**Environment variable for v0:**

```
AOD_AGENT_ID    # ID of the AoD agent to dispatch for coding tasks
                # v0 default: general-purpose-engineer's agent ID
                # future: replaced by user-managed agent config
```

---

## Resolution 3: PostHog funnel API — spike in parallel

**Decision:** The exact PostHog API call shape (endpoint, request body, response shape, flag-cohort filtering) is not yet verified. A research spike (`phase-4-spike-posthog-funnel-api`) is dispatched in parallel with slice-1. Its output will land in `research/posthog-funnel-api.md` in this repo before the Follow-up View build slice begins.

**Implication:** Slice-1 does NOT build the Follow-up View. The PostHog funnel client is deferred to a later slice, after the spike result is reviewed.

---

## Consequences

- **Enables:** Slice-1 can begin without further blockers. Multi-repo schema is settled; agent dispatch is configurable without being over-engineered; PostHog research runs in parallel.
- **Forecloses:** Hardcoding `agent_id` as a string literal in application source.
- **Accepted downside:** The `general-purpose-engineer` agent is broader than a coding-only agent and may exhibit scope-creep behaviors (writing docs, proposing refactors). The Tidepool engineer should scope the prompt tightly to mitigate this.

## Revisit when

- The user-defines-agents surface is scoped (opens the question of whether `AOD_AGENT_ID` env var or a DB-backed config row is the right primitive).
- The PostHog spike result arrives and contradicts the architecture's assumed API shape.

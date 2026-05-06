# Brief: PostHog funnel API — research spike

## Context
- Slice: phase-4-spike-posthog-funnel-api
- Bus repo: https://github.com/BinaryBourbon/the-product (clone this; work in branch `phase-4-spike-posthog/engineer`)
- Source-of-truth artifacts — read BOTH before researching:
  - `specs/v0/architecture.md` §5 (PostHog integration) — what Tidepool needs PostHog to do
  - `specs/v0/spec.md` (Follow-up phase acceptance criteria) — the exact data the UI needs

## This is a RESEARCH spike — no code, no Tidepool changes

You are NOT building anything. You are NOT touching the BinaryBourbon/tidepool repo. You are figuring out the exact PostHog API call shape so a future engineer can implement the Follow-up View without further research.

## Background

The architect identified (in `specs/v0/architecture.md §9`, spike #1) that the PostHog funnel API shape needs verification before the Follow-up View can be built. Specifically:

Tidepool's Follow-up View must display:
- A **before value**: the funnel conversion rate captured at plan time (when the user sets up the work item)
- An **after value**: the funnel conversion rate queried at follow-up time (24h+ after ship)
- A **numeric delta**: after − before, compared against the user's acceptance metric

The funnel is associated with a PostHog **feature flag** (e.g. `checkout-v2`). The before/after values should ideally be scoped to users who encountered the flag — i.e., the flag's rollout cohort — not the global funnel population.

## Research task

Using the PostHog API documentation (https://posthog.com/docs/api) and any other PostHog resources you can access, answer these questions precisely:

### 1. Funnel baseline at plan time

What is the exact API call to query a funnel conversion rate for a specific time window?
- Endpoint (full path)
- Required vs optional fields in the request body
- How to specify the funnel steps (event names — e.g. `checkout_page_loaded` → `checkout_payment_step_loaded`)
- How to set the time window (e.g., the 7 days before the work item was created)
- What the response looks like — key fields, where the conversion rate lives

### 2. Funnel after value at follow-up time

Same as above, but for the post-ship period (e.g., 24h after `shipped_at`). Are there any differences in how you query the "after" window vs the "before" window?

### 3. Filtering by feature flag cohort

The spec wants the funnel scoped to users who saw the feature flag. PostHog supports this via:
- **Feature flag filter on funnel**: does the Insights Funnel API support filtering by feature flag exposure? What is the exact filter structure?
- **Breakdown by feature flag**: alternatively, can you break down the funnel by flag value (`true`/`false`) and read the `true` cohort? What does the breakdown response look like?

### 4. Authentication

- What headers or params does the PostHog Insights API require?
- Is there any difference between the `POSTHOG_TOKEN` (project API key) and a personal API key for this endpoint?
- Any CORS or same-origin restrictions that would affect a server-side call from Next.js?

### 5. Rate limits

Does PostHog impose rate limits on Insights API calls? What are they? Any caching recommendations?

## Output

Write `research/posthog-funnel-api.md` in the BinaryBourbon/the-product bus repo. The document must contain:

1. **Baseline query** — exact endpoint, exact request body (with example values), exact relevant response fields, and the path to the conversion rate value
2. **After query** — same, for the post-ship time window; note any differences
3. **Flag cohort filter** — the exact filter or breakdown structure to scope the funnel to flag-exposed users; include an example response snippet showing where the cohort-scoped rate lives
4. **Auth** — headers/params required; note any difference between project key and personal key
5. **Rate limits** — numbers if available; any caching recommendation
6. **Open questions** — anything you couldn't resolve from docs (max 3 items); if the docs don't answer a question clearly, say so explicitly rather than guessing
7. **Sources** — URLs of every PostHog doc page or API reference you read

A Tidepool engineer reading this document must be able to write the PostHog API client without further research. If you are uncertain about something, say so explicitly — do not guess.

## Acceptance criteria

- [ ] `research/posthog-funnel-api.md` exists in `BinaryBourbon/the-product` on branch `phase-4-spike-posthog/engineer`
- [ ] Document covers all 5 sections above
- [ ] At least one complete example request body and response snippet are included
- [ ] Any uncertainty is explicitly flagged (not papered over)
- [ ] PR opened against `main` with title `phase-4-spike-posthog: funnel API research`

## Out of scope
- Do not write any code
- Do not touch BinaryBourbon/tidepool
- Do not implement any UI or API client
- Do not research PostHog's feature flag API (separate from funnels — Tidepool already knows how to enable flags via `PATCH /api/projects/{id}/feature_flags/{id}`)
- Do not research Honeycomb, GitHub, or any other tool

## Git workflow

```bash
git clone https://github.com/BinaryBourbon/the-product.git
cd the-product
git checkout -b phase-4-spike-posthog/engineer
mkdir -p research
# write research/posthog-funnel-api.md
git add research/posthog-funnel-api.md
git commit -m "phase-4-spike-posthog: PostHog funnel API research"
git push -u origin phase-4-spike-posthog/engineer
gh pr create --title "phase-4-spike-posthog: funnel API research" --body "..."
```

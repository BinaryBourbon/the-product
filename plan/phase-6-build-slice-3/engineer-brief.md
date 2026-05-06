# Slice-3 Engineer Brief — Follow-up Loop

**Phase:** phase-6-build-slice-3  
**Date:** 2026-05-06  
**Agent:** general-purpose-engineer (binarybourbon vault)  
**Target repo:** BinaryBourbon/tidepool (private)  
**Branch:** `slice-3/follow-up-loop` (branch from `main`)  
**PR target:** `main`

---

## Context

Tidepool is a Next.js 14 (App Router, TypeScript, Tailwind) web app that replaces the IDE for the plan/execute/follow-up shipping cycle. It is deployed on Render at **https://tidepool-web-77nd.onrender.com**.

Slices 1 and 2 are merged and deployed:
- **Slice-1 (foundation):** `work_items` + `agent_runs` tables, Prisma 5.22, basic CRUD UI, Render deploy pipeline.
- **Slice-2 (execute loop):** AoD dispatch (`POST /api/work-items/[id]/runs`), SSE stream proxy (`GET /api/runs/[id]/stream`) with `stream_log` persistence + Last-Event-ID replay, multi-turn follow-up (`POST /api/runs/[id]/prompts`), PR detection (polls GitHub, writes `agent_runs.github_pr_number`), `pull_requests` Prisma model.

This slice (slice-3) completes the v0 product by building the **follow-up loop**: the enable-flag action and the Follow-up View.

---

## Source of truth — read these before writing any code

All of the following files are in **BinaryBourbon/the-product** (public bus repo, main branch):

| File | What to extract |
|---|---|
| `decisions/0004-product-framing-v2.md` | Product definition: workspace replacing IDE; plan/execute/follow-up |
| `decisions/0006-execute-model.md` | Execute = prompt-driven; no IDE integration |
| `decisions/0007-agent-runtime.md` | AoD HTTP API patterns |
| `decisions/0008-v0-resolved-questions.md` | Multi-repo, agent_id config, PostHog spike resolved |
| `specs/v0/architecture.md` | §4 data model (follow_up_checks), §5 tool integrations (PostHog + Honeycomb), §8 state machine (`executing → shipped → done`) |
| `specs/v0/spec.md` | Screen 5 (Follow-up success), Screen 6 (Follow-up anomaly), all acceptance criteria for the Follow-up phase |
| `research/posthog-funnel-api.md` | **Authoritative PostHog API reference — do not re-research.** Exact request/response shapes for FunnelsQuery, flag cohort filter (Approach A), before/after date ranges, auth, rate limits. |

---

## Slice goal

Two parts, implemented together:

### Part A — Enable-flag action (in Execute View)

After a PR is merged (work_item state = `executing`, `pull_requests.state = merged`), show in the Execute View:

1. A **rollout % input** (integer 0–100, default 10).
2. An **"Enable feature flag" button**.

On submit:
1. **Capture baseline metric value** via PostHog FunnelsQuery (§1 of `research/posthog-funnel-api.md`). Use the work item's `feature_flag_name` to construct the funnel query. The date range for the baseline should be the 30 days *before* now (i.e., `date_from: -30d`, `date_to: shipped_at`). Compute `conversion_rate = results[last].count / results[0].count`. Write to `work_items.baseline_metric_value`.
2. **Enable the feature flag** via PostHog REST API: `PATCH /api/projects/{POSTHOG_PROJECT_ID}/feature_flags/{flag_key}/` with `{ "active": true, "rollout_percentage": <pct> }`. Note: PostHog's feature flag PATCH endpoint uses the flag key (string) in the URL, not a numeric ID. Look up the flag first via `GET /api/projects/{id}/feature_flags/?key=<flag_key>` to confirm existence; proceed with the PATCH.
3. **Transition state:** set `work_items.state = shipped`, `work_items.shipped_at = now()`, `work_items.posthog_rollout_pct = <pct>`.

API route: `POST /api/work-items/[id]/enable-flag` body `{ rolloutPct: number }`.

### Part B — Follow-up View (Screens 5 + 6 per spec.md)

Accessible from any work item in `shipped` or `done` state. Accessible on demand at any time (do not gate behind 24h timer — the spec says "on demand at any time").

On load, this view:
1. **Queries PostHog** for the after-value: same FunnelsQuery structure as the baseline but with `date_from: work_items.shipped_at`, `date_to: null`, `explicitDate: true`. Apply Approach A flag cohort filter (`properties: [{ type: "feature", key: work_items.feature_flag_name, operator: "exact", value: "true" }]`). Compute after conversion rate.
2. **Queries Honeycomb** for the post-ship window: `POST /1/queries/{HONEYCOMB_DATASET}` to get error rate and p99 latency since `shipped_at`. See §5 of `specs/v0/architecture.md` for auth (`X-Honeycomb-Team` header) and the polling pattern (`GET /1/query_results/{id}`).
3. **Persists a `follow_up_checks` row** with all fields per §4 of `specs/v0/architecture.md`: `posthog_after_value`, `posthog_delta` (after − baseline), `metric_passed` (parse the acceptance_metric string — treat it as a human-readable target, do a best-effort parse; if unparseable, default `metric_passed = null` and show raw delta), `honeycomb_error_rate_ok`, `honeycomb_latency_ok`, `anomaly_detected`, `raw_posthog_response`, `raw_honeycomb_response`.
4. **Serves cached results** on re-load within 5 minutes: if the most recent `follow_up_checks` row for this work item is < 5 minutes old, return it from the DB without hitting PostHog/Honeycomb again.

API route: `GET /api/work-items/[id]/follow-up` — fetches or computes the follow-up check, returns the data for the view.

#### Screen 5 — Success path (metric_passed = true, no anomaly)

Show:
- **Funnel widget:** before value (from `work_items.baseline_metric_value`), after value (`follow_up_checks.posthog_after_value`), delta (+N.Npp), acceptance metric string, pass/fail badge (✓/✗).
- **What shipped panel:** `pull_requests.github_pr_number` (linked to GitHub), `pull_requests.author_handle`, `pull_requests.merged_at`, `pull_requests.additions` / `pull_requests.deletions`.
- **Signals panel:** error rate OK / not OK, p99 latency OK / not OK. "No anomalies detected" if both are OK.
- **CTAs:** "Mark as Done" button, "Increase rollout" link (opens PostHog in new tab — no need to build rollout UI).

#### Screen 6 — Anomaly path (metric_passed = false OR anomaly_detected = true)

Show everything from Screen 5, plus:
- **Anomaly panel:** "Metric did not meet target" or "Signals regression detected."
- **Graph traversal pre-populated hop:** "PostHog event → this PR." Display the feature flag name (PostHog side) linked to the PR number (GitHub side). This is the first hop of the investigation path — assembled from `work_items.feature_flag_name` + `pull_requests.github_pr_number`. No graph database needed; this is a static join on `work_item_id`.
- **CTAs:** "Create follow-on work item" (creates a new work item pre-populated with title "Follow-up: <original title>", links to the original), "Roll back flag" (calls `PATCH /api/projects/{id}/feature_flags/{flag_key}/` with `{ "active": false }`).

#### Mark as Done

`POST /api/work-items/[id]/done` — sets `work_items.state = done`. After this, the work item shows as Done in the list view (update the list page filter/badge as needed).

---

## Database migrations needed

Add the `follow_up_checks` model to `prisma/schema.prisma`:

```prisma
model FollowUpCheck {
  id                    String    @id @default(uuid())
  workItemId            String    @map("work_item_id")
  checkedAt             DateTime  @default(now()) @map("checked_at")
  posthogAfterValue     Float?    @map("posthog_after_value")
  posthogDelta          Float?    @map("posthog_delta")
  metricPassed          Boolean?  @map("metric_passed")
  honeycombErrorRateOk  Boolean?  @map("honeycomb_error_rate_ok")
  honeycombLatencyOk    Boolean?  @map("honeycomb_latency_ok")
  anomalyDetected       Boolean   @default(false) @map("anomaly_detected")
  rawPosthogResponse    Json?     @map("raw_posthog_response")
  rawHoneycombResponse  Json?     @map("raw_honeycomb_response")
  workItem              WorkItem  @relation(fields: [workItemId], references: [id])

  @@map("follow_up_checks")
}
```

Add `followUpChecks FollowUpCheck[]` to the `WorkItem` model.

Run `npx prisma migrate dev --name follow-up-checks` to generate the migration.

---

## Render env vars to add

Add to `render.yaml` under `envVars` for `tidepool-web`:

```yaml
- key: POSTHOG_PROJECT_ID
  value: ""        # not a secret; fill in the actual project ID in Render dashboard
- key: HONEYCOMB_DATASET
  value: ""        # not a secret; fill in the dataset name in Render dashboard
```

**Verify before adding:** `POSTHOG_TOKEN` and `HONEYCOMB_KEY` are already in `render.yaml` from slice-1. Do NOT add them again. `AOD_VAULT_ID` was added in slice-2. Check the current `render.yaml` and only add what is missing.

---

## PostHog API reference (from research/posthog-funnel-api.md — use verbatim)

### Baseline query (before ship)

```
POST https://us.posthog.com/api/projects/{POSTHOG_PROJECT_ID}/query/
Authorization: Bearer {POSTHOG_TOKEN}
Content-Type: application/json
```

```json
{
  "query": {
    "kind": "FunnelsQuery",
    "series": [
      { "kind": "EventsNode", "event": "<step-1-event>" },
      { "kind": "EventsNode", "event": "<step-2-event>" }
    ],
    "dateRange": { "date_from": "-30d", "date_to": null },
    "filterTestAccounts": true
  }
}
```

Conversion rate: `results[results.length - 1].count / results[0].count`

### After query (post-ship, flag cohort, Approach A)

```json
{
  "query": {
    "kind": "FunnelsQuery",
    "series": [
      { "kind": "EventsNode", "event": "<step-1-event>" },
      { "kind": "EventsNode", "event": "<step-2-event>" }
    ],
    "dateRange": {
      "date_from": "<shipped_at ISO 8601>",
      "date_to": null,
      "explicitDate": true
    },
    "properties": [
      {
        "type": "feature",
        "key": "<feature_flag_name>",
        "operator": "exact",
        "value": "true"
      }
    ],
    "filterTestAccounts": true
  }
}
```

**Note on funnel series:** The `work_items.acceptance_metric` is a free-text string (e.g. "conversion +5%"). Tidepool does not know the PostHog event names for the funnel steps — these are not stored in the data model. For v0: use the `feature_flag_name` as a proxy. Construct a two-step funnel where both steps are the same event (`$pageview` is a safe generic default) scoped to the flag cohort. The delta signal will be approximate but valid for the anomaly/pass-fail determination. Document this limitation in the PR description. The exact funnel event names are a future configuration surface.

### Feature flag enable

```
PATCH https://us.posthog.com/api/projects/{POSTHOG_PROJECT_ID}/feature_flags/{flag_id}/
Authorization: Bearer {POSTHOG_TOKEN}
Content-Type: application/json

{ "active": true, "rollout_percentage": <pct> }
```

To get `{flag_id}` (numeric): `GET /api/projects/{POSTHOG_PROJECT_ID}/feature_flags/?key=<flag_key>` → take `results[0].id`.

---

## Honeycomb API reference (from architecture.md §5)

```
POST https://api.honeycomb.io/1/queries/{HONEYCOMB_DATASET}
X-Honeycomb-Team: {HONEYCOMB_KEY}
Content-Type: application/json
```

Query body for error rate since ship:
```json
{
  "time_range": <seconds since shipped_at>,
  "calculations": [
    { "op": "COUNT" },
    { "op": "RATE", "column": "error" }
  ],
  "filters": [],
  "granularity": 0
}
```

Query body for p99 latency since ship:
```json
{
  "time_range": <seconds since shipped_at>,
  "calculations": [
    { "op": "P99", "column": "duration_ms" }
  ],
  "granularity": 0
}
```

Poll for results: `GET https://api.honeycomb.io/1/query_results/{query_id}` with the same `X-Honeycomb-Team` header. Poll until `complete: true`.

**Thresholds (defensible defaults):** error rate > 1% → `honeycomb_error_rate_ok = false`. p99 > 2000ms → `honeycomb_latency_ok = false`. Document these thresholds in the PR description.

**If Honeycomb API is unavailable or returns empty results** (no data for the dataset): set both `honeycomb_error_rate_ok = true`, `honeycomb_latency_ok = true`, `raw_honeycomb_response = null`, and display "No Honeycomb data available" in the signals panel. Do NOT block the Follow-up View on Honeycomb availability.

---

## Acceptance criteria (testable end-to-end)

- [ ] From a work item in `executing` state with a merged PR, the enable-flag UI is visible in the Execute View
- [ ] Submitting rollout % calls PostHog, writes `baseline_metric_value` to the DB, enables the flag, transitions state to `shipped`, sets `shipped_at`
- [ ] The Follow-up View is accessible from a `shipped` work item on demand
- [ ] Follow-up View renders before/after funnel widget with real PostHog numbers (or placeholder "no data" if PostHog token is not set in Render — document which env vars are needed)
- [ ] Follow-up View renders Honeycomb signals panel (or "No Honeycomb data available" if key is not set)
- [ ] `follow_up_checks` row is persisted per view load; re-load within 5 minutes serves cached row
- [ ] Anomaly test: create a work item with `acceptance_metric = "conversion +50%"` and observe actual delta ~0%; verify Screen 6 renders with the anomaly panel and the pre-populated PostHog event → GitHub PR hop
- [ ] "Mark as Done" button transitions state to `done`; work item shows Done badge in list view
- [ ] All state transitions persist across browser refresh

---

## Git workflow

1. Branch from `main` of `BinaryBourbon/tidepool`: `git checkout -b slice-3/follow-up-loop`
2. Read existing code before modifying. Do not rewrite what slice-1/2 built.
3. Add `follow_up_checks` Prisma model and run migration.
4. Add/verify env vars in `render.yaml` (POSTHOG_PROJECT_ID, HONEYCOMB_DATASET — not secrets).
5. Implement API routes first, then UI components.
6. Commit in logical units (migration, API routes, UI).
7. Open PR against `main`. **PR description must include:**
   - Deployed URL (https://tidepool-web-77nd.onrender.com) and confirmation it is live post-deploy
   - Text proof (or screenshot) of: enable-flag transition working, Follow-up View rendering with data, anomaly path rendering when triggered, Mark as Done transition
   - Which Render env vars need to be set in the dashboard to make PostHog + Honeycomb live (list them explicitly)
   - Any defensible defaults chosen (funnel events, Honeycomb thresholds, caching window)
   - Anything not implemented and why (if any)

**Execute through to deployed URL + PR. Pause only on truly external choices the brief cannot anticipate. Pick defensible defaults and document them.**

# PostHog Funnel API Research

**Spike:** Phase-4 — verify PostHog funnel API shape before Follow-up View build  
**Addresses:** `specs/v0/architecture.md §9` open question #1  
**Date:** 2026-05-06

---

## 1. Baseline Query

### Endpoint

```
POST https://us.posthog.com/api/projects/{project_id}/query/
```

- Use `https://eu.posthog.com` for EU Cloud. For self-hosted, substitute your instance domain.
- `project_id` is the numeric project ID found in **PostHog → Project Settings**.

### Authentication

```
Authorization: Bearer {POSTHOG_PERSONAL_API_KEY}
Content-Type: application/json
```

A **personal API key** (`phx_…` prefix) with the **Query Read** permission is required. See §4 (Auth) for details on key types.

### Request body (baseline — before ship)

```json
{
  "query": {
    "kind": "FunnelsQuery",
    "series": [
      {
        "kind": "EventsNode",
        "event": "checkout_page_loaded",
        "name": "Checkout page loaded"
      },
      {
        "kind": "EventsNode",
        "event": "checkout_payment_step_loaded",
        "name": "Payment step loaded"
      }
    ],
    "dateRange": {
      "date_from": "2026-04-01",
      "date_to": "2026-04-30"
    },
    "filterTestAccounts": true
  }
}
```

**Required fields:**
| Field | Type | Notes |
|---|---|---|
| `query.kind` | string | Must be `"FunnelsQuery"` |
| `query.series` | array | Ordered list of funnel steps; minimum 2 items |
| `query.series[n].kind` | string | `"EventsNode"` for named events |
| `query.series[n].event` | string | The PostHog event name for this step |

**Optional but commonly used:**
| Field | Type | Default | Notes |
|---|---|---|---|
| `query.dateRange.date_from` | string | `-7d` | ISO 8601 (`2026-04-01`) or relative (`-30d`, `-1m`, `-1mStart`) |
| `query.dateRange.date_to` | string | now | ISO 8601 or relative; omit/null for "now" |
| `query.dateRange.explicitDate` | bool | false | When true, disables rounding to period boundaries |
| `query.filterTestAccounts` | bool | false | Excludes internal/test users |
| `query.funnelsFilter.funnelWindowInterval` | int | 14 | Conversion window size |
| `query.funnelsFilter.funnelWindowIntervalUnit` | string | `"day"` | `"minute"`, `"hour"`, `"day"`, `"week"`, `"month"` |
| `query.funnelsFilter.funnelOrderType` | string | `"ordered"` | `"ordered"`, `"strict"`, `"unordered"` |
| `query.samplingFactor` | float | null | e.g. `0.1` for 10% sampling |

### Response structure

```json
{
  "results": [
    {
      "action_id": "checkout_page_loaded",
      "name": "checkout_page_loaded",
      "custom_name": "Checkout page loaded",
      "order": 0,
      "people": [],
      "count": 1842,
      "type": "events",
      "average_conversion_time": null,
      "median_conversion_time": null
    },
    {
      "action_id": "checkout_payment_step_loaded",
      "name": "checkout_payment_step_loaded",
      "custom_name": "Payment step loaded",
      "order": 1,
      "people": [],
      "count": 1204,
      "type": "events",
      "average_conversion_time": 47.3,
      "median_conversion_time": 32.1
    }
  ],
  "is_cached": false,
  "timings": [...],
  "hogql": "...",
  "query_status": null
}
```

**Where the conversion rate lives:**

The API does **not** return a `conversion_rate` field. You compute it from the `count` fields:

```
conversion_rate = results[last_step].count / results[0].count
```

For the two-step checkout funnel above: `1204 / 1842 = 0.6536` → **65.4% conversion rate**.

For multi-step funnels, each step's rate relative to the first step is `results[n].count / results[0].count`.

`average_conversion_time` and `median_conversion_time` are in seconds. They are `null` for step 0 (the entry step).

---

## 2. After Query (Post-Ship)

The after-ship query is **identical in structure** to the baseline query. The only difference is the `dateRange`:

```json
{
  "query": {
    "kind": "FunnelsQuery",
    "series": [
      {
        "kind": "EventsNode",
        "event": "checkout_page_loaded",
        "name": "Checkout page loaded"
      },
      {
        "kind": "EventsNode",
        "event": "checkout_payment_step_loaded",
        "name": "Payment step loaded"
      }
    ],
    "dateRange": {
      "date_from": "2026-05-01T00:00:00Z",
      "date_to": "2026-05-06T23:59:59Z"
    },
    "filterTestAccounts": true
  }
}
```

**Key differences from baseline:**
- `date_from` is set to `work_items.shipped_at` (the timestamp when the flag was enabled).
- `date_to` is "now" or a specific end point for the measurement window (e.g., 24 hours after `shipped_at`).
- Using ISO 8601 with `explicitDate: true` prevents PostHog from rounding to day/week boundaries, which matters when `shipped_at` is mid-day.

**Recommended body for after query:**

```json
{
  "query": {
    "kind": "FunnelsQuery",
    "series": [ ... ],
    "dateRange": {
      "date_from": "<shipped_at ISO 8601>",
      "date_to": null,
      "explicitDate": true
    },
    "filterTestAccounts": true
  }
}
```

Setting `date_to: null` means "through now." Setting `explicitDate: true` prevents the start from rounding back to midnight.

The response shape and the conversion rate calculation (`results[last].count / results[0].count`) are identical to the baseline.

---

## 3. Flag Cohort Filter

There are two distinct approaches depending on the goal:

### Approach A — Filter the entire funnel to flag-exposed users (recommended for Tidepool)

Apply a `properties` filter on the top-level `FunnelsQuery` using a `FeaturePropertyFilter`. This restricts the funnel population to users who received the feature flag (i.e., users where `$feature/checkout-v2` was set to `true`).

```json
{
  "query": {
    "kind": "FunnelsQuery",
    "series": [
      { "kind": "EventsNode", "event": "checkout_page_loaded" },
      { "kind": "EventsNode", "event": "checkout_payment_step_loaded" }
    ],
    "dateRange": {
      "date_from": "<shipped_at>",
      "date_to": null,
      "explicitDate": true
    },
    "properties": [
      {
        "type": "feature",
        "key": "checkout-v2",
        "operator": "exact",
        "value": "true"
      }
    ],
    "filterTestAccounts": true
  }
}
```

The `type: "feature"` filter corresponds to the `FeaturePropertyFilter` schema in PostHog's schema. Per the source: `type: "feature"` is an *event property with `$feature/` prepended* — meaning it matches events where `$feature/checkout-v2 == "true"` was set on the event at capture time. This requires that the PostHog SDK or server-side capture sends `$feature/checkout-v2` as an event property when the flag is active (which PostHog SDKs do automatically when `send_feature_flags: true` is set, or when feature flags are explicitly included at capture time).

**Response for this approach:** Same flat `results` array as §1. The funnel population is automatically scoped to flag-exposed users.

```json
{
  "results": [
    { "action_id": "checkout_page_loaded", "count": 412, "order": 0 },
    { "action_id": "checkout_payment_step_loaded", "count": 289, "order": 1 }
  ]
}
```

Conversion rate: `289 / 412 = 0.7013` → 70.1%.

### Approach B — Breakdown by flag value (true vs false)

Use `breakdownFilter` with `breakdown_type: "event"` and `breakdown: "$feature/checkout-v2"` to split the funnel by flag variant. This returns a list of per-breakdown results.

```json
{
  "query": {
    "kind": "FunnelsQuery",
    "series": [
      { "kind": "EventsNode", "event": "checkout_page_loaded" },
      { "kind": "EventsNode", "event": "checkout_payment_step_loaded" }
    ],
    "dateRange": {
      "date_from": "<shipped_at>",
      "date_to": null,
      "explicitDate": true
    },
    "breakdownFilter": {
      "breakdown": "$feature/checkout-v2",
      "breakdown_type": "event"
    },
    "filterTestAccounts": true
  }
}
```

**Breakdown response shape:**

When a `breakdownFilter` is specified, `results` becomes a **list of lists** — one inner list per breakdown value:

```json
{
  "results": [
    [
      {
        "action_id": "checkout_page_loaded",
        "count": 412,
        "order": 0,
        "breakdown": "true",
        "breakdown_value": "true",
        "average_conversion_time": null,
        "median_conversion_time": null
      },
      {
        "action_id": "checkout_payment_step_loaded",
        "count": 289,
        "order": 1,
        "breakdown": "true",
        "breakdown_value": "true",
        "average_conversion_time": 44.1,
        "median_conversion_time": 30.5
      }
    ],
    [
      {
        "action_id": "checkout_page_loaded",
        "count": 1430,
        "order": 0,
        "breakdown": "false",
        "breakdown_value": "false",
        "average_conversion_time": null,
        "median_conversion_time": null
      },
      {
        "action_id": "checkout_payment_step_loaded",
        "count": 915,
        "order": 1,
        "breakdown": "false",
        "breakdown_value": "false",
        "average_conversion_time": 51.2,
        "median_conversion_time": 38.7
      }
    ]
  ]
}
```

**To extract the `true` cohort conversion rate:**

```typescript
const flagOnGroup = results.find(group => group[0]?.breakdown_value === "true");
const conversionRate = flagOnGroup
  ? flagOnGroup[flagOnGroup.length - 1].count / flagOnGroup[0].count
  : null;
```

**Recommendation for Tidepool:** Use **Approach A** (properties filter) for the after-value displayed in the Follow-up View. It returns a single conversion rate scoped to the flag-on population, which is the relevant metric. Approach B is useful if you want a side-by-side control/treatment comparison.

---

## 4. Auth

### Key types

| Key type | Prefix | Used for | How to pass |
|---|---|---|---|
| **Personal API key** | `phx_` | All private REST API calls (GET/POST/PATCH/DELETE); required for `/query/` endpoint | `Authorization: Bearer phx_...` header |
| **Project API key (project token)** | `phk_` | Public capture endpoints only (`/i/v0/e`, `/flags`); NOT for insights queries | `api_key` in request body or `token` query param |

**For Tidepool's server-side Next.js calls, use the personal API key** (`POSTHOG_TOKEN` env var should hold a personal API key). The project token is for event capture only and will not work with the `/query/` endpoint.

### Headers required

```
Authorization: Bearer {POSTHOG_TOKEN}
Content-Type: application/json
```

No additional headers are needed for server-side calls.

### CORS

The `/query/` endpoint is a private server-side endpoint. Since Tidepool calls it from a Next.js API route (server-side), CORS is not a concern — CORS restrictions only apply to browser-originated requests. There are no documented CORS restrictions for server-to-server calls to `https://us.posthog.com`.

### Self-hosted vs Cloud

Replace `https://us.posthog.com` with the self-hosted instance URL if applicable. The path structure is identical.

### Key security notes

- Personal API keys (`phx_`) are **automatically rolled** if GitHub detects them in a public repository.
- Store `POSTHOG_TOKEN` as a Render secret (marked `sync:false`) — never commit the value.

---

## 5. Rate Limits

The `/query/` endpoint has its own rate limit bucket, separate from analytics endpoints:

| Limit type | Limit |
|---|---|
| Requests per hour (`/query/`) | **2,400 / hour** |
| Requests per minute (`/query/`) | **240 / minute** |
| Concurrent queries | **3** |
| Maximum execution time | **10 seconds** |
| Analytics endpoints (e.g., older `/insights/` routes) | 240 / min, 1,200 / hour |
| CRUD endpoints | 480 / min, 4,800 / hour |

Limits are **team-wide** — they apply across all users and all personal API keys within the same PostHog organization.

### Caching recommendations

1. **Store the baseline value once.** The baseline metric is captured at plan time and written to `work_items.baseline_metric_value`. Never re-query PostHog for the baseline; read from the database.

2. **Cache the after-value per `follow_up_checks` row.** The `follow_up_checks.posthog_after_value` and `follow_up_checks.raw_posthog_response` columns snapshot the result at each view load. On repeated loads within a short window (e.g., page refresh), serve the most recent `follow_up_checks` row instead of hitting PostHog again.

3. **PostHog's own query cache.** The API response includes `"is_cached": true/false`. PostHog internally caches repeated identical queries, so identical requests within a short window will return a cached result and not count fully against rate limits. The `next_allowed_client_refresh` field in cached responses indicates when a fresh result will be available.

4. **Rate limit headroom.** At one PostHog call per Follow-up View load per work item, Tidepool is extremely unlikely to approach 2,400/hour under any realistic v0 load.

---

## 6. Open Questions

1. **`$feature/` property on events vs. person properties.** The `type: "feature"` filter targets event-level properties set at capture time (`$feature/flag-key`). This requires that PostHog SDKs send `send_feature_flags: true` or that the application explicitly sets `$feature/checkout-v2` on each event. If events were captured without this property, the filter will return zero results. The behavior of the filter when the property is absent from an event is not explicitly documented in the PostHog API docs — it is inferred from the schema source code description ("event property with `$feature/` prepended"). **Verify that the production PostHog project is capturing `$feature/checkout-v2` on checkout events before building the after-value query.**

2. **`results` shape for the non-breakdown case.** The `FunnelsQueryResponse.results` field is typed as `Any` in the PostHog schema (no typed Pydantic model for the individual step objects). The step object shape (`action_id`, `name`, `order`, `count`, `type`, `average_conversion_time`, `median_conversion_time`) is confirmed from source code (`_serialize_step` in `posthog/hogql_queries/insights/funnels/base.py`), but no official JSON schema or API reference documents this response shape. It is possible it changes across PostHog versions. **Pin the PostHog Cloud version or add a response validation layer.**

3. **Project ID retrieval.** The `/query/` endpoint requires a numeric `project_id` in the URL. The architecture spec stores `POSTHOG_TOKEN` but not `POSTHOG_PROJECT_ID`. The project ID is obtainable via `GET https://us.posthog.com/api/projects/` (authenticated with the personal API key), but this needs to become a required env var or a startup-time lookup. **Confirm whether `POSTHOG_PROJECT_ID` should be added to the Render environment variable list alongside `POSTHOG_TOKEN`.**

---

## 7. Sources

| URL | What was extracted |
|---|---|
| `https://raw.githubusercontent.com/posthog/posthog.com/master/contents/docs/api/index.mdx` | Authentication types, base URLs, rate limits, pagination, CORS, key prefixes |
| `https://raw.githubusercontent.com/posthog/posthog.com/master/contents/docs/api/queries.mdx` | `/query/` endpoint path, method, auth requirements, supported `kind` values, rate limits, async mode |
| `https://raw.githubusercontent.com/PostHog/posthog/master/posthog/schema.py` | `FunnelsQuery`, `FunnelsFilter`, `FunnelsQueryResponse`, `EventsNode`, `DateRange`, `BreakdownFilter`, `BreakdownType`, `FeaturePropertyFilter`, `FlagPropertyFilter`, `PersonPropertyFilter`, `EventPropertyFilter`, `PropertyOperator` class definitions |
| `https://raw.githubusercontent.com/PostHog/posthog/master/posthog/hogql_queries/insights/funnels/base.py` | `_serialize_step` method — confirms exact field names in step result objects (`action_id`, `name`, `custom_name`, `order`, `people`, `count`, `type`, `average_conversion_time`, `median_conversion_time`) |
| `https://raw.githubusercontent.com/PostHog/posthog/master/posthog/hogql_queries/insights/funnels/funnel.py` | `_format_single_funnel` — confirms that breakdown results are a list-of-lists and shows `breakdown`/`breakdown_value` fields |
| `https://raw.githubusercontent.com/PostHog/posthog/master/posthog/hogql_queries/insights/funnels/test/breakdown_cases.py` | Test assertions confirming breakdown response structure with `breakdown` and `breakdown_value` fields |
| `https://posthog.com/docs/api/flags` (via context7) | Confirms `$feature/flag-key` event property pattern for feature flag attribution |

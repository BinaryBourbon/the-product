# Tidepool v0 — Plan → Execute → Follow-up Flow

## Scenario

An engineer and PM align on shipping a checkout-page redesign behind a feature flag. The engineer describes the change as a prompt; a coding agent makes the change and opens a PR automatically. The engineer reviews and merges the PR inside Tidepool, then enables a PostHog flag at 10% rollout — all without leaving Tidepool. Twenty-four hours later, both check Tidepool: the checkout funnel conversion metric moved. The loop closes.

*(VS Code deep-linking was proposed in the original design and rejected at G2. See ADR-0006: execute is prompt-driven; no IDE integration of any kind.)*

## Flow diagram

```mermaid
flowchart TD
    START([User opens Tidepool]) --> HOME

    subgraph PLAN["Plan phase"]
        HOME[Work Item View\nCheckout redesign · flag: checkout-v2]
        HOME --> CTX[Context panel auto-loads\nRelated: PR #41 · PostHog funnel baseline\nPast decision: ADR on checkout flow]
        CTX --> METRIC[PM sets acceptance metric\n'checkout funnel conversion +5pp'\nstored on work item]
        METRIC --> READY{Context complete?}
        READY -- missing context --> TRAVERSE[Graph traversal\nPostHog event → GitHub commit\n→ Honeycomb trace]
        TRAVERSE --> READY
        READY -- yes --> EXECUTE_START[Click 'Start executing']
    end

    subgraph EXECUTE["Execute phase"]
        EXECUTE_START --> EX_VIEW[Execute View\nPrompt input + agent run surface]

        EX_VIEW --> STEP1[Step 1: Branch created\nTidepool calls GitHub API\nbranch: feat/checkout-v2]
        STEP1 --> STEP2[Step 2: Describe the change\nUser writes prompt in Tidepool\ne.g. 'Redesign checkout page per spec']
        STEP2 --> AGENT[Coding agent runs\nlive streamed progress\nfiles changed · tests · errors]
        AGENT --> PR_OPEN[PR opens automatically\nPR #47 — agent-authored\nappears in Tidepool PR-review surface]
        PR_OPEN --> REVIEW{Review outcome}
        REVIEW -- iterate: follow-up prompt --> STEP2
        REVIEW -- discard --> STEP2
        REVIEW -- approve + merge --> STEP4[Step 4: Enable feature flag\nTidepool shows PostHog flag: checkout-v2\nUser sets rollout → 10%\nTidepool calls PostHog API]
        STEP4 --> SHIPPED[Work item state → Shipped\nTimestamp recorded]
    end

    subgraph FOLLOWUP["Follow-up phase"]
        SHIPPED --> FU_VIEW[Follow-up View\n24h after ship]
        FU_VIEW --> METRIC_WIDGET[PostHog funnel widget\ncheckout conversion: before vs. after\nDelta: +6.2pp ✓]
        METRIC_WIDGET --> SIGNAL{Signal clear?}
        SIGNAL -- yes → metric moved --> CLOSED[Work item → Done\n'Shipped and confirmed']
        SIGNAL -- anomaly / regression --> GRAPH_HOP[Graph traversal surfaced\nPostHog event → GitHub PR #47\n→ Honeycomb trace for affected users]
        GRAPH_HOP --> DIAGNOSE[Engineer inspects trace\nfinds slow query introduced]
        DIAGNOSE --> NEW_ITEM[New work item created:\n'Fix slow query in checkout']
        NEW_ITEM --> HOME
    end

    CLOSED --> HOME
```

## States and transitions summary

| State | Phase | Entry | Exit |
|---|---|---|---|
| Work Item View | Plan | Open work item | Context complete → Start executing |
| Context Panel | Plan | Auto-load on open; manual traversal | Dismissed or fully resolved |
| Graph Traversal | Plan/Follow-up | Missing context or anomaly detected | Finding pinned to work item |
| Execute View | Execute | Start executing | PR merged + flag enabled → Shipped |
| Follow-up View | Follow-up | 24h after Shipped; or manually triggered | Metric confirmed → Done, or Anomaly → new work item |

## Integration touchpoints

| Tool | Phase | Action |
|---|---|---|
| GitHub | Execute | Create branch; coding agent opens PR; user merges via Tidepool |
| Coding agent | Execute | Receives prompt; writes code; opens PR; accepts follow-up prompts |
| PostHog | Execute | Enable/configure feature flag; set rollout % |
| PostHog | Follow-up | Pull funnel metric; before/after delta |
| Honeycomb | Follow-up (anomaly path) | Pull trace for affected cohort |

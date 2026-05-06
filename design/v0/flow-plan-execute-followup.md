# Tidepool v0 — Plan → Execute → Follow-up Flow

## Scenario

An engineer and PM align on shipping a checkout-page redesign behind a feature flag. The engineer writes the code (launching VS Code via deep-link), opens a PR, merges it, and enables a PostHog flag at 10% rollout. Twenty-four hours later, both check Tidepool: the checkout funnel conversion metric moved. The loop closes.

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
        EXECUTE_START --> EX_VIEW[Execute View\nChecklist of actions for this work item]

        EX_VIEW --> STEP1[Step 1: Create branch\nTidepool calls GitHub API\nbranch: feat/checkout-v2]
        STEP1 --> STEP2[Step 2: Write code\nClick 'Open in VS Code'\ndeep-link: vscode://file/...?tidepool_return=...]
        STEP2 --> IDE[VS Code opens\nUser writes & commits code\nexternal to Tidepool]
        IDE --> RETURN[User clicks 'Return to Tidepool'\nfrom VS Code extension panel]
        RETURN --> STEP3[Step 3: PR status\nTidepool polls GitHub\nPR #47 open → merged ✓]
        STEP3 --> STEP4[Step 4: Enable feature flag\nTidepool shows PostHog flag: checkout-v2\nUser sets rollout → 10%\nTidepool calls PostHog API]
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
| Execute View | Execute | Start executing | All checklist steps complete |
| VS Code (external) | Execute | Deep-link from Tidepool | Return deep-link → back to Tidepool |
| Follow-up View | Follow-up | 24h after Shipped; or manually triggered | Metric confirmed → Done, or Anomaly → new work item |

## Integration touchpoints

| Tool | Phase | Action |
|---|---|---|
| GitHub | Execute | Create branch; poll PR status; detect merge |
| PostHog | Execute | Enable/configure feature flag; set rollout % |
| PostHog | Follow-up | Pull funnel metric; before/after delta |
| Honeycomb | Follow-up (anomaly path) | Pull trace for affected cohort |

# Tidepool v0 — Wireframes

Three key states in the Plan → Execute → Follow-up loop. ASCII wireframes; no Figma required.

---

## Screen 1 — Plan View (Work Item)

The first thing a user sees when opening a work item. Scoping, context, and acceptance metric are set here before execution begins.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Tidepool                                    [Search]  [Engineer ▾]           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ← Back to workspace                                                           │
│                                                                                │
│  ┌──────────────────────────────────────────────────────┐  ┌────────────────┐ │
│  │ Checkout redesign                                    │  │ CONTEXT        │ │
│  │ Feature flag: checkout-v2                           │  │────────────────│ │
│  │ Owner: @ana  ·  PM: @rob                            │  │ GitHub         │ │
│  │                                                      │  │ • PR #41 merged│ │
│  │ What we're building                                  │  │   "old checkout│ │
│  │ ───────────────────────────────────────────          │  │    cleanup"    │ │
│  │ Redesign the checkout page to reduce drop-off        │  │ • 3 related    │ │
│  │ at the payment step. Ship behind flag                │  │   commits      │ │
│  │ checkout-v2, roll out at 10%.                        │  │                │ │
│  │                                                      │  │ PostHog        │ │
│  │ Acceptance metric                                    │  │ • Funnel:      │ │
│  │ ───────────────────────────────────────────          │  │   checkout →   │ │
│  │ [checkout funnel conversion +5pp              ✎ ]   │  │   payment      │ │
│  │                                                      │  │   52% baseline │ │
│  │                                                      │  │                │ │
│  │                                                      │  │ Past decisions │ │
│  │                                                      │  │ • ADR: payment │ │
│  │                                                      │  │   UX principles│ │
│  │                                                      │  │                │ │
│  │                                                      │  │ [+ Add context]│ │
│  │                                                      │  └────────────────┘ │
│  │                                                      │                     │
│  │           [ Start executing → ]                      │                     │
│  └──────────────────────────────────────────────────────┘                     │
│                                                                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Key elements:**
- Work item title, flag name, owners
- Editable acceptance metric (the PM's definition of done)
- Context panel: auto-populated from GitHub + PostHog, manually extensible
- Graph traversal available inline if context is incomplete (click any context item to expand the graph)
- `Start executing` is the single CTA; it locks the plan and transitions state

---

## Screen 2 — Execute View

The engineer's working state. A checklist of concrete actions drives progress. Code editing is delegated to VS Code via deep-link; Tidepool owns everything else.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Tidepool                                    [Search]  [Engineer ▾]           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ← Checkout redesign  ·  EXECUTING                                            │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ Execution checklist                                                        │ │
│  │                                                                            │ │
│  │  ✓  1.  Branch created                                                     │ │
│  │         feat/checkout-v2  ·  github.com/acme/web/tree/feat/checkout-v2    │ │
│  │                                                                            │ │
│  │  ◉  2.  Write code                                                         │ │
│  │         [ Open in VS Code ↗ ]                                              │ │
│  │         Opens feat/checkout-v2 at src/pages/checkout.tsx                  │ │
│  │         ← Return link active in VS Code extension panel                   │ │
│  │                                                                            │ │
│  │  ○  3.  PR merged                                                          │ │
│  │         Waiting for PR #47 · "Checkout redesign"                          │ │
│  │         Status: open · 2 approvals needed                                 │ │
│  │         ↻ Polling GitHub                                                   │ │
│  │                                                                            │ │
│  │  ○  4.  Enable feature flag                                                │ │
│  │         PostHog flag: checkout-v2                                          │ │
│  │         Rollout:  [  10  ]%    [ Enable flag ]                             │ │
│  │         (available after PR merged)                                        │ │
│  │                                                                            │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ Activity feed                                                 [collapse] │ │
│  │  09:14  Branch feat/checkout-v2 created via GitHub API                   │ │
│  │  09:31  PR #47 opened — "Checkout redesign"  · @ana                      │ │
│  │  10:05  Review requested from @ben, @carmen                               │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Key elements:**
- Checklist steps are sequential; each unlocks when the prior completes
- Step 2 is the IDE delegation moment: "Open in VS Code" fires a deep-link (`vscode://tidepool/open?branch=feat/checkout-v2&return=tidepool://...`); the VS Code Tidepool extension shows a persistent "Return to Tidepool" panel
- Steps 1 and 3 are driven by GitHub API (create branch; poll PR status); Tidepool acts, the user confirms
- Step 4 calls PostHog API to configure and activate the flag
- Activity feed provides a lightweight audit trail of what Tidepool has done on the user's behalf
- When step 4 completes, state transitions to Shipped → Follow-up View opens automatically

---

## Screen 3 — Follow-up View

Available 24 hours after ship (or on-demand). Answers: did it work?

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Tidepool                                    [Search]  [Engineer ▾]           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ← Checkout redesign  ·  SHIPPED  ·  flag at 10%  ·  shipped 24h ago         │
│                                                                                │
│  ┌───────────────────────────────────────┐  ┌────────────────────────────┐   │
│  │ Did it work?                          │  │ What shipped               │   │
│  │                                       │  │────────────────────────────│   │
│  │  Acceptance metric:                   │  │ PR #47 merged              │   │
│  │  checkout funnel conversion +5pp      │  │ "Checkout redesign"        │   │
│  │                                       │  │ @ana · yesterday 11:42     │   │
│  │  Before     After       Delta         │  │ +312 / -88 lines           │   │
│  │  ─────────────────────────────────    │  │                            │   │
│  │  52.1%  →  58.3%    +6.2pp  ✓         │  │ Flag: checkout-v2          │   │
│  │                                       │  │ 10% rollout · active       │   │
│  │  [PostHog funnel · checkout → payment]│  │                            │   │
│  │  ░░░░░░░░░░░░████████████████ 58.3%   │  │ Sample: 4,200 users        │   │
│  │  before: ░░░░░░░░░░░████████ 52.1%   │  │                            │   │
│  │                                       │  └────────────────────────────┘   │
│  │  Metric exceeded target (+1.2pp)      │                                    │
│  │  Confidence: 98% (n=4,200)            │  ┌────────────────────────────┐   │
│  │                                       │  │ Signals                    │   │
│  │                                       │  │────────────────────────────│   │
│  │  [ Mark as Done ]                     │  │ No anomalies detected      │   │
│  │  [ Increase rollout → 50% ]           │  │ Error rate: stable         │   │
│  │                                       │  │ p99 latency: stable        │   │
│  │                                       │  │                            │   │
│  │                                       │  │ [View Honeycomb traces ↗]  │   │
│  └───────────────────────────────────────┘  └────────────────────────────┘   │
│                                                                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Anomaly variant** — when the metric did not move or a regression is detected:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ← Checkout redesign  ·  SHIPPED  ·  flag at 10%                             │
│                                                                                │
│  ┌───────────────────────────────────────────────────────────────────────┐    │
│  │ ⚠  Anomaly detected                                                   │    │
│  │                                                                        │    │
│  │  Acceptance metric: checkout funnel conversion +5pp                   │    │
│  │  Actual delta: −1.1pp  ✗                                              │    │
│  │                                                                        │    │
│  │  Connected signal                                                      │    │
│  │  ─────────────────────────────────────────────                        │    │
│  │  PostHog event: checkout_payment_step_loaded                          │    │
│  │    └─▶  GitHub PR #47 (merged yesterday)                              │    │
│  │           └─▶  Honeycomb: p99 latency spike on /api/checkout          │    │
│  │                  → slow DB query in CheckoutService.findCart()        │    │
│  │                                                                        │    │
│  │  [ Create work item: Fix slow query in checkout ]                     │    │
│  │  [ Roll back flag to 0% ]                                             │    │
│  │                                                                        │    │
│  └───────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Key elements:**
- Primary widget answers "did it work?" against the acceptance metric set in Plan
- Before/after funnel numbers pulled from PostHog; statistical confidence shown
- "What shipped" panel connects the metric to the exact PR and flag — not inferred, directly linked
- Signals panel shows Honeycomb health (error rate, latency) — no anomaly = one line; anomaly = graph traversal surfaces inline
- CTAs: Mark as Done (closes loop), Increase rollout (next action), or (anomaly path) create follow-on work item + roll back

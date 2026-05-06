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

The engineer's working state. A prompt input drives a coding agent; Tidepool streams agent progress live and surfaces the resulting PR for review. No IDE or editor integration — per ADR-0006, execute is prompt-driven.

**2a — Prompt input (before agent run)**

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Tidepool                                    [Search]  [Engineer ▾]           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ← Checkout redesign  ·  EXECUTING                                            │
│                                                                                │
│  ✓  Branch created  ·  feat/checkout-v2                                       │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ Describe the change                                                        │ │
│  │                                                                            │ │
│  │  ┌────────────────────────────────────────────────────────────────────┐   │ │
│  │  │ Redesign the checkout page (src/pages/checkout.tsx) to reduce      │   │ │
│  │  │ drop-off at the payment step. New layout per the Figma attached     │   │ │
│  │  │ to this work item. Keep existing PostHog event names — only the    │   │ │
│  │  │ UI changes, not the tracking.                                       │   │ │
│  │  └────────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                            │ │
│  │                                              [ Run agent → ]               │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ Activity                                                      [collapse] │ │
│  │  09:14  Branch feat/checkout-v2 created                                  │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

**2b — Agent running (streamed progress)**

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Tidepool                                    [Search]  [Engineer ▾]           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ← Checkout redesign  ·  EXECUTING  ·  Agent running…                        │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ ◌  Agent run in progress                                    [cancel]     │ │
│  │                                                                            │ │
│  │  ✓  Read src/pages/checkout.tsx  (312 lines)                              │ │
│  │  ✓  Read src/components/PaymentForm.tsx                                   │ │
│  │  ✓  Read src/styles/checkout.css                                          │ │
│  │  ◌  Writing new layout…  ████████████░░░░░░░░  62%                        │ │
│  │     → Modifying checkout.tsx                                               │ │
│  │     → Adding CheckoutV2.css                                                │ │
│  │     ↳ keeping PostHog event names unchanged                                │ │
│  │                                                                            │ │
│  │  ○  Run tests                                                              │ │
│  │  ○  Open PR                                                                │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
│                                                                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

**2c — PR review surface (agent run complete)**

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ Tidepool                                    [Search]  [Engineer ▾]           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ← Checkout redesign  ·  EXECUTING  ·  Review PR                             │
│                                                                                │
│  ✓  Agent run complete  ·  PR #47 opened  ·  +287 / −91 lines                │
│                                                                                │
│  ┌─────────────────────────────────────────┐  ┌────────────────────────────┐ │
│  │ PR #47 — "Checkout redesign"            │  │ Follow-up prompt           │ │
│  │ feat/checkout-v2 → main                 │  │────────────────────────────│ │
│  │                                         │  │                            │ │
│  │ Files changed (3)                       │  │ ┌──────────────────────┐   │ │
│  │  ▸ src/pages/checkout.tsx    +231 −88   │  │ │ The payment step CTA │   │ │
│  │  ▸ src/styles/CheckoutV2.css  +54  −0   │  │ │ should be full-width.│   │ │
│  │  ▸ src/styles/checkout.css    +2   −3   │  │ │ Also fix the mobile  │   │ │
│  │                                         │  │ │ breakpoint at 375px. │   │ │
│  │ ── diff ──────────────────────────────  │  │ └──────────────────────┘   │ │
│  │  - <div className="checkout-cta">       │  │                            │ │
│  │  + <div className="checkout-cta full">  │  │   [ Send follow-up → ]     │ │
│  │    <PaymentForm />                      │  │                            │ │
│  │    …                                    │  └────────────────────────────┘ │
│  │                                         │                                  │
│  │  [ Merge PR ]   [ Discard ]             │                                  │
│  └─────────────────────────────────────────┘                                  │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ Activity                                                                  │ │
│  │  09:14  Branch feat/checkout-v2 created                                  │ │
│  │  09:31  Agent run dispatched                                              │ │
│  │  09:47  PR #47 opened by agent — "Checkout redesign"                     │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

**2d — After merge: enable feature flag**

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  ← Checkout redesign  ·  EXECUTING                                            │
│                                                                                │
│  ✓  Branch created  ·  feat/checkout-v2                                       │
│  ✓  PR #47 merged   ·  "Checkout redesign"  ·  @ana  ·  10:02               │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐ │
│  │ Enable feature flag                                                        │ │
│  │                                                                            │ │
│  │  PostHog flag: checkout-v2                                                 │ │
│  │  Rollout percentage:  [  10  ]%                                            │ │
│  │                                                                            │ │
│  │                                    [ Enable flag → mark as Shipped ]       │ │
│  └──────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Key elements:**
- Prompt input (2a) is the entry point for Execute; no IDE link anywhere
- Agent run (2b) streams live — files read, files written, tests, PR open — so the user can see progress and cancel if needed
- PR review surface (2c) keeps the user in Tidepool: diff view, merge/discard CTAs, and a follow-up prompt panel for iteration without re-opening the full prompt
- Sending a follow-up prompt restarts the agent on the same branch; the PR updates; the review surface refreshes
- After merge (2d), the flag-enable step unlocks exactly as in the original design — this part is unchanged
- Activity feed is a persistent audit trail of what the agent and Tidepool have done on the user's behalf

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

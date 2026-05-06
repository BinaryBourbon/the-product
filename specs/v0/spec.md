# Tidepool v0 — Engineering Spec

## Product definition

Tidepool is the working environment where engineers and product managers plan, execute, and follow up on work — replacing the IDE as the primary surface for the full shipping cycle.

---

## Concrete scenario

An engineer and PM ship a checkout-page redesign behind a PostHog feature flag (`checkout-v2`). The engineer describes the change as a prompt in Tidepool; a coding agent makes the changes and opens a GitHub PR automatically. The engineer reviews the PR inside Tidepool, iterates with a follow-up prompt if needed, merges, and enables the flag at 10% rollout — all without leaving Tidepool. Twenty-four hours later, both check Tidepool's Follow-up View: the PostHog checkout funnel shows +6.2pp conversion. The PM marks the loop done. If a latency regression had appeared instead, a one-step graph traversal (PostHog event → GitHub PR → Honeycomb trace) would surface the cause inline and prompt a new work item.

---

## Execute model: prompt-driven coding agents (per ADR-0006)

**Decision:** The Option A vs. Option B (embedded editor vs. launched IDE) debate is moot — both were rejected at G2. The v0 Execute phase is **prompt-driven**: users describe the change they want in natural language; a coding agent makes the change and opens a PR automatically; users review, iterate via follow-up prompts, and merge — without leaving Tidepool.

Manual editing is a deliberate leak-through: if a user needs to edit specific code by hand, they clone the branch themselves. No Tidepool UI is provided for this.

The specific coding agent runtime (Claude Code subprocess, AoD, Cursor agent API, custom wrapper, etc.) is an **open question for the architect at G2**. See **ADR-0006** for the full decision record and the human's verbatim rationale.

---

## Screen inventory

| # | Screen / State | Phase | Description |
|---|---|---|---|
| 1 | Work Item View (Plan) | Plan | The primary planning surface. Shows work item title, description, owner, feature flag name, and an editable acceptance metric. A context panel auto-loads related GitHub history, PostHog baselines, and past decisions. The user completes this view before execution begins. |
| 2 | Context Panel (expanded) | Plan | An inline panel surfacing auto-linked context: related GitHub PRs/commits, PostHog funnel baselines, and linked past decisions. Manually extensible. Graph traversal is available from here if context is missing. |
| 3 | Graph Traversal View | Plan / Follow-up | A one-step-per-hop traversal of the cross-tool latent graph. Entry: any identifier (commit SHA, event name, PR number). Shows linked objects across GitHub, PostHog, and Honeycomb. Findings can be pinned to the work item or used to create a new work item. |
| 4 | Execute View | Execute | prompt input → agent run progress (streamed) → PR review surface → iterate via follow-up prompt or merge. Steps: (1) describe the change as a prompt, (2) agent works (live status), (3) PR opens automatically, (4) review and approve / iterate / discard, (5) enable feature flag. |
| 5 | Follow-up View (success) | Follow-up | Shown 24h after ship (or on demand). Primary widget: before/after PostHog funnel metric vs. acceptance metric. Secondary panel: what shipped (PR, flag config). Signals panel: Honeycomb error rate and latency health. CTAs: Mark as Done or Increase rollout. |
| 6 | Follow-up View (anomaly) | Follow-up | Shown when the metric did not move or a regression is detected. Inline graph traversal surfaces the chain: PostHog event → GitHub PR → Honeycomb trace. CTAs: create follow-on work item, roll back flag. |

---

## Acceptance criteria

The v0 slice is done when all of the following are true:

**Plan phase**
- [ ] A user can create a work item with a title, description, feature flag name, and at least one owner
- [ ] A user can set an acceptance metric (free-text string) on the work item
- [ ] The context panel auto-loads at least one linked GitHub PR and one PostHog funnel baseline when the work item is opened (based on flag name or keyword match)
- [ ] A user can manually add context items to the panel
- [ ] Clicking "Start executing" transitions the work item state from Plan → Executing and renders the Execute View

**Execute phase**
- [ ] A prompt input field is available on the work item in Executing state
- [ ] Submitting a prompt dispatches a coding agent run; the run's live progress streams into the Execute View
- [ ] The coding agent's PR appears in Tidepool's PR-review surface automatically when the agent opens it
- [ ] A user can send a follow-up prompt to iterate on the agent's work without leaving Tidepool
- [ ] A user can manually merge the PR from Tidepool's PR-review surface
- [ ] Tidepool calls the GitHub API to create a branch named `feat/<work-item-slug>` when execution begins
- [ ] A user can set a rollout percentage and call the PostHog API to enable the feature flag from within Tidepool
- [ ] Enabling the flag transitions work item state to Shipped and records a ship timestamp

**Follow-up phase**
- [ ] The Follow-up View is accessible from the work item 24 hours after the Shipped timestamp (and on-demand at any time)
- [ ] The PostHog funnel widget shows a before value (baseline captured at plan time) and an after value (queried at follow-up time), with a numeric delta
- [ ] The delta is compared to the acceptance metric and renders as pass (✓) or fail (✗)
- [ ] The "what shipped" panel shows the merged PR number, author, and merge timestamp, linked to GitHub
- [ ] The signals panel shows at least one Honeycomb health indicator (error rate or p99 latency) for the period since ship; "no anomalies" is an acceptable and explicit state
- [ ] If the metric delta is negative or the Honeycomb signal shows a regression, the anomaly path is shown with at least one graph traversal hop pre-populated (PostHog event → GitHub PR)
- [ ] A user can click "Mark as Done" to close the work item loop (state → Done)

**Invariants**
- [ ] All user-visible copy uses the name `Tidepool` (never `[PRODUCT NAME]` or any variant)
- [ ] The work item and its context, execution log, and follow-up data persist across browser sessions
- [ ] The GitHub, PostHog, and Honeycomb integrations each require only an API token to connect (no OAuth flow required for v0)

---

## Out of scope for v0

These are real Tidepool features explicitly excluded from this slice:

| Feature | Reason for deferral |
|---|---|
| Editor of any kind (embedded OR launched) | ADR-0006 — execute is prompt-driven; manual editing is leak-through (user clones branch locally if needed) |
| Multi-user collaboration (shared work items, comments, @-mentions) | Adds real-time sync complexity; single-user loop proves the concept |
| Full graph traversal UI (multi-hop explorer, graph visualization) | v0 needs one pre-populated hop in the anomaly path; a full traversal explorer is a later feature |
| Linear, Sentry, Datadog, PagerDuty integrations | Out of scope per ADR-0004; GitHub + PostHog + Honeycomb only |
| Notification system (email, Slack, push) | Adds integration surface; Follow-up View is pull-based in v0 |
| Onboarding, auth, billing, settings | Explicitly out of scope per brief |
| Design system, color palette, typography | Explicitly out of scope per brief |
| PM-only flows (no-code ticket movement, roadmap views) | v0 scenario is engineer-led; PM participates in Plan and Follow-up but does not have a distinct primary path |
| Feature flag targeting rules (user segments, percentage cohorts beyond simple rollout) | PostHog flag configuration is simplified to rollout percentage only |
| Work item history / audit log beyond the activity feed | Full history is a later feature; activity feed covers the v0 need |
| Mobile / tablet layout | Desktop-first for v0 |

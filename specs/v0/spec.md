# Tidepool v0 — Engineering Spec

## Product definition

Tidepool is the working environment where engineers and product managers plan, execute, and follow up on work — replacing the IDE as the primary surface for the full shipping cycle.

---

## Concrete scenario

An engineer and PM ship a checkout-page redesign behind a PostHog feature flag (`checkout-v2`). The engineer writes code in VS Code (launched via deep-link from Tidepool), opens and merges a GitHub PR, and enables the flag at 10% rollout — all tracked in a single Tidepool work item. Twenty-four hours later, both check Tidepool's Follow-up View: the PostHog checkout funnel shows +6.2pp conversion. The PM marks the loop done. If a latency regression had appeared instead, a one-step graph traversal (PostHog event → GitHub PR → Honeycomb trace) would surface the cause inline and prompt a new work item.

---

## Code-editing position: Option B — Launched IDE with deep-linking

**Decision:** v0 delegates code editing to the user's existing IDE (VS Code in the primary case) via a deep-link. Tidepool does not include an embedded editor in v0.

**Rationale:**

1. **Prove the loop before proving the editor.** The core Tidepool claim is that plan, execute, and follow-up belong in one continuous surface. That claim can be validated — and challenged — before solving the hard problems of a production code editor (LSP integration, git conflict resolution, extension ecosystem). v0 should prove the loop is valuable; a later slice can pull code editing in-product if the evidence supports it.

2. **Build cost.** A production-quality embedded editor (Monaco + LSP + terminal + git UI) is a multi-sprint investment that competes directly with VS Code and Cursor on ground they have spent years on. Option B ships a testable v0 with a fraction of that investment.

3. **The "replaces the IDE" claim does not require owning every pixel.** What Tidepool replaces is the IDE as the *organizing surface* — the place where work is planned, context lives, and outcomes are measured. The IDE as a *text editing tool* is a narrower claim and a lower-priority wedge. Deep-linking means Tidepool remains the organizing surface while VS Code handles the text editing surface.

4. **Precedent.** Linear opens GitHub PRs in GitHub. Notion opens Figma in Figma. These products own the *work graph*, not every tool in the graph. Tidepool's version of this is: own the plan/follow-up loop; launch the editor for the write-code step; pull the engineer back via a return link.

**Challenge this.** If the G2 engineer believes Option A is the right v0 wedge — that the embedded editor is what makes Tidepool legibly different from a project tracker — this should be debated before engineering begins. The counter-argument (Option A) is that without owning the editor surface, Tidepool is a wrapper around GitHub + PostHog + a task tracker, and users will not adopt a new "organizing surface" unless it also removes the friction of their current text editor. That argument is plausible and worth testing.

**Re-evaluate when:** A user research session shows that engineers adopt tools bottom-up from the edit loop outward (i.e., they would only use Tidepool if it were also their editor), not top-down from planning.

---

## Screen inventory

| # | Screen / State | Phase | Description |
|---|---|---|---|
| 1 | Work Item View (Plan) | Plan | The primary planning surface. Shows work item title, description, owner, feature flag name, and an editable acceptance metric. A context panel auto-loads related GitHub history, PostHog baselines, and past decisions. The user completes this view before execution begins. |
| 2 | Context Panel (expanded) | Plan | An inline panel surfacing auto-linked context: related GitHub PRs/commits, PostHog funnel baselines, and linked past decisions. Manually extensible. Graph traversal is available from here if context is missing. |
| 3 | Graph Traversal View | Plan / Follow-up | A one-step-per-hop traversal of the cross-tool latent graph. Entry: any identifier (commit SHA, event name, PR number). Shows linked objects across GitHub, PostHog, and Honeycomb. Findings can be pinned to the work item or used to create a new work item. |
| 4 | Execute View | Execute | A sequential checklist of concrete actions: (1) branch created, (2) write code → Open in VS Code, (3) PR merged (auto-detected), (4) enable feature flag. Each step has a status (pending / active / done) and is driven by Tidepool API calls where possible. |
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
- [ ] Tidepool calls the GitHub API to create a branch named `feat/<work-item-slug>` when execution begins
- [ ] "Open in VS Code" fires a deep-link that opens VS Code at the correct branch; the link is functional on macOS with VS Code installed
- [ ] A VS Code extension (or URI handler) provides a "Return to Tidepool" button that re-focuses the Tidepool browser tab
- [ ] Tidepool polls the GitHub API for PR status; the checklist step auto-completes when the PR is detected as merged
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
| Embedded code editor (Option A) | High build cost; loop can be validated without it; explicit architectural bet for later slice |
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

# Tidepool v0 — Architecture Proposal

- **Status:** proposed
- **Date:** 2026-05-06
- **Slice:** phase-3-architecture
- **Author:** architect agent

---

## 1. Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | Next.js (React, App Router) | Full-stack in one repo; built-in SSE support; proven at scale; Render deploys it as a Node service with zero config |
| Backend | Next.js API routes (Node.js / TypeScript) | Collocated with the frontend; eliminates a second service for v0; TypeScript throughout reduces type-boundary bugs |
| Database | PostgreSQL (Render Managed Postgres) | Relational model fits the entity graph; Render provides managed Postgres as a first-class service type; migration-based schema is easy to reason about |
| Real-time transport | Server-Sent Events (SSE) | One-directional stream (server → client) is all agent output requires; SSE is simpler than WebSocket, reconnects automatically via the browser's native `EventSource`, and works on Render's standard web service without any protocol upgrade |

The stack is boring by design. Next.js + Postgres + SSE covers every v0 requirement without introducing a separate API server, message broker, or WebSocket upgrade infrastructure.

---

## 2. Deployment

Tidepool v0 deploys on **Render** using two service types:

### Service topology

| Render service | Type | Purpose |
|---|---|---|
| `tidepool-web` | Web Service (Node) | Serves the Next.js frontend and all API routes, including the SSE streaming endpoint |
| `tidepool-db` | PostgreSQL | Managed Postgres database; Render injects `DATABASE_URL` automatically |

A **Background Worker** service is not needed for v0. Agent runs are spawned as child processes by the web service and stream output back over SSE directly. If agent run concurrency becomes a problem (see §9), the worker can be extracted later.

### Environment variables

All integration credentials are environment variables set in the Render dashboard:

```
DATABASE_URL          # injected by Render
ANTHROPIC_API_KEY     # for Claude Code CLI / API
GITHUB_TOKEN          # GitHub API token
POSTHOG_TOKEN         # PostHog API token
HONEYCOMB_KEY         # Honeycomb API key
```

### Release process

1. Push to `main` triggers a Render auto-deploy of `tidepool-web`.
2. On each deploy, Render runs `npm run build` then starts the Next.js server.
3. Database migrations run as a pre-deploy command (e.g., `npx prisma migrate deploy`) before Render swaps traffic to the new instance.
4. Render performs a zero-downtime swap; existing SSE connections from active agent runs are drained before the old instance is terminated (or reconnected automatically by the client's `EventSource`).

---

## 3. Coding Agent Runtime — Claude Code as a subprocess

**Decision: Claude Code CLI spawned as a sandboxed subprocess.**

### Rationale

ADR-0006 established that v0 Execute is prompt-driven: a user writes a prompt, a coding agent reads the repo, makes changes, and opens a PR. The runtime choice is load-bearing because it determines quality on multi-file changes, operational complexity, and vendor surface area.

**Why Claude Code:**

- **Quality on multi-file changes.** Claude Code is designed for exactly this: it reads repo context, reasons across files, runs tests, and opens PRs. The other options (AoD, Codex, custom wrapper) all require building or extending the agentic loop that Claude Code provides out of the box.
- **Dogfooding.** The team already uses Claude Code. Tidepool running Claude Code under the hood is the most direct form of dogfooding the product's own infrastructure.
- **Multi-turn support.** Follow-up prompts (the iterate loop in ADR-0006 and the wireframes) are naturally supported: each follow-up spawns a new Claude Code invocation on the same branch; the agent reads the current state of the branch (including the open PR) and applies the incremental change.
- **PR opening is native.** Claude Code can open PRs via the GitHub CLI (`gh pr create`) as part of its tool loop. No custom PR-opening logic is needed.

**Why not the alternatives:**

- **AoD** — tuned for single-turn specialist work; the multi-turn coding loop would require non-trivial extension. The AoD infrastructure is valuable but not the right fit for an interactive coding agent.
- **OpenAI Codex / Responses API** — introduces a second AI vendor with no clear quality advantage for repo-aware multi-file changes; file-system tooling is newer and less battle-tested than Claude Code's.
- **Custom thin wrapper** — building a reliable agentic loop (tool routing, error recovery, test running, PR opening) is non-trivial. Claude Code is the custom wrapper, already built.

### How it works

1. When the user submits a prompt, the backend clones the work item's GitHub branch into a temporary directory (`/tmp/runs/<run-id>`).
2. The backend spawns `claude --dangerously-skip-permissions -p "<prompt>"` as a child process in that directory, with `ANTHROPIC_API_KEY` in the environment.
3. stdout lines are read incrementally and streamed to the client via SSE (see §7).
4. When Claude Code completes, it will have committed changes and opened a PR via `gh`. The backend scans the stdout for the PR URL, extracts the PR number, and writes a `PullRequest` record to the database.
5. The temporary clone is deleted after the run completes or fails.

### Key dependency

The Render web service environment must have:
- `claude` CLI installed (via npm: `npm install -g @anthropic-ai/claude-code`)
- `gh` CLI installed (GitHub CLI, for PR creation)
- `git` available (standard in Render's build environment)

This is installed in the Docker build step / Render build command. See §9 for the spike needed to validate this.

---

## 4. Data Model

All entities persist in PostgreSQL. External APIs (GitHub, PostHog, Honeycomb) are the system of record for their own data; Tidepool mirrors what it needs locally to avoid round-trips and to survive API outages gracefully.

### `work_items`

The central entity. Everything else hangs off it.

| Field | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `title` | text | |
| `description` | text | |
| `slug` | text unique | derived from title; used as branch name suffix |
| `owner_handles` | text[] | GitHub handles |
| `feature_flag_name` | text | PostHog flag key (e.g. `checkout-v2`) |
| `acceptance_metric` | text | free-text definition of done |
| `state` | enum | `plan \| executing \| shipped \| done` |
| `baseline_metric_value` | float | PostHog funnel value captured at plan time |
| `posthog_rollout_pct` | int | 0–100; set when flag is enabled |
| `shipped_at` | timestamptz | set when flag is enabled |
| `github_branch` | text | `feat/<slug>` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `agent_runs`

One record per prompt submission (initial or follow-up).

| Field | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `work_item_id` | UUID FK → work_items | |
| `prompt` | text | the user's prompt |
| `status` | enum | `pending \| running \| complete \| failed \| cancelled` |
| `stream_log` | jsonb | array of `{t: timestamp, line: string}` entries; appended as agent streams |
| `github_pr_number` | int nullable | set when Claude Code opens the PR |
| `started_at` | timestamptz | |
| `completed_at` | timestamptz | |

### `pull_requests`

Mirrored from GitHub. Single source of truth for PR state inside Tidepool.

| Field | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `work_item_id` | UUID FK → work_items | |
| `github_pr_number` | int | |
| `github_repo` | text | `owner/repo` |
| `title` | text | |
| `state` | enum | `open \| merged \| closed` |
| `head_branch` | text | |
| `base_branch` | text | |
| `author_handle` | text | |
| `additions` | int | |
| `deletions` | int | |
| `opened_at` | timestamptz | |
| `merged_at` | timestamptz nullable | |
| `last_synced_at` | timestamptz | |

The diff itself is not stored. It is fetched from the GitHub API on demand when the PR review surface renders (see §6).

### `follow_up_checks`

One record per follow-up view load (snapshots the metric at a point in time).

| Field | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `work_item_id` | UUID FK → work_items | |
| `checked_at` | timestamptz | |
| `posthog_after_value` | float | funnel value at check time |
| `posthog_delta` | float | after − baseline |
| `metric_passed` | bool | delta meets acceptance metric |
| `honeycomb_error_rate_ok` | bool | |
| `honeycomb_latency_ok` | bool | |
| `anomaly_detected` | bool | true if metric failed or either Honeycomb signal is bad |
| `raw_posthog_response` | jsonb | for debugging and future replay |
| `raw_honeycomb_response` | jsonb | |

### Cross-tool correlation model

`work_item_id` is the anchor for all cross-tool correlation. The design is intentionally flat:

- **GitHub** — the branch (`feat/<slug>`) is created by Tidepool before execution begins; the PR is linked back to `work_item_id` via `pull_requests.work_item_id`.
- **PostHog** — `work_items.feature_flag_name` is the correlation key; Tidepool queries PostHog using this key for both flag control and funnel metrics.
- **Honeycomb** — queried by time range (`work_items.shipped_at` + window); no flag-key correlation is needed because the Honeycomb query is scoped to "the period after this work item shipped."

For the anomaly graph traversal: the pre-populated hop (PostHog event → GitHub PR) is assembled by joining `work_items.feature_flag_name` (PostHog side) to `pull_requests.github_pr_number` (GitHub side) via the shared `work_item_id`. The Honeycomb trace hop is a time-range query against the same window. No graph database is needed for v0's single-hop traversal.

---

## 5. Tool Integrations

### GitHub

| Dimension | Detail |
|---|---|
| Auth model | Static API token (`GITHUB_TOKEN` env var); passed as `Authorization: Bearer` header |
| Key API calls | `POST /repos/{owner}/{repo}/git/refs` — create branch at execution start; `GET /repos/{owner}/{repo}/pulls?head={branch}` — find PR opened by agent; `GET /repos/{owner}/{repo}/pulls/{pr}/files` — fetch diff for PR review surface; `PUT /repos/{owner}/{repo}/pulls/{pr}/merge` — merge from Tidepool |
| Real-time strategy | **Poll**: when a `PullRequest` record is in `open` state, the backend polls `GET /repos/{owner}/{repo}/pulls/{pr}` every 30 seconds. On merge, updates `pull_requests.state` and `merged_at`. Webhooks are better for latency but require a public endpoint registration per repo — too much setup surface for v0. |

### PostHog

| Dimension | Detail |
|---|---|
| Auth model | Static API token (`POSTHOG_TOKEN` env var); `Authorization: Bearer` header for REST API calls |
| Key API calls | `GET /api/projects/{id}/insights/funnel/` — query funnel baseline at plan time and after-value at follow-up; `PATCH /api/projects/{id}/feature_flags/{id}` — enable flag and set rollout percentage |
| Real-time strategy | **Pull on demand**: funnel metrics are fetched when the Follow-up View loads, not pushed. No webhook needed. |

### Honeycomb

| Dimension | Detail |
|---|---|
| Auth model | Static API key (`HONEYCOMB_KEY` env var); `X-Honeycomb-Team` header |
| Key API calls | `POST /1/queries/{dataset}` — create a query for error rate and p99 latency over the post-ship time window; `GET /1/query_results/{id}` — poll for query result |
| Real-time strategy | **Pull on demand**: Honeycomb results are fetched when the Follow-up View loads (or the anomaly panel opens). No webhook. |

---

## 6. PR Review Surface

**Decision: pull the diff from the GitHub API and render it server-side in Tidepool.**

The three options considered:

1. **GitHub API diff + server-side render (chosen)** — `GET /repos/{owner}/{repo}/pulls/{pr}/files` returns a list of changed files with `patch` (unified diff) content per file. The backend transforms this into a structured JSON payload; the frontend renders it as a styled diff view. The user never leaves Tidepool.
2. **iframe the GitHub PR** — simple, but requires the user to have a GitHub session in the browser; breaks the Tidepool UI chrome; blocks merge/discard CTAs from working without postMessage hacks.
3. **Local diff renderer** — would require cloning the branch on the server, which adds storage and compute cost; the GitHub API already has the diff; this is redundant.

The chosen approach keeps the user fully in Tidepool, requires only the existing `GITHUB_TOKEN`, and surfaces exactly the data needed (file list, additions/deletions, patch hunks). The diff renderer in the frontend is a thin component that formats unified-diff patches into colored `+`/`-` line displays — no diff library heavier than `diff2html` or a hand-rolled component is needed for v0.

The **Merge PR** CTA calls `PUT /repos/{owner}/{repo}/pulls/{pr}/merge` directly from the backend (not the GitHub web UI), then updates `pull_requests.state` in the database.

---

## 7. Streaming Agent Output

**Transport: Server-Sent Events (SSE)**

### Why SSE over WebSocket or long-poll

- Agent output is one-directional (server → client). SSE is designed for exactly this; WebSocket's bidirectionality is unnecessary and adds upgrade complexity.
- The browser's native `EventSource` reconnects automatically on drop (using the `Last-Event-ID` header and the SSE `retry` field), with no client-side reconnect logic needed beyond opening the `EventSource`.
- SSE works over standard HTTP/1.1 and HTTP/2 — no special Render configuration required.

### Stream path

```
Claude Code subprocess
  │ (stdout, line-by-line)
  ▼
Next.js API route (/api/runs/[id]/stream)
  │ reads stdout via Node.js readline
  │ appends each line to agent_runs.stream_log (JSONB)
  │ emits SSE event to connected client
  ▼
Browser EventSource
  │ receives events, appends to agent run UI
  ▼
Execute View (streamed progress)
```

### Reconnection

The SSE endpoint accepts a `?lastEventId=<n>` query param (mapped from the `Last-Event-ID` header). On reconnect, the backend reads `agent_runs.stream_log` from line `n` onward and replays the missed lines before resuming live streaming. This covers browser tab refreshes and dropped connections mid-run without losing log history.

### Run cancel

The **Cancel** button in the wireframes sends `DELETE /api/runs/[id]`. The backend sends `SIGTERM` to the Claude Code child process, marks the run `cancelled`, and closes the SSE stream.

---

## 8. State Persistence

### What lives in the database

| Entity | Persisted in DB? | Notes |
|---|---|---|
| Work item state machine | Yes — fully | `work_items.state` is the authoritative state; transitions are recorded by the backend on each action |
| Agent run log | Yes — fully | `agent_runs.stream_log` accumulates all streamed lines; this is how replays and reconnects work |
| PR review state | Partially | `pull_requests` mirrors GitHub's state; the diff itself is not stored |
| PostHog follow-up values | Yes — per check | `follow_up_checks` snapshots the metric at each view load; the raw response is stored for debugging |
| Honeycomb signals | Yes — per check | Same as PostHog: snapshotted in `follow_up_checks` |
| Context panel items | Yes | Stored as JSONB on `work_items` (a `context_items` array); manually added items persist, auto-loaded items are re-fetched but cached |

### What is re-fetched from external APIs on load

| Data | Source | When re-fetched |
|---|---|---|
| PR diff (file patches) | GitHub API | On demand when PR review surface opens |
| PostHog funnel metric (after-value) | PostHog API | Each time Follow-up View loads |
| Honeycomb health signals | Honeycomb API | Each time Follow-up View loads |
| PR merge status | GitHub API (via polling) | Every 30s while PR is open |

### Work item state machine

```
plan → executing → shipped → done
              ↑ (follow-up prompt loops back to executing, same branch)
```

State transitions are:
- `plan → executing`: user clicks "Start executing" (branch created in GitHub)
- `executing → shipped`: user enables feature flag (PostHog API called, `shipped_at` recorded)
- `shipped → done`: user clicks "Mark as Done" in Follow-up View

State is authoritative in the database. On page load, Tidepool reads the work item from the DB and renders the correct view — no external API calls are needed to determine state.

---

## 9. Open Questions for the Engineering Slice

These are decisions the spec and design cannot resolve — each requires a spike before the architecture can be considered final.

1. **Claude Code CLI in Render's build environment.** Can `claude` and `gh` CLIs be installed in a Render Web Service's build step and used in a subprocess? Render's build environment is a standard Linux container, but subprocess execution of long-running CLI tools (with stdin/stdout streaming) at request time needs validation. The spike: write a minimal Render Web Service that spawns `claude --version` and streams stdout back as SSE.

2. **PostHog funnel API shape for before/after queries.** The spec requires a "baseline captured at plan time" and an "after value queried at follow-up time." PostHog's Insights API supports funnel queries, but the exact parameters for time-windowed funnel conversion rates (date ranges, breakdown by flag cohort) need to be verified against the PostHog API docs before building the Follow-up View widget.

3. **Claude Code non-interactive output format.** When spawned as `claude -p "<prompt>"` (non-interactive / print mode), does Claude Code emit structured output (JSON lines) or free-form text? The stream log renderer and PR number extraction logic depend on the answer. The spike: run `claude -p "list files in the current directory"` in a test repo and inspect stdout format.

4. **GitHub repository scope.** The v0 spec implies a single GitHub repo per Tidepool instance. This needs to be made explicit: is the target repo a global config setting, or can different work items target different repos? The data model above has `github_repo` on `pull_requests` (which allows multiple repos), but the GitHub token and branch-creation flow assume one repo. A decision here affects the settings surface (even though onboarding/settings are out of scope for v0, the data model must accommodate the answer).

5. **Temporary clone storage on Render.** Claude Code needs a local git clone to operate. Render Web Services have ephemeral local disk; large repos may be slow to clone on every agent run. The spike: measure clone time for a representative repo size. If clone time exceeds ~30s, a shallow clone strategy (`--depth=1 --branch=<branch>`) or a warm-clone cache is needed.

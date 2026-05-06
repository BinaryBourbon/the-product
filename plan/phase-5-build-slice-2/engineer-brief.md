# Brief: Tidepool v0 — Slice 2: Execute loop (AoD dispatch + SSE proxy + PR detection)

## Context
- Slice: phase-5-build-slice-2
- Target repo: **BinaryBourbon/tidepool** — clone main (slice-1 is merged); branch from main
- Branch name: `slice-2/execute-loop`
- PR target: `main` of `BinaryBourbon/tidepool`
- Bus repo source-of-truth: https://github.com/BinaryBourbon/the-product — read these before writing any code:
  - `decisions/0004-product-framing-v2.md` — product framing
  - `decisions/0006-execute-model.md` — **prompt-driven via AoD; no IDE**
  - `decisions/0007-agent-runtime.md` — AoD HTTP API patterns (dispatch, multi-turn, stream, PR detection, auth)
  - `decisions/0008-v0-resolved-questions.md` — `agent_id` from `AOD_AGENT_ID` env var; multi-repo schema
  - `specs/v0/architecture.md` §3 (AoD runtime), §4 (data model), §7 (streaming), §5 (GitHub integration)
  - `specs/v0/spec.md` — Execute View screen (Screen 4) and Execute phase acceptance criteria (as amended for prompt-driven model)
  - `design/v0/wireframes.md` — Screen 2 sub-states (2a prompt input, 2b agent running, 2c PR review surface, 2d flag enable)

## Slice goal

Wire the prompt-driven execute loop end-to-end so a Tidepool user can:
1. Open a work_item, write a prompt, submit it
2. Watch the AoD agent's output stream live in the browser
3. Send a follow-up prompt to iterate
4. See the PR the agent opened appear in Tidepool's UI

This slice does NOT implement: PR diff rendering, merge-from-Tidepool, feature flag enable, or the Follow-up View. Those are slice-3.

---

## What to build

### A. New env vars

Add these to `render.yaml` and `README.md` env var table. **Do NOT commit values.**

```
AOD_VAULT_ID   # binarybourbon vault id; sync: false
               # Resolution: curl $AOD_BASE_URL/api/vaults -H "Authorization: Bearer $AOD_TOKEN"
               #             | jq -r '.data[] | select(.name=="binarybourbon") | .id'
```

`AOD_BASE_URL`, `AOD_TOKEN`, `AOD_AGENT_ID` are already in render.yaml from slice-1. Document the resolution pattern for `AOD_AGENT_ID` in README if not already there:

```bash
curl $AOD_BASE_URL/api/agents -H "Authorization: Bearer $AOD_TOKEN" \
  | jq -r '.data[] | select(.name=="general-purpose-engineer") | .id'
```

### B. API routes

#### `POST /api/work-items/[id]/runs`

Create an agent run and dispatch to AoD.

1. Validate `prompt` field from request body (non-empty string)
2. Insert `agent_runs` row: `{ work_item_id: id, prompt, status: 'pending' }`
3. POST to AoD:
   ```
   POST $AOD_BASE_URL/api/conversations
   Authorization: Bearer $AOD_TOKEN
   Content-Type: application/json

   { "agent_id": "$AOD_AGENT_ID", "vault_id": "$AOD_VAULT_ID", "prompt": "<user prompt>" }
   ```
4. On success: update `agent_runs` row with `{ aod_conversation_id: data.id, status: 'running' }`
5. Return `{ runId, aodConversationId }` to client
6. Immediately start the background stream-read loop (see §C below) — do not block the HTTP response on it

#### `POST /api/runs/[id]/prompts`

Send a follow-up prompt to an existing AoD conversation.

1. Look up `agent_runs` by `id`; verify `aod_conversation_id` is set and `status === 'running'`
2. POST to AoD:
   ```
   POST $AOD_BASE_URL/api/conversations/<aod_conversation_id>/prompts
   Authorization: Bearer $AOD_TOKEN
   Content-Type: application/json

   { "prompt": "<follow-up prompt>" }
   ```
3. Return `200 OK`; client continues listening on the existing SSE stream

#### `GET /api/runs/[id]/stream`

SSE proxy: connects to AoD's stream, persists to `stream_log`, re-emits to browser.

```
Response headers:
  Content-Type: text/event-stream
  Cache-Control: no-cache
  Connection: keep-alive
```

**Stream path:**

```
GET $AOD_BASE_URL/api/conversations/<aod_conv_id>/stream?wait=true
  │  Authorization: Bearer $AOD_TOKEN
  │  (SSE: AoD events)
  ▼
Next.js API route /api/runs/[id]/stream
  │  for each event:
  │    append { t: Date.now(), data: rawEventData } to agent_runs.stream_log (JSONB)
  │    emit SSE line to browser: "data: <rawEventData>\n\n"
  ▼
Browser EventSource('/api/runs/<id>/stream')
```

**Reconnection via Last-Event-ID:**
- Assign each event an incrementing integer ID (its index in `stream_log`)
- Emit `id: <n>` on each SSE event
- On new connection with `Last-Event-ID: <n>` header, read `agent_runs.stream_log` and replay entries from index `n+1` onward before resuming live proxy
- When AoD conversation is already `complete`, set `wait=false` on the AoD stream URL to drain the replay quickly

**AoD conversation status polling (background):**
- After dispatching, poll `GET $AOD_BASE_URL/api/conversations/<aod_conv_id>` every 5s
- When `data.status` transitions to a terminal state (`complete`, `failed`, `stopped`), update `agent_runs.status` accordingly and trigger PR detection (see §D)

#### `GET /api/runs/[id]`

Return `agent_runs` row (id, status, prompt, stream_log length, github_pr_number, started_at, completed_at). Client polls this to learn when a PR has been detected.

### C. Background stream-read + status-poll

In Next.js App Router, long-running background work can be done in a route handler that doesn't hold the request. For slice-2, the simplest approach:

- When `POST /api/work-items/[id]/runs` returns, kick off an async function (do not `await` it) that:
  1. Opens the AoD SSE stream and persists lines to `stream_log`
  2. Polls AoD conversation status every 5s
  3. On terminal status → triggers PR detection → closes

Note: on Render's free tier, the Node process may be recycled between requests. If the background reader is killed, the client's `EventSource` reconnect (with `Last-Event-ID`) will re-trigger the proxy, which will replay from `stream_log` and re-open the AoD stream if the conversation is still running. Build the SSE route handler so it handles this: if the AoD conversation is still `running`, open the live stream; if `complete`, drain the stored log with `wait=false`.

### D. PR detection

After `agent_runs.status` reaches `complete`:
1. Read `agent_runs.workItem.githubBranch` (the work item's branch name, `feat/<slug>`)
2. Read `agent_runs.workItem.githubRepo` (`owner/repo`)
3. Poll `GET https://api.github.com/repos/{owner}/{repo}/pulls?head={owner}:{branch}&state=open` every 10s for up to 2 minutes
   - Header: `Authorization: Bearer $GITHUB_TOKEN`
   - If a PR is found: insert `pull_requests` row, set `agent_runs.github_pr_number = pr.number`
   - If no PR after 2 min: mark `agent_runs.github_pr_number = null` and log a warning — the agent may not have opened one
4. Update the UI via the run status API (`GET /api/runs/[id]` poll)

Note: `pull_requests` table is in the architecture data model (§4) but was not in the Prisma schema from slice-1 — add it in this slice's migration.

### E. Execute View UI (Screen 4)

Add to the work_item view page (`/work-items/[id]`):

**2a — Prompt input (when no active run)**
- Textarea: "Describe the change"
- Button: "Run agent →"
- On submit: POST `/api/work-items/[id]/runs`, get back `runId`, navigate into streaming view

**2b — Agent running (streaming output)**
- Show `runId` and status badge (running / complete / failed)
- A `<pre>` or scrolling div that receives SSE lines appended in real time via `EventSource`
- "Cancel" button: `DELETE /api/runs/[id]` → calls `POST $AOD_BASE_URL/api/conversations/<aod_conv_id>/terminate`, marks run cancelled

**2c — Iterate (follow-up prompt)**
- Below the streaming output, a smaller textarea + "Send follow-up →" button
- On submit: POST `/api/runs/[id]/prompts`; SSE output continues on the same stream

**2d — PR detected banner**
- Poll `GET /api/runs/[id]` every 5s from the client
- When `github_pr_number` is set: show a banner "PR #N opened by agent — [view on GitHub ↗]"
- Link: `https://github.com/{githubRepo}/pull/{github_pr_number}`

Styling: Tailwind utility classes only, no design system. Functional, not beautiful.

---

## The dogfood test (mandatory — required for acceptance)

As part of building this slice, you must run the execute loop against a real target. Here is the exact test:

1. On your deployed Tidepool instance, create a work_item:
   - Title: `Slice-2 dogfood test`
   - `github_repo`: `BinaryBourbon/the-product`
   - Feature flag name: (leave blank)

2. Submit this prompt:
   ```
   Add a one-line comment to README.md that says: "verified via Tidepool slice-2 dispatch"
   Do not change anything else. Open a PR.
   ```

3. Observe streaming output in the Tidepool UI. Confirm that:
   - AoD output streams in real time in the browser
   - A PR is detected on BinaryBourbon/the-product and appears in the Tidepool banner

4. **Include in your PR description:**
   - The deployed Render URL
   - A link to the PR the agent opened on BinaryBourbon/the-product (proof the execute loop works end-to-end)
   - Any defensible-default flags you made

If the dogfood test fails (agent doesn't open a PR, stream breaks, etc.), debug and fix before opening the slice-2 PR. The PR description is the tech-lead's sign-off signal.

---

## `pull_requests` table (add in this slice's migration)

```prisma
model PullRequest {
  id              String    @id @default(uuid())
  workItemId      String    @map("work_item_id")
  githubPrNumber  Int       @map("github_pr_number")
  githubRepo      String    @map("github_repo")
  title           String
  state           String    @default("open")   // "open" | "merged" | "closed"
  headBranch      String    @map("head_branch")
  baseBranch      String    @map("base_branch")
  authorHandle    String?   @map("author_handle")
  additions       Int?
  deletions       Int?
  openedAt        DateTime  @map("opened_at")
  mergedAt        DateTime? @map("merged_at")
  lastSyncedAt    DateTime  @default(now()) @map("last_synced_at")
  workItem        WorkItem  @relation(fields: [workItemId], references: [id])

  @@map("pull_requests")
}
```

---

## Acceptance criteria

- [ ] Prompt input is present on the work_item view page
- [ ] Submitting a prompt creates an `agent_runs` row and dispatches to AoD; `aod_conversation_id` is set on the row
- [ ] AoD streaming output appears in the browser in real time via `EventSource`
- [ ] `agent_runs.stream_log` accumulates all events; refreshing the page replays from `Last-Event-ID`
- [ ] A follow-up prompt sends to the same AoD conversation and more output streams in
- [ ] After the agent run completes, Tidepool polls GitHub and detects the PR; `agent_runs.github_pr_number` is set
- [ ] A PR banner appears in the UI with a link to the GitHub PR
- [ ] `pull_requests` table exists in schema and a row is written on PR detection
- [ ] `render.yaml` includes `AOD_VAULT_ID` (sync: false) alongside existing AoD env vars
- [ ] Dogfood test PR on BinaryBourbon/the-product is linked in the slice-2 PR description
- [ ] No IDE integration, no `claude` CLI, no `ANTHROPIC_API_KEY` anywhere in the codebase

## Out of scope for this slice
- PR diff rendering inside Tidepool (slice-3)
- Merge-from-Tidepool (slice-3)
- Feature flag enable (slice-3)
- Follow-up View / PostHog widget (slice-3)
- Auth, design system, multi-user

## Constraints (non-negotiable)
- No IDE integration — ADR-0006
- `AOD_TOKEN`, `AOD_VAULT_ID`, `GITHUB_TOKEN` must never be committed — sync: false
- Agent_id comes from `process.env.AOD_AGENT_ID`, never hardcoded — ADR-0008

## Git workflow

```bash
git clone https://github.com/BinaryBourbon/tidepool.git
cd tidepool
git checkout main && git pull
git checkout -b slice-2/execute-loop
# implement, run dogfood test
npx prisma migrate dev --name add_pull_requests
git add .
git commit -m "slice-2: execute loop (AoD dispatch, SSE proxy, PR detection)"
git push -u origin slice-2/execute-loop
gh pr create \
  --title "slice-2: execute loop (AoD dispatch + SSE proxy + PR detection)" \
  --body "..."   # include: Render URL, dogfood test PR link, defensible-default flags
```

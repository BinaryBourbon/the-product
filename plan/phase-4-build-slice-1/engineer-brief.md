# Brief: Tidepool v0 — Slice 1: Foundation (repo, schema, CRUD, minimal UI)

## Context
- Slice: phase-4-build-slice-1
- Target repo: **BinaryBourbon/tidepool** (you must create this — see First Step below)
- Bus repo (source of truth docs): https://github.com/BinaryBourbon/the-product
- Source-of-truth artifacts — read ALL of these before writing a single line of code:
  - `decisions/0004-product-framing-v2.md` — what Tidepool is
  - `decisions/0005-product-name.md` — canonical spelling: **Tidepool** (capital T, lowercase p)
  - `decisions/0006-execute-model.md` — Execute is prompt-driven via AoD; no IDE
  - `decisions/0007-agent-runtime.md` — AoD HTTP API; env vars; no Claude CLI
  - `decisions/0008-v0-resolved-questions.md` — multi-repo schema; agent_id configurable; PostHog deferred
  - `specs/v0/architecture.md` — tech stack, data model, deployment (Render), env vars
  - `specs/v0/spec.md` — screen inventory, acceptance criteria, out-of-scope list

## First step: create the target repo

```bash
gh repo create BinaryBourbon/tidepool --private --description "Tidepool — workspace replacing the IDE"
```

If this fails due to permissions, **stop immediately and report to the tech-lead**. Do not work around it.

## Task

Build the **foundation slice** of Tidepool: the scaffolding, schema, and minimal UI that every future slice will build on top of. This slice does NOT implement agent dispatch, GitHub integration, PostHog, Honeycomb, or authentication. It proves the stack works end-to-end on Render and that the core data model is solid.

### What to build

**1. Next.js project scaffold**

- App Router, TypeScript, Tailwind CSS (for basic styling — no design system)
- `src/` directory structure per Next.js conventions
- ESLint + Prettier configured

**2. Database — Postgres via Prisma**

Use Prisma as the ORM (fits Next.js well; migration-based). Schema must implement the `work_items` table from `specs/v0/architecture.md §4` exactly. Include all fields specified there, including:

- `github_repo TEXT` — required per ADR-0008 (multi-repo from v0); free-text; no foreign key
- All state machine fields (`state` enum: `plan | executing | shipped | done`)
- `baseline_metric_value`, `posthog_rollout_pct`, `shipped_at` — present in schema, nullable for v0, NOT populated by slice-1

Also create the `agent_runs` table per architecture.md §4 with `aod_conversation_id UUID` — slice-1 does not write to it, but the schema must be complete so future slices don't require migrations that break existing data.

**3. API routes**

Implement four JSON API endpoints under `/api/work-items/`:
- `POST /api/work-items` — create; validates required fields; returns created record
- `GET /api/work-items` — list all; returns array ordered by `created_at DESC`
- `GET /api/work-items/[id]` — get single; 404 if not found
- `PATCH /api/work-items/[id]` — update title, description, acceptance_metric, github_repo, feature_flag_name

**4. Minimal UI — three pages**

Do not use a design system. Basic HTML structure with Tailwind utility classes is fine. The UI must be functional, not beautiful.

| Page | Route | What it does |
|------|-------|--------------|
| Create | `/work-items/new` | Form with all plan-phase fields; on submit, POST to API, redirect to view |
| List | `/work-items` | Table of all work items; each row links to view page |
| View | `/work-items/[id]` | Shows all fields; state badge; no actions yet beyond what's available |

**Form fields on Create:**
- `title` (required, text input)
- `description` (optional, textarea)
- `github_repo` (required, text input, placeholder: `owner/repo`) — per ADR-0008, must be present even though no GitHub API calls happen in slice-1
- `feature_flag_name` (optional, text input)
- `acceptance_metric` (optional, text input)
- `owner_handles` (optional, comma-separated text input → stored as text array)

**5. Render deployment**

Create `render.yaml` at repo root provisioning:
- `tidepool-web` — Web Service (Node), build: `npm run build`, start: `npm start`, env: `DATABASE_URL` (from group), `AOD_BASE_URL`, `AOD_TOKEN` (sync: false), `AOD_AGENT_ID`, `GITHUB_TOKEN`, `POSTHOG_TOKEN`, `HONEYCOMB_KEY`
- `tidepool-db` — PostgreSQL (Render managed)

Pre-deploy command: `npx prisma migrate deploy`

**6. README**

Root `README.md` documenting:
- What Tidepool is (one paragraph — use the press release framing)
- How to run locally (env vars needed, `npm run dev`)
- Env var table: `DATABASE_URL`, `AOD_BASE_URL`, `AOD_TOKEN` (secret), `AOD_AGENT_ID`, `GITHUB_TOKEN`, `POSTHOG_TOKEN`, `HONEYCOMB_KEY` — descriptions only, no values

## Constraints (non-negotiable)
- **No IDE integration** (ADR-0006) — not even a mention
- **No auth** — explicitly out of scope per spec
- **No agent dispatch** — that is slice-2
- **No PostHog/Honeycomb API calls** — deferred
- **No design system** — Tailwind utility classes only
- `AOD_TOKEN` must never be committed — it is a Render `sync: false` secret
- Do NOT edit anything in the BinaryBourbon/the-product bus repo — your only output is BinaryBourbon/tidepool

## Acceptance criteria

The slice is done when:
- [ ] `BinaryBourbon/tidepool` repo exists (private)
- [ ] App deploys successfully to Render (no build or runtime errors)
- [ ] The Render URL is reachable and renders the list page
- [ ] A user can create a work item via the form; it persists in the DB
- [ ] The list page shows all created work items
- [ ] The view page shows a single work item's fields
- [ ] All fields persist correctly across page reloads (server-rendered or fetched fresh from DB)
- [ ] `github_repo` field is present on the create form and persists

## PR instructions

- Branch: `slice-1/foundation`
- Open PR against `main` of `BinaryBourbon/tidepool`
- PR title: `slice-1: foundation (Next.js + Postgres + CRUD + Render deploy)`
- **PR description must include the deployed Render URL** — the tech-lead will verify it

## Git workflow

```bash
gh repo create BinaryBourbon/tidepool --private --description "Tidepool — workspace replacing the IDE"
git clone https://github.com/BinaryBourbon/tidepool.git
cd tidepool
git checkout -b slice-1/foundation
# scaffold the app, build, test locally
git add .
git commit -m "slice-1: foundation (Next.js + Postgres + CRUD + Render deploy)"
git push -u origin slice-1/foundation
gh pr create --title "slice-1: foundation (Next.js + Postgres + CRUD + Render deploy)" \
  --body "..."  # include Render URL
```

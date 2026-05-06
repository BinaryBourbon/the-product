# ADR-0007: Coding agent runtime — jhgaylor/aod-ex (AoD)

- **Status:** accepted
- **Date:** 2026-05-06
- **Decider:** human (G2 override)
- **Slice:** phase-3-architecture
- **Supersedes:** the Claude Code subprocess choice in the architect's spec (specs/v0/architecture.md §3)

## Context

The architect's spec (PR #7) chose Claude Code CLI as a subprocess and rejected AoD on the grounds that it was "tuned for single-turn specialist work; the multi-turn coding loop would require non-trivial extension." The human overrode this at G2. The architect's AoD critique was incorrect: AoD already supports multi-turn via `POST /api/conversations/<id>/prompts`, which captures `runtime_session_id` and resumes the same agent session. This pattern is used by this very project's tech-lead orchestration loop.

The human's exact words:

> "use jhgaylor/aod-ex for any prompting you need to do. dont subprocess out to claude code."

## Decision

Tidepool's coding agent runtime is **jhgaylor/aod-ex (AoD)**, accessed via its HTTP API. Not Claude Code subprocess; not Codex; not a custom wrapper.

## How AoD maps to Tidepool's Execute model

**Dispatch (initial prompt)**
`POST $AOD_BASE_URL/api/conversations` with body `{agent_id, vault_id, prompt}`. Response includes a `conversation.id`; Tidepool persists this on `agent_runs.aod_conversation_id`.

**Multi-turn (follow-up prompt)**
`POST $AOD_BASE_URL/api/conversations/<id>/prompts` with body `{prompt}`. AoD resumes the same agent session via `runtime_session_id` capture. The architect's single-turn critique was incorrect — this API exists and is in active use.

**Streaming**
`GET $AOD_BASE_URL/api/conversations/<id>/stream` — SSE endpoint. Tidepool proxies this stream: receives AoD SSE events, persists each to `agent_runs.stream_log`, and re-emits to the browser client as Tidepool's own SSE. `Last-Event-ID` reconnection still works because Tidepool can replay from `stream_log`.

**PR detection**
The AoD agent opens its own PR via its GitHub MCP integration (using its configured vault, e.g. binarybourbon). Tidepool does not parse stdout for a PR URL. Instead, Tidepool polls `GET /repos/{owner}/{repo}/pulls?head={branch}` after the agent run completes to detect the PR — clean separation of concerns.

**Auth**
Tidepool sends `Authorization: Bearer $AOD_TOKEN` to AoD. AoD-side agents use their own vault (e.g. binarybourbon) for GitHub credentials. Tidepool never holds the agent's GitHub token.

## AoD instance

The instance Tidepool will use: **https://jake-bagzz.sprites.app** (already deployed; binarybourbon-owned). This value is safe to reference in source code as the default for `AOD_BASE_URL`.

## Environment variables

| Variable | Type | Notes |
|---|---|---|
| `AOD_BASE_URL` | Regular env var | Defaults to `https://jake-bagzz.sprites.app`; safe in source |
| `AOD_TOKEN` | **Render sync:false secret** | Set in Render dashboard only; **NEVER commit the value to the repo** |

## Dogfooding note

Tidepool runs on its operator's own AoD instance. The product's Execute phase dispatches agents through the same infrastructure used to build the product. Two layers, same platform.

## Open question (deferred to first build slice)

Does Tidepool dispatch the existing `general-purpose-engineer` agent, or a new Tidepool-specific agent definition with an execute-only system prompt? The general-purpose agent is broader and may scope-creep (it has the habit of opening ADRs, updating roadmaps, etc.). A Tidepool-specific agent would be tighter but adds another definition to maintain. Decide before the first build slice begins.

## Consequences

- **Enables.** Simpler Tidepool backend: no subprocess management, no sandboxed clones, no `claude` or `gh` CLI in the Render build. The AoD HTTP API is all Tidepool needs.
- **Forecloses.** Claude Code CLI subprocess as the Execute runtime.
- **Accepted downside.** Tidepool is now dependent on the AoD instance being available. If `https://jake-bagzz.sprites.app` is down, Execute is down. For v0 (single-operator usage) this is acceptable.

## Revisit when

- The AoD instance uptime becomes a concern for user-facing reliability.
- A user research session reveals that the AoD agent's output quality on coding tasks is materially worse than alternatives.

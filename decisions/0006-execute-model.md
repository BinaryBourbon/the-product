# ADR-0006: v0 Execute model — prompt-driven coding agents, no IDE integration

- **Status:** accepted
- **Date:** 2026-05-06
- **Decider:** human (G2 gate decision)
- **Slice:** phase-2-design
- **Supersedes:** the Option A vs. Option B debate in the designer's spec (specs/v0/spec.md)

## Context

The designer's spec (PR #6) proposed Option B: launched IDE (VS Code) with deep-linking. At G2 review, the human rejected both Option A (embedded editor) and Option B (launched IDE) in favor of a third path not considered by the designer. The human's exact words:

> "we shouldnt integrate with vscode. they should be doing the bulk of their work with prompts opening prs. if they need to edit specific code manually we can trust the user to figure out how to do that on their own sanely."

## Decision

The v0 Execute phase is **prompt-driven via coding agents**.

- Engineers and PMs describe the change they want in natural language.
- A coding agent interprets the prompt and makes the change, then opens a pull request automatically.
- The user reviews the PR inside Tidepool, sends follow-up prompts to iterate, and merges when satisfied.
- **Manual editing is a deliberate leak-through**: if a user needs to edit specific code by hand, they clone the branch themselves and use whatever editor they want. This is not a Tidepool v0 feature and requires no UI.

The Option A vs. Option B debate in the designer's spec is moot. Both assumed a human writes code; the correct v0 model is that a coding agent writes the code in response to a prompt.

## What is explicitly NOT in scope

- Any VS Code deep-link, URI handler, or extension
- Any embedded code editor (Monaco, CodeMirror, etc.)
- Any "Open in editor" affordance of any kind
- Any Tidepool-managed IDE launch

## Open question for the architect (G2)

The specific coding agent runtime is **not decided by this ADR**. Options include:

- Claude Code as a subprocess (Anthropic API, agentic loop)
- AoD (the same Agent-on-Demand infrastructure this team already uses)
- Cursor's agent API
- OpenAI Codex API / Responses API
- A thin custom wrapper around any of the above

The architect at G2 must pick one with rationale. The choice affects: API cost, latency, self-hostability, quality on multi-file changes, and whether the coding agent can be the product's own dogfood.

## Consequences

- **Enables.** A dramatically simpler v0 Execute surface: a prompt input field, a streaming agent-run view, and a PR review panel — no editor, no extension, no deep-link infrastructure.
- **Forecloses.** Coding-as-typing in Tidepool for v0. If research reveals that prompt-driven execute fails for a class of changes that engineers actually need to make (e.g., changes requiring interactive debugging), this decision must be revisited.
- **Accepted downside.** Prompt-driven execute works well for bounded, describable changes. It works poorly for exploratory, iterative edits where the engineer doesn't know the shape of the change upfront. The human's framing ("they should be doing the bulk of their work with prompts") accepts this limitation as appropriate for the product's target workflow.

## Revisit when

- User research shows that a significant fraction of engineering work items cannot be described as a prompt (i.e., the task requires interactive exploration before the shape of the change is known).
- The coding agent runtime chosen at G2 proves unreliable enough that users regularly fall back to manual editing, and the leak-through ("just clone it") is causing meaningful friction.

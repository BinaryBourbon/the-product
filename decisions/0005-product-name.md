# ADR-0005: Product name — Tidepool

- **Status:** accepted
- **Date:** 2026-05-06
- **Decider:** human (G1 approval)
- **Slice:** phase-1-narrative

## Context

At G1 approval the human named the product. Prior artifacts used `[PRODUCT NAME]` as a placeholder. This ADR locks the name and establishes canonical spelling for all future agents.

## Decision

The product is named **Tidepool**.

- ADR-0004's `[PRODUCT NAME]` references are now read as Tidepool.
- The press release at `narrative/phase-1-press-release.md` has been updated accordingly.

## Canonical spelling

`Tidepool` — one word, capital T, lowercase p. Not:

- ~~Tide Pool~~ (two words)
- ~~TidePool~~ (camel case)
- ~~TIDEPOOL~~ (all caps)
- ~~tidepool~~ (all lowercase, except in code identifiers where snake/kebab case applies)

Future agents must use `Tidepool` verbatim in all user-facing copy, documentation, and PR titles. Code identifiers (repo names, npm packages, env vars) may use `tidepool`, `tide-pool`, or `TIDEPOOL` as the convention for that context requires — this ADR governs human-readable naming only.

## Consequences

- All future briefs, ADRs, specs, designs, and press materials use `Tidepool`.
- The `[PRODUCT NAME]` placeholder is retired.

## Revisit when

Human requests a name change.

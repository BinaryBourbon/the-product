# Slice: <kebab-case-name>

- **Status:** proposed | designed | building | shipped | measured
- **Owner conversation (build):** <aod conv id when assigned>
- **PR:** <link when open>
- **Depends on:** <list of prior slices that must ship first, or "none">

## User-visible capability

One sentence. After this slice ships, a user can: ___.

If you can't fill that blank cleanly, the slice is too big or too vague — go back to the tech-lead.

## Acceptance criteria

Concrete, testable. The release-validator works off these.

- [ ] <criterion 1>
- [ ] <criterion 2>
- [ ] <criterion 3>

## Out of scope

Things this slice deliberately does NOT do. Often more important than what it does.

## Design notes

Pointer to `design/<slice>/` if applicable. Inline sketches if it's small.

## Technical notes

The minimum the engineer needs to start. Not an exhaustive spec — just enough to disambiguate. If a real architectural choice has to be made, it's an ADR, not a note here.

## Measurement

How we'll know this worked once it's live.

- **Primary metric:** <name, source (PostHog event / Honeycomb derived column / etc.)>, target: <value or direction>
- **Guardrail metrics:** <metrics we don't want to regress>
- **Read at:** <after how long live>

## Related decisions

- ADR-NNNN: <title>
- ...

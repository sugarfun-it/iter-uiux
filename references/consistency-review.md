# Consistency review (Phase 4)

After all units in Phase 3 have converged (or been marked blocked / skipped), run one final loop that examines the project as a whole.

## Inputs

- Every Unit section already written in `CHANGELOG.md`.
- Final screenshots from each unit: pass the latest round's default-state screenshot per unit as image attachments to codex. For units with multiple representative states (e.g., button: default, pressed, disabled), pass all of them; codex needs to see variants to flag inconsistency.

## Loop

Same advisor / reviewer iteration shape as Phase 3:

1. **Round 1 (advisor)** — codex returns cross-unit issues + need_more.
2. **Round N (reviewer)** — Claude applies fixes (may span multiple files / multiple units), recaptures any affected units' screenshots, sends back diff + mapping_notes + screenshots.
3. Loop until `done: true`.

The anti-loop guard from `failure-recovery.md` applies here too.

## What codex looks for (cross-unit)

- **Design-token drift:** different border radii / spacing / type scale / colors for the same semantic role across units (e.g., nav uses radius 0, ticket uses 12, banner uses 8).
- **Shared-component candidates:** the same visual element re-implemented in multiple places that should be extracted.
- **Interaction-feedback drift:** ripple in one place, opacity in another, nothing elsewhere — for the same kind of element.
- **Copy/voice drift:** sentence case vs title case in CTAs; terminology mismatches ("Sign in" vs "Log in"); error tone inconsistency.
- **Flow seams:** redundant chrome (e.g., nav appearing inside a modal), back-button placement inconsistency, stacked top bars.
- **Accessibility consistency:** focus order across units; landmark roles; semantic-label conventions.

## Recording

Write the global section under `## Global Consistency Review` per the format in `changelog-format.md`. Each round writes the same fields a unit round writes (codex JSON, files modified, mapping notes, verdicts). Ensure all changes maintain design system coherence.

## Convergence

Codex returning `done: true` ends Phase 4. Proceed to Phase 5 (Summary).

If forced-convergence triggers (anti-loop), record it as in `failure-recovery.md` and proceed anyway — Phase 5 records the unresolved item count.

# Image mode

Triggered when `/iter-uiux` is invoked with one or more attached images. The first image defines the single unit to iterate on; additional images (if any) are extra reference states for the **same** unit, not separate units.

## Phase deltas vs. standard mode

| Phase | Standard | Image mode |
|---|---|---|
| 0 | Sandbox entry interview | Same one open question, plus one extra ask: **"how do I navigate the running app to the state shown in the attached image — which trigger / button / route brings up this view?"** Persist the answer to `.iter-uiux/.runtime-state.json` under `sandbox_anchor.image_navigation`. |
| 1 | Environment prep | unchanged |
| 2 | Full unit scan | **Skipped.** Unit = the UI element shown in the image. Append a single "Detected Units" row to `CHANGELOG.md` with a name inferred from the image (use `assets/unit-dictionary.yaml` if an obvious match exists; otherwise coin a short kebab-case name) and `source = user-supplied image`. |
| 3 | Per-unit iteration | Loop only on the single image-defined unit. **Round 1** attaches the user-supplied image(s) directly (no viewport/fullpage split — use the image as given). **Round N ≥ 2** recaptures via `screenshot-protocol.md` (viewport + fullpage) using `sandbox_anchor.image_navigation`; the user-supplied image is no longer attached. |
| 4 | Global consistency review | **Skipped.** Only one unit — nothing to cross-compare. |
| 5 | Summary | unchanged |

## Annotation semantics

If the user-supplied image contains a visible overlay annotation (red box, arrow, circle, or similar) marking a sub-region, treat the marked region as a **user-confirmed problem area** — the user has already decided this part is broken and wants it fixed. Do not re-frame it as a "focus hint".

Inject this paragraph into the advisor user prompt body, right after the `FLOW OUTLINE` section:

```
USER ANNOTATION
The attached image contains a red box / arrow / circle marking a region the
user has already identified as problematic and wants fixed. You MUST produce
concrete issues and suggested_fix entries targeting that region. Elements
outside the marked region are context only — do not flag issues there unless
they directly relate to the marked problem.
```

If no annotation is detected in the image, omit this paragraph entirely; codex then treats the whole image as the unit under review.

## Detection of annotations

A lightweight check is sufficient — codex will read the image anyway:

- If the image has any near-pure-red (`#E0–FF` red, low green/blue) rectangular outline, arrow shape, or circle that is visibly an overlay (sharp edges, not part of the underlying UI palette), treat as annotated.
- When in doubt, include the USER ANNOTATION paragraph; false positives are cheap (codex still works) but false negatives lose the user's intent.

## Round 1 codex call (image-mode advisor)

Same shape as the standard advisor in `codex-prompt.md`, with:

- `unit.name` = the inferred name (see Phase 2 row above).
- `unit.navigation` = filled from `sandbox_anchor.image_navigation`.
- `unit.entry_state` = the state visible in the image (e.g., `has-new-release`, `empty-list`, `error`).
- Image attachment = the user-supplied image(s). Attach in the order the user provided them.
- USER ANNOTATION paragraph included iff the image is annotated.

## Round N codex call (image-mode reviewer)

Identical to the standard reviewer in `codex-prompt.md`. Recapture follows `screenshot-protocol.md` (both `-viewport.png` and `-fullpage.png`) using the navigation captured in Phase 0. The user-supplied image is no longer attached — only the fresh captures.

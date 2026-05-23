# Image mode

Triggered when `/iter-uiux` is invoked with one or more attached images. The first image defines the single unit to iterate on; additional images (if any) are extra reference states for the **same** unit, not separate units.

**One-shot, always.** Image mode is a single advisor call → apply `suggested_fix` → done. No interview, no app launch, no recapture, no reviewer round. If the user wants another pass, they paste a new screenshot and re-trigger — that *is* the loop, paced by the user.

## Phase deltas

| Phase | Standard | Image mode |
|---|---|---|
| 0 | Sandbox entry interview | **Skipped.** No navigation needed — Claude never launches the app. |
| 1 | Full environment prep | Minimal: only verify `codex exec --version` and detect the image flag. Skip app-launch checks and screenshot-toolchain checks. |
| 2 | Full unit scan | **Skipped.** Unit = the UI shown in the image. Infer a short kebab-case name (use `assets/unit-dictionary.yaml` if it matches). Source files = grep the codebase for visible text strings in the image to locate the component(s). |
| 3 | Per-unit loop | **Single advisor call.** Attach the user-supplied image(s) as-is, plus RELATED SOURCE excerpts found via grep, plus USER ANNOTATION / USER INTENT paragraphs if applicable (see below). Codex returns `issues` + `suggested_fix`. Claude applies every `suggested_fix` to the working tree. |
| 4 | Global consistency review | **Skipped.** |
| 5 | Summary | **Do not write CHANGELOG.md.** Reply in-chat with a brief: which files changed, which fixes were applied, any issue codex flagged without an actionable fix. `git diff` is the durable record. |

## Annotation and intent semantics

Two optional signals can ride along with the image:

- **Annotation** — a visible overlay on the image (red box, arrow, circle, or similar) marking a sub-region. Marked region = **user-confirmed problem area**; the user has already decided this part is broken. Not a "focus hint".
- **Intent text** — any free-form text the user typed in the trigger alongside the slash command (e.g. `/iter-uiux <image> 多餘`). This describes the **direction** the fix should take (remove, consolidate, restyle, rewrite copy, etc.).

Behavior matrix:

| Image | Annotation | Intent text | Codex behavior |
|---|---|---|---|
| ✓ | ✗ | ✗ | Review the entire image. |
| ✓ | ✓ | ✗ | Marked region is broken — find the cause and propose fixes targeting it. |
| ✓ | ✗ | ✓ | Review the entire image; bias fix direction toward the stated intent. |
| ✓ | ✓ | ✓ | Marked region is broken **and** fix direction follows the stated intent. |

### Prompt injection

Inject the following paragraphs into the advisor user prompt body, right after `FLOW OUTLINE`. Include each one only when its signal is present; omit otherwise.

If annotation present:

```
USER ANNOTATION
The attached image contains a red box / arrow / circle marking a region the
user has already identified as problematic and wants fixed. You MUST produce
concrete issues and suggested_fix entries targeting that region. Elements
outside the marked region are context only — do not flag issues there unless
they directly relate to the marked problem.
```

If intent text present (verbatim, do not paraphrase):

```
USER INTENT
<paste the user's text exactly, e.g. "多餘">

This is the user's stated direction for the fix (e.g. "多餘" → propose
removal / consolidation, not restyling; "對齊怪" → alignment; "看不懂" →
copy / information hierarchy). Bias issues and suggested_fix toward this
intent. Only deviate if the intent is technically impossible or would create
a worse problem; if you deviate, explain why in the issue body.
```

### Detection of annotations

A lightweight check is sufficient — codex will read the image anyway:

- If the image has any near-pure-red (`#E0–FF` red, low green/blue) rectangular outline, arrow shape, or circle that is visibly an overlay (sharp edges, not part of the underlying UI palette), treat as annotated.
- When in doubt, include the USER ANNOTATION paragraph; false positives are cheap (codex still works) but false negatives lose the user's intent.

### Extracting intent text

The intent text is whatever the user typed in the trigger **after** the slash command and any image attachments. Strip leading/trailing whitespace; if the remainder is empty, there is no intent text. Do not interpret or rewrite it before injection.

## Advisor call

Same shape as the standard advisor in `codex-prompt.md`, with:

- `unit.name` = the inferred name (see Phase 2 row above).
- `unit.navigation` = `n/a (one-shot)`.
- `unit.entry_state` = the state visible in the image (e.g., `has-new-release`, `empty-list`, `error`).
- Image attachment = the user-supplied image(s). Attach in the order the user provided them.
- USER ANNOTATION and/or USER INTENT paragraphs included per the matrix above.

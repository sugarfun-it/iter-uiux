---
name: iter-uiux
description: |
  Drive a strict codex-CLI-supervised UI/UX review-and-fix loop over the current
  project's operation units (nav, topbar, ticket, banner, alert, toast, ...).
  Trigger when the user runs /iter-uiux or asks to iterate UI/UX with codex
  supervision. Operates fully automatically; produces .iter-uiux/CHANGELOG.md.
---

# iter-uiux

You are about to run a strict UI/UX iteration loop on the current project. Codex CLI acts as a strict UI/UX supervisor; you (Claude) operate the app, capture screenshots, apply edits, and maintain the changelog. The user does not intervene during the run.

## Modes

- **Standard mode** (default): trigger has no attached images. Full lifecycle (Phases 0–5), scans all units.
- **Image mode**: trigger has one or more attached images. Single-unit iteration on the UI shown in the image; Phase 2 and Phase 4 are skipped; if the image carries a red box / arrow / circle annotation, that region is treated as user-confirmed broken. See `references/image-mode.md` for the full phase deltas.

## Lifecycle

```
Phase 0  Sandbox entry interview  (only interactive phase)
Phase 1  Environment prep
Phase 2  Unit scan
Phase 3  Per-unit iteration (sequential)
Phase 4  Global consistency review
Phase 5  Summary
```

### Phase 0 — Sandbox entry interview
- Do **not** pre-scan the project. Ask the user **one** open question in a single message, covering all of:
  - how to start dev (command / URL)
  - which route / page / screen is the safe anchor to iterate on
  - login or test account, if any
  - rough scope hint (which area is fair game to mess with)
  - **(image mode only)** how to navigate the running app to the state shown in the attached image
- Read the user's free-form reply and try to parse out `launch`, `anchor_url`, `login`, `scope` (and `image_navigation` in image mode). If a critical field is missing or genuinely ambiguous, ask **one** follow-up question targeting only the missing piece; otherwise proceed silently (one short line of confirmation is fine).
- Persist the parsed result to `.iter-uiux/.runtime-state.json` under key `sandbox_anchor`. All later phases must read from it and operate only within this anchor.

### Phase 1 — Environment prep
- Verify cwd is the target project root (look for pubspec.yaml / package.json / index.html / etc).
- Detect project type and verify the app can be launched. Prefer the launch command from Phase 0's `sandbox_anchor.launch` if present; otherwise fall back to `references/unit-scan.md` (project-type detection + launch commands). Also see `references/screenshot-protocol.md` (screenshot toolchain pre-check).
- Verify `codex exec --version` works.
- Detect the codex image flag and cache it to `.iter-uiux/.runtime-state.json`. See `references/codex-prompt.md` § Image flag detection.
- Prepare `.iter-uiux/`:
  - If `.iter-uiux/CHANGELOG.md` exists, rename it to `.iter-uiux/CHANGELOG.<UTC-timestamp>.md`.
  - Create a fresh `.iter-uiux/CHANGELOG.md` and write the header + Environment block (see `references/changelog-format.md`).

### Phase 2 — Unit scan
- **Image mode: skip this phase.** Append a single "Detected Units" row whose unit is inferred from the attached image (see `references/image-mode.md`).
- Follow `references/unit-scan.md`. Map raw labels onto `assets/unit-dictionary.yaml`.
- Scope the scan to the sandbox anchor: navigate to `sandbox_anchor.anchor_url` (logging in via `sandbox_anchor.login` if given) and only enumerate units reachable from there within `sandbox_anchor.scope`.
- Append the "Detected Units" table to `CHANGELOG.md`.

### Phase 3 — Per-unit iteration (sequential)
For each detected unit, in order:
1. Round 1 (advisor): capture default-state screenshot, gather related source, draft flow outline. Call `codex exec` with the advisor prompt. See `references/codex-prompt.md`. **Image mode**: skip the capture step for Round 1 — attach the user-supplied image(s) as given (see `references/image-mode.md`).
2. Round N (N ≥ 2, reviewer): respond to `need_more`, apply fixes, recapture screenshots, build the response payload, call `codex exec` with the reviewer prompt.
3. Loop until codex returns `done: true` ("no further suggestions"). The only forced exit is the anti-loop guard in `references/failure-recovery.md`.
4. Append the unit's section to `CHANGELOG.md` per `references/changelog-format.md`.

### Phase 4 — Global consistency review
- **Image mode: skip this phase** (only one unit — nothing to cross-compare).
- Follow `references/consistency-review.md`. Same advisor/reviewer iteration model, scoped to cross-unit consistency.

### Phase 5 — Summary
- Append the Summary block (units reviewed / converged, total rounds, files modified, duration).

## Reference map

| When you are doing… | Read |
|---|---|
| project detection, app launch, scanning units | `references/unit-scan.md` |
| capturing screenshots on any platform | `references/screenshot-protocol.md` |
| any `codex exec` call (advisor / reviewer / global) | `references/codex-prompt.md` |
| writing or appending to CHANGELOG.md | `references/changelog-format.md` |
| any failure / unexpected state | `references/failure-recovery.md` |
| Phase 4 specifics | `references/consistency-review.md` |
| trigger had attached image(s) — image mode | `references/image-mode.md` |

## Invariants

1. **Fully automated after Phase 0.** Phase 0 is the only point that takes user input (one open question, plus at most one clarifying follow-up). Phases 1–5 must not ask the user for input.
2. **Fresh scan every run.** Always re-scan all units. Never carry previous-run context into codex.
3. **Archive, don't accumulate.** Each run starts a fresh `CHANGELOG.md`; old changelog files are renamed with a timestamp suffix. Archives are not fed to codex.
4. **Codex decides convergence.** Iterate until codex returns `done: true`. Only the anti-loop guard (failure-recovery.md) overrides this.
5. **Auto-diagnose, then halt.** When something fails, try the recovery steps in `references/failure-recovery.md`. Only stop when truly blocked.
6. **Full evaluation scope.** Codex covers visual / interaction / copy / structure / a11y / RWD / i18n / design-system consistency. Codex itself decides what depth and what extra data to demand.

## Red lines (Claude must NOT do these autonomously)

- Install missing SDKs or CLI tools.
- Fabricate mock login credentials or testing users.
- `git commit`, `git push`, or open PRs. This skill only modifies the working tree.
- Delete or rename existing files unless a codex `suggested_fix` explicitly says to remove or merge that file/component.
- Write to anything outside `.iter-uiux/` other than source files that codex's `suggested_fix` directs you to edit.

## Entry

1. Confirm cwd is a project root (not the iter-uiux skill repo itself, not a parent dir).
2. Enter Phase 0.

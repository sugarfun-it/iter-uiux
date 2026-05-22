# iter-uiux — Design Spec

- **Date:** 2026-05-22
- **Author:** Eric
- **Status:** Approved (brainstorming phase complete)
- **Target repo:** `D:/github/iter-uiux/` (Claude Code plugin, future github release)

## 1. Purpose

Build a Claude Code plugin SKILL named `iter-uiux` that drives an automated UI/UX review-and-fix loop over a target project. Each "operation unit" of the project's UI (nav, topbar, slide, ticket, banner, alert, confirm, toast, …) is iteratively reviewed by Codex CLI acting as a strict UI/UX supervisor; Claude performs the code edits. A CHANGELOG records every round.

## 2. Roles

- **Codex CLI** — strict UI/UX supervisor. Reads screenshots + source + design system + flow descriptions, returns structured issues and demands for more data. Also acts as reviewer of Claude's fixes. Advises and reviews; does NOT modify code.
- **Claude (this SKILL's runtime)** — orchestrator and implementer. Scans the project for units, drives the app, captures screenshots, calls `codex exec`, applies edits, maintains CHANGELOG.
- **User** — invokes `/iter-uiux` and reads the final CHANGELOG. No per-round approval prompts.

## 3. High-Level Lifecycle

```
/iter-uiux  (in target project root)

Phase 1  Environment prep
  - Detect project type (pubspec.yaml / package.json / index.html / ...)
  - Verify codex CLI available (codex exec --version)
  - Launch app if not running
  - Prepare .iter-uiux/ working dir
      If CHANGELOG.md exists -> rename to CHANGELOG.<UTC-timestamp>.md
      Create fresh CHANGELOG.md

Phase 2  Unit scan
  - Extract operation units from routes / pages / widget tree / component tree
  - Map raw labels onto the standard unit dictionary
  - Write "Detected Units" table to CHANGELOG.md

Phase 3  Per-unit iteration (sequential, for unit in units)
  Round 1 (advisor mode):
    Claude captures default-state screenshot + collects related source +
    drafts flow outline -> codex exec (advisor prompt) -> structured issues
    + need_more demands
  Round N (reviewer mode, N >= 2):
    Claude responds to need_more / fixes issues -> recaptures the same
    state set + diff + per-issue mapping notes -> codex exec (reviewer
    prompt) -> per-issue verdict + new issues
  Converged when codex returns "no further suggestions"
  Append unit section to CHANGELOG.md

Phase 4  Global consistency review
  Feed all unit CHANGELOG sections + final screenshots to codex (global
  prompt). Codex finds cross-unit inconsistencies. Claude fixes, recaptures
  affected units, codex re-reviews. Iterate to convergence (same rule as
  per-unit).

Phase 5  Summary
  Write summary block at end of CHANGELOG.md
```

## 4. Invariants

1. Fully automated — no per-round user approval (decision Q9 A).
2. Each `/iter-uiux` invocation re-scans all units; no carry-over from past runs (Q16).
3. Old `CHANGELOG.md` is archived to `CHANGELOG.<timestamp>.md` before each new run; archives are not fed to codex (Q17 B).
4. Convergence rule: codex must literally return "no further suggestions" — no fixed round cap (Q10 A, Q20 B). Exception: anti-loop guard (see §8).
5. Claude actively diagnoses and fixes obstacles. Only halts when truly blocked (Q13). See §8 for the failure ladder and red lines.
6. Codex's review scope is full-spectrum: visual + interaction + copy + structure + a11y + RWD + i18n + design system rationalization (Q18 E, Q21 E). Codex itself decides how deep to dig and what extra screenshots / source / flow info to demand.

## 5. Codex Invocation (three modes via `codex exec`)

All three modes use `codex exec` (non-interactive) directly. Images are passed via `codex exec`'s image flag (verify the flag name against the installed codex CLI version at runtime).

All three modes require codex to return a **structured response** (JSON; schema in `assets/codex-output-schema.json`):

```json
{
  "issues": [
    {
      "id": "string",
      "severity": "high|med|low",
      "area": "visual|interaction|copy|structure|a11y|rwd|i18n|design_system",
      "description": "string",
      "suggested_fix": "string"
    }
  ],
  "need_more": [
    { "kind": "screenshot|source|flow|state", "detail": "string" }
  ],
  "verdicts": [
    { "issue_id": "string", "status": "addressed|partially|not_addressed|regressed", "note": "string" }
  ],
  "done": false
}
```

When codex is satisfied, it returns `{ "issues": [], "need_more": [], "done": true }` — that is the "no further suggestions" signal.

### 5.1 Advisor mode (per-unit Round 1)

Inputs:
- Role preamble: "strict UI/UX supervisor; cover visual / interaction / copy / structure / a11y / RWD / i18n / design system; you may demand additional screenshots, states, source, or flow descriptions"
- Unit name + raw label + navigation path
- Entry-state screenshot
- Related source excerpts (Claude picks the unit's component file(s) and design-token files)
- Flow outline (Claude's initial draft of "what happens after each interactive element is triggered")
- Prior-unit CHANGELOG sections from the *current run* (for cross-unit consistency awareness; never previous runs)

Output: `issues[]`, `need_more[]`, possibly `done`.

### 5.2 Reviewer mode (per-unit Round N >= 2)

Inputs:
- The codex output from the previous round of this unit
- Claude's response payload:
  - `diff` — patches applied since last round
  - `mapping_notes[]` — for each prior `issues[i]`, which file/lines address it and why
  - `screenshots[]` — same state set, recaptured (plus any new states from prior `need_more`)
  - `extra_data[]` — any source / flow info requested via prior `need_more`

Output: `verdicts[]` (one per prior issue), optional new `issues[]`, optional `need_more[]`, possibly `done`.

### 5.3 Global mode (Phase 4)

Inputs:
- All unit CHANGELOG sections from this run
- Final screenshots of every unit
- Role preamble emphasizing cross-unit consistency: shared-component drift, design-token inconsistencies, copy/voice mismatches, flow seams between units

Output: same schema. Iterates with Claude until `done: true`.

## 6. Unit Scan Strategy

`references/unit-scan.md` will encode:

- **Project-type detection** — `pubspec.yaml` -> Flutter; `package.json` + react/vue/nuxt -> Web; `index.html` only -> static HTML; else heuristic + fail loud per §8.
- **Per-platform scan recipes:**
  - Flutter: parse `lib/**/*.dart` for `MaterialApp`/`CupertinoApp`/`GoRouter` route maps; identify `Scaffold` / `AppBar` / `BottomNavigationBar` / `Dialog` / `SnackBar` / `Banner` / etc.
  - React/Next: parse `pages/` or `app/` for routes; identify `<Dialog>`/`<Toast>`/`<Banner>`/`<Navbar>`/`<Sidebar>`/`<TopBar>`/`<Modal>` imports.
  - Vue/Nuxt: similar; parse `.vue` components.
  - Static HTML: parse `<nav>`/`<header>`/`<dialog>`/`<aside>`/`role="alert"`.
- **Standard unit dictionary** (`assets/unit-dictionary.yaml`):
  ```
  nav, topbar, bottombar, sidebar, drawer, tabbar,
  banner, alert, confirm, dialog, modal, sheet,
  toast, snackbar, tooltip, popover,
  card, list, listitem, ticket, slide, carousel,
  form, input, button, fab,
  empty_state, error_state, loading_state, skeleton
  ```
- **Unit model:**
  ```yaml
  name: nav                        # standardised
  raw_label: BottomNavigationBar
  source_files: [lib/widgets/bottom_nav.dart]
  navigation: "app launch -> home tab"
  entry_state: default
  related_states: [pressed, with_badge, disabled_item]   # Claude initial guess; codex may add
  flow_outline: |
    tap home -> home screen
    tap profile -> profile screen
    long press -> unknown   # marked uncertain; codex will probe
  ```
- Units whose `raw_label` does not map to the dictionary keep their raw label as `name` and are tagged `(unknown_type)` in CHANGELOG.

## 7. CHANGELOG Format

Single file at `<target_project>/.iter-uiux/CHANGELOG.md`, written in order:

```markdown
# iter-uiux run @ <ISO timestamp> (project: <name>)

## Environment
- platform: ...
- app launched via: ...
- codex CLI: ...

## Detected Units
| name | raw_label | source | navigation |
|---|---|---|---|
| nav | BottomNavigationBar | lib/widgets/bottom_nav.dart | app launch -> home |
...

---

## Unit: <name>
- rounds: N
- states reviewed: ...
- screenshots: .iter-uiux/screenshots/<run-ts>/<unit>/...

### Round 1 (advisor)
**Codex issues:** ...
**Codex need_more:** ...

### Round 2 (claude action)
**Diff:** ...
**Claude notes:** (issue_id -> change rationale)
**Screenshots:** ...

### Round 2 (reviewer)
**Codex verdict:** (per issue_id: addressed/partially/not_addressed/regressed)
**New issues:** ...

...

### Round K (reviewer)
**Codex verdict:** no further suggestions
**Converged at round K**

---

## Unit: ...

---

## Global Consistency Review
### Round 1 ... ### Round M (reviewer) -> Converged

---

## Summary
- units reviewed: ...
- units converged: ...
- total rounds: ...
- global review rounds: ...
- files modified: ...
- duration: ...
```

Screenshots are stored under `.iter-uiux/screenshots/<run-timestamp>/<unit>/round<N>-<state>.png`. They are NOT archived/moved between runs — only `CHANGELOG.md` is renamed; screenshot dirs accumulate by timestamp under `screenshots/`.

## 8. Failure Handling and Red Lines

`references/failure-recovery.md` will encode the full ladder. Summary:

| Failure | Auto-fix steps | Stop / fallback |
|---|---|---|
| App launch fails | Try project's usual command (`flutter run` / `npm run dev` / `yarn dev`); if port conflict, change port | 3 retries -> halt, write "blocker: cannot launch app" to CHANGELOG |
| Cannot navigate to unit | Grep source for correct route; check for `dev_login` / mock auth fixtures / seed scripts | 2 retries -> mark unit `blocked`, write reason, continue next unit. Never fabricate credentials. |
| Screenshot tooling missing (adb / simctl / playwright not installed) | Detect via `adb devices` / `simctl list` etc. | Do NOT auto-install SDKs; halt entire run, report. |
| Screenshot blank / all black | Wait 500ms, retry once | Still fails -> skip unit |
| codex not authed | (do nothing) | Halt entire run immediately, report |
| codex rate limit | Exponential backoff: 5s, 30s, 2m, 5m (up to 4 attempts) | After 4 -> halt run |
| codex rejects image (too large) | Downscale to 1920x1080 and retry | Still fails -> unit `blocked` |
| codex returns unparseable output | Send one corrective re-prompt ("please reply in the structured schema") | 2 failures -> unit `blocked` |
| Build / lint fails after Claude edit | Run lint --fix if pure lint; on build error revert this round's edits, tell codex "I tried fix X but build broke for reason Y, want to revise?" | The originating issue is marked `attempt_failed` and the unit proceeds to the next round |
| Unit not converging (codex looping) | Detect when codex's current-round `issues[]` is byte-identical to two rounds prior | Force convergence; write "diverged, stopped at round N" |

**Red lines — Claude must NOT do any of these autonomously:**

- Install missing SDKs / CLI tools
- Fabricate mock login credentials or testing users
- `git commit` / `git push` / open PRs (this SKILL only modifies the working tree)
- Delete or rename existing files, *unless* the codex issue's `suggested_fix` explicitly says to remove or merge that file/component (Claude must not "tidy" on its own initiative)
- Write to anything outside `.iter-uiux/` (no README, no root-level CHANGELOG, no `.gitignore` edits)

## 9. Repository Layout (flat skill, not plugin)

```
iter-uiux/                        # repo root == skill root
  README.md                       # github-facing usage doc + install instructions
  SKILL.md                        # short driver
  references/
    unit-scan.md
    codex-prompt.md
    screenshot-protocol.md
    changelog-format.md
    failure-recovery.md
    consistency-review.md
  assets/
    codex-output-schema.json
    unit-dictionary.yaml
  2026-05-22-iter-uiux-design.md  # this spec
  2026-05-22-iter-uiux-plan.md    # implementation plan
  examples/                       # (future) sample CHANGELOG outputs
```

**Installation:** Users clone this repo and copy/symlink the contents into `~/.claude/skills/iter-uiux/`. No `plugin.json`, no `/plugin install` workflow.

### 9.1 SKILL.md (driver)

Short orchestrator. Frontmatter:

```yaml
---
name: iter-uiux
description: |
  Drive a strict codex-CLI-supervised UI/UX review-and-fix loop over the
  current project's operation units (nav, topbar, ticket, banner, alert,
  toast, ...). Trigger when user runs /iter-uiux or asks to iterate UI/UX
  with codex supervision.
---
```

Body contents:
1. Lifecycle diagram (Phases 1-5 — verbatim from §3)
2. Phase -> reference doc lookup table
3. Invariants list (§4)
4. Red lines list (§8)
5. Entry instructions: verify cwd is the target project root, then enter Phase 1

### 9.2 References (one per concern)

- `references/unit-scan.md` — §6
- `references/codex-prompt.md` — §5 (full templates, schema reference, image-flag handling)
- `references/screenshot-protocol.md` — platform-specific capture (Flutter: `flutter-device-iter`-style adb / simctl; Web: playwright)
- `references/changelog-format.md` — §7
- `references/failure-recovery.md` — §8
- `references/consistency-review.md` — Phase 4 details, global-mode prompt, convergence rule

## 10. Open Items (resolved as design defaults; revisit if needed)

- **Codex CLI image flag** — exact flag name and size limits are determined at runtime by the installed codex CLI version. `references/codex-prompt.md` will document detection logic.
- **Codex output JSON parsing robustness** — schema in `assets/codex-output-schema.json` is the contract; the corrective re-prompt path (§8) handles non-conforming replies.

## 11. Out of Scope (explicit)

- Generating new UI from scratch — this SKILL only refines existing UI.
- Backend / API / data-layer review.
- Performance profiling.
- Test generation.
- Committing or pushing changes — user owns git operations.
- Multi-project orchestration — one invocation operates on one project (cwd).

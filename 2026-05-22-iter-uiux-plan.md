# iter-uiux Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a flat Claude Code skill repository at `D:/github/iter-uiux/` whose contents (when copied/symlinked into `~/.claude/skills/iter-uiux/`) drive a Codex-CLI-supervised UI/UX review-and-fix loop over any target project.

**Architecture:** Markdown skill repo. `SKILL.md` is a short driver that points Claude at the right `references/*.md` for each lifecycle phase. JSON schema in `assets/` defines the structured contract with `codex exec`. No executable code; verification is via schema/YAML parsing and a final dry-run walkthrough.

**Tech Stack:** Markdown (skill files), YAML (unit dictionary), JSON Schema draft-07 (codex output contract). Validation tools: `node` for JSON parse / ajv-style validation, `python` with PyYAML for YAML parse.

**Reference spec:** `D:/github/iter-uiux/2026-05-22-iter-uiux-design.md` — all design decisions live there. Each task below cites the spec sections it implements.

---

## File Inventory

| Path | Purpose | Implementing spec section |
|---|---|---|
| `README.md` | github-facing usage + install | §9 |
| `SKILL.md` | driver / orchestrator | §3, §4, §8, §9.1 |
| `references/codex-prompt.md` | three codex-exec prompt modes + JSON schema usage | §5 |
| `references/unit-scan.md` | project-type detection + per-platform scan recipes + unit model | §6 |
| `references/screenshot-protocol.md` | platform-specific screenshot capture | §3 (Phase 1, 3) |
| `references/changelog-format.md` | `.iter-uiux/CHANGELOG.md` layout + archive rules | §7 |
| `references/failure-recovery.md` | failure ladder + red lines | §8 |
| `references/consistency-review.md` | Phase 4 global review + convergence rule | §3 (Phase 4), §4 invariant 4 |
| `assets/codex-output-schema.json` | JSON Schema for `{issues, need_more, verdicts, done}` | §5 |
| `assets/unit-dictionary.yaml` | standard unit name list | §6 |

---

## Task 1: Repo scaffolding

**Files:**
- Create: `D:/github/iter-uiux/README.md` (stub)
- Create: `D:/github/iter-uiux/.gitignore`
- Create empty dirs: `D:/github/iter-uiux/references/`, `D:/github/iter-uiux/assets/`, `D:/github/iter-uiux/examples/`

- [ ] **Step 1: Create directories**

```bash
mkdir -p D:/github/iter-uiux/references D:/github/iter-uiux/assets D:/github/iter-uiux/examples
```

- [ ] **Step 2: Create `.gitignore`**

Write to `D:/github/iter-uiux/.gitignore`:
```
.DS_Store
Thumbs.db
*.swp
*.swo
node_modules/
.venv/
__pycache__/
```

- [ ] **Step 3: Write README stub (will be finalized in Task 11)**

Write to `D:/github/iter-uiux/README.md`:
```markdown
# iter-uiux

Codex-CLI-supervised UI/UX review loop for any project. Acts as a Claude Code skill.

Status: in development. See `2026-05-22-iter-uiux-design.md` for the full spec.
```

- [ ] **Step 4: Verify layout**

Run: `ls D:/github/iter-uiux/`
Expected: `2026-05-22-iter-uiux-design.md`, `2026-05-22-iter-uiux-plan.md`, `README.md`, `.gitignore`, `references/`, `assets/`, `examples/`

---

## Task 2: `assets/unit-dictionary.yaml`

**Files:**
- Create: `D:/github/iter-uiux/assets/unit-dictionary.yaml`

Implements spec §6 (standard unit dictionary).

- [ ] **Step 1: Write the validation command first**

This is the test — run before and after writing the YAML.
```bash
python -c "import yaml,sys; data=yaml.safe_load(open('D:/github/iter-uiux/assets/unit-dictionary.yaml')); assert isinstance(data,dict) and 'units' in data and isinstance(data['units'], list) and all(isinstance(u,dict) and 'name' in u and 'category' in u for u in data['units']); print('OK', len(data['units']), 'units')"
```
Expected (before file exists): FAIL with FileNotFoundError.

- [ ] **Step 2: Write the YAML**

Write to `D:/github/iter-uiux/assets/unit-dictionary.yaml`:
```yaml
# Standard operation-unit names. Spec §6.
# Categories are for grouping in scan output; not used as filters.
units:
  - { name: nav,            category: navigation,    aliases: [navigation] }
  - { name: topbar,         category: navigation,    aliases: [header, appbar, top_bar] }
  - { name: bottombar,      category: navigation,    aliases: [bottom_nav, bottom_navigation, BottomNavigationBar] }
  - { name: sidebar,        category: navigation,    aliases: [side_nav, side_panel] }
  - { name: drawer,         category: navigation,    aliases: [nav_drawer, Drawer] }
  - { name: tabbar,         category: navigation,    aliases: [tabs, TabBar] }

  - { name: banner,         category: feedback,      aliases: [MaterialBanner] }
  - { name: alert,          category: feedback,      aliases: [AlertDialog, alertbox] }
  - { name: confirm,        category: feedback,      aliases: [confirm_dialog, ConfirmDialog] }
  - { name: dialog,         category: feedback,      aliases: [Dialog, modal_dialog] }
  - { name: modal,          category: feedback,      aliases: [Modal] }
  - { name: sheet,          category: feedback,      aliases: [BottomSheet, action_sheet, ActionSheet] }
  - { name: toast,          category: feedback,      aliases: [Toast] }
  - { name: snackbar,       category: feedback,      aliases: [SnackBar] }
  - { name: tooltip,        category: feedback,      aliases: [Tooltip] }
  - { name: popover,        category: feedback,      aliases: [Popover, PopupMenu] }

  - { name: card,           category: container,     aliases: [Card] }
  - { name: list,           category: container,     aliases: [ListView] }
  - { name: listitem,       category: container,     aliases: [ListTile, list_item] }
  - { name: ticket,         category: container,     aliases: [TicketCard, ticket_card] }
  - { name: slide,          category: container,     aliases: [slide_page, PageView] }
  - { name: carousel,       category: container,     aliases: [Carousel, swiper] }

  - { name: form,           category: input,         aliases: [Form] }
  - { name: input,          category: input,         aliases: [TextField, TextFormField] }
  - { name: button,         category: input,         aliases: [ElevatedButton, TextButton, Button] }
  - { name: fab,            category: input,         aliases: [FloatingActionButton, floating_action_button] }

  - { name: empty_state,    category: state,         aliases: [EmptyState, empty] }
  - { name: error_state,    category: state,         aliases: [ErrorState, error_view] }
  - { name: loading_state,  category: state,         aliases: [Loading, loader] }
  - { name: skeleton,       category: state,         aliases: [Skeleton, shimmer] }
```

- [ ] **Step 3: Run the validation command**

Run the command from Step 1.
Expected: `OK 30 units` (or current count).

- [ ] **Step 4: Commit (deferred)**

git init is Task 13. Skip commit here.

---

## Task 3: `assets/codex-output-schema.json`

**Files:**
- Create: `D:/github/iter-uiux/assets/codex-output-schema.json`

Implements spec §5 (structured response contract).

- [ ] **Step 1: Write the validation command first**

```bash
node -e "const s=JSON.parse(require('fs').readFileSync('D:/github/iter-uiux/assets/codex-output-schema.json','utf8')); if(s['$schema']!=='http://json-schema.org/draft-07/schema#') throw new Error('wrong draft'); if(!s.properties.issues||!s.properties.need_more||!s.properties.verdicts||!('done' in s.properties)) throw new Error('missing keys'); console.log('OK')"
```
Expected (before file exists): FAIL with ENOENT.

- [ ] **Step 2: Write the JSON schema**

Write to `D:/github/iter-uiux/assets/codex-output-schema.json`:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://github.com/<user>/iter-uiux/assets/codex-output-schema.json",
  "title": "iter-uiux codex exec output",
  "description": "Structured response returned by codex acting as UI/UX supervisor in any of the three modes (advisor, reviewer, global). Spec §5.",
  "type": "object",
  "required": ["issues", "need_more", "verdicts", "done"],
  "additionalProperties": false,
  "properties": {
    "issues": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "severity", "area", "description", "suggested_fix"],
        "additionalProperties": false,
        "properties": {
          "id":            { "type": "string", "minLength": 1 },
          "severity":      { "enum": ["high", "med", "low"] },
          "area":          { "enum": ["visual", "interaction", "copy", "structure", "a11y", "rwd", "i18n", "design_system"] },
          "description":   { "type": "string", "minLength": 1 },
          "suggested_fix": { "type": "string", "minLength": 1 }
        }
      }
    },
    "need_more": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["kind", "detail"],
        "additionalProperties": false,
        "properties": {
          "kind":   { "enum": ["screenshot", "source", "flow", "state"] },
          "detail": { "type": "string", "minLength": 1 }
        }
      }
    },
    "verdicts": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["issue_id", "status"],
        "additionalProperties": false,
        "properties": {
          "issue_id": { "type": "string", "minLength": 1 },
          "status":   { "enum": ["addressed", "partially", "not_addressed", "regressed"] },
          "note":     { "type": "string" }
        }
      }
    },
    "done": { "type": "boolean" }
  }
}
```

- [ ] **Step 3: Run validation**

Run the command from Step 1.
Expected: `OK`.

---

## Task 4: `SKILL.md` driver

**Files:**
- Create: `D:/github/iter-uiux/SKILL.md`

Implements spec §3 (lifecycle), §4 (invariants), §8 red lines, §9.1 driver layout.

- [ ] **Step 1: Write the structural validation command**

```bash
python -c "
import re
t = open('D:/github/iter-uiux/SKILL.md', encoding='utf-8').read()
assert t.startswith('---'), 'missing frontmatter'
assert re.search(r'^name:\s*iter-uiux', t, re.M), 'missing name'
assert re.search(r'^description:', t, re.M), 'missing description'
for h in ['Lifecycle','Phase 1','Phase 2','Phase 3','Phase 4','Phase 5','Invariants','Red lines','Reference map']:
    assert h in t, f'missing section: {h}'
for ref in ['references/unit-scan.md','references/codex-prompt.md','references/screenshot-protocol.md','references/changelog-format.md','references/failure-recovery.md','references/consistency-review.md']:
    assert ref in t, f'missing reference link: {ref}'
print('OK')
"
```
Expected (before file): FAIL with FileNotFoundError.

- [ ] **Step 2: Write `SKILL.md`**

Write to `D:/github/iter-uiux/SKILL.md`:
````markdown
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

## Lifecycle

```
Phase 1  Environment prep
Phase 2  Unit scan
Phase 3  Per-unit iteration (sequential)
Phase 4  Global consistency review
Phase 5  Summary
```

### Phase 1 — Environment prep
- Verify cwd is the target project root (look for pubspec.yaml / package.json / index.html / etc).
- Detect project type and ensure the app can be launched. See `references/screenshot-protocol.md`.
- Verify `codex exec --version` works.
- Prepare `.iter-uiux/`:
  - If `.iter-uiux/CHANGELOG.md` exists, rename it to `.iter-uiux/CHANGELOG.<UTC-timestamp>.md`.
  - Create a fresh `.iter-uiux/CHANGELOG.md` and write the header + Environment block (see `references/changelog-format.md`).

### Phase 2 — Unit scan
- Follow `references/unit-scan.md`. Map raw labels onto `assets/unit-dictionary.yaml`.
- Append the "Detected Units" table to `CHANGELOG.md`.

### Phase 3 — Per-unit iteration (sequential)
For each detected unit, in order:
1. Round 1 (advisor): capture default-state screenshot, gather related source, draft flow outline. Call `codex exec` with the advisor prompt. See `references/codex-prompt.md`.
2. Round N (N ≥ 2, reviewer): respond to `need_more`, apply fixes, recapture screenshots, build the response payload, call `codex exec` with the reviewer prompt.
3. Loop until codex returns `done: true` ("no further suggestions"). The only forced exit is the anti-loop guard in `references/failure-recovery.md`.
4. Append the unit's section to `CHANGELOG.md` per `references/changelog-format.md`.

### Phase 4 — Global consistency review
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

## Invariants

1. **Fully automated.** No per-round user approval. Do not ask the user for input during Phases 1–5.
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
2. Enter Phase 1.
````

- [ ] **Step 3: Run structural validation**

Run the command from Step 1. Expected: `OK`.

---

## Task 5: `references/codex-prompt.md`

**Files:**
- Create: `D:/github/iter-uiux/references/codex-prompt.md`

Implements spec §5 (three modes, prompt templates, image flag handling, JSON schema usage).

- [ ] **Step 1: Write the validation command**

```bash
python -c "
t = open('D:/github/iter-uiux/references/codex-prompt.md', encoding='utf-8').read()
for s in ['Advisor mode','Reviewer mode','Global mode','codex exec','codex-output-schema.json','image','done']:
    assert s in t, f'missing: {s}'
print('OK')
"
```
Expected (before file): FAIL.

- [ ] **Step 2: Write `references/codex-prompt.md`**

Write to `D:/github/iter-uiux/references/codex-prompt.md`:
````markdown
# Codex prompt reference

All codex calls go through `codex exec` (non-interactive). Three modes: advisor (per-unit Round 1), reviewer (per-unit Round N ≥ 2), global (Phase 4). All three use the same JSON output schema (`assets/codex-output-schema.json`).

## Image flag detection

Codex CLI's image flag name has changed across versions. Before the first `codex exec` call in a run, detect it:

1. Run `codex exec --help 2>&1` and capture the output.
2. Search for image-related flags in this priority order:
   - `--image` / `-i`
   - `--attach`
   - `--input-image`
3. Cache the detected flag for the rest of the run (write it to `.iter-uiux/.runtime-state.json`).
4. If none is found, halt the run and report: "codex CLI does not advertise an image input flag; please upgrade codex or run with a version that supports image input."

## Common rules

- Always end the user prompt with: `Reply ONLY with JSON matching the schema at <skill_root>/assets/codex-output-schema.json. No prose, no markdown fences.`
- Pass screenshots via the detected image flag. Multiple images: repeat the flag.
- Pass source / CHANGELOG context inline in the prompt (as fenced code blocks). Keep total prompt under codex's context limit; if oversize, trim oldest CHANGELOG sections first.
- After the call returns, parse stdout as JSON. On parse failure follow the corrective re-prompt path in `failure-recovery.md`.

## Advisor mode (per-unit Round 1)

System / role preamble (prepend to user prompt):

```
You are a strict UI/UX supervisor. Cover visual, interaction, copy,
structure, a11y, RWD, i18n, and design-system consistency. You may
demand additional screenshots, states, source, or flow descriptions
via the need_more field. Be thorough; do not soften findings.
```

User prompt body — fill in placeholders:

```
UNIT
  name: <unit.name>
  raw_label: <unit.raw_label>
  navigation: <unit.navigation>
  entry_state: <unit.entry_state>

INITIAL STATE LIST (Claude's guess; you may demand more via need_more)
  <unit.related_states as comma list>

RELATED SOURCE (truncated to relevant excerpts)
<fenced code blocks per file, with file path on the fence info line>

FLOW OUTLINE (Claude's draft; mark uncertainties with "?")
<flow text>

PRIOR-UNIT CHANGELOG SECTIONS FROM THIS RUN (consistency context — do not be
constrained by them, but flag conflicts)
<changelog sections so far, or "none yet" if first unit>

Reply ONLY with JSON matching the schema at <abs path to codex-output-schema.json>.
No prose, no markdown fences.
```

Pass via image flag: the entry-state screenshot.

Expected output shape: `{ issues: [...], need_more: [...], verdicts: [], done: false }` (or `done: true` if nothing to fix).

## Reviewer mode (per-unit Round N ≥ 2)

System / role preamble: same as advisor.

User prompt body:

```
UNIT
  name: <unit.name>
  round: <N>

YOUR PRIOR-ROUND OUTPUT
<JSON of last codex response>

CLAUDE'S RESPONSE PAYLOAD
  diff: |
<unified diff of code changes since last round>
  mapping_notes:
    - issue_id: <id>
      files_changed: [<paths>]
      rationale: <why this addresses the issue>
    ...
  extra_data: <any source/flow info you asked for in last need_more>

SCREENSHOTS ATTACHED: <state list, in order of image flags>

Now produce verdicts for each prior issue_id (addressed | partially |
not_addressed | regressed). You may also emit new issues if Claude's
changes introduced new problems. If everything is addressed and there
are no new issues and no further need_more, set done: true.

Reply ONLY with JSON matching the schema at <abs path>. No prose.
```

Pass via image flag: the recaptured screenshots (same state set as the unit's tracked states, plus any new states from prior need_more).

## Global mode (Phase 4)

System / role preamble:

```
You are a strict UI/UX supervisor performing a CROSS-UNIT consistency review.
Look for: shared-component drift, inconsistent design tokens (radius, color,
spacing, type, motion), copy-voice/terminology mismatches, mismatched
interaction feedback, and flow seams between units. Do not re-flag
per-unit issues already resolved.
```

User prompt body:

```
ALL UNIT CHANGELOG SECTIONS FROM THIS RUN
<concat of every unit's section>

ATTACHED SCREENSHOTS: <unit -> state mapping, one image per attachment>

Emit issues that span multiple units, or are about the design system as a
whole. Use the same JSON schema. When no cross-unit inconsistencies remain,
set done: true.

Reply ONLY with JSON. No prose.
```

Iterates in advisor/reviewer pattern with Claude's response payload describing cross-unit edits.

## Output parsing

```python
# pseudocode for Claude's runtime
raw = run("codex exec <prompt-file> <image flags...>")
try:
    data = json.loads(raw)
    validate(data, schema=<assets/codex-output-schema.json>)
except (JSONDecodeError, ValidationError):
    # see failure-recovery.md → "codex returns unparseable output"
    ...

if data["done"]:
    converged()
else:
    handle_issues_and_need_more(data)
```
````

- [ ] **Step 3: Run validation**

Expected: `OK`.

---

## Task 6: `references/unit-scan.md`

**Files:**
- Create: `D:/github/iter-uiux/references/unit-scan.md`

Implements spec §6.

- [ ] **Step 1: Write validation command**

```bash
python -c "
t = open('D:/github/iter-uiux/references/unit-scan.md', encoding='utf-8').read()
for s in ['Project-type detection','Flutter','React','Vue','HTML','Unit model','unit-dictionary.yaml','navigation','flow_outline']:
    assert s in t, f'missing: {s}'
print('OK')
"
```

- [ ] **Step 2: Write `references/unit-scan.md`**

Write to `D:/github/iter-uiux/references/unit-scan.md`:
````markdown
# Unit scan reference

How to detect project type, launch the app, scan for operation units, and produce the canonical unit model.

## Project-type detection

Probe in this order; first match wins:

| Probe | Verdict |
|---|---|
| `pubspec.yaml` exists | Flutter |
| `package.json` exists and lists `react`, `next` in deps | React/Next |
| `package.json` exists and lists `vue`, `nuxt` in deps | Vue/Nuxt |
| `package.json` exists, no SPA framework match, has `index.html` | Generic web |
| Only `index.html` at root | Static HTML |
| `Cargo.toml`, `go.mod`, etc., with no web UI | unsupported → halt per failure-recovery.md |

Cache detected type in `.iter-uiux/.runtime-state.json`.

## Per-platform scan recipes

### Flutter
- Routes: parse `lib/**/*.dart`; look for `MaterialApp(routes:)`, `MaterialApp.router`, `GoRouter(routes:)`, `Navigator.push` call sites.
- Components: grep `lib/**/*.dart` for the alias list in `assets/unit-dictionary.yaml`. Examples: `Scaffold`, `AppBar`, `BottomNavigationBar`, `Drawer`, `TabBar`, `Dialog`/`AlertDialog`, `SnackBar`, `MaterialBanner`, `BottomSheet`, `Tooltip`, `Card`, `ListTile`, `PageView`, `TextField`, `ElevatedButton`/`TextButton`, `FloatingActionButton`.
- App launch: `flutter run -d <device>` once a device is up. See `screenshot-protocol.md`.

### React / Next
- Routes: parse `pages/` or `app/` for file-based routes; also grep for `<Routes>`/`<Route>`/`createBrowserRouter` if react-router.
- Components: grep `**/*.{jsx,tsx,js,ts}` for tag names matching aliases. Examples: `<Modal>`, `<Dialog>`, `<Drawer>`, `<Toast>`, `<Banner>`, `<Tooltip>`, `<Navbar>`, `<Sidebar>`, `<TopBar>`, `<Tabs>`, `<Card>`.
- App launch: `npm run dev` / `yarn dev` / `pnpm dev`. Detect from `package.json`'s `scripts.dev`.

### Vue / Nuxt
- Routes: parse `pages/` (Nuxt) or `vue-router` config.
- Components: grep `**/*.vue` for tag aliases (similar set to React).
- App launch: `npm run dev` (Nuxt) or `npm run serve` (Vue CLI).

### Static / generic HTML
- Parse `index.html` (and other top-level html files) for `<nav>`, `<header>`, `<aside>`, `<dialog>`, `[role="alert"]`, `[role="dialog"]`, `[role="tooltip"]`, etc.
- App launch: open in browser via the screenshot tool (e.g., playwright `page.goto('file://...')`).

## Mapping raw labels to dictionary names

Load `assets/unit-dictionary.yaml`. For each found component:
1. Lowercase + underscore-normalize the raw label.
2. Match against `name` directly, then against `aliases[]`.
3. On match, set `unit.name` to the dictionary's `name`, and `unit.raw_label` to the original.
4. On no match, set `unit.name = <raw_label_normalized>` and tag `(unknown_type)` in CHANGELOG.

## Unit model

Each detected unit yields:

```yaml
name: <dictionary name or unknown_type>
raw_label: <verbatim source token>
source_files: [<absolute paths>]
navigation: <how to reach entry state, free text>
entry_state: default
related_states: <Claude's initial guess: e.g. [pressed, with_badge, disabled_item]>
flow_outline: |
  <free text — what each interactive element does next; mark uncertainties with "?">
```

Push the full list into `.iter-uiux/.runtime-state.json` keyed by `units`.

## Navigation drafting

For each unit, write a short navigation path from app launch to the unit's entry state. This goes into the codex advisor prompt and the CHANGELOG "Detected Units" table.

Examples:
- `app launch -> home tab`
- `app launch -> profile -> settings -> notifications`
- `home -> tap any ticket card -> ticket_detail`

If a unit requires login and no dev fixture exists, mark `navigation: blocked: needs auth` and follow the "Cannot navigate to unit" entry in failure-recovery.md.

## Initial state guess

Per unit type, default `related_states` guesses:

| unit category | default initial states |
|---|---|
| navigation | default, pressed, with_badge (if items can have badges), disabled_item |
| feedback (dialog/toast/...) | default, with_long_copy |
| feedback (banner/alert) | default, dismissed |
| container (card/list/listitem) | default, pressed, with_long_copy |
| container (carousel/slide) | default, first, middle, last |
| input (button) | default, pressed, focused, disabled, loading |
| input (form/input) | empty, filled, error, focused |
| state (empty/error/loading/skeleton) | default |

These are seeds. Codex will demand more via `need_more`.
````

- [ ] **Step 3: Run validation**

Expected: `OK`.

---

## Task 7: `references/screenshot-protocol.md`

**Files:**
- Create: `D:/github/iter-uiux/references/screenshot-protocol.md`

Implements spec §3 capture details across platforms.

- [ ] **Step 1: Write validation command**

```bash
python -c "
t = open('D:/github/iter-uiux/references/screenshot-protocol.md', encoding='utf-8').read()
for s in ['Flutter','adb','simctl','playwright','Naming','round','state']:
    assert s in t, f'missing: {s}'
print('OK')
"
```

- [ ] **Step 2: Write `references/screenshot-protocol.md`**

Write to `D:/github/iter-uiux/references/screenshot-protocol.md`:
````markdown
# Screenshot protocol

Platform-specific recipes for launching the app, navigating to a unit state, and capturing a screenshot. All produced files land under `.iter-uiux/screenshots/<run-ts>/<unit>/round<N>-<state>.png`.

## Naming convention

```
.iter-uiux/screenshots/<run-timestamp>/<unit>/round<N>-<state>.png
```

Examples:
- `.iter-uiux/screenshots/2026-05-22T1430Z/nav/round1-default.png`
- `.iter-uiux/screenshots/2026-05-22T1430Z/ticket/round3-with_long_copy.png`
- `.iter-uiux/screenshots/2026-05-22T1430Z/button/round2-disabled.png`

`<run-timestamp>` is set at the start of the run and reused for all screenshots in that run. Reuse the **same** state names across rounds so the reviewer can compare round-N vs round-N+1 directly.

## Flutter

Prefer existing patterns from `flutter-device-iter` if it is installed in the operator's environment.

### Android emulator / device
```
adb devices                                 # confirm one device
adb shell input tap <x> <y>                 # navigate
adb shell screencap -p /sdcard/cap.png      # capture
adb pull /sdcard/cap.png <local>            # retrieve
```

For widget-tree-aware navigation, prefer `flutter drive` with a test driver, or `adb shell uiautomator dump` + parse.

### iOS simulator (macOS only)
```
xcrun simctl list devices booted
xcrun simctl io booted screenshot <local>.png
```

If `xcrun` is not available (non-macOS), iOS capture is unsupported — proceed with Android only and mark in CHANGELOG.

## Web (React / Vue / static HTML)

Use playwright if installed (`npx playwright --version` to detect). Else use puppeteer (`npx puppeteer --version`).

```js
// scripts/.iter-uiux-capture.mjs (generated at Phase 1)
import { chromium } from 'playwright';
const [url, outPath] = process.argv.slice(2);
const b = await chromium.launch();
const p = await b.newPage();
await p.setViewportSize({ width: 1440, height: 900 });
await p.goto(url);
await p.screenshot({ path: outPath, fullPage: false });
await b.close();
```

Run: `node scripts/.iter-uiux-capture.mjs <url> <outpath>`.

For interactive states (hover / focus / pressed), extend the script: `await p.hover('selector')` / `await p.focus(...)` / `await p.click(...)` (without releasing for pressed needs special handling — emulate via `page.dispatchEvent`).

For responsive states (RWD), repeat with different viewport sizes; name the state `<base>__<width>x<height>`.

## Capture sequence per state

For each `(unit, round, state)`:
1. Bring the app to entry state for that unit (use `unit.navigation`).
2. Trigger the state interaction (pressed = mousedown without mouseup, etc.).
3. Wait for any animation: 300ms default; 800ms if state name contains `loading`, `skeleton`, or `error`.
4. Capture to the named path.
5. If the captured image is all-black or all-white, wait 500ms and retry once. Persistent failure → see `failure-recovery.md`.

## Pre-capture environment check

At Phase 1, before scanning units, verify the toolchain for the detected platform:

| Platform | Required | Check command |
|---|---|---|
| Flutter (Android) | `adb` | `adb version` |
| Flutter (iOS) | `xcrun simctl` (macOS only) | `xcrun simctl help` |
| Web | `node` + `npx playwright` (or `npx puppeteer`) | `npx playwright --version` |
| Static HTML | same as Web | same |

Missing tools → halt run, report which tool is missing. Do NOT auto-install.
````

- [ ] **Step 3: Run validation**

Expected: `OK`.

---

## Task 8: `references/changelog-format.md`

**Files:**
- Create: `D:/github/iter-uiux/references/changelog-format.md`

Implements spec §7.

- [ ] **Step 1: Write validation command**

```bash
python -c "
t = open('D:/github/iter-uiux/references/changelog-format.md', encoding='utf-8').read()
for s in ['Header','Detected Units','Unit:','Round','Global Consistency','Summary','Archive']:
    assert s in t, f'missing: {s}'
print('OK')
"
```

- [ ] **Step 2: Write `references/changelog-format.md`**

Write to `D:/github/iter-uiux/references/changelog-format.md`:
````markdown
# CHANGELOG format

`.iter-uiux/CHANGELOG.md` is the canonical artifact of every run. Append in lifecycle order. Never edit a prior section after appending the next one (rounds are immutable once written).

## Header (Phase 1)

```markdown
# iter-uiux run @ <ISO 8601 timestamp> (project: <basename of cwd>)

## Environment
- platform: <flutter|react|vue|web|static-html>
- app launched via: <exact command>
- screenshot tool: <adb|simctl|playwright|puppeteer>
- codex CLI: <`codex exec --version` output>
- image flag detected: <flag name>
```

## Detected Units (Phase 2)

```markdown
## Detected Units
| name | raw_label | source | navigation |
|---|---|---|---|
| nav | BottomNavigationBar | lib/widgets/bottom_nav.dart | app launch -> home |
| ticket | TicketCard | lib/features/ticket/ticket_card.dart | nav -> my_tickets |
| ...
```

## Unit section (Phase 3, one per unit)

```markdown
---

## Unit: <name>
- rounds: <N>
- states reviewed: <comma-separated final state list>
- screenshots: .iter-uiux/screenshots/<run-ts>/<unit>/

### Round 1 (advisor)
**Codex output (raw JSON):**
```json
{ "issues": [...], "need_more": [...], "verdicts": [], "done": false }
```
**Issue summary:**
- [<severity>/<area>] <id>: <description>

### Round 2 (claude action)
**Files modified:**
- <path> (+<lines added>/-<lines removed>)
**Mapping notes:**
- <issue_id>: <what was changed, why it addresses this issue>
**Screenshots captured:** <state list>

### Round 2 (reviewer)
**Codex output (raw JSON):**
```json
{ "issues": [...], "need_more": [...], "verdicts": [...], "done": false }
```
**Verdicts:**
- <issue_id>: <status>  — <note if any>
**New issues:** (if any)
- [<severity>/<area>] <id>: <description>

... (repeat for each subsequent round)

### Round K (reviewer)
**Converged at round K.** Codex returned `done: true`.
```

## Global Consistency Review (Phase 4)

```markdown
---

## Global Consistency Review
### Round 1 (advisor)
... (same shape as a unit round; "Files modified" may span many units)
### Round M (reviewer)
**Converged at round M.**
```

## Summary (Phase 5)

```markdown
---

## Summary
- units detected: <count>
- units converged: <count>
- units blocked: <count> (see <unit names>)
- per-unit total rounds: <sum>
- global review rounds: <count>
- files modified (unique): <count>
- run duration: <hh:mm:ss>
```

## Archive rules

- At the start of every `/iter-uiux` invocation, if `.iter-uiux/CHANGELOG.md` exists, rename it to `.iter-uiux/CHANGELOG.<UTC-timestamp>.md` using the timestamp format `YYYY-MM-DDThhmmZ` (no colons, for filename safety on Windows).
- Screenshots from prior runs are NOT moved — they already live under their own `<run-timestamp>/` directory, so they accumulate side-by-side.
- Archived CHANGELOG files are never fed back to codex. They exist purely for the user's audit trail.

## Anti-loop annotation

If the anti-loop guard from `failure-recovery.md` forces convergence, write into the round's reviewer section:

```markdown
### Round K (reviewer)
**Forced convergence.** Codex's issues[] was byte-identical to round K-2's. Marked `diverged, stopped at round K`.
```
````

- [ ] **Step 3: Run validation**

Expected: `OK`.

---

## Task 9: `references/failure-recovery.md`

**Files:**
- Create: `D:/github/iter-uiux/references/failure-recovery.md`

Implements spec §8 (full ladder + red lines).

- [ ] **Step 1: Write validation command**

```bash
python -c "
t = open('D:/github/iter-uiux/references/failure-recovery.md', encoding='utf-8').read()
for s in ['App launch','navigate','Screenshot','codex','Build','converging','Red lines','blocked','halt']:
    assert s in t, f'missing: {s}'
print('OK')
"
```

- [ ] **Step 2: Write `references/failure-recovery.md`**

Write to `D:/github/iter-uiux/references/failure-recovery.md`:
````markdown
# Failure recovery

When something fails, follow the auto-fix steps below before halting. Only halt when truly blocked. Every halt or skip MUST be recorded in `CHANGELOG.md` with a clear reason.

## Failure ladder

### App launch fails
1. Detect the project's launch command from `package.json` `scripts.dev` / `flutter run` / etc.
2. If port conflict (`EADDRINUSE` or equivalent), pick an unused port and retry.
3. If launch hangs > 60s, kill and retry up to 3 times total.
4. On final failure: halt run. Write to CHANGELOG:
   ```
   ## BLOCKER: cannot launch app
   - command attempted: ...
   - error output: ...
   ```

### Cannot navigate to unit
1. Grep source for the route name in `unit.navigation`. Correct if a near-miss exists.
2. Check for dev-login / mock-auth fixtures: look for `.env*`, `seed.*`, `fixtures/`, `dev_login`, `mock_auth`.
3. If found, apply them. If not, **do not fabricate credentials**.
4. After 2 retries with no success: mark `navigation: blocked: <reason>` in the unit's CHANGELOG section, skip the unit, continue to the next one.

### Screenshot tooling missing
1. Confirm which tool is missing via the Phase-1 environment check (see `screenshot-protocol.md`).
2. **Do NOT auto-install SDKs / CLIs.**
3. Halt the run. Write to CHANGELOG:
   ```
   ## BLOCKER: screenshot tooling missing
   - missing: <tool name>
   - platform: <platform>
   ```

### Screenshot blank / all black
1. Wait 500ms, recapture once.
2. If still blank: skip this state. Mark in the unit's CHANGELOG round:
   ```
   - state <name>: skipped (capture failed twice)
   ```
3. Proceed with whatever states succeeded. If all states for a unit fail, mark the unit blocked and skip.

### codex CLI not authed
1. Detect via codex stderr (`not logged in` / `authentication required`).
2. **Do not attempt auth.** Halt run immediately. Write to CHANGELOG:
   ```
   ## BLOCKER: codex CLI not authenticated
   ```

### codex CLI rate limit
1. Detect via stderr (`rate limit`, `429`, `quota`).
2. Exponential backoff: 5s, 30s, 2m, 5m. Retry the same call after each wait.
3. After 4 failed attempts: halt run. Write blocker to CHANGELOG.

### codex rejects image (too large)
1. Detect via stderr (`payload too large` / `413` / image-size error).
2. Downscale the image to max 1920x1080 (preserve aspect) and retry the call.
3. If still rejected: mark the affected state skipped; if all states fail for this unit, mark unit blocked.

### codex returns unparseable output
1. Send one corrective re-prompt:
   ```
   Your prior response did not match the required JSON schema at
   <abs path to assets/codex-output-schema.json>. Reply ONLY with JSON
   matching that schema. No prose, no fences.
   ```
2. Re-validate against the schema.
3. After 2 corrective failures: mark unit `blocked: codex output invalid`, continue with next unit.

### Build / lint fails after Claude edit
1. If purely lint warnings: run the project's lint --fix once (e.g., `dart fix --apply`, `eslint --fix`, `prettier -w`).
2. If build/compile error: revert THIS ROUND's edits only (use `git stash` if a git repo is present; else manual revert per saved diff). Do NOT touch prior rounds.
3. Mark the originating issue `attempt_failed` in the next reviewer call's mapping_notes. The unit proceeds to the next round; codex may suggest a different approach.

### Unit not converging (codex loops)
1. After each reviewer round, compare codex's `issues[]` to the same field two rounds prior (canonicalize: sort by id; compare as JSON).
2. If byte-identical: force convergence. Stop the unit's loop. Write to CHANGELOG:
   ```
   ### Round K (reviewer)
   **Forced convergence.** issues[] identical to round K-2.
   Marked `diverged, stopped at round K`.
   ```
3. Continue to the next unit.

## Red lines (Claude must NOT do these autonomously)

- Install missing SDKs / CLI tools.
- Fabricate mock login credentials or testing users.
- `git commit`, `git push`, open PRs.
- Delete or rename existing files unless a codex `suggested_fix` explicitly says to remove or merge that file/component. No proactive tidying.
- Write to anything outside `.iter-uiux/` or source files that codex explicitly told you to edit.

## Halt vs skip

- **Halt** = stop the entire `/iter-uiux` run. Write `## BLOCKER:` section to CHANGELOG. User must intervene before next run.
- **Skip** = mark this unit (or state) blocked, continue with the next unit. Recorded in that unit's CHANGELOG section.

Use halt only for environmental failures (tools missing, codex not authed, codex rate-limit exhausted, app cannot launch). Use skip for unit-level failures (navigation blocked, repeated capture failure, codex unparseable for this unit).
````

- [ ] **Step 3: Run validation**

Expected: `OK`.

---

## Task 10: `references/consistency-review.md`

**Files:**
- Create: `D:/github/iter-uiux/references/consistency-review.md`

Implements spec §3 Phase 4 + §4 invariant 4 (global convergence rule).

- [ ] **Step 1: Write validation command**

```bash
python -c "
t = open('D:/github/iter-uiux/references/consistency-review.md', encoding='utf-8').read()
for s in ['Phase 4','Global','convergence','done','design system','cross-unit']:
    assert s in t, f'missing: {s}'
print('OK')
"
```

- [ ] **Step 2: Write `references/consistency-review.md`**

Write to `D:/github/iter-uiux/references/consistency-review.md`:
````markdown
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

Write the global section under `## Global Consistency Review` per the format in `changelog-format.md`. Each round writes the same fields a unit round writes (codex JSON, files modified, mapping notes, verdicts).

## Convergence

Codex returning `done: true` ends Phase 4. Proceed to Phase 5 (Summary).

If forced-convergence triggers (anti-loop), record it as in `failure-recovery.md` and proceed anyway — Phase 5 records the unresolved item count.
````

- [ ] **Step 3: Run validation**

Expected: `OK`.

---

## Task 11: Final `README.md`

**Files:**
- Modify: `D:/github/iter-uiux/README.md`

Replace the Task-1 stub with a complete README aimed at github visitors.

- [ ] **Step 1: Write validation command**

```bash
python -c "
t = open('D:/github/iter-uiux/README.md', encoding='utf-8').read()
for s in ['iter-uiux','Install','~/.claude/skills/iter-uiux','codex','/iter-uiux','CHANGELOG','spec']:
    assert s in t, f'missing: {s}'
print('OK')
"
```

- [ ] **Step 2: Write final `README.md`**

Write to `D:/github/iter-uiux/README.md`:
````markdown
# iter-uiux

A Claude Code skill that drives a strict UI/UX review-and-fix loop on any project, using Codex CLI as a tough UI/UX supervisor.

Run `/iter-uiux` in your project root. Claude scans the UI for operation units (nav, topbar, banner, alert, ticket, toast, ...), iterates Codex review → Claude edit → Codex re-review until each unit converges, then runs a final cross-unit consistency pass. Every step is logged to `.iter-uiux/CHANGELOG.md`.

Fully automated — no per-round approval prompts.

## Install

```bash
git clone https://github.com/<user>/iter-uiux.git
# either symlink:
ln -s "$(pwd)/iter-uiux" ~/.claude/skills/iter-uiux
# or copy:
cp -r iter-uiux/* ~/.claude/skills/iter-uiux/
```

After install, the `iter-uiux` skill is auto-discoverable. Invoke with `/iter-uiux` in any Claude Code session.

## Requirements

- `codex` CLI installed and authenticated (`codex exec --version` must work)
- For Flutter projects: `adb` (Android) and/or `xcrun simctl` (iOS, macOS only)
- For Web projects: `node` + `npx playwright` (or puppeteer)
- A running app (the skill will try to launch it for you)

## Usage

```bash
cd path/to/your/project
claude
> /iter-uiux
```

The skill operates fully automatically. When it finishes, read `.iter-uiux/CHANGELOG.md` for the full record.

Re-running `/iter-uiux` rotates the old CHANGELOG to a timestamped archive (`.iter-uiux/CHANGELOG.<UTC-timestamp>.md`) and starts fresh. Old screenshots accumulate under `.iter-uiux/screenshots/<run-timestamp>/`.

## How it works

Five phases:

1. **Environment prep** — detect project type, launch app, prepare working dir
2. **Unit scan** — extract operation units from routes / components
3. **Per-unit iteration** — Codex advises → Claude fixes → Codex reviews, loop until Codex says "no further suggestions"
4. **Global consistency review** — Codex finds cross-unit design-system inconsistencies; Claude fixes
5. **Summary**

See [`2026-05-22-iter-uiux-design.md`](./2026-05-22-iter-uiux-design.md) for the full spec.

## What Codex evaluates

Visual, interaction, copy, structure, accessibility (a11y), responsive (RWD), internationalisation (i18n), and design-system consistency. Codex itself decides how deep to go and demands additional screenshots / states / source / flow descriptions as needed.

## What Claude will NOT do autonomously

- Install missing SDKs or CLIs
- Fabricate test credentials
- Commit, push, or open PRs (the skill only modifies the working tree — you own git)
- Delete / rename files unless Codex explicitly directs it
- Touch anything outside `.iter-uiux/` or the source files Codex flagged

## Status

In active development. Tracking the implementation plan in [`2026-05-22-iter-uiux-plan.md`](./2026-05-22-iter-uiux-plan.md).
````

- [ ] **Step 3: Run validation**

Expected: `OK`.

---

## Task 12: End-to-end dry-run validation

**Files:** none (read-only checks)

Verify the skill structure is internally consistent and Claude can navigate it without dead ends.

- [ ] **Step 1: Confirm all expected files exist**

```bash
python -c "
import os
root = 'D:/github/iter-uiux'
expected = [
    'README.md','SKILL.md',
    '2026-05-22-iter-uiux-design.md','2026-05-22-iter-uiux-plan.md',
    '.gitignore',
    'references/codex-prompt.md',
    'references/unit-scan.md',
    'references/screenshot-protocol.md',
    'references/changelog-format.md',
    'references/failure-recovery.md',
    'references/consistency-review.md',
    'assets/codex-output-schema.json',
    'assets/unit-dictionary.yaml',
]
missing = [p for p in expected if not os.path.exists(os.path.join(root,p))]
assert not missing, f'missing: {missing}'
print('OK', len(expected), 'files')
"
```
Expected: `OK 13 files`.

- [ ] **Step 2: Link check — every reference SKILL.md cites must exist**

```bash
python -c "
import re, os
root = 'D:/github/iter-uiux'
t = open(os.path.join(root,'SKILL.md'), encoding='utf-8').read()
refs = re.findall(r'references/[a-z\-]+\.md', t)
for r in set(refs):
    p = os.path.join(root, r)
    assert os.path.exists(p), f'SKILL.md cites missing file: {r}'
print('OK', len(set(refs)), 'unique refs')
"
```
Expected: `OK 6 unique refs`.

- [ ] **Step 3: Schema parse — confirm JSON is valid and YAML is valid**

```bash
node -e "JSON.parse(require('fs').readFileSync('D:/github/iter-uiux/assets/codex-output-schema.json','utf8')); console.log('JSON OK')"
python -c "import yaml; yaml.safe_load(open('D:/github/iter-uiux/assets/unit-dictionary.yaml')); print('YAML OK')"
```
Expected: `JSON OK`, `YAML OK`.

- [ ] **Step 4: Cross-reference — every reference doc mentioned in design spec §9 exists**

```bash
python -c "
import os, re
root = 'D:/github/iter-uiux'
spec = open(os.path.join(root,'2026-05-22-iter-uiux-design.md'), encoding='utf-8').read()
mentioned = set(re.findall(r'references/[a-z\-]+\.md', spec))
for m in mentioned:
    assert os.path.exists(os.path.join(root,m)), f'spec mentions missing file: {m}'
print('OK', len(mentioned), 'spec-mentioned refs all exist')
"
```
Expected: `OK <N> spec-mentioned refs all exist`.

- [ ] **Step 5: Dry-walkthrough — read SKILL.md as if you were Claude and answer:**

For each lifecycle phase, can you answer "which reference doc do I read?" without ambiguity? Run through the Reference map table in SKILL.md. If any phase has no clear reference target, fix SKILL.md.

This is a manual check. No command.

Expected: every phase maps to exactly one reference doc.

---

## Task 13: git init + initial commit

**Files:**
- Create: `D:/github/iter-uiux/.git/` (via git init)

- [ ] **Step 1: Confirm with the user before git init**

git was not initialized at brainstorming time (per the spec note: "Is a git repository: false"). Before proceeding, ask the user once: "Initialize this repo as git now and make the initial commit?"

If user says no → stop here. If yes → continue.

- [ ] **Step 2: git init**

```bash
cd D:/github/iter-uiux && git init -b main
```
Expected: `Initialized empty Git repository in D:/github/iter-uiux/.git/`.

- [ ] **Step 3: Stage**

```bash
cd D:/github/iter-uiux && git add .
```

- [ ] **Step 4: Initial commit**

```bash
cd D:/github/iter-uiux && git commit -m "$(cat <<'EOF'
feat: initial iter-uiux skill scaffold

Spec, plan, SKILL.md driver, six reference docs, JSON schema + YAML
dictionary, and README. Ready for first end-to-end shakedown against a
real project.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```
Expected: `[main (root-commit) ...] feat: initial iter-uiux skill scaffold`.

- [ ] **Step 5: Verify**

```bash
cd D:/github/iter-uiux && git status && git log --oneline
```
Expected: clean working tree; one commit on `main`.

---

## Self-review

After all tasks complete, verify spec coverage:

| Spec section | Covered by |
|---|---|
| §1 Purpose | README.md, SKILL.md frontmatter |
| §2 Roles | SKILL.md preamble |
| §3 Lifecycle | SKILL.md + each references/*.md |
| §4 Invariants | SKILL.md Invariants section |
| §5 Codex modes | references/codex-prompt.md + assets/codex-output-schema.json |
| §6 Unit scan | references/unit-scan.md + assets/unit-dictionary.yaml |
| §7 CHANGELOG | references/changelog-format.md |
| §8 Failure / red lines | references/failure-recovery.md + SKILL.md red lines |
| §9 Repo layout | this plan's file inventory |
| §10 Open items | resolved-in-place: image flag detection in codex-prompt.md |
| §11 Out of scope | README "What Claude will NOT do" |

If any cell is empty when implementation is done, add a task to fill it.

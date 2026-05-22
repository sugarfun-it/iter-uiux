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

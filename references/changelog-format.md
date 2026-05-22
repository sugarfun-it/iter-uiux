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

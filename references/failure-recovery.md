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

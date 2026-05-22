# iter-uiux

A Claude Code skill that drives a strict UI/UX review-and-fix loop on any project, using Codex CLI as a tough UI/UX supervisor.

Run `/iter-uiux` in your project root. Claude scans the UI for operation units (nav, topbar, banner, alert, ticket, toast, ...), iterates Codex review → Claude edit → Codex re-review until each unit converges, then runs a final cross-unit consistency pass. Every step is logged to `.iter-uiux/CHANGELOG.md`.

Fully automated — no per-round approval prompts.

## Install

```bash
git clone https://github.com/sugarfun-it/iter-uiux.git
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

Initial implementation complete. See [`2026-05-22-iter-uiux-plan.md`](./2026-05-22-iter-uiux-plan.md) for the implementation record. First real-project shakedown still pending.

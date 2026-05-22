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
// .iter-uiux/capture.mjs (generated at Phase 1)
import { chromium } from 'playwright';
const [url, outPath] = process.argv.slice(2);
const b = await chromium.launch();
const p = await b.newPage();
await p.setViewportSize({ width: 1440, height: 900 });
await p.goto(url);
await p.screenshot({ path: outPath, fullPage: false });
await b.close();
```

Run: `node .iter-uiux/capture.mjs <url> <outpath>`.

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

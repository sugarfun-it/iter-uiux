# Screenshot protocol

Platform-specific recipes for launching the app, navigating to a unit state, and capturing screenshots. Every `(unit, round, state)` produces **two** images by default: a `viewport` shot (initial fold) and a `fullpage` shot (entire scroll height), so codex can judge both first-impression and overall layout / cross-section consistency.

## Naming convention

```
.iter-uiux/screenshots/<run-timestamp>/<unit>/round<N>-<state>-<view>.png
```

`<view>` ∈ `{viewport, fullpage}`.

Examples:
- `.iter-uiux/screenshots/2026-05-22T1430Z/nav/round1-default-viewport.png`
- `.iter-uiux/screenshots/2026-05-22T1430Z/nav/round1-default-fullpage.png`
- `.iter-uiux/screenshots/2026-05-22T1430Z/ticket/round3-with_long_copy-fullpage.png`

`<run-timestamp>` is set at the start of the run and reused for all screenshots in that run. Reuse the **same** state names across rounds so the reviewer can compare round-N vs round-N+1 directly.

## Flutter

Prefer existing patterns from `flutter-device-iter` if it is installed in the operator's environment.

Native screencap is inherently viewport-only. For Flutter, write the device shot to `<outBase>-viewport.png` and **also copy** it to `<outBase>-fullpage.png` so downstream filename expectations hold. For screens whose meaningful content extends beyond the viewport (long lists, scrollable forms), capture multiple scroll positions as separate states named `<base>__scroll<idx>` (e.g. `default__scroll0`, `default__scroll1`); each gets its own viewport/fullpage pair.

### Android emulator / device
```
adb devices                                 # confirm one device
adb shell input tap <x> <y>                 # navigate
adb shell screencap -p /sdcard/cap.png      # capture
adb pull /sdcard/cap.png <outBase>-viewport.png
cp <outBase>-viewport.png <outBase>-fullpage.png
```

For widget-tree-aware navigation, prefer `flutter drive` with a test driver, or `adb shell uiautomator dump` + parse.

### iOS simulator (macOS only)
```
xcrun simctl list devices booted
xcrun simctl io booted screenshot <outBase>-viewport.png
cp <outBase>-viewport.png <outBase>-fullpage.png
```

If `xcrun` is not available (non-macOS), iOS capture is unsupported — proceed with Android only and mark in CHANGELOG.

## Web (React / Vue / static HTML)

Use playwright if installed (`npx playwright --version` to detect). Else use puppeteer (`npx puppeteer --version`).

```js
// .iter-uiux/capture.mjs (generated at Phase 1)
// Produces BOTH viewport and fullpage shots. Pass the outPath without the
// `-viewport.png` / `-fullpage.png` suffix; the script appends them.
import { chromium } from 'playwright';
const [url, outBase] = process.argv.slice(2);
const b = await chromium.launch();
const p = await b.newPage();
await p.setViewportSize({ width: 1440, height: 900 });
await p.goto(url);
await p.screenshot({ path: `${outBase}-viewport.png`, fullPage: false });
await p.screenshot({ path: `${outBase}-fullpage.png`, fullPage: true });
await b.close();
```

Run: `node .iter-uiux/capture.mjs <url> <outBase>` where `<outBase>` is the path **without** the `-viewport.png` / `-fullpage.png` suffix (e.g. `.iter-uiux/screenshots/.../round1-default`).

For interactive states (hover / focus / pressed), extend the script: `await p.hover('selector')` / `await p.focus(...)` / `await p.click(...)` (without releasing for pressed needs special handling — emulate via `page.dispatchEvent`).

For responsive states (RWD), repeat with different viewport sizes; name the state `<base>__<width>x<height>`.

## Capture sequence per state

For each `(unit, round, state)`:
1. Bring the app to entry state for that unit (use `unit.navigation`).
2. Trigger the state interaction (pressed = mousedown without mouseup, etc.).
3. Wait for any animation: 300ms default; 800ms if state name contains `loading`, `skeleton`, or `error`.
4. Capture **both** `<outBase>-viewport.png` and `<outBase>-fullpage.png` (per platform recipe above).
5. If either captured image is all-black or all-white, wait 500ms and retry once. Persistent failure → see `failure-recovery.md`.

## Pre-capture environment check

At Phase 1, before scanning units, verify the toolchain for the detected platform:

| Platform | Required | Check command |
|---|---|---|
| Flutter (Android) | `adb` | `adb version` |
| Flutter (iOS) | `xcrun simctl` (macOS only) | `xcrun simctl help` |
| Web | `node` + `npx playwright` (or `npx puppeteer`) | `npx playwright --version` |
| Static HTML | same as Web | same |

Missing tools → halt run, report which tool is missing. Do NOT auto-install.

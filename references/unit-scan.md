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

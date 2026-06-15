# CLAUDE.md

Guidance for AI assistants (and humans) working in this repository.

## What this is

**记账 (jìzhàng, "bookkeeping")** — a tiny, installable PWA for tracking
daily personal expenses. The UI language is **Simplified Chinese**.

It is a **zero-dependency, no-build, single-page app**. The entire application
lives in one file: `index.html` (HTML + CSS + vanilla JS, ~495 lines). There is
no framework, bundler, package manager, transpiler, or test runner. Data is
stored locally in the browser via `localStorage`; there is no backend and no
network API.

## File layout

| File | Purpose |
|------|---------|
| `index.html` | The whole app — markup, styles (`<style>`), and logic (`<script>`). Edit this for any feature/UI/logic change. |
| `manifest.json` | PWA manifest (name 记账, theme color, icons, standalone display). |
| `sw.js` | Service worker. Cache-first offline strategy. Cache name is versioned: `jizhang-vN`. |
| `icon-180.png` | Apple touch icon (180×180). |
| `icon-512.png` | PWA icon (512×512, `any maskable`). |

There are no other source directories, config files, or CI workflows.

## Architecture (inside `index.html`)

The `<script>` block is organized into clearly commented sections:

- **数据 (Data)** — `STORE_KEY = 'jizhang_records_v1'`, the `CATS` category
  list, and mutable module state: `records`, `amount`, `selectedCat`,
  `editingId`, `statMonth`.
- **工具 (Utils)** — formatting/date helpers: `money`, `pad`, `todayStr`,
  `curMonth`, `parseDate`, `dateLabel`, `uid`, `toast`, `escapeHtml`.
- **记账页 (Add tab)** — the entry form: category grid, a custom **calculator
  keypad** (`amount` holds an expression string like `"12.5+30-2"`, evaluated by
  `evalExpr`), date, note, save/update/delete.
- **明细页 (List tab)** — today/month summary plus records grouped by day;
  tapping a row opens it for editing. Also JSON export/import (backup).
- **统计页 (Stats tab)** — month navigation, category breakdown rendered as a
  hand-built **SVG donut** (`donutSVG`) + legend, and a per-day bar chart
  (`renderBars`).
- **导航 (Nav)** — `switchTab(name)` toggles the three `<section class="tab">`
  panels and the bottom nav; it re-renders the target tab on switch.
- **绑定 & 启动 (Bind & boot)** — wires DOM event handlers, runs the initial
  render, and registers the service worker (only over http/https, skipped on
  `file://`).

### Record shape

A record stored in `records` (and serialized to `localStorage`):

```js
{ id, amount /* number */, category /* CATS key */, date /* 'YYYY-MM-DD' */, note, ts /* ms epoch */ }
```

### Categories

`CATS` is the single source of truth for categories — each has `key`, `nm`
(name), `emo` (emoji), and `color` (used by the donut/legend). Records reference
a category by `key`; `catOf(key)` resolves it and falls back to `other`.

## Conventions

- **Single file:** keep app logic in `index.html`. Do not introduce a build step,
  npm dependencies, or split files unless explicitly asked.
- **Vanilla JS only:** plain DOM APIs, template strings for rendering, direct
  `el.onclick = ...` handlers. Match the existing terse, comment-as-you-go
  style. Comments are in Chinese — keep new user-facing strings and comments in
  Chinese to match.
- **Render pattern:** functions named `render*` rebuild a section's `innerHTML`
  from state and re-attach handlers. Mutate the module-level state, then call
  the relevant `render*` / `persist()`.
- **Persistence:** call `persist()` after any change to `records`; it writes
  `localStorage` and refreshes the header subtitle.
- **Escaping:** user-supplied text (notes) is inserted via `escapeHtml()` — keep
  doing this when injecting user data into `innerHTML`.
- **Money/dates:** format money with `money()` and use `todayStr()` /
  `curMonth()` / `pad()` for dates rather than re-deriving formats.

### Versioning caches and storage

- If you change the offline asset list or want to force clients to update,
  **bump the cache name** in `sw.js` (`jizhang-v2` → `jizhang-v3`). The
  `activate` handler deletes old caches automatically.
- If you change the persisted record schema in a breaking way, bump
  `STORE_KEY` (`jizhang_records_v1` → `_v2`) and handle migration, since old
  data lives in users' browsers.

## Running / testing

No build and no test suite. To run locally you must serve over HTTP so the
service worker registers (opening `index.html` as `file://` works for the app
but skips the SW):

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

Manual verification checklist after changes:

1. **记账** — enter an amount via the keypad (try an expression like `12+5-3`,
   then `=`), pick a category, save → toast appears, lands on 明细.
2. **明细** — record shows under the correct day; tap it to edit/delete;
   today/month summaries update. Export then re-import the JSON backup.
3. **统计** — donut, legend percentages, and daily bars render; month
   navigation (`‹` / `›`) works and the empty state shows for empty months.
4. Reload the page — data persists (localStorage).

## Git / workflow

- Active development branch: `claude/claude-md-docs-d7w2si`. Develop, commit, and
  push there; open a **draft PR** after pushing.
- Commit messages in this repo are short and imperative (e.g. `add sw.js`).
- Keep changes minimal and focused; this is a small, self-contained app.

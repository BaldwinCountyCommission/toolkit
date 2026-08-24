# 08 — Front-End: Theme, Navigation, Conventions

Everything about the client side: how panes switch, how deep-links work, the theme tokens, and the house rules for adding UI so new work matches the existing look and dark mode.

## Loaded libraries
Bootstrap (grid, form controls, utilities) and Font Awesome (icons) from CDN, plus Google Fonts. No framework — plain DOM + IIFEs. Monospace numerals use **JetBrains Mono**.

## Theme tokens (CSS custom properties)
The entire look is driven by CSS variables on `:root`, with a light/dark mode. Use these variables — never hard-coded colors — so new UI themes correctly in both modes.

Core tokens:
- Text: `--text`, `--text2`, `--text3` (primary → most muted)
- Surfaces: `--surface`, `--surface2`, `--surface3`, `--bg`, `--bg2`, `--card-bg`
- Borders: `--border`, `--border2`, `--card-border`
- Accent (blue): `--accent`, `--accent-bright`, `--accent-dim`, `--accent-glow`
- Status: `--green`/`--green-bright`/`--green-dim`, `--amber`, `--red`/`--red-bright`/`--red-dim`, plus `--indigo*`
- Inputs/modals: `--input-bg`, `--input-border`, `--input-focus-bg`, `--modal-bg`, `--overlay`
- Sidebar: `--sb-bg`, `--sb-border`, `--sb-w`
- Motion: `--t-fast`, `--t-med`, `--ease`

### Dark-mode pitfall (learned the hard way)
Bootstrap's `.table` class sets `--bs-table-color` / `--bs-table-bg` at a specificity that overrides the theme and renders light-on-light in dark mode. **Do not put `.table`/`.table-sm` on data tables.** Use a self-styled table class (like `.nr-table` / `.wr-table`) with explicit `color: var(--text2)` and `background: transparent` on cells. The Ninja Reporting table is the corrected reference.

## Panes and the sidebar
Each feature is a `<div class="tool-pane" id="pane-…">`. The sidebar (`.sb-item` with `data-target="pane-…"`) switches the active pane. Panes gated by role are wrapped server-side in `<% If canSeeSecOps Then %> … <% End If %>` so they aren't even emitted for unauthorized users.

## Navigation core & URL slugs
A single navigation IIFE handles pane switching and URL state. Two maps:
- **`PANE_SLUGS`**: `pane-id → url-slug` (and its inverse `SLUG_PANES`). E.g. `pane-ninjarpt → 'ninjareporting'`, `pane-tools → 'tools'`. Adding a deep-linkable pane means adding it here.
- **`SB_TARGETS`**: slug → action for **shadowbox** deep-links, e.g. `'phonetic': () => document.getElementById('btnPhonetic')?.click()`.

Deep-link forms:
- `?pane=<slug>` opens a pane on load.
- `?sb=<slug>` opens a shadowbox tool on load (via `SB_TARGETS`).
- Report Center sub-panes also have a restore list so a refresh while on a sub-pane returns there and re-loads its data.

## Report Center hub pattern
- Hub tiles: `<button class="rc-open" data-rc-target="pane-…">`.
- Sub-pane back button: `<button class="rc-back">` → returns to `pane-reportcenter`.
- Sub-pane data loads on first tile click (each report IIFE binds to its `.rc-open`).

## Dynamic link tiles
The `#dyn-tools`, quicklinks, infographics, secops, and sysadmin grids are populated at runtime from the GitHub JSON files. When they finish loading, a **`dynLinksLoaded`** DOM event fires. Features that depend on tiles (global search index, shadowbox button wiring like the phonetic tool) **re-bind on this event**, because the tiles are re-created each load. If you add a feature that hooks a dynamically-rendered button, bind on load *and* on `dynLinksLoaded`.

## Shadowbox (in-page modal) pattern
Used by the dashboard, county map, Copilot board, and the phonetic tool. A hidden `.shadowbox-backdrop` + `.shadowbox` markup block; an IIFE opens it (on the tile's button id), closes on the ×, backdrop click, or Escape, and toggles `?sb=<slug>` in the URL. To add one: drop the hidden markup near the other shadowboxes, add an opener IIFE, and register the slug in `SB_TARGETS`.

## Custom DOM events in use
- `dynLinksLoaded` — GitHub link tiles finished loading/re-rendering.
- `ocDataReloaded` — org chart roster was refreshed (search re-pulls people).

## Global search
Top-bar box (Ctrl/Cmd+K or `/`). Indexes live DOM entries plus org-chart people. Tiered scoring (exact > prefix > word-start > substring > subsequence) for primary text; a **stricter** scorer (no subsequence) for secondary haystacks like email/extension so long concatenated fields don't over-match. People results route through `window.ocFocusPerson`.

## Conventions for new UI
1. Use theme variables, never literal colors.
2. Self-style data tables; avoid Bootstrap `.table`.
3. Gate privileged markup server-side **and** re-check the role in any handler it calls.
4. Give deep-linkable panes a slug; give shadowboxes an `SB_TARGETS` entry.
5. Re-bind anything attached to dynamically-rendered tiles on `dynLinksLoaded`.
6. Keep each feature in its own IIFE; communicate across features with DOM events, not globals (except the few documented `window.*` APIs like `ocEnsureData`/`ocFocusPerson`).
7. Money/number/time columns: monospace (`JetBrains Mono`), right-aligned.

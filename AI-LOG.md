# AI Development Log

Running log of AI-assisted work on this theme. One entry per task/PR: what was done, what the
merchant/reviewer changed or rejected, and open questions.

---

## 2026-10-08 — Native color_scheme_group migration (4 PRs)

**Why:** reviewing the announcement bar and card blocks surfaced that several places in the theme
(`sections/header-announcements.liquid`, `blocks/_product-card.liquid`/`product-card.liquid`,
`blocks/_collection-card.liquid`/`collection-card.liquid`, `blocks/_card.liquid`) exposed Horizon's
native, *unconstrained* `background_color` swatch — any hex color, with text contrast computed
dynamically — rather than being limited to the Beast & Branch brand palette. This was a real
problem beyond what the earlier design-system phases covered (the "additional color schemes"
phase only addressed `sections/section.liquid` and `sections/footer.liquid`, via a custom
`color_mood` select + static CSS classes, `bb-schemes.css`).

Confirmed against Shopify's current docs (via the shopify-dev-mcp tools) that `color_scheme_group`
is still fully supported, current theme architecture — not deprecated — and that a scheme group's
`definition` array is fully custom, not limited to Dawn's classic background/text/button set. This
meant the theme's complete token set (surface, ornament, border-strong, every button/input role)
could be represented as real, merchant-editable swatches via the native mechanism, with no loss of
fidelity versus the CSS-class approach.

**Decision (confirmed with reviewer):** migrate fully to native `color_scheme_group`/`color_scheme`
settings, retiring the custom `color_mood`/`bb-schemes.css` approach entirely rather than running
two parallel scheme systems. Done as four sequenced commits/PRs:

1. **Define the scheme group** — `color_schemes` color_scheme_group in
   `config/settings_schema.json`/`settings_data.json`, six schemes (Parchment, Ink, Beast,
   Midnight, Branch, Moss), each with the full ~24-token set. Values copied verbatim from
   `design/Beast & Branch Design System/tokens/colors.css`.
2. **Migrate section/footer** — `sections/section.liquid` and `sections/footer.liquid` off
   `color_mood` onto the native `color_scheme` setting (removing their free `background_color`
   swatch entirely, since letting both coexist on one element would fight over which one paints
   it). Added `snippets/bb-color-scheme-styles.liquid`, which loops over `settings.color_schemes`
   and emits a `.color-scheme-{id}` class per scheme — retires the static `bb-schemes.css` asset.
   `<body>` now carries `color-scheme-scheme-parchment` so brand-only tokens (ornament, surface,
   etc.) are available sitewide. `sections/_blocks.liquid` (Shopify's auto-generated AI-block
   wrapper) was deliberately left on the old `background_color`/contrast-override path, since it
   doesn't define `color_scheme` — the shared `snippets/section.liquid` checks for `color_scheme`
   first and falls back cleanly.
3. **Announcement bar** — same treatment, the section that prompted this work. Defaults to
   `scheme-ink`.
4. **Card blocks** — same treatment for `_product-card`/`product-card`, `_collection-card`/
   `collection-card`, and `_card`. The two shared snippets (`product-card.liquid`,
   `collection-card.liquid`) each needed the same color_scheme-first-else-fallback logic, since
   one of their callers (`_featured-product.liquid`) defines neither setting.

Each step validated with `shopify theme check` (0 new errors/warnings vs. the `staging` baseline
each time) and Shopify's `validate_theme` MCP tool.

**Open questions / flags carried forward:**
- Not yet visually verified in the theme editor (no linked store in this environment) — worth a
  manual pass per scheme per section type before considering this fully done.
- The announcement bar and generic sections still allow picking *any* of the 6 schemes freely;
  if certain schemes should only be available on certain section types (e.g. Midnight/Moss only
  as "feature section" variants, per the original design intent), that's a further restriction
  not yet applied — right now all 6 are offered everywhere a `color_scheme` setting exists.
- Variant swatch colors (`--color-variant-*`) still aren't scheme-aware (same gap noted in the
  earlier design-system phase) — carried forward, not addressed here.
- Everything from the Phases 1–7 log below still applies; that work predates this migration and
  is unaffected by it except where explicitly noted above.

---

## 2026-09-26 — Beast & Branch design system, Phases 1–7 (PRs #1–#7)

Implemented the Beast & Branch design system (`/design`) on top of Horizon, planned as 7 small
PRs, one per phase, each merged into `staging` individually:

1. **Parchment default palette + self-hosted fonts** — theme settings for the Parchment color
   palette; self-hosted Grenze/Alegreya/Alegreya SC (not in Shopify's font library) via a new
   `bb-fonts` snippet overriding Horizon's font-role CSS variables.
2. **Buttons** — radius/border-width settings, plus `bb-buttons.css` for the design's inset
   "hairline" and press-state effects Horizon doesn't natively support.
3. **Product card & badges** — image-only border, square ratio, subtle-zoom hover tuning, badge
   position/colors via existing settings.
4. **Ornament dividers** — new `_ornament-divider` block/section (hairline + glyph, CSS/Unicode
   only), registered into the footer's existing block slot.
5. **Additional color schemes** (Ink, Beast, Midnight, Branch, Moss) — built as CSS classes
   (`bb-schemes.css`) applied via a custom `color_mood` select setting. **Superseded 2026-10-08**
   — see the entry above.
6. **Header/footer decorative touches** — stars divider above the footer, double ornament-color
   rule under the header via CSS override (Horizon's border settings can't express `double`).
7. **New badge kinds + ornate frame** — extracted badge markup into a shared snippet to add
   New/Beast/Branch/Clyde's Pick kinds (tag-driven); added a reusable `ornate-frame` snippet
   (double-ruled border, lozenge corners) for future Lore Card/Notice Board-style components.

One merge conflict (Phase 3 vs. Phase 2, both touching `settings_data.json`/`theme.liquid`) was
resolved as a union of both phases' changes — no logic lost either side.

**Open questions / flags carried forward:**
- Non-English locale files were *not* touched for any new custom-brand strings (badge kind
  labels, ornament-divider settings) — these are hardcoded English literals rather than
  machine-translated placeholders, a deliberate choice over guessing at 22 languages.
- No Monty Manticore or Branch-side art exists yet (design system's own gap, not a code gap).
- The Wordmark/logo ampersand is an explicit design stand-in, not final.
- `ornate-frame.liquid` is not yet wired into any section/block (Theme Check's `OrphanedSnippet`
  warning on it is expected).
- Homepage (`templates/index.json`) is still Horizon's stock demo content — none of the actual
  Beast & Branch homepage (hero, Beast/Branch collections, Clyde's Picks, Bestiary, Notice Board)
  has been assembled. A `LoreCard` component (for the Bestiary) doesn't exist yet, and per-card
  scheme switching (one card dark, one light, side by side) isn't supported — `color_mood` only
  reached the section level (now superseded, see above).

---

## 2026-10-08 — CLAUDE.md workflow rules (PR #8)

Committed the project context and workflow rules (feature branches, small PRs, don't touch
merchant-owned `settings_data.json`/template JSON unless required, log tasks here, prefer new
`bb-*` files over core edits) that had been drafted directly in CLAUDE.md but not yet committed.

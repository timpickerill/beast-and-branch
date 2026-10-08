# AI Development Log

Running log of AI-assisted work on this theme. One entry per task/PR: what was done, what the
merchant/reviewer changed or rejected, and open questions.

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
   (`bb-schemes.css`) applied via a custom `color_mood` select setting, since this Horizon fork
   doesn't use Shopify's native `color_scheme_group` mechanism. **Superseded 2026-10-08** — see
   below.
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
  reached the section level.

---

## 2026-10-08 — CLAUDE.md workflow rules (PR #8)

Committed the project context and workflow rules (feature branches, small PRs, don't touch
merchant-owned `settings_data.json`/template JSON unless required, log tasks here, prefer new
`bb-*` files over core edits) that had been drafted directly in CLAUDE.md but not yet committed.

---

## 2026-10-08 — Native color_scheme_group migration (in progress)

**Why:** reviewing the announcement bar and card blocks surfaced that several places in the theme
(`sections/header-announcements.liquid`, `blocks/_product-card.liquid`,
`blocks/_collection-card.liquid`, `blocks/_card.liquid`) expose Horizon's native, *unconstrained*
`background_color` swatch — any hex color, with text contrast computed dynamically — rather than
being limited to the Beast & Branch brand palette. This is a real problem beyond what Phase 5
covered (Phase 5 only addressed `sections/section.liquid` and `sections/footer.liquid`, via a
custom `color_mood` select + static CSS classes).

Confirmed against Shopify's current docs that `color_scheme_group` is still fully supported,
current theme architecture (not deprecated), and — importantly — that a scheme group's
`definition` array is fully custom, not limited to Dawn's classic background/text/button set. This
means our complete token set (surface, ornament, border-strong, every button/input role) can be
represented as real, merchant-editable swatches via the native mechanism, not just a reduced
subset.

**Decision (confirmed with reviewer):** migrate fully to native `color_scheme_group`/`color_scheme`
settings, in four sequenced steps, retiring the Phase 5 custom `color_mood`/`bb-schemes.css`
approach entirely rather than running two parallel scheme systems:
1. Define the `color_scheme_group` (6 schemes, full token set) in `config/settings_schema.json` /
   `settings_data.json`.
2. Migrate `sections/section.liquid` and `sections/footer.liquid` off `color_mood` onto the native
   `color_scheme` setting.
3. Add `color_scheme` to the announcement bar (the original complaint).
4. Add `color_scheme` to the three card blocks.

Each step as its own small commit/PR, per the new workflow rules.

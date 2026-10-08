# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Shopify theme forked from **Horizon** (Shopify's flagship first-party theme), built on Liquid Storefronts with theme blocks. There is no build step — Liquid, CSS, and JS assets are served directly by the Shopify CLI / Shopify servers. There is no root `package.json`, bundler, or test suite in this repo.

The repo tracks Shopify's Horizon repo as an `upstream` remote so updates can be pulled in:

```sh
git remote -v          # origin = this repo, upstream = Shopify/horizon
git fetch upstream
git pull upstream main
```

When resolving merge conflicts from an upstream pull, preserve local customizations rather than blindly taking upstream's side.

## Commands

There is no npm/build tooling. Development happens via the Shopify CLI:

```sh
shopify theme dev       # local dev server with live reload, connected to a store
shopify theme check     # lint Liquid/JSON via Theme Check
shopify theme push      # push local theme to a store
shopify theme pull      # pull theme from a store
```

Theme Check is the only "linter"/"test" tool in this repo — there's no `.theme-check.yml` override, so it runs with default rules. Run it before committing Liquid changes. Horizon (upstream) also runs Theme Check in CI on every commit via `Shopify/theme-check-action`, though that workflow config isn't present in this repo.

## Architecture

### Directory roles (standard Shopify theme layout)

- `layout/theme.liquid` — root HTML shell; wires up `content_for_header`, global snippets (`stylesheets`, `scripts`, `fonts`, `meta-tags`), and renders section groups via `{% sections 'header-group' %}` / `{% sections 'footer-group' %}`.
- `sections/` — top-level, placeable sections. Files prefixed with `_` (e.g. under `blocks/`) are private/non-addable partials, not directly insertable in the theme editor.
- `blocks/` — theme blocks (the newer Liquid Storefronts block architecture), composable inside sections via `{% content_for 'blocks' %}`. Most are private (`_`-prefixed) building blocks composed by sections.
- `snippets/` — reusable Liquid partials (145+), rendered via `{% render 'name' %}`.
- `templates/` — JSON templates (most) or `.liquid` (legacy, e.g. `gift_card.liquid`, `password.liquid`) that compose sections for each page type.
- `assets/` — all JS/CSS/SVG. No bundler: JS ships as native ES modules loaded via an import map (see below). SVGs are inlined via `render 'icon-x'`-style snippets or `asset_url`.
- `config/settings_schema.json` / `settings_data.json` — theme editor settings definitions and current values.
- `locales/` — translation JSON, split into storefront strings (`xx.json`) and theme-editor/schema strings (`xx.schema.json`). `en.default.json` / `en.default.schema.json` are the source of truth for new keys.

### JavaScript architecture

- **No bundler.** JS is loaded as native ES modules via a `<script type="importmap">` defined in `snippets/scripts.liquid`, mapping bare specifiers like `@theme/component` to `{{ 'component.js' | asset_url }}`. When adding a new shared module, register it in that import map.
- **Base component class**: `assets/component.js` exports `Component`, the base class for the theme's custom elements. It auto-wires `ref="..."` attributes into `this.refs`, supports declarative event listeners, and handles declarative shadow DOM hydration for components mounted post-initial-render.
- **Custom elements** are defined at the bottom of their owning file via `customElements.define('tag-name', ClassName)` (e.g. `header.js` → `header-component`, `variant-picker.js` → `variant-picker`, `product-form.js` → `add-to-cart-component`/`product-form-component`). Grep `customElements.define` to find a component's implementation.
- **Section rendering**: `section-renderer.js` and `section-hydration.js` handle re-rendering/hydrating sections client-side (used for cart/facets/variant updates without full page reloads); `morph.js` provides DOM diffing for these updates.
- Type-checking is JSDoc-based, not TypeScript: see `assets/jsconfig.json` (path aliases `@theme/*` → `./assets/*`, plus `@shopify/events`) and `assets/global.d.ts` / `standard-events.d.ts`. Files are still `.js`; add JSDoc types rather than converting to `.ts`.
- `@shopify/events` (`standard-actions.js` bundle) is loaded from Shopify's CDN, not a local asset — `standard-actions-override.js` registers theme-specific behavior against it and depends on running after that bundle loads.

### Liquid/schema conventions

- Sections and blocks each end in a `{% schema %}` JSON block defining theme-editor settings; keep new settings' `id`s consistent with how they're referenced in the Liquid above.
- Media/slot logic in sections (e.g. `hero.liquid`) commonly toggles between image/video pairs per breakpoint (`image_1`/`video_1` + `_mobile` variants gated by `media_type_*` settings) — check both the picker value and the type flag before treating a slot as active, since the theme editor may leave stale hidden values from a prior media-type toggle.
- RTL support matters: `request.locale.direction` drives `dir` on `<html>`, and layout/positioning should use logical CSS properties (start/end) rather than hardcoded left/right.
- New user-facing strings go in `locales/en.default.json` (storefront) or `locales/en.default.schema.json` (theme editor/schema labels), referenced via Liquid's `t:` / `| t` filters — not hardcoded into the schema or template.

## Project

Beast and Branch is a fantasy/TTRPG print-on-demand store, built as a portfolio demo of
AI-assisted Shopify development with human review. Code quality, commit history and
documentation are part of the deliverable.

## Workflow rules

- Always `git pull` before starting work. The Shopify GitHub integration commits theme
  editor changes (template JSON, `config/settings_data.json`) back to `main`.
- Work on a feature branch; never commit directly to `main`. Keep PRs small and focused.
- Write a PR description summarizing what changed, why, and anything I should check by hand.
- Don't edit `config/settings_data.json` or template JSON unless the task requires it; those
  are merchant-owned settings.
- After each task, add an entry to `AI-LOG.md`: what you did, what I changed or rejected,
  open questions.

## Build approach

- Configure existing Horizon sections, blocks and settings before writing custom code.
- Prefer new files (`sections/bb-*.liquid`, `blocks/bb-*.liquid`, `assets/bb-*.js/css`)
  over modifying Horizon core files, so upstream merges stay clean. If a core file must
  change, note it in the PR.
- Every new section or block must expose theme editor settings for anything a merchant
  would reasonably want to change (text, images, colors, layout options).
- Store-specific content (creatures, realms, product details) comes from metaobjects and
  metafields, not hardcoded Liquid.
- Design references live in `/design`. Match them; flag anything that needs significant
  custom work or can't be matched cleanly before building it.
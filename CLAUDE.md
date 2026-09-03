# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

CEBUQ is a static business directory site for Cebu, Philippines (business.cebuq.com), deployed via
GitHub Pages (see `CNAME`). There is no build step, package manager, bundler, or test suite — every
`.html` file in the repo is served as-is. There is no local dev server config; open files directly in
a browser or serve the directory with any static file server to preview.

## Runtime architecture (important — read before editing any `.html`)

Every page is **not** plain HTML. Each page loads `support.js` (or `../support.js` from `directory/`),
a bundled runtime (originally generated elsewhere from a `dc-runtime` TypeScript project — that source
is not part of this repo, so treat `support.js` as a vendored, do-not-hand-edit dependency) that:

- Parses a custom `<x-dc>...</x-dc>` block in the page body as a template, using a small JSX-like DSL:
  - `{{ expr }}` interpolation
  - `<sc-if value="{{ expr }}">...</sc-if>` conditionals
  - `<sc-for list="{{ expr }}" as="item">...</sc-for>` loops
  - `data-r="name"` attributes are stable hooks used by the inline `<style>` block's responsive
    media queries (mobile layout overrides) — keep them intact when editing markup.
- Reads a sibling `<script type="text/x-dc" data-dc-script data-props="...">` tag containing a
  `class Component extends DCLogic { state = {...}; componentDidMount() {...} ... }` — this is the
  page's actual logic (React-like lifecycle methods, `this.setState`, event handlers referenced from
  the template via `{{ onClick }}`-style bindings).
- Boots this component against the `<x-dc>` template and mounts it into the page.

So **the real logic for a page lives in the trailing `<script data-dc-script>` block**, not in
separate JS files. When fixing a bug or adding a feature to a listing page, look there first.

## Data model

- Each top-level category (e.g. `food-drink`, `health-medical`, `hotels-resorts`, ...) has:
  - `*.html` — the category landing page (e.g. `health-medical.html`)
  - `*.json` — the category's data: `{slug, name, subs: [...], seo, subSeo: {sub: text}, items: [...]}`
  - `directory/*.html` — one page per sub-category (e.g. `directory/dentists.html`), fetching both
    `../directory.json` (for the nav menu / category list) and the parent category JSON
    (e.g. `../health-medical.json`), then filtering `items` client-side by `sub`.
- `directory.json` is the master index: list of `areas`, per-area `totals`, and `groups` of
  categories/sub-categories with per-area counts — drives the directory mega-menu and the
  `/directory.html` overview page.
- `search-index.json` is a flattened, site-wide search index fetched by the search UI
  (`search.html`, and the in-page search overlay present on most pages).
- `home.json` feeds the homepage (`index.html`) featured/collections sections.
- Listing items share a common shape: `{id, name, tier, subs, sub, area, address, price, rating,
  reviews, lat, lng, maps, website, instagram, phone, ..., desc, prices, checked, photo, logo}`.
  - `tier` is one of `Free`, `Verified`, `Business` — controls card styling, badge, and sort rank
    (see `TIER` / `RANK` constants duplicated in each page's script block).
  - Google Place ID is reused as `id`; `maps` is a prebuilt Google Maps search URL.
- Region/area filtering uses a fixed `AREAS` list (`All Cebu`, `Mactan`, `Cebu City`, `Mandaue City`,
  `Moalboal`, `Badian`, `Oslob`) and reads an initial `?area=` query param (slugified) to preselect a
  region — used for deep-linking from the homepage/directory into a pre-filtered category page.

## Conventions to preserve when editing

- `CURRENT_SLUG`, `CURRENT_SUB`, `SUB_SLUGS`, `CATEGORIES`, `ASSET_PREFIX` constants are duplicated
  per-page (there's no shared module system) — when adding a new category or sub-category, update
  these constants consistently across: the new page(s), `directory.json`, and any other page's
  `CATEGORIES`/menu list that enumerates all categories (e.g. header mega-menu in every page).
- Paths are relative: root pages use `assets/...`, `directory.json`, etc.; pages under `directory/`
  use `../assets/...`, `../directory.json`, `../support.js`, `../sw.js`.
- Maps use Leaflet (`unpkg.com/leaflet`) with Esri "Canvas/World_Light_Gray_Base" tiles, custom grey
  vs. purple pin icons (`assets/pin-grey.png` / `pin-purple.png`) to distinguish free vs. paid tiers,
  and a `MAX_PINS` cap that prioritizes paid listings over free ones.
- `sw.js` is a hand-maintained service worker (not generated) — bump `VERSION` when changing its
  caching strategy so old caches get evicted. It deliberately uses network-first for navigations and
  `*.json` data (so listings don't go stale offline) and cache-first for other static assets.
- Some inline code comments are in Russian — this reflects the site owner's working language; match
  the existing style (don't blanket-translate) unless asked.
- Google Analytics (`gtag.js`, `AW-18395931672`) and the service worker registration snippet are
  boilerplate repeated in every page's `<helmet>` — keep them in sync if changing tracking/SW setup.

## Making content changes

Because there's no build step, category/listing data updates are direct edits to the relevant
`*.json` file(s) (and `directory.json` counts if item counts change). There is no automation in this
repo to regenerate JSON from an external source (e.g. Google Places) — assume such regeneration, if
any, happens outside this repo.

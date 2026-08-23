# NetWise V3 — Design audit & multi-style architecture (Stage 1–2)

*August 2026 · Analysis only — no production theme code was modified. The browser-viewable
prototypes live in [`/v3-styles/`](../v3-styles/) in this branch.*

This document summarises the audit of the NetWise V3 theme (private `NetWise-UK/netwise-v3`
repo) and recommends how V3 should support multiple professional visual styles.
A small number of security/escaping findings from the audit are deliberately **not** listed
here because this repository is public; they have been reported separately.

---

## A. Current V3 architecture

- Classic PHP theme (Underscores + Bootstrap 4.5.3 base), SCSS/Gulp build, `dist/` committed.
  ACF Pro and Kirki are bundled; client sites update via Plugin Update Checker pointed at the
  public `NetWise-V3-releases` repo (release-asset zips built by GitHub Actions on `v*` tags).
- Two generations coexist:
  - **Legacy layer** — Kirki theme mods (typography, footer, layout), Bootstrap templates,
    the `frontpage.php` ACF flexible-content homepage, `src/scss/theme/`.
  - **Civic layer** (current direction) — `inc/civic/` (33 modules) + `src/scss/civic/`
    (~7,400 lines): ACF options pages consolidated into the **NetWise → Site layout** admin
    screen, the Estuary homepage (default) and the section-builder homepage (feature-flagged),
    native breadcrumbs, notification bar, GOV.UK support.
- CPTs: documents, members, galleries, directory, notices (news = core posts, relabelled).
  Events and meetings come from separate NW plugins and are consumed by the theme.
- Strong backwards-compatibility discipline already in place: feature flags default off,
  whitelist-sanitised getters with fallbacks, one-shot guarded migrations, theme-side text
  defaults never written to the DB, forwards-compatible config parsing, diagnostic body classes.

## B. Current design system

V3 already has a **four-layer variation system**, delivered as CSS custom properties:

1. **Markup layer** — shared section templates (`nw_civic_render_sections()` etc.). Markup
   never changes per style.
2. **Style layer** — `inc/styles.php` `nw_styles()` registry (currently `civic`, `estuary`):
   non-colour tokens only (display/body font, display weight, letter-spacing, six radius
   tokens, border/rule weights), printed as `:root{}` in `wp_head`. Each style declares which
   colour schemes it permits ("styles and schemes are paired, not freely mixed").
3. **Colour-scheme layer** — `inc/civic/colour-schemes.php`: nine schemes (civic, heritage,
   maritime, claret, estuary + hidden slate/coastal/minimal/govuk) emitting ~22 `--nw-*`
   variables, with server-side WCAG contrast maths.
4. **Brand-override layer** — per-site primary/secondary brand colours, contrast-corrected
   server-side.

Orthogonal to those: **7 header styles** (body class + one template part each),
**7 hero styles** (shared `hero_slides` data), and **3 homepage section-order presets**.

## C. Reusable components (shared across all future styles)

Section templates (next meeting, quick links, news, events, documents, councillors,
directory, contact), the header/nav component set (one nav + off-canvas mobile menu shared by
all header styles), breadcrumbs, pagination, notification bar, document rows, page-title bar,
footer columns, forms, search. All are already styled through `--nw-*` tokens or are close to
it — they are the natural shared chassis for every visual style.

## D. Problems / opportunities

- **Token vocabulary is split** across three generations (`_tokens.scss` SCSS vars — largely
  unused; `:root` `--nw-*`; homepage-scoped `.nw-hp` variables with different names, e.g.
  `--nw-accent` vs `--nw-brass`). Needs one canonical set before more styles are added.
- **Per-style CSS has no home**: styles currently differ only by token values; genuinely
  different component treatments (card vs rule-based lists, different section chrome) need a
  scoped, per-style stylesheet convention.
- **Performance**: the committed `bundle.css` is a dev build (~1.5 MB with inline sourcemap,
  no minify pass); whole Bootstrap 4 + whole Font Awesome ship unpurged; one CSS file still
  loads from a third-party CDN; a Google Fonts `@import` remains at the top of the bundle.
  These should be fixed before styles multiply the CSS.
- **Heading/landmark gaps** on some inner pages (missing `h1`s, search-result headings,
  duplicated main landmarks, pagination `aria-current`) — worth fixing once in the shared
  chassis so every style inherits the fix.

## E. Recommended architecture: extend what V3 already has

**One theme, one markup chassis, N styles** — exactly the direction `inc/styles.php` began:

1. **`nw_styles()` stays the single source of truth.** Extend each entry to a fuller token
   set (type scale, spacing density, shadow level, image treatment) plus per-style defaults:
   default header style, default hero style, default section-order preset, permitted colour
   schemes. Selecting a style = one option write; everything else is derived but remains
   individually overridable (a style is a *curated bundle* over the existing orthogonal axes,
   not a new axis).
2. **Tokens stay runtime CSS custom properties** printed in `wp_head` (no per-style
   stylesheet request, no cache-busting issue, client brand overrides keep working through
   the existing contrast-corrected pipeline).
3. **Add one scoped SCSS partial per style** (`src/scss/styles/_{slug}.scss`, keyed off the
   existing `theme-{scheme}`-style body class, e.g. `nw-style-{slug}`) for the component
   treatments tokens can't express (rule-based lists vs cards, masthead vs banded header).
   Budget: a few hundred lines per style, compiled into the bundle — never a duplicated theme.
4. **New styles ship dormant.** A style not selected adds ~0 bytes of runtime cost and cannot
   change existing sites; the default resolution (`colour_scheme → style`) is untouched, so
   every live site renders exactly as before.
5. **Names** (working set, matching the prototypes): `Modern Civic`, `Community`,
   `Editorial`, `Minimal`, `Contemporary Classic` — alongside the existing `Civic` (a.k.a.
   Classic) and `Estuary`. Contemporary Classic is the natural evolution of today's Civic;
   Community is a sibling of Estuary.

**Prerequisite hardening (small, safe, high-value):** produce `dist/` with the `--prod`
pipeline; self-host the remaining CDN stylesheet; remove the Google Fonts `@import`; trim
Font Awesome/Bootstrap; unify the token names. None of this changes rendered output.

---

## The prototypes

`/v3-styles/index.html` is a self-contained, dependency-free showroom (no build, no server):
a gallery landing page plus all five directions × ten views (homepage, content page, news
listing/article, meetings, documents, contact, search, 404, component sheet) rendered from
one shared, realistic parish-council content model, with a live style/page switcher and a
390 px mobile preview. Open it in any browser.

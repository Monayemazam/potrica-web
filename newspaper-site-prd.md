# Bilingual Online Newspaper — Product Requirements Document

**Status:** Final, build-ready
**Audience:** The person building the site (developer/team). Every section is written so it can be built directly from this document.
**Public languages:** Bangla (default) and English
**Admin/backend language:** English only

---

## Table of contents

1. Overview
2. Confirmed decisions
3. Scope
4. Technology and architecture
5. Performance and lightweight requirements
6. Language and localization
7. Design system (light and dark only)
8. Public site: header, ticker, footer, pages
9. Reader accounts and comments
10. Admin panel (CMS) modules
11. Roles and permissions
12. Media management
13. Ad management
14. SEO
15. Admin CRUD coverage checklist
16. Data model summary
17. Emails and notifications
18. Security, privacy, accessibility
19. Analytics and reporting
20. Infrastructure and deployment
21. Build phases
22. Acceptance criteria
23. Risks
24. Pending inputs

---

## 1. Overview

### 1.1 Product
A new bilingual (Bangla + English) online newspaper. Readers see Bangla by default and can switch to English at any time. Editors, writers and admins work in an English-only admin panel. The site is expected to receive **high traffic**, so it must be **lightweight and fast** above everything else after content quality.

### 1.2 Goals
1. Publish and manage news quickly, with a clear editorial workflow.
2. Read comfortably for all ages: large type, high contrast, calm design, no clutter.
3. Be very fast and cheap to serve under heavy traffic (mostly from cache).
4. Rank well in search (Google Search, Google News, Discover) in both languages.
5. Make **every** visible component editable from the admin panel (full CRUD).
6. Earn revenue through a flexible, lightweight ad system.
7. Give readers an optional account with comments and saved articles, in a separate dashboard.

### 1.3 Non-goals
- No data migration (this is a brand-new site).
- No themes other than **light** and **dark**.
- No click tracking on the ticker.
- No mobile app, no paid subscriptions/paywall, no live-blog engine, no e-paper, in this build.
- No social sign-in in the first build (email + password only; see Pending inputs).

---

## 2. Confirmed decisions

| # | Topic | Decision |
|---|---|---|
| 1 | Migration | None. New site. |
| 2 | Languages | Bangla (default, at `/`) and English (at `/en/`). Full coverage in both for UI, navigation, pages and content. |
| 3 | Translation | Human translation is the standard. Optional machine-translation suggestion (Admin toggle), always marked "Needs review". |
| 4 | Readers | Reader accounts and comments at launch. Separate dashboard (`/account`), separate login, separate table and guard. Only necessary features. |
| 5 | Themes | **Light and dark only.** No sepia, no other themes. |
| 6 | Header brand | Logo image option **and** text option. Admin chooses: logo only, logo + text, or text only. |
| 7 | Weight | Lightweight by design: server-rendered pages, full-page caching, minimal JavaScript, CSS-only ticker. |
| 8 | Ticker | "Latest News" running text below the main menu. Manual, automatic or mixed. Sort order option. Per-page on/off. Colour-coded tags. Full CRUD. No click tracking. |
| 9 | Brand | Logo and visual identity are **not designed yet**. Site must work with a placeholder and let the owner upload the final logo later. |
| 10 | Traffic | Not decided. Build for high traffic; targets in section 5. |
| 11 | Hosting | Not a concern. Requirements in section 20 are given so any host can be chosen. |

---

## 3. Scope

### 3.1 In scope
- Public site in Bangla and English.
- Header with logo/text brand, main menu, language switcher, theme toggle, search, reader login.
- Latest News ticker.
- Home, section, sub-section, article, tag, author, search, gallery, video, poll, static pages, contact, 404.
- Reader accounts, reader dashboard, comments, comment moderation.
- Admin panel: articles, sections, tags, authors, media, pages, menus, homepage layout, ticker, ads, polls, galleries, videos, redirects, SEO, users and roles, translations, settings, audit logs.
- SEO (meta, schema.org, sitemaps, news sitemap, hreflang, redirects).
- Ad management.
- Newsletter sign-up (email collection and export; sending is done by an external provider).

### 3.2 Out of scope
Native apps, paywall, e-paper, live blog, advertiser self-service portal, user-generated articles, push notifications (first build).

---

## 4. Technology and architecture

### 4.1 Stack
| Layer | Choice |
|---|---|
| Language/runtime | PHP 8.3+ |
| Framework | Laravel 12 or 13 (use the current stable release at start) |
| Front-end | Vue 3 + Inertia.js (Laravel starter kit, Vue) |
| Styling | Tailwind CSS (JIT, purged) + DaisyUI (only the components used; theme config limited to `light` and `dark`) |
| Database | MySQL 8, `utf8mb4_unicode_ci` |
| Cache/sessions/queues | Redis |
| Search | MySQL FULLTEXT (ngram parser for Bangla) at launch; Meilisearch via Laravel Scout when needed |
| Roles (staff only) | spatie/laravel-permission |
| Rich text editor | TipTap (in admin) |
| Images | Intervention Image or Spatie Media Library with queued conversions; WebP + AVIF |
| Queue | Laravel queue on Redis, Supervisor-managed workers |
| Scheduler | Laravel scheduler via cron every minute |

### 4.2 Rendering strategy (important for weight)
- **Public pages are server-rendered Blade/Inertia SSR HTML** and cached as full pages. Visitors who are not logged in receive cached HTML with no per-user content in it.
- Use **Inertia + Vue** for the admin panel and reader dashboard (full app experience).
- For the public site, ship **Vue only as small islands** for interactive parts (search box, theme toggle, language switch is a plain link, comments, save button, poll, gallery lightbox, mobile menu). Pages that need none of these ship **no framework JavaScript**.
- If Inertia SSR is used for public pages, the public bundle must still meet the JavaScript budget in section 5. If it cannot, public pages use Blade + islands and only admin/account use full Inertia. Decide in Phase 0 by measuring against the budget; the budget wins.

### 4.3 Three areas of the application
| Area | URL | Users | Table / guard | Language |
|---|---|---|---|---|
| Public site | `/` and `/en/` | Everyone | none | Bangla + English |
| Reader dashboard | `/account`, `/en/account` | Readers | `readers`, guard `reader` | Bangla + English |
| Admin panel | `/admin` | Staff | `users`, guard `web` | English only |

Staff and readers never share sessions, cookies, tables, login pages or password-reset flows. A staff member who also wants to comment needs a separate reader account.

### 4.4 Code organisation
- Domain folders under `app/Domain/{Articles,Sections,Media,Ads,Ticker,Readers,Comments,Seo,Menus,Pages,Settings}`.
- Thin controllers; business logic in action/service classes; form requests for validation; policies for authorization; API resources for JSON output.
- Every user-visible string goes through the translation layer (no hard-coded Bangla or English in templates).
- Code, comments, database names, admin UI and migrations are in English.

---

## 5. Performance and lightweight requirements

This site will get a lot of traffic. These are **requirements**, not suggestions. Phase 0 sets up measurement; every phase must stay within budget.

### 5.1 Targets
| Metric | Target (mobile, mid-range phone, 4G) |
|---|---|
| Largest Contentful Paint (LCP) | < 2.5 s (goal 2.0 s) |
| Interaction to Next Paint (INP) | < 200 ms |
| Cumulative Layout Shift (CLS) | < 0.1 |
| Time to First Byte, cached page | < 200 ms from CDN, < 80 ms at origin |
| Lighthouse mobile performance | ≥ 90 (minimum 85) on home and article |

### 5.2 Page weight budgets (first load, uncached, mobile)
| Page | HTML (gzip/br) | CSS | JavaScript | Fonts | Images above fold | Total (without ads) |
|---|---|---|---|---|---|---|
| Home | ≤ 40 KB | ≤ 25 KB | ≤ 60 KB | ≤ 120 KB | ≤ 150 KB | ≤ 450 KB |
| Article | ≤ 35 KB | ≤ 25 KB | ≤ 50 KB | ≤ 120 KB | ≤ 120 KB | ≤ 400 KB |
| Section | ≤ 35 KB | ≤ 25 KB | ≤ 50 KB | ≤ 120 KB | ≤ 120 KB | ≤ 400 KB |

JavaScript and CSS sizes are compressed transfer sizes. Ads are excluded from the totals but are governed by 5.6. CI fails the build if a budget is exceeded (use Lighthouse CI and a bundle-size check).

### 5.3 Caching (the main tool for high traffic)
1. **CDN in front of the site** (any provider). Public HTML for anonymous visitors is cached at the edge, per language, with `Cache-Control: public, s-maxage=…, stale-while-revalidate=…`.
2. **Origin full-page cache** (Redis or file) as the second layer. Cache key includes locale and path; it does **not** include theme (theme is applied client-side) or user (no per-user content in cached HTML).
3. **Cache lifetimes (defaults, editable in settings):** home 60 s, section 120 s, article 300 s, tag/author 300 s, static pages 3600 s, ticker API 30–60 s.
4. **Instant purge on change:** publishing, editing, unpublishing or deleting an article purges that article, its sections, the home page, its tags and authors, and both language versions as needed. Ticker, menu, homepage layout, ad and settings changes purge the affected pages.
5. **Stale-while-revalidate** so a purge or expiry never makes visitors wait on a cold render.
6. **Stampede protection:** only one request regenerates a page; the others get the stale copy (cache lock).
7. **Per-user and dynamic parts are loaded separately** after the page is shown (small JSON/HTML fragments, never inside the cached page): comments, login state (header "Login"/avatar), saved state, poll results, reader theme if needed.
8. Query results (menus, settings, homepage blocks, ticker, sections list) are cached in Redis and purged on edit.
9. Logged-in staff previews and the admin panel are never cached publicly (`Cache-Control: private, no-store`).

### 5.4 Front-end weight rules
- **No JS frameworks on pages that do not need them.** The article reading page works fully with JavaScript disabled (comments and save button are the only enhancements).
- **No jQuery, no heavy libraries.** No icon font; use inline SVG sprites, only the icons used.
- Tailwind is purged to used classes; DaisyUI only the components used. One critical CSS block is inlined for above-the-fold; the rest loads normally. Target ≤ 25 KB compressed.
- **Vue islands are lazy:** hydrate on idle/visible (comments when scrolled near; search on focus; lightbox on first click).
- **Fonts:**
  - Self-hosted, WOFF2, **subset** to Bangla + Latin + digits + punctuation.
  - Maximum **two families** (a serif for headlines, a sans for body), maximum **two weights each** in the initial load. Candidates: Noto Serif Bengali and Noto Sans Bengali (or a lighter Bangla web font if tested better).
  - `font-display: swap`; preload only the body regular font file. Define a close fallback stack with matched metrics (`size-adjust`) to avoid layout shift.
- **Images:**
  - Responsive `srcset`/`sizes`, WebP with AVIF where supported, JPEG fallback.
  - `loading="lazy"` and `decoding="async"` for everything below the fold; the main (LCP) image is `fetchpriority="high"` and not lazy.
  - Width/height (or aspect-ratio) always set; space reserved.
  - Generated sizes: 160, 320, 480, 640, 800, 1024, 1280, 1600 widths (only those needed per slot). Quality ~75 for WebP, ~55 for AVIF.
  - Low-quality blurred placeholder optional and tiny (< 1 KB, inline colour or tiny blur).
- **Third-party scripts** (analytics, ads, embeds): loaded after load/idle, never blocking render; embeds (YouTube, X, Facebook) use click-to-load placeholders ("facade"), not inline iframes.
- No auto-playing video. No carousels that load many images at once (home slider is a single hero image, not a slideshow).
- HTTP/2 or HTTP/3, Brotli compression, long-cache immutable fingerprinted assets (1 year).
- Prefetch the next likely page only on hover/touch-start for desktop and never on slow connections (`navigator.connection.saveData`).
- Respect `Save-Data` and reduced-motion preferences.

### 5.5 Back-end efficiency
- Avoid N+1 queries: eager loading everywhere; a query-count guard in tests (article page ≤ 12 queries uncached, 0–2 cached).
- Indexes on every filter/sort column used by public queries (listed in the data model).
- Counters (views, comment counts) are **not** written per request: views are buffered in Redis and flushed to the database every minute by a scheduled job.
- Heavy work (image conversion, sitemap generation, search indexing, emails, machine translation, cache warming) runs on queues.
- Pagination everywhere (no unbounded lists); section pages use "load more"/numbered pages with a hard maximum.
- Database: read replica support configured (optional), connection pooling friendly, slow-query log enabled in production.
- Octane (Swoole/RoadRunner/FrankenPHP) is **optional**; the system must perform to target without it thanks to caching.

### 5.6 Ads and weight
- Ads load **after** content, lazily when near the viewport, in slots with reserved size (no layout shift).
- Total ad script and creative weight per page is capped by policy in settings (default: max 4 slots per page on mobile, max 6 on desktop).
- Ad code is loaded with `async`/`defer`; one ad provider script is loaded once per page.
- Direct-sold (self-hosted image/HTML) ads are served from cached, fingerprinted files; the ad list for a page is part of the cached HTML (rotation handled client-side if needed).
- A global "ads off" switch and per-page-type switches exist.

### 5.7 Resilience under spikes
- If the origin or database is slow, the CDN serves stale pages (up to a configurable maximum, default 1 hour).
- A static "lightweight mode" can be enabled in settings during extreme load: disables comments loading, ticker polling, related-article queries, and ad refresh, and serves cached pages longer. (Switch in Admin → Settings → Performance.)
- Rate limiting on search, login, comments, newsletter and contact endpoints.
- Search and comment endpoints are protected by caching and limits so they cannot take the site down.

### 5.8 Monitoring and testing
- Real-user monitoring of Core Web Vitals (lightweight, own endpoint or a privacy-friendly service).
- Uptime checks, error tracking (Sentry or similar), slow-query log, queue-length alerts, cache hit-ratio dashboard (goal ≥ 95% CDN hit on anonymous traffic).
- Load test before launch (k6 or similar) against home, section, article, search and ticker: sustain the agreed traffic target (see Pending inputs) with p95 < 500 ms and no errors, with CDN on and with CDN bypassed at 10% of the target.
- Lighthouse CI on every merge against home, article, section in both languages and both themes.

---

## 6. Language and localization

### 6.1 Languages
| Language | Code | Public URL prefix | Direction | Default |
|---|---|---|---|---|
| Bangla | `bn` | none (`/`) | LTR | Yes |
| English | `en` | `/en/` | LTR | No |

The `languages` table lets more languages be added later without code change, but only these two are enabled at launch.

### 6.2 Language switcher
- Visible in the header on every page (two compact options: "বাংলা" and "English" or "EN"), as plain links (works without JavaScript).
- On an article, the switcher links to that article's other-language version **if it exists**; if not, it links to the other-language home page and shows a small notice "This article is not available in English."
- The choice is remembered in a cookie (`lang`, 1 year). First visit to `/` is never auto-redirected by browser language (this protects SEO and caching); a small dismissible banner may suggest English for visitors whose browser language is English.
- The same language persists for the reader account area and emails sent to that reader.

### 6.3 URLs and slugs
- Bangla: `/`, `/news/{id}/{slug}`, `/section/{slug}`, `/tag/{slug}`, `/author/{slug}`, `/search`, `/page/{slug}`.
- English: `/en/`, `/en/news/{id}/{slug}`, `/en/section/{slug}`, etc.
- `{id}` is shared across languages for the same article; `{slug}` is per-language. Bangla slugs may be in Bangla script (UTF-8, percent-encoded in the URL) or transliterated; the editor can choose; the system auto-suggests a transliterated slug.
- If the slug in the URL is wrong but the `{id}` is correct, redirect (301) to the canonical slug.

### 6.4 What is translated
| Item | Bangla | English |
|---|---|---|
| Article (title, subtitle, summary, body, SEO fields, image captions/alt) | Required | Optional per article (shown if present) |
| Section, tag, author bio, page, menu item, homepage block titles, ticker items, ticker tags, poll, gallery, video, ad labels | Required | Expected (a coverage report flags gaps) |
| Interface text (buttons, labels, error messages, emails, date formats) | Required | Required |
| Admin interface | English only | English only |

### 6.5 Translation workflow tools
1. **Side-by-side editor:** Bangla on the left, English on the right, for articles and other translatable items.
2. **Status per translation:** Missing, Draft, Needs review, Published.
3. **"Source changed" alert:** when the Bangla (source) text changes after an English version was approved, the English version is flagged **Needs review**.
4. **Translation queue:** list of items missing or needing English, filterable by section, assignee and age; assignable to a **Translator** role.
5. **Coverage report:** percentage of articles/sections/pages translated, per section and per month.
6. **Optional machine suggestion:** Admin → Settings → Translation. When on, a "Suggest English" button fills the English fields with a machine translation (provider chosen later; stored as an API key setting). Suggestions are labeled "Machine-assisted — needs review" and cannot be published without a human marking them reviewed (default; can be relaxed by an Admin).
7. **UI string manager:** all interface strings editable in the admin (Admin → Settings → Interface texts) with both languages side by side and missing-string highlighting.

### 6.6 Formatting
- Numerals: Bangla pages show Bangla digits (০–৯) by default; admin setting allows switching to Western digits. English pages always use Western digits.
- Dates: Bangla pages show Bangla month names (e.g., "৯ অক্টোবর ২০২৬") and relative times ("৫ মিনিট আগে"); English pages show "9 October 2026". Time zone default `Asia/Dhaka` (setting).
- Optional Bangla calendar date line in the header (setting, off by default).

---

## 7. Design system (light and dark only)

### 7.1 Principles
Dense, news-packed and readable for all ages. The page should show as many real headlines as comfortably fit: tight spacing, compact cards, text-only lists, thin dividers, almost no empty space. Text stays large enough to read, with strong contrast, a restrained neutral palette and no distracting animation. Newspaper feel: serif headlines, ruled section headers, plain white (or deep grey) surfaces. **The full density rules, page layouts, card types and type scale are in `newspaper-site-design-ux.md`, which governs visual design.**

### 7.2 Themes
- **Only two themes: light and dark.** DaisyUI config contains exactly two themes: `light` and `dark`; all other built-in themes are disabled.
- Toggle in the header (sun/moon icon button, 44 px, with accessible label "Switch to dark mode"/"Switch to light mode" in the current language).
- **First visit:** follows the operating system (`prefers-color-scheme`). **After the toggle is used:** the choice is stored (cookie + localStorage) and wins.
- **No flash of wrong theme:** a tiny inline script (< 300 bytes) in `<head>` sets `data-theme` on `<html>` before first paint. Cached HTML is theme-neutral.
- Reader accounts may also save the preference to their profile (light/dark/follow system); logged-in readers get it applied on the next load.
- Images are not inverted in dark mode. Logos use their dark variant (7.5). Ads are never recoloured.

### 7.3 Colour tokens (CSS variables; values are the starting palette, editable in code only — not in the admin)
| Token | Light | Dark |
|---|---|---|
| `--bg` page background | #F4F3EF | #101214 |
| `--surface` cards | #FFFFFF | #181B1F |
| `--surface-muted` | #ECEAE4 | #20242A |
| `--border` | #DEDBD3 | #2C3137 |
| `--border-strong` | #C9C5BB | #444B53 |
| `--ink` headlines | #15181B | #F3F1EC |
| `--text` body | #22282E | #E2E0DB |
| `--text-soft` | #3A424A | #C9CDD2 |
| `--text-muted` | #5A636D | #A3AAB2 |
| `--accent` links/buttons | #0E5A6B | #5CC0D3 |
| `--accent-contrast` text on accent | #FFFFFF | #0B1A1E |
| `--alert` (live/urgent) | #C0392B | #F0776A |
| `--success` | #1E7B4B | #5FCB91 |
| `--warning` | #8A5A00 | #E6B450 |

All text/background pairs must meet **WCAG AA** (4.5:1 for body, 3:1 for large text) in both themes; checked in CI with a contrast script on the token table. Final brand colours may replace these when a logo/brand exists; only token values change.

### 7.4 Typography
| Role | Family | Size desktop / phone | Weight | Line height |
|---|---|---|---|---|
| Body (articles) | Noto Sans Bengali (or chosen) | 20 px / 19 px | 400 | 1.85 |
| Body (UI, lists) | same | 17–18 px | 400/500 | 1.6 |
| H1 article title | Noto Serif Bengali | 38 px / 28 px | 700 | 1.35 |
| H2 section title | Noto Serif Bengali | 26 px / 22 px | 700 | 1.4 |
| Card title | Noto Serif Bengali | 20–24 px / 19 px | 700 | 1.45 |
| Meta (date, author) | Sans | 15 px | 400 | 1.5 |

- Bangla needs more line height than Latin; never below 1.6 for Bangla body.
- Readers can change article text size with an "A− A A+" control (3 steps: 100%, 115%, 130%), remembered in a cookie. English text uses a Latin font pair (e.g., system serif/sans or the same families' Latin glyphs).
- Maximum article line length ≈ 70 characters (about 680 px).

### 7.5 Header and brand (logo option)
The header brand block is controlled from **Admin → Settings → Brand**.

| Setting | Details |
|---|---|
| Brand display mode | `Logo only`, `Logo + text`, or `Text only` (default `Text only` until a logo is uploaded). |
| Logo (light theme) | Image upload. SVG (preferred), PNG or WebP. Max 1 MB. Recommended height 80 px (2× of 40 px). Transparent background. |
| Logo (dark theme) | Optional separate upload for dark backgrounds. If missing, the light logo is used and a warning shows in the admin ("No dark logo — may be hard to read in dark mode"). |
| Logo per language | Optional separate logos for Bangla and English (e.g., Bangla script logo vs. Latin logo). If not provided, the same logo is used for both. |
| Logo alt text | Per language (e.g., "সংবাদ — হোম পেজ"). Required when a logo is set. |
| Brand text (site name) | Per language. Shown for `Text only` and `Logo + text` modes, and used in `<title>`, schema and emails. |
| Tagline | Optional, per language, shown under or beside the name on desktop only. |
| Logo link | Always links to the home page of the current language. |
| Logo size | Desktop height 40–56 px (admin chooses within range; default 48). Phone height 32–40 px (default 36). Width is automatic, max 220 px desktop / 160 px phone. |
| Favicon | Upload one 512×512 PNG; the system generates 16, 32, 48 ICO/PNG, Apple touch icon 180, Android 192/512. |
| Social/default share image | 1200×630, used when an article has no image. |
| Footer logo | Uses the same logo; option to choose a different (e.g., monochrome) one. |

Rendering rules:
- Logo is an `<img>` with explicit width/height and `fetchpriority="high"` (it can be the LCP element) — **not** a CSS background. SVG is inlined only if < 4 KB, otherwise served as a cached file.
- Light/dark logo switching: render two `<img>` elements, the light logo (`fetchpriority="high"`) and the dark logo (`loading="lazy"`), and show one with CSS based on `data-theme`. Lazy images inside a `display:none` parent do not download, so a visitor only downloads the logo for their theme. If only one logo exists, render a single `<img>`.
- The brand block must not shift layout when the logo loads.
- Before a logo exists, the default is a styled text masthead (serif, large) using the Brand text.

### 7.6 Layout
- Max content width 1200 px (desktop), 12-column grid, 16 px gutters (max container 1280 px); phone side padding 12 px (16 px for article text).
- Breakpoints: 360 (small phone), 480, 768 (tablet), 1024 (laptop), 1280 (desktop), 1536 (wide).
- Dense spacing scale 2/4/6/8/12/16/20/24/32 px: gap between page sections 20–24 px, between cards 12–16 px, card padding 0–12 px. Radius 6 px (cards, buttons), 4 px (tags), full for pills/avatars.
- Section header pattern: 3 px ink-coloured rule + serif title (+ "সব দেখুন / View all" link).
- Minimum touch target 44×44 px. Visible focus ring (2 px accent, 2 px offset) on all interactive elements.
- Dark mode uses the same layout; shadows are replaced by borders.

### 7.7 Components (all built once, reused everywhere)
Header (brand, menu, tools), mega/dropdown menu, mobile drawer menu, ticker, article cards (lead, standard, compact, list, with-number), section header, breadcrumb, byline, tag chip, share bar, pagination/load-more, newsletter box, poll, gallery grid + lightbox, video card (facade), quote box, related list, comment block, reader avatar, buttons, form fields, alerts/toasts, modal, tabs, accordion, skeleton loaders (for dynamic fragments only), ad slot, footer.

### 7.8 Motion
Only essential: ticker scroll, 150 ms hover/focus transitions, drawer slide. All disabled under `prefers-reduced-motion`.

---

## 8. Public site: header, ticker, footer and pages

### 8.1 Header (top to bottom)
1. **Utility bar (desktop only, optional, admin on/off):** date (Bangla/English format, optional Bangla calendar), weather (optional, off by default, external lightweight API cached server-side), social icons, language switcher, theme toggle, reader login/avatar.
2. **Brand row:** logo/text brand (7.5) on the left or centre (admin setting: left/centre), search button, language switcher, theme toggle, login/avatar on the right. On phone: hamburger, brand, search icon, theme toggle; language switcher and login are inside the drawer **and** a compact "বাং | EN" toggle stays visible.
3. **Main menu:** driven by the Menus module (up to 12 top items, two-level dropdown on desktop; accordion in the phone drawer). Sticky on scroll: a slim version (brand mark small + menu) appears when scrolling up (setting on/off). The sticky bar must not cause layout shift.
4. **Latest News ticker** (8.2) sits directly **below the main menu** (the old "breaking news" position). Not at the very top of the page.

### 8.2 Latest News ticker
**Purpose:** a continuous, infinite running line of the latest headlines/notices.

**Visual:** white (surface) card, 1 px border, 8 px radius, fixed height (48 px desktop, 44 px phone). Left: red "সর্বশেষ" / "Latest" label with a small pulsing dot (static under reduced motion). Centre: scrolling text. Right: a pause/play button (44 px target).

**Behaviour:**
- Seamless infinite loop built with **CSS only**: the item track is rendered twice; animate `transform: translateX(0 → -50%)`; linear; default duration scales with content length (about 50 px/second; admin setting for speed: Slow/Normal/Fast).
- Pauses on hover, on keyboard focus within, and via the pause button. The pause state is remembered in `sessionStorage` (optional, silently ignored if unavailable).
- `prefers-reduced-motion`: no animation; the ticker becomes a horizontally scrollable list (no auto movement).
- The duplicate copy is `aria-hidden="true"` and has no links/tabbable elements. The ticker has `role="region"` with an accessible name ("Latest news") and is not an `aria-live` region (it would be noisy).
- Items are links (to an article, a URL, or none). Separators between items are a small dot or slash.
- If there are no active items for the current language, the ticker is **not rendered** at all (no empty bar).
- First paint is server-rendered with the current items (part of the cached page). A 60 s poll of `GET /api/ticker?locale=bn|en` (cached 30–60 s in Redis, purged on dashboard change) updates the track only if items changed, replacing it between loop cycles to avoid a visible jump. Polling stops when the tab is hidden and when lightweight mode is on.
- No click tracking.

**Ticker data fields (per item):**
| Field | Rules |
|---|---|
| Text (bn / en) | Required for Bangla; English optional (item is hidden on `/en/` if no English text). Max 140 characters. Plain text only. |
| Link type | None / Article (search-select) / Custom URL (internal or external). External links open in a new tab with `rel="noopener noreferrer"`. |
| Tag | Optional; picks from the colour-coded tag list (below). |
| Status | Active / Inactive. |
| Pinned | Pinned items always come first (ordered among themselves). |
| Start / End | Optional date-time; item is shown only inside this window (time zone Asia/Dhaka). |
| Order | Drag-and-drop order (used when sort mode is Manual). |
| Source | Manual (typed) or Article-linked (text defaults to the article headline; can be overridden). |

**Ticker tags (CRUD):** name (bn/en), background colour, text colour, active flag. The form shows a live preview and warns if the contrast ratio is below 4.5:1. Examples: জরুরি/Urgent (red), খেলা/Sports (green), আপডেট/Update (blue). In dark mode the tag colours are used as set (admin should pick colours that work on both; the tag form previews both light and dark surfaces).

**Ticker settings (Admin → Ticker → Settings):**
| Setting | Options |
|---|---|
| Master switch | On / Off for the whole ticker. |
| Source mode | **Manual only** / **Automatic (latest articles)** / **Mixed** (manual items first, then automatic fill up to the limit). |
| Automatic rules | Which sections to include/exclude; only articles flagged "Add to ticker" (yes/no); max age (default 24 h); max items (default 10). |
| Sort order | **Manual order** / **Newest first** / **Oldest first** (applies within non-pinned items; pinned always first). |
| Maximum items shown | 3–30 (default 12). |
| Speed | Slow / Normal / Fast. |
| Label text | "সর্বশেষ" / "Latest" (per language). |
| Show on pages | Individual on/off switches for: Home, Section pages, Article pages, Tag pages, Author pages, Search, Static pages, Reader account pages. Defaults: on for all except Reader account pages. |
| Show on devices | Desktop / Phone toggles. |

Articles have an "Add to ticker" checkbox (and an optional short ticker text) so editors can push headlines to the ticker directly from the article editor (subject to permission, section 11).

### 8.3 Footer
Brand block (logo/text + short description), footer menus (up to 4 columns, from Menus module), social links, newsletter box, copyright text, language switcher, theme toggle, "Back to top". All editable (Menus, Settings, Pages). Footer is plain HTML and cached.

### 8.4 Home page
Blocks are controlled from **Admin → Homepage layout** (drag-and-drop order, enable/disable, per-block settings, per-language titles). Default structure:
1. Hero: one lead story (large image + headline + summary) and 4 secondary headlines.
2. Top stories list (5–8 compact items).
3. Ad slot (leaderboard).
4. Most read (24 h / 7 days, from buffered counters) — tab switch is client-side but both lists are in the HTML.
5. Section blocks (any number): each shows section header + 1 large + 3–4 small stories. Types: "Grid 1+4", "List", "Thumbnails row".
6. Opinion/editorial strip with author photos.
7. Photo gallery strip, Video strip (facade).
8. Poll block.
9. Newsletter block.
10. Latest list (infinite "Load more" button, not automatic scroll).
Each block has: type, source (manual picks, latest of section(s), tag, most read), item count, title (bn/en), "view all" link, visibility window, device visibility.

### 8.5 Section page
`/section/{slug}`: section header (name, description), optional sub-section chips, lead story + grid, paginated list (20 per page, numbered pagination for SEO), sidebar (most read, ad, newsletter) on desktop, ads inline. Canonical, breadcrumb schema, `rel=prev/next` not required.
Section fields: name (bn/en), slug (per language), description, parent (one level of sub-sections), colour accent (optional, one of a fixed palette that passes contrast), icon (optional), order, show in menu, show on home, SEO fields, status.

### 8.6 Article page
Order of elements:
1. Breadcrumb (Home › Section).
2. Label chips (Opinion, Exclusive, Live-update, Sponsored) — configurable "article labels".
3. Headline (H1), sub-headline.
4. Byline: author(s) with small avatar, "Published" and "Updated" dates (both shown if updated), reading time, location/dateline (optional field), language switcher link to translation.
5. Share bar (copy link, Facebook, X, WhatsApp, Messenger, Email; plain share URLs, no SDK scripts) and text-size control.
6. Main image (LCP, with caption and credit) or video.
7. Body: paragraphs, subheads (H2/H3), lists, quotes/pull-quotes, inline images with captions, galleries, embeds (facade), tables, related-article inline box, ad slots after paragraphs 3 and 8 (setting).
8. Tags.
9. Author box (for opinion pieces and when enabled).
10. Reader tools: "Save" (logged-in), reactions are **not** included.
11. Comments (section 9), loaded lazily.
12. Related articles (same section/tags, 4–6), "More from this author" (optional), most read.
13. Ad slots (sidebar sticky on desktop, in-content, bottom).

Article rules: published URL `/news/{id}/{slug}`; AMP is **not** used. Print stylesheet included. Schema: `NewsArticle`. Updated articles show "Updated" time and an optional "Correction" note block (editor-written, shown at top or bottom).

### 8.7 Tag page, Author page
- Tag: name, description, list of articles (paginated), related tags, `noindex` automatically if fewer than 3 articles (setting).
- Author: photo, name, role, bio (bn/en), social links, article list (paginated). Schema `Person`.

### 8.8 Search
- Search box in header (opens an overlay on phone). Results page `/search?q=` (both languages share the endpoint; results filter by current language by default with a "Search in both" option).
- Filters: section, date range, sort (relevance/newest). Highlight matched words. Minimum 2 characters. `noindex`.
- Suggestions (autocomplete) optional, from cached top searches; off in lightweight mode.
- Rate limited (30/min per IP). Query logging (anonymised, setting) for the "popular searches" report.
- Bangla search must work with ngram FULLTEXT; handle Bangla/English digits and common spelling variants through an editable synonyms list (Admin → Settings → Search synonyms).

### 8.9 Other pages
- **Photo gallery** (list and single page): grid, lightbox (lazy-loaded JS), captions, credit.
- **Video** (list and single): YouTube/Vimeo or uploaded MP4 link; facade thumbnails; transcript optional.
- **Polls:** one question, 2–8 options, open/close dates, one vote per browser (cookie) and per IP window; results shown after voting; results fetched as a small JSON (cached 30 s).
- **Static pages:** About, Contact, Editorial policy, Ethics, Privacy, Terms, Advertise, Corrections, Team. Editable in Pages (rich text, SEO fields, bn/en).
- **Contact form:** name, email, subject, message, honeypot + rate limit; stored in the admin inbox and emailed to the configured address.
- **Newsletter sign-up:** email only (double opt-in email), unsubscribe link, export CSV from admin.
- **404 page:** friendly, search box, latest stories; logs 404 paths for the redirect manager. **500 page:** static, lightweight.
- **Archive page:** by date (`/archive/2026/10/09`) listing that day's articles (indexable).
- **RSS feeds:** `/feed` and `/en/feed` (latest 30), plus per-section feeds.
- **Maintenance page:** static, with a bypass key for staff.

---

## 9. Reader accounts and comments

### 9.1 Separate area
Readers have their own login, registration, password reset, profile and dashboard at `/account` (Bangla) and `/en/account` (English). It is visually consistent with the public site (same header/footer, same theme toggle), but is a different guard, different table (`readers`), different session cookie, no roles and no access to `/admin`. Account pages are `noindex`, not cached publicly, and not shown in sitemaps.

### 9.2 Registration and login
- Registration: name (display name), email, password (min 8 chars, with strength hint), accept terms/privacy checkbox. Honeypot + rate limit (5/hour/IP). **Email verification required** before commenting (verification link valid 24 h; resend available).
- Login: email + password, "remember me", rate limited (5 failures per minute per account+IP, then a temporary lockout with a clear message).
- Password reset by email (token valid 60 minutes).
- Optional later: Google/Facebook sign-in (Pending inputs).
- Logout works on all pages. Login state in the header is loaded by a tiny cached-safe request (the page itself is anonymous HTML).
- Account deletion: reader can delete their account (confirm with password). Their comments are anonymised to "Deleted user" (or removed; Admin setting).

### 9.3 Reader dashboard (only what is necessary)
| Page | Contents |
|---|---|
| Overview | Greeting, counts (saved articles, comments), shortcuts. |
| Profile | Display name, avatar (upload, 1 MB, square-cropped, optional), short bio (optional, 160 chars), preferred language, theme preference (light / dark / follow system). |
| Saved articles | List of saved articles with remove button; paginated 20 per page; save/unsave from any article page. |
| My comments | List of own comments with status (Published / Pending / Rejected / Removed), edit (within window) and delete. |
| Notifications settings | Email me when someone replies to my comment (on/off); newsletter subscription (on/off). |
| Security | Change email (re-verify), change password, log out other sessions, delete account. |
Not included: following authors, achievements, points, messaging, reading history.

### 9.4 Comments: rules
- Only **verified, logged-in readers** can comment. Everyone can read comments.
- Plain text only (no HTML, no images); line breaks allowed; max **1000 characters**; min 2 characters. Links are not clickable by default (auto-held if they contain URLs).
- One level of replies (a reply to a reply attaches to the same parent).
- Comments are **shared across both language versions** of an article (a comment in Bangla is visible on the English page and vice versa); each shows its language tag if different from the page.
- Comments can be turned off per article (checkbox "Allow comments") and globally; an article's comments auto-close after N days (default 14, setting; 0 = never).
- Sorting: Newest (default), Oldest, Most replies (setting).
- Pagination: first 10 top-level comments, "Load more" button.
- **Not part of cached HTML:** comments load as a separate request when the block approaches the viewport; the article page itself only contains a placeholder and the count (the count comes from the cached counter).
- Edit window: 15 minutes (setting); edited comments show "edited". Delete own comment: always allowed (shows "Comment deleted by reader" if replies exist).
- Rate limits: 1 comment per 20 seconds, 10 per hour per reader; new accounts (< 24 h) limited to 3 comments per hour.

### 9.5 Moderation
- **Modes (Admin → Settings → Comments):** Post-moderation (publish immediately, review later) or Pre-moderation (hold every comment until approved). Default: **Post-moderation** for verified readers with ≥ 1 approved comment, **hold** for first-time commenters (setting).
- **Automatic holds:** banned-word match (editable list, bn/en), contains a link, duplicate of recent comment, too many capital letters, reader flagged by moderators.
- **Reports:** every comment has a "Report" action for logged-in readers (reasons: abuse, spam, hate, misinformation, other). Comments reaching **N reports** (default 3) are auto-hidden pending review.
- **Moderation dashboard** (Admin → Comments): tabs — Pending, Reported, Published, Spam, Removed. Bulk approve/reject/spam/delete. Filters by article, section, reader, date, language. Each comment shows context (article link, thread). Actions: approve, reject, mark spam, remove (with reason), edit (removes personal data/profanity, marks "edited by moderator"), ban reader (suspend commenting for X days or permanently).
- **Reader suspension:** suspended readers can log in and read but cannot comment; they see the reason.
- **Audit log** of every moderation action (who, what, when, reason).
- **Policy text:** a "Comment policy" static page, linked next to the comment box (text supplied by the owner; see Pending inputs).
- Comment links, if enabled by an admin, use `rel="ugc nofollow noopener"`.
- Moderator notification: daily digest and optional instant email when pending > threshold.

### 9.6 Reader emails
Verify email, welcome, password reset, email-change confirmation, reply notification (respecting settings), newsletter double opt-in, account-deleted confirmation. Bilingual by reader's language; plain, light HTML (< 30 KB), sent through the queue.

---

## 10. Admin panel (CMS) modules

The admin panel lives at `/admin` (English). It is desktop-first but must be usable on tablets and phones for reviewing/publishing. It has its own light/dark theme (same two themes). It is not cached; it can be heavier than the public site but should still load < 2 s on a normal connection.

### 10.1 Dashboard
Per-role widgets: today's published count, drafts waiting, pending review, scheduled posts, pending comments, translation backlog, top articles (24 h), traffic summary (from internal counters), failed jobs/queue status (Admin), quick-create buttons.

### 10.2 Articles
**List:** columns — title, section, author, status, language coverage (BN/EN badges), published time, views, comments; filters — status, section, author, language, date, "has ticker", "featured"; search; bulk actions (publish, unpublish, move section, add tag, delete, change author); saved filter views.

**Editor (tabs: Content, Media, Settings, SEO, Translation, History):**
| Field | Notes |
|---|---|
| Title (bn / en) | Required in Bangla. Max 160 chars; character counter. |
| Sub-headline | Optional. |
| Summary/excerpt | 1–2 sentences, max 300 chars; used on cards and meta. |
| Slug (per language) | Auto-suggested; editable until first publish (then change creates automatic redirect). |
| Body | TipTap: headings H2/H3, bold, italic, underline, link, lists, quote, pull-quote, table, image (from Media library with caption/credit/alt), gallery, video/embeds (YouTube, Vimeo, Facebook, X, Instagram as facades), horizontal rule, "related article" inline box, "info box", clear-formatting, paste-from-Word cleanup, Bangla typing friendly (Unicode, optional phonetic input toggle). Autosave every 30 s (draft revision). |
| Main image | From media library; caption, credit, alt text (per language); focal-point selector for cropping. |
| Section (primary) + additional sections | Required primary. |
| Tags | Autocomplete, create-on-the-fly (permission). |
| Author(s) | One or more author profiles (not just the logged-in user); optional "byline override" text. |
| Dateline/location | Optional. |
| Labels | Opinion, Exclusive, Analysis, Sponsored, Fact-check, etc. (CRUD list). |
| Display options | Featured on home, "Breaking/Top" flag, show in "most read", allow comments, add to ticker (+ ticker text), hide author, hide share bar. |
| Related articles | Auto (by tags/section) or manual picks. |
| Publish | Status: Draft / In review / Approved / Scheduled / Published / Unpublished / Archived. Publish time (default now, can schedule; time zone Asia/Dhaka). Optional "expires at". |
| SEO | Meta title, meta description, canonical override, noindex/nofollow, OG title/description/image, Twitter card, schema type (NewsArticle default; Opinion → `OpinionNewsArticle`; fact-check optional), Google News keywords, live SERP preview and length meters (bn/en). |
| Translation | Status per language, side-by-side view, "Suggest English" (if enabled), assignee, "Needs review" flag. |
| History | Revisions (every save and autosave, last 50 kept), diff view, restore, "who changed what". Editorial comments thread (internal, not public). |

**Editorial workflow:** Writer saves draft → submits for review → Editor reviews (inline internal comments, request changes/approve) → Editor/Admin publishes or schedules. Roles with direct-publish permission can skip review. Locking: if two users open the same article, the second sees a warning and opens in read-only unless they take over (soft lock with 2-minute heartbeat).

**Corrections:** "Add correction note" action (text + date, appears on the article, logged).

**Duplicates/safety:** warns on identical slug/title; confirm before deleting; deletes are soft (Trash, 30 days) except for Admin permanent delete.

### 10.3 Sections
CRUD, drag-and-drop ordering, parent/child (one level), fields in 8.5, merge/move articles when deleting a section (must choose a target).

### 10.4 Tags
CRUD, merge tags, bulk delete unused, bn/en names and slugs, description, SEO fields, "noindex if empty" behaviour.

### 10.5 Authors
CRUD author profiles (separate from staff users, but may be linked to a staff user): name (bn/en), slug, photo, role/title, bio (bn/en), email (private), social links, status.

### 10.6 Pages
CRUD static pages (rich text editor, bn/en, SEO fields, template choice: default / wide / contact), status, show in footer/menu helper.

### 10.7 Menus
CRUD menus by location (Main header, Footer 1–4, Utility/top, Mobile extra). Items: label (bn/en), link type (section, page, tag, article, custom URL, category archive), open in new tab, icon (optional), children (one level, mega-menu optional), visibility window, order via drag-and-drop. Menu changes purge cached pages.

### 10.8 Homepage layout
Block manager as in 8.4 with live preview link (opens the public home in preview mode showing unpublished/scheduled blocks).

### 10.9 Ticker
Items list (drag-to-order, filters, bulk activate/deactivate/delete), item form, tags CRUD, settings (8.2), live preview strip showing exactly how the ticker will look in light and dark.

### 10.10 Media library
See section 12.

### 10.11 Galleries, Videos, Polls
CRUD for each with bn/en fields, status, scheduling. Polls show results and can be exported; open/close dates; "show on home" toggle.

### 10.12 Ads
See section 13.

### 10.13 Comments and readers
Moderation dashboard (9.5) plus **Readers** list: search, status (Active/Suspended/Unverified), comment count, last login, actions (suspend, unsuspend, resend verification, delete). Personal data is limited to name, email, avatar; admins with permission only.

### 10.14 Newsletter and messages
Subscribers list (status, source, date), export CSV, unsubscribe, bulk delete. Contact messages inbox (read/unread, reply by email, archive, delete, spam).

### 10.15 SEO tools
Global SEO settings (site title patterns, default meta, default share image, robots.txt editor, sitemap controls, Google Search Console/Bing verification meta tags, analytics IDs), **Redirect manager** (301/302, from→to, hit counter, import/export CSV, bulk), **404 log** (top missing URLs with one-click "create redirect"), structured-data defaults (Organization info, logo, social profiles), **per-page SEO audit** hints (missing meta description, title too long, missing alt text, missing image).

### 10.16 Users and roles
Staff users CRUD (name, email, password reset/send invite, role, assigned sections, status, 2FA enforcement), roles CRUD with permission matrix editor (section 11), login history, force logout.

### 10.17 Translations
Interface strings manager, translation queue, coverage report (6.5).

### 10.18 Settings (all editable)
| Group | Contents |
|---|---|
| General | Site name (bn/en), tagline, contact emails, time zone, default language, number format (Bangla/Western digits), date formats. |
| Brand | Logo mode, logos (light/dark/per language), alt text, sizes, favicon, share image, footer logo (7.5). |
| Header & footer | Utility bar on/off, weather on/off, date line, sticky header on/off, brand alignment, footer text, copyright, social links. |
| Appearance | Only: default theme behaviour (follow system / default light / default dark), article text-size default. No other theme options. |
| Performance | Cache lifetimes, lightweight mode switch, image quality, max ads per page, polling on/off, preload settings. |
| Comments & readers | Moderation mode, banned words, report threshold, rate limits, edit window, auto-close days, registration on/off, deletion behaviour. |
| Translation | Machine suggestion on/off, provider key, review required toggle. |
| Ticker | Section 8.2. |
| Search | Synonyms, minimum characters, logging on/off. |
| Email | Sender name/address, SMTP/provider credentials, templates (bn/en), digest settings. |
| Integrations | Analytics ID, Search Console, social profile URLs, ad provider scripts, translation provider, Sentry DSN. |
| Legal | Cookie notice text, privacy/terms page selection. |
| Maintenance | Maintenance mode, bypass key, cache purge buttons (all / home / sections / one URL), rebuild sitemaps, clear image cache. |

### 10.19 Audit log
Every create/update/delete/publish/login/permission change recorded: who, what, old/new values (summary), IP, time. Filter and export. Retained 12 months (setting).

---

## 11. Roles and permissions

### 11.1 Staff roles (spatie/laravel-permission)
| Role | Purpose |
|---|---|
| **Super Admin** | Everything, including roles, settings, ads, users, maintenance. At least one always exists; cannot be deleted by others. |
| **Admin** | All content and site management; cannot change Super Admin accounts or system-critical settings. |
| **Editor-in-Chief** | All content in all sections, publish/unpublish anything, homepage layout, ticker, menus, comment moderation. |
| **Section Editor** | Edit/approve/publish content **only in assigned sections**; manage tags; ticker for own sections; moderate comments in own sections. |
| **Reporter / Writer** | Create and edit **own** drafts; submit for review; cannot publish (unless granted). |
| **Contributor** | Like Writer but limited to submitting; cannot upload to shared folders. |
| **Translator** | Edit English translations only; cannot change Bangla source or publish originals; can mark translations as ready for review. |
| **Photographer/Media Manager** | Manage media library; no article publishing. |
| **Ad Manager** | Manage ads, placements, advertisers, reports; no content access. |
| **SEO Manager** | SEO settings, redirects, 404 log, per-article SEO fields; no publishing. |
| **Moderator** | Comment moderation, readers list (suspend), contact messages. |
| **Analyst (read-only)** | View reports and dashboard only. |
Custom roles can be created by Super Admin by combining permissions. Roles apply to **staff only**; readers have no roles.

### 11.2 Permission matrix (✔ = allowed, own = own items only, sec = assigned sections only)
| Capability | Super Admin | Admin | Editor-in-Chief | Section Editor | Reporter | Translator | Media Mgr | Ad Mgr | SEO Mgr | Moderator |
|---|---|---|---|---|---|---|---|---|---|---|
| Create article | ✔ | ✔ | ✔ | sec | ✔ | – | – | – | – | – |
| Edit article | ✔ | ✔ | ✔ | sec | own (draft) | EN only | – | – | SEO fields | – |
| Submit for review | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ (translations) | – | – | – | – |
| Approve/publish | ✔ | ✔ | ✔ | sec | – | – | – | – | – | – |
| Unpublish/delete | ✔ | ✔ | ✔ | sec | – | – | – | – | – | – |
| Add to ticker | ✔ | ✔ | ✔ | sec | – | – | – | – | – | – |
| Manage ticker items/settings | ✔ | ✔ | ✔ | items (sec) | – | – | – | – | – | – |
| Manage sections/tags | ✔ | ✔ | ✔ | tags | – | – | – | – | – | – |
| Menus / homepage layout | ✔ | ✔ | ✔ | – | – | – | – | – | – | – |
| Media upload / edit | ✔ | ✔ | ✔ | ✔ | upload own | – | ✔ | – | – | – |
| Media delete | ✔ | ✔ | ✔ | – | – | – | ✔ | – | – | – |
| Ads and placements | ✔ | ✔ | – | – | – | – | – | ✔ | – | – |
| SEO settings, redirects | ✔ | ✔ | – | – | – | – | – | – | ✔ | – |
| Comments moderation | ✔ | ✔ | ✔ | sec | – | – | – | – | – | ✔ |
| Readers (suspend/delete) | ✔ | ✔ | – | – | – | – | – | – | – | ✔ |
| Users and roles | ✔ | users (not Super Admin) | – | – | – | – | – | – | – | – |
| Settings | ✔ | most | – | – | – | – | – | – | – | – |
| Audit log | ✔ | ✔ | – | – | – | – | – | – | – | – |
| Reports/dashboard | ✔ | ✔ | ✔ | sec | own | – | – | ads | SEO | comments |

Policies enforce every rule on the server (never rely on hidden buttons). Permission names follow `resource.action` (e.g., `articles.publish`, `ticker.manage`, `comments.moderate`).

### 11.3 Staff security
Strong passwords (min 10), optional/enforceable 2FA (TOTP) for roles Admin and above, session timeout (default 8 h idle 2 h), login rate limit, new-device email alert, IP allow-list option for `/admin` (setting), force logout.

---

## 12. Media management

### 12.1 Library
- Grid/list views, folders (nested), tags, search (filename, title, caption, alt), filters (type, date, uploader, folder, unused).
- Upload: drag and drop, multiple files, paste from clipboard, progress, resumable for large videos (chunked). Max sizes: image 10 MB, document 20 MB, audio 50 MB, video 500 MB (setting; large video should be hosted on YouTube/Vimeo instead).
- Allowed types: JPG, PNG, WebP, AVIF, GIF (static preferred), SVG (sanitised), PDF, MP4, MP3. Server validates MIME and content, strips EXIF location data (keeps copyright), scans file names, rejects scripts.
- Auto-processing (queued): original kept, resized copies at the widths in 5.4, WebP/AVIF versions, optimisation, dominant-colour placeholder. Duplicate detection by hash.
- Metadata per file: title, alt text (bn/en, required for images used in articles), caption (bn/en), credit/source, license note, focal point, uploader, dates, usage list ("used in 12 articles").
- Editing: crop (preset ratios 16:9, 4:3, 1:1, 3:4, free), rotate, focal point, replace file (keeps URL and references), restore original.
- Delete protection: files in use show where they are used; deletion requires confirmation; soft delete 30 days.
- Storage: local disk or S3-compatible object storage via Laravel filesystem; public URLs are CDN-friendly, versioned and immutable.
- Bulk actions: move, tag, delete, download.

### 12.2 Image rules for editors (shown in the uploader)
Recommended main image 1600×900 minimum; the system warns on small or very large files; alt text reminder; an "Accessibility" checklist badge on articles missing alt text.

### 12.3 Embeds
YouTube/Vimeo/Facebook/X/Instagram/TikTok by URL; rendered as lightweight click-to-load placeholders; allow-list of domains; no arbitrary iframe/script embeds by writers (Admin can allow a trusted HTML block).

---

## 13. Ad management

### 13.1 Concepts
- **Placement (slot):** a named location on the site, defined in code with a fixed size and reserved space (list below). Admin can enable/disable and change which ads fill it.
- **Ad:** creative + targeting + schedule. Types: **Image** (upload + link + alt), **HTML/JS code** (trusted, Admin/Ad Manager only, e.g., Google AdSense/GAM tags), **Text/sponsored link**.
- **Advertiser:** name, contact, notes, ads list.
- **Campaign (optional grouping):** name, advertiser, dates, budget notes.

### 13.2 Default placements
| Code | Where | Desktop size | Phone size |
|---|---|---|---|
| `header_leaderboard` | Below ticker on home/section | 728×90 | 320×100 |
| `home_mid_1`, `home_mid_2` | Between home blocks | 728×90 | 300×250 |
| `sidebar_top`, `sidebar_sticky` | Right sidebar | 300×250, 300×600 | not shown |
| `article_top` | Above article body | 728×90 | 320×100 |
| `article_inline_1`, `article_inline_2` | After paragraph N (setting) | 300×250 / responsive | 300×250 |
| `article_bottom` | Below article | 728×90 | 320×100 |
| `section_inline` | Between list items | 728×90 | 300×250 |
| `mobile_sticky_bottom` | Phone bottom bar (dismissable) | – | 320×50 |
| `interstitial` | Disabled by default; not recommended | – | – |
New placements can be added by developers; admin can reorder/disable existing ones.

### 13.3 Ad fields
Title (internal), type, creative (image per size/code), destination URL (with `rel="sponsored noopener"`, optional UTM), alt text, **placements** (many), **start/end** date-time, **priority/weight** (for rotation), **targeting** (language bn/en/both, sections, article labels, device desktop/phone, page type), **frequency cap** (optional, per browser), status (Draft/Active/Paused/Expired), notes.
- Rotation: weighted among eligible ads, chosen on the server per cache refresh, plus optional client-side rotation (off by default to protect cache).
- Labelling: every ad shows a small "বিজ্ঞাপন / Advertisement" label; sponsored articles show "Sponsored" label.
- Ad blockers: no anti-adblock wall in this build.

### 13.4 Reporting
Impressions and clicks per ad/placement/day, counted by a lightweight beacon (batched, buffered in Redis, flushed to `ad_stats_daily`); CTR; date-range filter; CSV export; expiring-soon alerts (7 days); Ad Manager dashboard.
Impression counting uses a visible-viewport beacon sent after the slot is seen; bots are filtered by user-agent list; counts are approximate and labelled as such.

### 13.5 Controls
Global ads switch, per-page-type switch, per-article "no ads" flag (for sensitive content), maximum slots per page, ads disabled on 404/account/login pages, consent-aware loading if a cookie notice is enabled.

---

## 14. SEO

### 14.1 On-page
- Unique `<title>` and meta description per page; patterns editable (e.g., `{article title} | {site name}`), per language.
- One H1 per page; semantic HTML (`header`, `nav`, `main`, `article`, `aside`, `footer`); breadcrumbs.
- Canonical URL on every page (self-canonical; pagination pages self-canonical; filtered/search pages `noindex`).
- Open Graph and Twitter Card tags with image (article main image or default share image); `og:locale` = `bn_BD` / `en_US` plus `og:locale:alternate`.
- **hreflang**: each page with a translation links both ways (`bn`, `en`, `x-default` → Bangla). Pages without a translation do not output hreflang for the missing language.
- `robots` meta: `noindex` for search, account, login, tag pages with < 3 articles, preview, and admin; `max-image-preview:large` on articles (Discover).
- Image `alt` text required; descriptive file names; `width`/`height` always set.

### 14.2 Structured data (JSON-LD, server-rendered)
`Organization` (with logo from the Brand setting) and `WebSite` (with SearchAction) on home; `NewsArticle` (headline, image array, datePublished, dateModified, author `Person`, publisher, `inLanguage`, `articleSection`, `mainEntityOfPage`) on articles; `BreadcrumbList` on article/section pages; `Person` on author pages; `ImageGallery`/`VideoObject` where relevant; `FAQPage` only when the article has an FAQ block.

### 14.3 Sitemaps and feeds
- `sitemap.xml` index → per-language sitemaps (articles split into chunks of ≤ 5,000 URLs, sections, tags, authors, pages) with `lastmod` and hreflang alternates.
- **Google News sitemap** per language: articles from the last 48 hours (`news:publication`, `news:publication_date`, `news:title`).
- Image sitemap optional. Sitemaps are generated by queued jobs and cached; updated within 1 minute of publishing.
- `robots.txt` editable in admin; references sitemaps; blocks `/admin`, `/account`, `/search`, `/api`.
- RSS per language and per section.
- Auto-ping/IndexNow (optional setting) on publish.

### 14.4 URLs and redirects
Clean URLs (6.3); 301 to canonical slug; slug change → automatic 301 from old URL; trailing slash and uppercase normalisation; redirect manager (10.15); 410 for permanently deleted articles (setting); soft 404 avoidance.

### 14.5 Technical SEO
Core Web Vitals targets (section 5) are part of SEO; server-rendered content visible to crawlers without JavaScript; lazy-loaded comments are not indexed (fine); no duplicate content between languages beyond hreflang; pagination as normal links; no infinite scroll without links ("Load more" has an `href` fallback to the next page).

---

## 15. Admin CRUD coverage checklist

Every item below must have Create, Read/List (with search, filter, pagination), Update and Delete (soft delete where noted) in the admin, with validation, audit logging and cache purge on change.

| Component | C | R | U | D | Notes |
|---|---|---|---|---|---|
| Articles | ✔ | ✔ | ✔ | ✔ (trash) | Revisions, workflow |
| Article labels | ✔ | ✔ | ✔ | ✔ | |
| Sections | ✔ | ✔ | ✔ | ✔ | Reassign on delete |
| Tags | ✔ | ✔ | ✔ | ✔ | Merge |
| Authors | ✔ | ✔ | ✔ | ✔ | |
| Pages | ✔ | ✔ | ✔ | ✔ (trash) | |
| Menus and items | ✔ | ✔ | ✔ | ✔ | Drag order |
| Homepage blocks | ✔ | ✔ | ✔ | ✔ | |
| Ticker items | ✔ | ✔ | ✔ | ✔ | |
| Ticker tags | ✔ | ✔ | ✔ | ✔ | Colours |
| Ticker settings | – | ✔ | ✔ | – | |
| Media files and folders | ✔ | ✔ | ✔ | ✔ (trash) | |
| Galleries | ✔ | ✔ | ✔ | ✔ | |
| Videos | ✔ | ✔ | ✔ | ✔ | |
| Polls | ✔ | ✔ | ✔ | ✔ | |
| Ads, placements, advertisers, campaigns | ✔ | ✔ | ✔ | ✔ | |
| Redirects | ✔ | ✔ | ✔ | ✔ | Import/export |
| Comments | – (readers create) | ✔ | ✔ (moderate/edit) | ✔ | |
| Banned words | ✔ | ✔ | ✔ | ✔ | |
| Readers | – | ✔ | ✔ (suspend) | ✔ | |
| Subscribers | ✔ (manual) | ✔ | ✔ | ✔ | Export |
| Contact messages | – | ✔ | ✔ (status) | ✔ | |
| Staff users | ✔ | ✔ | ✔ | ✔ (deactivate) | |
| Roles and permissions | ✔ | ✔ | ✔ | ✔ | System roles protected |
| UI strings | ✔ | ✔ | ✔ | – | |
| Search synonyms | ✔ | ✔ | ✔ | ✔ | |
| Settings (all groups) | – | ✔ | ✔ | – | |
| Languages | ✔ | ✔ | ✔ | – | Only bn/en enabled |
| Brand (logo, favicon) | ✔ (upload) | ✔ | ✔ (replace) | ✔ (remove) | |

---

## 16. Data model summary

(A full MySQL schema is a separate document and must be aligned with this PRD. This section lists the entities and key rules.)

**Conventions:** `id` bigint unsigned; `created_at`/`updated_at`; soft deletes where noted; every translatable entity has an `<entity>_translations` table (`<entity>_id`, `locale`, translated fields, unique `(<entity>_id, locale)`); all foreign keys indexed; all text `utf8mb4`.

| Entity | Key fields |
|---|---|
| `languages` | code, name, native_name, is_default, is_active, sort |
| `settings` | group, key, value (JSON), locale nullable |
| `ui_translations` | key, locale, value |
| `users` (staff) | name, email, password, status, two_factor fields, last_login |
| roles/permissions | spatie tables; `section_user` for assigned sections |
| `readers` | name, email, password, email_verified_at, avatar_media_id, bio, locale, theme_pref (`light`/`dark`/`system`), status (active/suspended), suspended_until, suspend_reason, last_login |
| `saved_articles` | reader_id, article_id, created_at (unique pair) |
| `sections` | parent_id, slug base, sort, accent, icon, show_in_menu, show_on_home, status |
| `articles` | section_id, status, published_at, expires_at, scheduled flags, featured, is_top, allow_comments, add_to_ticker, no_ads, label_id, main_media_id, dateline, hide_author, hide_share, view_count (flushed), comment_count, created_by, updated_by, soft delete |
| `article_translations` | article_id, locale, title, subtitle, summary, body (long text, sanitised HTML), slug, meta fields, og fields, schema_type, translation_status (missing/draft/needs_review/published), is_machine_assisted, source_changed_at, assigned_translator_id |
| `article_revisions` | article_id, locale, snapshot JSON, user_id, created_at |
| `article_review_comments` | article_id, user_id, body, resolved |
| `article_section` (extra), `article_tag`, `article_author` | pivot tables |
| `tags` (+translations), `authors` (+translations) | as described |
| `media`, `media_folders`, `mediables` | file data, variants JSON, alt/caption translatable, focal point, hash, usage |
| `pages` (+translations), `menus`, `menu_items` (+translations) | |
| `homepage_blocks` (+translations) | type, config JSON, sort, visibility window, devices |
| `ticker_items` (+translations) | tag_id, link_type, article_id, url, status, pinned, starts_at, ends_at, sort, source |
| `ticker_tags` (+translations) | bg_color, text_color, active |
| `galleries`, `videos`, `polls`, `poll_options`, `poll_votes` | |
| `comments` | article_id, reader_id, parent_id, body, status (pending/published/rejected/spam/removed/hidden), locale, report_count, edited_at, moderated_by, moderation_reason |
| `comment_reports` | comment_id, reader_id, reason, created_at (unique reader+comment) |
| `banned_words` | word, locale, action (hold/reject) |
| `ads`, `ad_placements`, `ad_placement_ad`, `advertisers`, `campaigns`, `ad_stats_daily` | targeting JSON, schedule, weight |
| `redirects`, `not_found_logs` | from, to, code, hits |
| `article_views_daily` | article_id, date, views |
| `subscribers`, `contact_messages` | |
| `audit_logs` | user_id, action, entity, entity_id, changes JSON, ip, created_at |
| `search_logs` (optional) | term, locale, results_count, date (anonymous) |

**Indexes that matter for speed:** `articles (status, published_at)`, `articles (section_id, status, published_at)`, `article_translations (locale, slug)` unique, `article_translations (article_id, locale)` unique, FULLTEXT on `article_translations (title, summary, body)` with ngram, `comments (article_id, status, parent_id, created_at)`, `ticker_items (status, starts_at, ends_at, sort)`, `article_tag (tag_id, article_id)`, `redirects (from_path)` unique.

---

## 17. Emails and notifications

| Email | Recipient | Trigger |
|---|---|---|
| Staff invite / password reset | Staff | Admin action / request |
| New-device login alert | Staff | Login from unseen device |
| Reader verify / welcome / reset / email change / deleted | Reader | Account events |
| Comment reply | Reader | Reply to their comment (if enabled) |
| Comment pending digest | Moderators | Daily + threshold instant |
| Article submitted for review / approved / changes requested | Editors / writers | Workflow |
| Translation assigned / needs review | Translator / editors | Translation workflow |
| Newsletter double opt-in | Subscriber | Sign-up |
| Contact message received | Configured address | Contact form |
| Ad expiring soon | Ad Manager | 7 days before end |
| Failed jobs / queue backlog | Super Admin | Threshold |
Templates are editable (bn/en for reader-facing ones), light HTML + plain-text version, sent via the queue through the configured provider. In-admin notifications bell shows workflow events.

---

## 18. Security, privacy and accessibility

### 18.1 Security
- HTTPS everywhere with HSTS; secure, HttpOnly, SameSite cookies; CSRF on all forms; CSP header (no inline scripts except the tiny theme script via nonce/hash); X-Frame-Options/`frame-ancestors`; Referrer-Policy; Permissions-Policy.
- Input validation on server; HTML sanitisation (allow-list) for article bodies and static pages; comment text escaped, no HTML.
- File upload hardening (12.1); SVG sanitised; uploads stored outside executable paths.
- Rate limiting and bot protection (honeypot; optional CAPTCHA/Turnstile when abuse appears) on registration, login, comments, contact, newsletter, search, poll.
- Password hashing (bcrypt/argon2); breached-password check optional.
- SQL injection safe via Eloquent/bindings; authorization via policies; mass-assignment protection.
- Dependency updates monthly; security headers tested in CI; secrets in environment variables only.
- Backups: database daily (retain 14 days) + weekly (retain 8), media backups, restore tested before launch and quarterly.
- Admin audit log (10.19). Optional IP allow-list for admin.

### 18.2 Privacy
- Collect only what is needed: staff (name, email), readers (name, email, optional avatar/bio), newsletter (email), comments, contact messages.
- Privacy policy and terms pages; cookie notice (short, lightweight) if analytics/ads need consent in the target audience; essential cookies only by default (session, theme, language, text size).
- Reader data export and deletion on request (account delete is self-service; export by admin tool).
- Server logs and search logs are anonymised/retained for a limited period (default 90 days).
- No click tracking on the ticker; analytics are privacy-friendly.

### 18.3 Accessibility (WCAG 2.2 AA)
Keyboard navigation everywhere; visible focus; skip-to-content link; semantic landmarks; alt text; form labels and error messages linked to fields; colour contrast AA in both themes; text resize to 200% without loss; motion respects reduced-motion; ticker pausable; dialogs trap focus and close with Esc; `lang` attribute set per page (`bn`/`en`) and on mixed-language fragments; tap targets ≥ 44 px. Automated checks (axe) in CI plus manual testing with a screen reader (TalkBack and NVDA) before launch.

### 18.4 Browser/device support
Last 2 versions of Chrome, Safari, Firefox, Edge; Android Chrome 90+; iOS Safari 14+; Samsung Internet; low-end Android (2 GB RAM) must stay smooth. Core reading works without JavaScript.

---

## 19. Analytics and reporting

- **Internal counters (lightweight):** article views (buffered in Redis → daily table), most read lists, section views, search terms, ad impressions/clicks, poll votes. Bots filtered.
- **External analytics (optional):** one privacy-friendly or standard analytics script, loaded after idle; ID set in Settings.
- **Admin reports:** top articles (24 h/7 d/30 d), by section, by author, publishing volume per day, translation coverage, comment volume and moderation times, ad performance, search terms with zero results, 404 report, Core Web Vitals summary (if RUM is enabled). Export CSV.
- Counters never block a request: failures are swallowed and logged.

---

## 20. Infrastructure and deployment

Hosting is not decided; any provider that supports the following is acceptable.

| Requirement | Detail |
|---|---|
| Servers | Linux; PHP-FPM 8.3+ with OPcache (preload optional); Nginx. Start with 1 web + 1 worker + 1 DB; design for horizontal scaling of web nodes (stateless; sessions/cache in Redis; media on shared/object storage). |
| Database | MySQL 8 with automated backups; optional read replica. |
| Redis | One instance (cache/sessions/queue); persistence enabled for queue. |
| CDN | Required for production (HTML caching + images/assets). |
| Storage | Local or S3-compatible bucket for media. |
| Queue worker | Supervisor with auto-restart; separate queues: `default`, `media`, `mail`, `seo`. |
| Scheduler | Cron every minute. |
| Environments | Local, Staging (mirror of production, password-protected, `noindex`), Production. |
| CI/CD | On every merge: tests, static analysis, lint, bundle-size and Lighthouse checks; deploy script with zero-downtime release (symlink switch), migrations, cache warm and CDN purge. |
| Observability | Error tracking, uptime monitor, log aggregation, queue and cache dashboards. |
| Capacity planning | Decided after traffic target is set (Pending inputs); load test (5.8) sizes the servers. |

---

## 21. Build phases

Order is designed so the site is fast from the first commit and every phase is releasable internally.

| Phase | Weeks | Contents | Exit criteria |
|---|---|---|---|
| **0. Foundation** | 2 | Laravel + Vue/Inertia starter, Tailwind + DaisyUI (light/dark only), CI, environments, Redis/queue, languages routing (`/`, `/en/`), UI string system, base layout, theme toggle script, font subsetting, performance budget checks, Lighthouse CI, staff auth + roles skeleton, audit log | Blank bilingual page passes budgets; roles work; CI green |
| **1. Core CMS** | 4 | Sections, tags, authors, media library (upload, variants, alt/captions), article editor (TipTap, revisions, workflow, scheduling), pages, menus, settings (General, Brand incl. logo upload) | Editors can write, review and publish an article in Bangla |
| **2. Public site** | 4 | Header (logo/text brand, menu, switcher, theme toggle), footer, home with blocks, section, article, tag, author, static pages, 404, search (FULLTEXT), RSS, full-page cache + purge, image pipeline, responsive QA, dark mode QA | Public pages meet performance budgets in both themes and languages |
| **3. Ticker, English and translation** | 3 | Ticker (items, tags, settings, API, CSS marquee), English content, side-by-side translation editor, coverage report, translator role, optional machine suggestion, hreflang | Ticker CRUD complete; English site live with fallbacks |
| **4. Readers and comments** | 3 | Reader auth, dashboard, saved articles, comments, moderation tools, reports, rate limits, emails | Comment lifecycle tested end to end; staff/reader separation verified |
| **5. Ads and SEO** | 3 | Ad placements, ads, advertisers, reports, global switches; SEO settings, schema, sitemaps, news sitemap, redirects, 404 log, robots, SEO audit hints | Ads render lazily with no CLS; sitemaps validate |
| **6. Polish, security, launch** | 3 | Galleries, videos, polls, newsletter, contact; accessibility audit; security review; load test; backups/restore test; content seeding; launch checklist | All acceptance criteria met; launch checklist signed |
Total ≈ 22 weeks for one small team; phases 3 and 5 can overlap. A usable Bangla-only newspaper can launch after Phase 2 if needed.

### 21.1 Launch checklist
DNS/SSL; CDN rules; robots/sitemaps submitted (Search Console, Bing, Google News publisher centre); analytics verified; email domain (SPF/DKIM/DMARC); backups verified; admin accounts and 2FA; default content (about, contact, privacy, terms, comment policy); logo, favicon and share image uploaded; ticker seeded; ads configured; load test passed; monitoring alerts on.

---

## 22. Acceptance criteria

**Brand, themes, language**
1. Only light and dark themes exist; no other theme is selectable or present in the build.
2. First visit follows OS theme; toggle persists; no flash of wrong theme on reload; cached HTML is identical for both themes.
3. Admin can upload a light logo and a dark logo, choose "Logo only / Logo + text / Text only", set alt text per language, and the header and footer update after cache purge; missing dark logo shows a warning; the logo causes no layout shift.
4. With no logo uploaded, the text masthead is shown.
5. Language switcher works on every page; article switcher goes to the translation if it exists; Bangla is default at `/`; English at `/en/`; hreflang correct.

**Performance**
6. Home and article pages meet Lighthouse mobile ≥ 85 (goal 90), LCP < 2.5 s, CLS < 0.1, INP < 200 ms in both themes and languages.
7. Page-weight budgets (5.2) pass in CI.
8. Anonymous page views are served from cache (≥ 95% CDN hit in load test); origin query count for uncached article ≤ 12.
9. Article page is readable and usable with JavaScript disabled.
10. Load test sustains the agreed target (Pending inputs) with p95 < 500 ms and zero errors.
11. Lightweight mode switch disables comments loading, ticker polling and ad refresh and reduces origin load.
12. Publishing or editing an article updates the public page within 10 seconds (cache purge).

**Content and CMS**
13. A Writer can draft and submit; an Editor can review, comment, approve and schedule; scheduled posts publish on time (±1 min).
14. Every item in the CRUD checklist (section 15) can be created, viewed, edited and deleted from the admin.
15. Media upload produces responsive WebP/AVIF variants; alt text is required for images in articles; replacing a file keeps references.
16. Revisions can be compared and restored; two editors on the same article get a soft-lock warning.
17. Roles and permissions match the matrix; a Section Editor cannot see or edit other sections' articles; permissions are enforced by policies (tested).

**Ticker**
18. Ticker appears below the main menu, loops seamlessly, pauses on hover/focus/button, respects reduced motion, and is hidden when no items exist for the language.
19. Ticker items can be created, edited, deleted, pinned, scheduled and ordered; source modes manual/automatic/mixed and sort options work; per-page switches work; colour-coded tags display with correct contrast; changes appear within 60 seconds.
20. No click tracking exists for the ticker.

**Readers and comments**
21. Readers register, verify email, log in, reset password, edit profile, save articles, view/edit/delete own comments; the reader area is separate from `/admin` (different login, guard, table, session).
22. Only verified readers can comment; limits, rate limits, edit window, replies (one level) and auto-close work.
23. Moderation modes, banned words, reports and auto-hide, bulk actions, reader suspension and audit log work.
24. Comments never appear inside cached HTML and load lazily.

**SEO**
25. Every page type has correct title, description, canonical, OG/Twitter, hreflang (where translated) and structured data; validated with a rich-results test.
26. Sitemaps and Google News sitemap are valid per language and update within 1 minute of publishing.
27. Slug changes create 301 redirects; 404 log and redirect manager work.

**Ads**
28. Ads can be created, scheduled, targeted and assigned to placements; they load lazily with reserved space (no CLS); impressions/clicks are reported; global and per-page-type switches work; ads never load on account/login/404 pages.

**Quality**
29. WCAG 2.2 AA checks pass (automated + manual), including both themes' contrast.
30. Security checklist (section 18.1) passes; backup restore tested.
31. English coverage report and translation queue work; "Needs review" appears when Bangla source changes after English approval.

---

## 23. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Traffic far above estimate | Slowdowns, cost | CDN HTML caching, lightweight mode, load test, stateless web nodes |
| Bangla fonts are heavy | Slow first load | Subset, 2 families, WOFF2, swap, preload one file |
| Bangla search quality (FULLTEXT ngram) | Poor results | Synonym list, move to Meilisearch if needed |
| Ads hurt speed and layout | Lower SEO and UX | Lazy load, reserved space, slot caps, one provider script |
| Comment abuse/spam | Moderation load, legal risk | Verified accounts, rate limits, holds, reports, banned words, audit log |
| Translation backlog | English site looks thin | Queue, coverage report, translator role, optional machine suggestions |
| Logo/brand not ready | Launch delay | Text masthead default; logo can be uploaded any time |
| Cache invalidation bugs | Stale news | Event-based purge tests, short TTL with stale-while-revalidate |
| Scope growth | Late delivery | Phases; anything outside this PRD goes to a backlog |
| Legal/policy text missing (comments, privacy) | Launch blocker | Owner supplies drafts before Phase 6 |

---

## 24. Pending inputs (do not block building)

| # | Item | Needed by | Default until decided |
|---|---|---|---|
| 1 | Final logo, brand colours, favicon | Phase 2 launch | Text masthead and token palette in 7.3 |
| 2 | Expected traffic (peak concurrent / daily page views) | Phase 5 (load test, hosting) | Build to the targets in section 5 |
| 3 | Machine-translation provider and budget | Phase 3 | Feature off |
| 4 | Comment policy and privacy/terms text (bn/en) | Phase 6 | Placeholder text |
| 5 | Social sign-in for readers (Google/Facebook) | After launch | Email + password only |
| 6 | Ad provider(s) (AdSense/GAM/direct only) | Phase 5 | Direct-sold image/HTML ads |
| 7 | Hosting/CDN provider | Phase 0 (staging) | Any that meets section 20 |
| 8 | Analytics choice | Phase 5 | Internal counters only |
| 9 | Newsletter sending provider | Phase 6 | Collect and export emails only |
| 10 | Weather widget on/off | Phase 2 | Off |

---

*End of document.*

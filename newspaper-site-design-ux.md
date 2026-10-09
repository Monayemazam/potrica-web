# Bilingual Online Newspaper — Design and UX Guide

**Aligned with:** `newspaper-site-prd.md` (final) and `newspaper-site-database-schema.md`
**Direction:** a **dense, news-packed** newspaper layout that is still easy to read. Light and dark themes only. Logo or text brand in the header. Lightweight by design.
**Note:** this guide sets the density rules for the whole site. Where the PRD's design section (7.1, 7.6) says "generous spacing", the spacing in this guide applies instead. Everything else in the PRD (themes, logo option, ticker, performance budgets) is unchanged.

---

## 1. Design direction

### 1.1 In one line
A classic, trustworthy newspaper front page: **many headlines visible at once, clear hierarchy, almost no empty space**, in a neutral palette that suits every age and taste.

### 1.2 Principles
1. **News first.** Every screen should show as many real headlines as comfortably fit. Whitespace is used to separate, never to decorate.
2. **Hierarchy does the work.** Size, weight, a thin rule and an image tell the reader what matters. Not colour, not effects.
3. **Neutral and timeless.** Black/ink text, off-white paper, one accent colour, one alert red. Nothing trendy. It must look right to a 20-year-old and a 70-year-old.
4. **Dense but legible.** Dense means tight spacing and compact cards, **not small text**. Headlines and body keep their reading sizes (section 4).
5. **Predictable.** The same card types, the same section header, the same positions everywhere.
6. **Fast.** Fewer decorative images, small thumbnails, text-heavy lists, no sliders (PRD section 5).
7. **Equal quality in both languages and both themes.**

### 1.3 What "dense" means in numbers
| Item | Airy (old) | Dense (this guide) |
|---|---|---|
| Gap between page sections | 48 px | **20–24 px** |
| Gap between cards in a grid | 24 px | **12–16 px** |
| Card inner padding | 20–24 px | **0–12 px** (many cards have none, just a divider) |
| Page side padding (phone) | 16 px | **12 px** (still 16 px for article text) |
| Headlines visible on home (desktop, first 2 screens) | ~25 | **60–90** |
| Headlines visible on home (phone, first 2 screens) | ~8 | **18–25** |
| Section blocks on home | 4–6 | **10–14** |

---

## 2. Theme tokens (light and dark only)

### 2.1 Colours
| Token | Light | Dark | Use |
|---|---|---|---|
| `--bg` | #F4F3EF | #101214 | Page background |
| `--surface` | #FFFFFF | #181B1F | Cards, ticker, header |
| `--surface-muted` | #ECEAE4 | #20242A | Tags, zebra rows, quote boxes |
| `--border` | #DEDBD3 | #2C3137 | Dividers, card borders |
| `--border-strong` | #C9C5BB | #444B53 | Section rules, inputs |
| `--ink` | #15181B | #F3F1EC | Headlines |
| `--text` | #22282E | #E2E0DB | Body |
| `--text-muted` | #5A636D | #A3AAB2 | Meta (time, author) |
| `--accent` | #0E5A6B | #5CC0D3 | Links, buttons, active tab |
| `--accent-contrast` | #FFFFFF | #0B1A1E | Text on accent |
| `--alert` | #C0392B | #F0776A | "Latest" label, urgent tag |
| `--success` / `--warning` | #1E7B4B / #8A5A00 | #5FCB91 / #E6B450 | Admin and form states |
All pairs pass WCAG AA in both themes. Brand colours from a future logo replace `--accent` only; the rest stays.

### 2.2 Section colours (small, optional)
Each section may have one accent from a fixed palette of 10 hues (used only for the small bar before the section title and the tiny section label on cards). Hues are tuned per theme to pass 4.5:1 on the surface. Suggested: Bangladesh (teal), International (indigo), Politics (maroon), Economy (olive), Sports (green), Entertainment (magenta), Technology (blue), Opinion (ink grey), Education (amber), Health (cyan). Colour is never the only signal; the section name is always written.

### 2.3 Theme behaviour
Follows the device on first visit, remembers the toggle, no flash on load (PRD 7.2). Shadows are not used in either theme; borders and background steps separate things. Photos are never inverted or dimmed except a subtle 5% darkening in dark mode (CSS filter, optional).

---

## 3. Layout system

### 3.1 Grid
| Screen | Container | Columns | Gutter | Side padding |
|---|---|---|---|---|
| ≥ 1280 px | max 1280 px | 12 | 16 px | 16 px |
| 1024–1279 px | fluid | 12 | 16 px | 16 px |
| 768–1023 px | fluid | 8 | 12 px | 12 px |
| 480–767 px | fluid | 4 | 12 px | 12 px |
| < 480 px | fluid | 4 | 8–12 px | 12 px |
Wide screens (≥ 1536 px) keep the 1280 px container; the page background shows on the sides, with optional skin/branding ad space off by default.

### 3.2 Spacing scale (dense)
2, 4, 6, 8, 12, 16, 20, 24, 32. Typical use: list item padding 8–10 px vertical; card gap 12–16; section gap 20–24; header block gap 8.

### 3.3 Radius, borders, elevation
Radius 6 px (cards, buttons, inputs), 4 px (tags), full (avatars, pills). 1 px borders. No shadows. Section header uses a 3 px ink rule.

### 3.4 Z-order
Sticky header 40, drawer 50, modal 60, toast 70. Ads never overlay content except the dismissable bottom sticky on phones.

---

## 4. Typography (dense but readable)

| Role | Font | Desktop | Phone | Weight | Line height |
|---|---|---|---|---|---|
| Lead headline (hero) | Serif Bangla | 34 px | 26 px | 700 | 1.3 |
| Secondary headline | Serif | 22–24 px | 20 px | 700 | 1.35 |
| Standard card headline | Serif | 19–20 px | 18 px | 700 | 1.4 |
| Compact list headline | Serif/Sans | 17 px | 17 px | 600 | 1.4 |
| Summary line (2 lines max) | Sans | 16 px | 15.5 px | 400 | 1.55 |
| Meta (time, section, author) | Sans | 14 px | 13.5 px | 400 | 1.4 |
| Section title | Serif | 22 px | 20 px | 700 | 1.3 |
| Menu items | Sans | 17 px | 17 px | 600 | 1.2 |
| **Article body** | Sans | **20 px** | **19 px** | 400 | **1.85** |
| Article H1 | Serif | 36 px | 27 px | 700 | 1.35 |
| Article H2 / H3 | Serif | 26 / 22 px | 22 / 20 px | 700 | 1.4 |
Minimum size anywhere readable by readers: **13.5 px** (meta only). Bangla line height never below 1.35 in headlines or 1.6 in text blocks. Headlines are limited to 2–3 lines with a clamp (`line-clamp`) on lists. Fonts as in PRD 5.4 (two families, subsetted WOFF2, swap). English uses the Latin side of the same families.
Text-size control (A− A A+) affects **article body only**.

---

## 5. Header, brand and navigation

### 5.1 Desktop header (top → bottom)
```
┌────────────────────────────────────────────────────────────────────────────┐
│ ৯ অক্টোবর ২০২৬, শুক্রবার    [f][X][▶]        বাংলা | English   ☀/☾   লগইন │  Utility bar 32 px (optional)
├────────────────────────────────────────────────────────────────────────────┤
│ [LOGO]  (optional text/tagline)              [ 🔍 অনুসন্ধান…        ]       │  Brand row 64–72 px
├────────────────────────────────────────────────────────────────────────────┤
│ প্রচ্ছদ  বাংলাদেশ  রাজনীতি  অর্থনীতি  আন্তর্জাতিক  খেলা  বিনোদন  প্রযুক্তি  ▾ আরও │  Main menu 44 px
├────────────────────────────────────────────────────────────────────────────┤
│ [সর্বশেষ ●]  লেখা চলছে… • লেখা চলছে… • লেখা চলছে…                    [ ❚❚ ]   │  Ticker 44–48 px
└────────────────────────────────────────────────────────────────────────────┘
```
Total header height ≈ 190 px on desktop (utility bar off: ≈ 158 px). The main menu is **sticky** (alone, 44 px) after scrolling; brand row and utility bar scroll away to save space.

### 5.2 Brand block (logo option)
- Modes (Admin → Brand): logo only, logo + text, text only. Left- or centre-aligned.
- Logo height 40–56 px desktop (default 48), 32–40 px phone (default 36); max width 220 / 160 px. SVG preferred. Separate light/dark logos (PRD 7.5); fallback is a text masthead in the serif face, 30 px bold.
- In the sticky bar the logo shrinks to a small mark/text (height 28 px) on the left of the menu.
- Alt text and tagline per language.

### 5.3 Phone header
```
┌──────────────────────────────────┐
│ ☰   [LOGO small]     বাং|EN  🔍 ☾│  Brand row 52 px (sticky)
├──────────────────────────────────┤
│ প্রচ্ছদ বাংলাদেশ রাজনীতি অর্থনীতি → │  Scrollable chips row 42 px (sticky with brand row)
├──────────────────────────────────┤
│ [সর্বশেষ●] running text…     ❚❚  │  Ticker 44 px (scrolls away)
└──────────────────────────────────┘
```
- The scrollable section chips row replaces the need to open the drawer for common sections; the last chip "আরও" opens the drawer.
- The drawer (☰) has: search, all sections (accordion), pages, language, theme, login, social. 
- Logo + language + search + theme stay visible; login and the rest are in the drawer.

### 5.4 Menu behaviour
- Desktop: hover/click opens a dropdown (sub-sections list; "mega" variant shows 4 latest headlines of that section). Keyboard accessible; opens on focus/Enter, closes on Esc.
- Overflow: items that do not fit go into "আরও / More".
- Active section is underlined with a 3 px accent line.

### 5.5 Latest News ticker (visual spec)
- Placement: **directly below the main menu** on every page type that has it enabled.
- Card: `--surface` background, 1 px border, 6 px radius, 48 px (desktop) / 44 px (phone) high, page-width.
- Left label: `--alert` background, white text "সর্বশেষ"/"Latest", 15 px bold, with a 8 px dot (pulses; static under reduced motion). Fixed width, never scrolls.
- Text: 17 px, `--ink`, items separated by a small "•" in `--text-muted`. Optional colour tag chip before an item (admin-defined colours; 13 px bold, 4 px radius, 2×8 px padding).
- Right: pause/play button 44×44 px.
- Motion: seamless CSS marquee at about 50 px/second, pauses on hover/focus/button, reduced-motion → scrollable list. Items are links; hover underlines in accent.
- No ticker on pages where the admin has turned it off; no empty bar.

---

## 6. Components (dense versions)

### 6.1 Section header
`3 px ink rule on top` → small colour bar (4×18 px, section colour) + serif title + right-aligned "সব দেখুন ›" link. Height 36 px. Gap below: 10 px.

### 6.2 Article cards
| Card | Anatomy | Used for |
|---|---|---|
| **Lead** | Image 16:9 (full column width), section label, headline 34 px, 2-line summary, meta | Home hero, section top |
| **Large** | Image 16:9, headline 22–24, 2-line summary | Home secondary (2–4), section grid |
| **Standard** | Image 3:2 top, headline 19–20, meta | 3- or 4-column grids |
| **Side-thumb** | Thumb 96×64 (left) + headline 17–18 (3 lines max) + meta | Mixed lists, phone default |
| **Text-only** | Headline 17 + meta, divider below | Dense lists, "more news" |
| **Numbered** | Big number (28 px serif, muted) + headline | Most read |
| **Opinion** | Round author photo 56 px + name + headline | Opinion strips |
| **Photo/Video** | Image with small play/camera badge, headline overlay-free (text below) | Gallery/video strips |
Cards have **no outer border or background by default** on lists; grids use a 1 px bottom divider. Boxes (`--surface` + 1 px border) are used only for: ticker, ad slots, most-read panel, newsletter, poll, comments.
Meta line: `section label · time ago` (e.g., "খেলা · ৫ মিনিট আগে"); icons for comment count (💬 12) and video/photo shown only when relevant. Time ago switches to a date after 24 h.

### 6.3 Labels and badges
`Opinion`, `Exclusive`, `Analysis`, `Sponsored`, `Fact-check` as small 13 px pills with the label colours from admin. "Updated" shows a small clock icon with time. No "NEW" badges.

### 6.4 Buttons, inputs
Primary button: accent background, 44 px high, 6 px radius, 17 px text. Secondary: 1 px border-strong. Link buttons: accent text, underline on hover. Inputs: 44 px, 1 px border-strong, 17 px text, visible label above (never placeholder-only). Focus ring 2 px accent + 2 px offset everywhere.

### 6.5 Pagination
Numbered pagination on section/tag/author/search/archive pages (SEO) with Previous/Next; the home "latest" list uses a **Load more** button (real link fallback).

### 6.6 Ad slots (see section 10)
Boxed, labelled "বিজ্ঞাপন", reserved size, `--surface-muted` placeholder while loading.

### 6.7 Share bar, tags, breadcrumbs
Share: 6 compact icon buttons (copy link, Facebook, X, WhatsApp, Messenger, Email), 40 px with 4 px gap, shown in one row (sticky left rail on desktop article pages ≥ 1280 px; inline row elsewhere). Tag chips: 14 px, `--surface-muted`, 4 px radius. Breadcrumb: 14 px, one line, truncated.

### 6.8 Skeletons and empty states
Skeleton blocks only for dynamic fragments (comments, login state, search suggestions). Empty states are short and useful ("এই মুহূর্তে কোনো মন্তব্য নেই। প্রথম মন্তব্য করুন।").

---

## 7. Page layouts

### 7.1 Home — desktop (≥ 1024 px)
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ [header + ticker]                                                           │
├──────────────┬──────────────────────────────────────────┬───────────────────┤
│ LEFT 3 cols  │ CENTRE 6 cols                            │ RIGHT 3 cols      │
│              │ ┌──────────────────────────────────────┐ │                   │
│ সর্বশেষ       │ │ LEAD image 16:9 + headline 34 px     │ │ সর্বাধিক পঠিত      │
│ (time-stamped│ │ + 2-line summary                     │ │ 1 …               │
│  list, 12    │ └──────────────────────────────────────┘ │ 2 …               │
│  text-only   │ ┌─────────────────┬────────────────────┐ │ 3 …  (8–10 items) │
│  headlines   │ │ Large 22 px     │ Large 22 px        │ │                   │
│  with time)  │ │ image + title   │ image + title      │ │ [AD 300×250]      │
│              │ └─────────────────┴────────────────────┘ │                   │
│ [AD 300×250] │ Side-thumb list ×5 (headline 18 px)      │ মতামত (4 items    │
│              │                                          │  with photos)     │
│ জনপ্রিয় ট্যাগ │                                          │ পোল / নিউজলেটার    │
├──────────────┴──────────────────────────────────────────┴───────────────────┤
│ [AD leaderboard 728×90 centered]                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│ SECTION BLOCKS — 2 blocks per row, each 6 cols:                             │
│ ┌──────────────── বাংলাদেশ ─────────┐ ┌──────────────── আন্তর্জাতিক ─────────┐ │
│ │ big image + title │ 4 text-only   │ │ big image + title │ 4 text-only   │ │
│ └───────────────────────────────────┘ └───────────────────────────────────┘ │
│ (repeat: রাজনীতি | অর্থনীতি, খেলা | বিনোদন, প্রযুক্তি | শিক্ষা, স্বাস্থ্য | জীবনযাপন)  │
├─────────────────────────────────────────────────────────────────────────────┤
│ ছবি ও ভিডিও strip (4 thumbs each)                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│ আরও সংবাদ (Load more): 3-column text/side-thumb list                         │
├─────────────────────────────────────────────────────────────────────────────┤
│ [footer]                                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```
- The three columns are separated by 1 px vertical borders (`--border`) like a newspaper.
- At 1024–1279 px the right column drops below the centre as a 3-column strip.
- Hero must be server-rendered and its image is the LCP element (high priority, correct size).

### 7.2 Home — tablet (768–1023 px)
Two columns: (lead + large cards) on the left (5/8), "সর্বশেষ" + "সর্বাধিক পঠিত" tabs on the right (3/8). Section blocks 1 per row with image + 4 text headlines. Ads 728×90 or 300×250.

### 7.3 Home — phone
1. Header + chips + ticker.
2. Lead card (image 16:9 full width, headline 26 px, one-line summary).
3. 4 side-thumb cards.
4. Ad (320×100 or 300×250).
5. Tabs: **সর্বশেষ | সর্বাধিক পঠিত** (both lists in HTML; tab switch needs no network): 8 items each.
6. Section blocks, each: header, 1 image card + 3 text-only rows. (Section chips at the top are a shortcut to each.)
7. Photo/video strip as a horizontal scroll row (snap) of 6 small thumbs.
8. Opinion strip.
9. Poll/newsletter (compact).
10. "আরও সংবাদ" list + Load more.
11. Footer (collapsible link groups to save height).
Typical phone density: 3–4 headlines visible per screen without feeling crowded.

### 7.4 Section page
- Desktop: section header + sub-section chips; row 1: Lead (6 cols) + 2 large cards (3+3); then a 2-column area: list (9 cols, side-thumb cards with 2-line summaries) + sidebar (3 cols: most read in section, ad, newsletter). Numbered pagination (20 per page).
- Phone: chips row, Lead card, side-thumb list, ad every 6 items.

### 7.5 Article page
```
Desktop (≥ 1280):
┌────┬────────────────────────────────────────────┬───────────────────┐
│Share│ breadcrumb · label chips                  │ SIDEBAR 300 px    │
│rail │ H1 36 px                                  │ [AD 300×250]      │
│     │ sub-headline 20 px                         │ সর্বশেষ (8 text)   │
│     │ byline row: avatar · author · date · time │ [AD sticky 300×600]│
│     │ · reading time · [EN/বাং link] · A− A A+   │ সর্বাধিক পঠিত      │
│     │ main image 16:9 + caption/credit          │                   │
│     │ BODY 20 px, max width 680 px               │                   │
│     │ … inline related box after ¶ 4             │                   │
│     │ … ad after ¶3 and ¶8                        │                   │
│     │ tags · share row · author box              │                   │
│     │ COMMENTS (lazy)                            │                   │
│     │ Related (6 side-thumb cards, 2 cols)       │                   │
└────┴────────────────────────────────────────────┴───────────────────┘
```
- Content column 680 px max (≈ 70 characters), sidebar 300 px with 24 px gap; below 1024 px the sidebar content moves after the comments.
- Phone: single column, 12 px sides for headers and 16 px for body text; byline in two lines; share row horizontally scrollable; text-size control near the byline; "related" shown as 4 side-thumb rows; bottom sticky ad (dismissable) and a slim "back to top" button.
- Reading progress bar (2 px accent line) under the sticky bar — pure CSS/scroll, optional.
- Pull-quote: 24 px serif with a 4 px accent left bar. Inline image: full column width with caption 14 px. Info box: `--surface-muted`, 12 px padding.
- Correction note: `--surface-muted` box with warning icon, above the body.

### 7.6 Tag, author, archive, search
- **Tag/author:** header block (name, short description, author photo and bio), then the same list as the section page. Author page shows 2-line bio, social icons.
- **Archive (by date):** compact text-only list grouped by section.
- **Search:** big input at top, filter chips (section, date, sort), results as side-thumb cards with highlighted terms; "Search in both languages" toggle; no-result state shows latest news and popular searches.

### 7.7 Gallery, video, poll, static pages
Gallery: 3–4 column thumb grid (2 on phone), lightbox with caption and keyboard arrows. Video: 16:9 facade with play button, title, description. Poll: boxed card, option rows 44 px, result bars after voting. Static pages: single column 720 px, plain typography. Contact: 2-column form + address on desktop.

### 7.8 404 and error pages
Short message, search box, 6 latest headlines. 500 page is static text with a link home.

### 7.9 Footer
Four link columns (desktop) / accordions (phone), brand block with logo (or text), short description, social icons, newsletter input (single row), copyright, language and theme toggles, back-to-top. Background `--surface-muted`; compact (padding 24 px). Footer is plain HTML.

---

## 8. Reader area and comments

### 8.1 Comments block (on article)
```
মন্তব্য (24)                                   সাজান: নতুন ▾
┌──────────────────────────────────────────────────────────┐
│ [avatar] আপনার মন্তব্য লিখুন…                  (0/1000)   │  (logged in)
│ [মন্তব্য করুন]                                             │
└──────────────────────────────────────────────────────────┘
(Not logged in: "মন্তব্য করতে লগইন করুন" + [লগইন] [নিবন্ধন])
 ───────────────────────────────────────────────────────────
 [avatar] রহিম  · ২ ঘণ্টা আগে · EN badge if English
 comment text (17 px)
 উত্তর দিন · রিপোর্ট
   └ [avatar] করিম · …  (one level of replies, indented 32 px)
 ───────────────────────────────────────────────────────────
 [আরও মন্তব্য দেখুন]
```
- Compact rows (12 px vertical padding), avatar 36 px, text 17 px.
- Pending comment shows to its author only: "আপনার মন্তব্য পর্যালোচনার জন্য অপেক্ষমাণ।"
- Own comments: Edit (within 15 min) and Delete. Report opens a small popover with reasons.
- A link to the comment policy sits under the box.
- Loaded lazily when near the viewport; skeleton while loading.

### 8.2 Reader pages (`/account`)
Same header/footer and theme as the public site. Inside: left menu (desktop) or top tabs (phone): **Overview, Profile, Saved articles, My comments, Notifications, Security**. Layout is a simple single column of cards (max width 820 px): forms with labels above fields, clear save buttons, success/error toasts. Saved articles use side-thumb rows with a remove button. Login/register pages are single centred forms (max 420 px) with show-password toggle, a clear error line, and links to reset/register. Account pages are quiet: no ads, no ticker (default off).

---

## 9. Admin panel design (English)

### 9.1 Principles
Fast for editors working many hours: dense tables, keyboard shortcuts, minimal clicks, clear status colours, never loses work. Light and dark themes (same two). Desktop first; usable on tablet; basic review/publish on phone.

### 9.2 Layout
- **Left sidebar** (240 px, collapsible to 64 px icons): Dashboard, Articles, Sections, Tags, Authors, Media, Pages, Menus, Homepage, Ticker, Comments, Readers, Galleries/Videos/Polls, Ads, SEO, Translations, Users and roles, Settings, Audit log. Items visible by permission only.
- **Top bar:** global search (articles, media, users), quick-create (+), notification bell, theme toggle, profile menu, "View site" link.
- **Content area:** page title + primary action on the right, filters row, table or form.
- Tables: sticky header, row height 44 px, bulk-select with a floating action bar, saved filters, column chooser, 25/50/100 per page.
- Forms: two-column on desktop (main fields left, settings right), sticky save bar at the bottom with Save draft / Submit / Publish.

### 9.3 Article editor
- Tabs: **Content, Media, Settings, SEO, Translation, History**.
- Content tab: title (large input with counter), sub-headline, summary, TipTap editor (toolbar sticky), word count and reading time, autosave status ("Saved 10:42").
- Right panel: status badge, publish time, section, tags, authors, labels, display switches (featured, top, comments, **add to ticker**, no ads).
- Translation tab: side-by-side Bangla/English with status chips, "Suggest English" (if enabled), "Mark reviewed".
- SEO tab: Google-style preview (light/dark), length meters, OG preview card.
- Soft-lock banner if another editor has it open.
- Shortcuts: Ctrl/Cmd+S save, Ctrl+Enter submit, Ctrl+K link, Ctrl+/ help.

### 9.4 Status colours
Draft grey, In review amber, Approved blue, Scheduled purple, Published green, Unpublished/Archived muted. Each has an icon and text (not colour alone).

### 9.5 Key admin screens
- **Dashboard:** widgets by role (see PRD 10.1), compact cards in a 3–4 column grid.
- **Media library:** grid with thumbnails (160 px), folder tree on the left, detail drawer on the right (alt text per language, caption, credit, focal-point picker, usage list), drag-and-drop upload area on top.
- **Homepage layout:** list of blocks with drag handle, type icon, enable switch, edit drawer; "Preview" button opens the live page in preview mode.
- **Ticker:** items table (drag to reorder, pin, status, tag, schedule, link), tags manager with live colour preview (light and dark) and contrast warning, settings tab (source mode, sort order, per-page switches, speed), a live strip preview at the top.
- **Comments moderation:** tabs (Pending, Reported, Published, Spam, Removed), card-per-comment rows with context link, Approve / Reject / Spam / Remove / Edit / Ban reader, bulk actions, keyboard shortcuts (A/R/S).
- **Brand settings:** logo upload slots for light, dark and per language with a live header preview in both themes.
- **Roles:** matrix grid of permissions with group toggles.

### 9.6 Feedback and safety
Toasts for success, inline errors on fields, confirm dialogs for destructive actions (type the name for bulk delete), undo for trash moves, unsaved-changes warning.

---

## 10. Ads in the layout

- Every ad sits in a labelled box ("বিজ্ঞাপন" 12 px muted, centred), with reserved dimensions and `--surface-muted` background until loaded.
- Desktop slots: leaderboard under the ticker on home/section (728×90), between home blocks (728×90), sidebar 300×250 and sticky 300×600, in-article after ¶3 and ¶8, below the article.
- Phone slots: 320×100 under the ticker, 300×250 inside lists every ~6 items and inside articles, 320×50 sticky bottom (dismissable, with a clear close button).
- Maximum slots per page from admin settings (default 4 phone / 6 desktop). Ads never push the first headline below the first screen on phones; the first ad is placed after the lead and four stories.
- Ad frames do not use brand colours or borders that look like article cards.

---

## 11. Interaction and motion

- Hover: underline headline in accent; no zoom effects on images.
- Focus: always visible; skip link "মূল বিষয়বস্তুতে যান".
- Motion: ticker, 150 ms colour/underline transitions, drawer slide 200 ms. All off with reduced-motion.
- No carousels, no auto-play video, no pop-ups. A language suggestion banner (for English browsers) is a slim dismissible bar under the header.
- Back-to-top button appears after 2 screens (44 px, bottom right, not over the sticky ad).
- Tabs and dropdowns work by keyboard (arrow keys, Esc).

---

## 12. Accessibility and age-friendliness

- WCAG 2.2 AA; contrast checked for tokens in both themes.
- Text size control for articles; browser zoom to 200% without breaking layout (dense grids collapse to fewer columns).
- 44 px touch targets even in dense lists (the whole row is the tap area).
- Clear labels, icons always paired with text for key actions (language, search, login).
- Simple language in the interface, consistent wording from the UI string manager.
- Screen reader order matches visual order; landmarks (`header`, `nav`, `main`, `aside`, `footer`); ticker is a labelled region and not a live region.
- Bangla specifics: no letter-spacing, no all-caps styling, no italic for Bangla text, adequate line height, proper `lang="bn"`.

---

## 13. Language and RTL/LTR notes

Both languages are left-to-right. The language switcher is always visible (header utility bar, phone brand row, footer). Layouts are the same in both languages; English headlines are slightly narrower so more fit, which is fine. Date/number formats follow PRD 6.6. Long English words never overflow cards (`overflow-wrap:anywhere` on headlines).

---

## 14. Performance rules inside the design

- Hero image 800–1280 px wide AVIF/WebP; thumbs 96×64 (192×128 for 2×) in lists; no image in text-only rows.
- At most 1 high-priority image per page; the rest lazy.
- Home page ships ≤ 60 KB JavaScript (PRD 5.2); tabs, accordion and theme toggle use tiny vanilla/Vue islands; section blocks are pure HTML.
- Dense does not mean heavy: use text-only rows heavily; every extra thumbnail must be justified.
- Icons: inline SVG sprite, about 30 icons.

---

## 15. Design deliverables and build notes

1. **Tokens file** (CSS variables for light/dark) and the DaisyUI theme config with only `light` and `dark`.
2. **Component library** in Vue/Blade: header, menu, ticker, cards (8 types), section header, ad slot, pagination, comments, forms, admin components.
3. **Page templates:** home, section, article, tag/author, search, gallery, video, static, account pages, 404.
4. **Visual QA checklist** per page: both themes, both languages, widths 360 / 390 / 768 / 1024 / 1280 / 1536, with a logo and without a logo, with ticker on and off, with long Bangla headlines (3-line clamp) and with missing images.
5. **Final brand pass** once the logo exists: update `--accent`, section colours if needed, and check contrast.

*End of document.*

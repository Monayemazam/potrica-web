# Bilingual Newspaper Website — Complete AI Development Prompt Sequence

## MASTER RULES — APPLY TO EVERY PROMPT

You are acting as a **Senior Laravel Architect, Senior Full-Stack Web Developer, Database Engineer, UI/UX Engineer, Performance Engineer, and QA Engineer**.

You are responsible for developing a production-ready, high-traffic bilingual online newspaper. You must follow these three project documents throughout the entire development process:

1. `newspaper-site-prd.md`
2. `newspaper-site-database-schema.md`
3. `newspaper-site-design-ux.md`

### Document authority

- The PRD is the primary authority for functionality, architecture, security, workflows, and acceptance criteria.
- The database schema defines the database structure, relationships, indexes, constraints, and migration order. If it conflicts with the PRD, the PRD wins. Document any resulting schema discrepancy rather than silently changing either specification.
- The Design & UX Guide governs the visual design, content density, page layouts, typography, spacing, responsive behaviour, and component appearance.
- Do not invent replacement requirements or silently omit documented features.
- If a requirement is genuinely ambiguous, document it and use the least disruptive interpretation consistent with the files.

### Mandatory development rules

1. Inspect the existing repository before changing any code.
2. Do not rebuild the application from scratch simply because a page or feature is broken.
3. Do not delete, rename, replace, or rewrite working components, routes, migrations, database tables, or business logic without a demonstrated need.
4. Do not create duplicate components, controllers, migrations, tables, settings, or routes when existing implementations can be extended.
5. Preserve all existing working features and user data.
6. Never use destructive database commands, reset migrations, truncate production tables, or drop tables to resolve ordinary development problems.
7. Do not run `migrate:fresh` or equivalent destructive operations.
8. Follow the specified stack: PHP 8.3+, Laravel 12 or 13 according to the existing compatible installation, Vue 3, Inertia.js, Tailwind CSS, DaisyUI, MySQL 8, Redis, TipTap, and Spatie Laravel Permission.
9. Public content must support Bangla by default at `/` and English at `/en/`. The admin interface must remain English-only.
10. Only two themes are allowed: `light` and `dark`.
11. Public pages must be server-rendered and cacheable, with minimal JavaScript. Use lightweight interactive components only where needed.
12. Every required visible content component must be editable through the appropriate admin CMS module.
13. Implement real functionality, database persistence, validation, authorization, error handling, and tests. Do not use fake buttons, placeholder CRUD, hard-coded demo content presented as production content, or success messages for operations that have not succeeded.
14. Use Laravel policies, form requests, service/action classes, Eloquent relationships, and queued jobs where appropriate. Keep controllers thin.
15. Every feature must work on desktop, tablet, and mobile, and in both languages and themes where applicable.
16. After each prompt, run the relevant tests and checks, inspect the changed files, and report any remaining issues.
17. Do not proceed to the next phase if the current phase has unresolved critical errors, broken existing features, or failed mandatory tests.
18. Do not claim that a feature, test, migration, or performance target is complete unless you have actually verified it.

### Required response after every prompt

Report:

- What was inspected.
- What was implemented.
- Files created or modified.
- Database changes and migrations, if any.
- Tests and commands executed, with actual results.
- Existing features verified as still working.
- Outstanding bugs or blockers.
- The exact next prompt that should be executed.

Work incrementally. Make small, reviewable changes. Never perform a complete redesign or architectural rewrite unless explicitly authorized.

---

# PROMPT 0 — Existing Project Audit and Safe Development Plan

Before changing anything, thoroughly inspect the existing repository and all three specification files.

Do not modify application code during this phase.

### Tasks

1. Identify the existing framework versions, PHP version, Node version, installed packages, database configuration, frontend architecture, routing, authentication guards, and build commands.
2. Inspect the current public website, admin panel, reader authentication pages, database migrations, models, controllers, middleware, policies, Vue components, layouts, stylesheets, and tests.
3. Review Git status and existing commits if Git is available. Never assume a backup or clean working tree exists.
4. If Git is available, identify uncommitted work and preserve it. Do not reset, revert, or discard user changes.
5. Verify whether the three specification files exist in the project repository. If they do not, use the provided copies as the requirements for this session and report that they should be retained in the repository.
6. Create a feature inventory covering every module in the PRD, every database entity, and every public/admin layout in the UX Guide.
7. For every feature, classify it as:
   - Working and verified.
   - Partially implemented.
   - Implemented but broken.
   - Missing.
   - Not yet verified.
8. Inspect the existing migrations against the database schema. Identify missing columns, duplicate tables, incorrect relationships, unsafe migration history, and schema conflicts.
9. Check for duplicated homepage sections, repeated components, broken images, route conflicts, CSS collisions, missing styles, JavaScript errors, and failed API requests.
10. Inspect staff authentication separately from reader authentication.
11. Establish a baseline for tests, application errors, frontend builds, database health, and performance where tooling is available.
12. Map each feature to its relevant PRD section, schema tables, and UX section.
13. Identify dependencies and propose the safest implementation order based on the project's actual current state.

### Deliverable

Produce a development audit and feature-completion matrix. Include a list of critical issues, existing working features that must not be disturbed, required database changes, and a phase-by-phase implementation plan.

Do not implement anything yet. Wait for approval before applying changes.

---

# PROMPT 1 — Safe Foundation, Architecture and Design System

Implement or repair only the foundation necessary for the remaining project phases. Reuse the current application wherever possible.

### Tasks

1. Establish a consistent Laravel domain structure using the PRD's recommended organization, including Articles, Sections, Media, Ads, Ticker, Readers, Comments, SEO, Menus, Pages, and Settings.
2. Configure the Vue 3 and Inertia.js architecture using the existing Laravel starter kit.
3. Configure Tailwind CSS and DaisyUI with exactly two themes: `light` and `dark`. Disable other DaisyUI themes.
4. Implement the design tokens from the Design & UX Guide:
   - Light and dark background, surface, border, text, accent, alert, success, and warning colours.
   - Fixed, accessible section-colour palette.
   - Dense spacing scale and six-pixel standard radius.
   - Newspaper-style rules and borders instead of decorative shadows.
5. Configure bilingual routing:
   - Bangla: `/`
   - English: `/en/`
   - Locale-aware route generation and middleware.
6. Build the centralized UI translation system using the `ui_translations` table and language configuration.
7. Ensure all user-facing strings use the translation system rather than hard-coded interface text.
8. Configure the initial staff authentication architecture using the `web` guard.
9. Prepare the staff role and permission infrastructure using Spatie Laravel Permission.
10. Configure Redis for cache, sessions, and queues according to the environment.
11. Establish shared public, admin, and reader layout boundaries.
12. Configure the correct timezone behaviour: store timestamps in UTC and display them using the configurable timezone, defaulting to `Asia/Dhaka`.
13. Implement the initial theme behaviour:
    - Follow the operating system on first visit.
    - Persist the user's theme selection.
    - Prevent a flash of the wrong theme during initial rendering.
    - Keep theme selection independent of public HTML cache keys.
14. Establish code formatting, static analysis, automated testing, build checks, and performance-budget tooling appropriate to the current project.

### Safety constraints

- Do not replace the existing authentication system blindly.
- Do not change database schema unrelated to the foundation.
- Do not introduce a third theme.
- Do not build the entire site in this phase.

### Acceptance criteria

The application builds successfully, locale routing works, the shared layouts render correctly, themes persist without a flash, and existing functionality remains intact.

---

# PROMPT 2 — Database Architecture, Migrations and Seed Data

Implement and verify the database architecture using `newspaper-site-database-schema.md` as the structural reference and the PRD as the authority when conflicts arise.

### Tasks

1. Compare every required table, column, foreign key, index, unique constraint, translation table, soft-delete field, and pivot table against the existing database.
2. Follow the schema document's migration order.
3. Implement missing tables and fields through safe, incremental Laravel migrations.
4. Preserve existing data and existing migration history.
5. Use MySQL 8, InnoDB, and `utf8mb4_unicode_ci`.
6. Use unsigned BIGINT primary keys where specified and maintain explicit foreign-key delete behaviour.
7. Implement the separate `users` and `readers` tables and preserve their independent authentication guards.
8. Implement translation tables with the required unique `(entity_id, locale)` constraints.
9. Implement required article relationships, including sections, tags, authors, related articles, revisions, corrections, review comments, and media usage.
10. Implement database entities for:
    - Languages, settings and UI translations.
    - Staff, permissions, notifications and login history.
    - Readers, saved articles, email changes and suspensions.
    - Sections, tags, authors and article labels.
    - Articles, translations, revisions and editorial workflow.
    - Pages, menus, menu items and homepage blocks.
    - Ticker items and ticker tags.
    - Media, folders, translations, tags and usage tracking.
    - Comments, reports, moderation logs and banned words.
    - Galleries, videos, polls, options and votes.
    - Advertisers, campaigns, ads, placements and statistics.
    - Redirects, 404 logs, search synonyms, search logs and article view statistics.
    - Subscribers, contact messages and audit logs.
11. Implement all other tables documented in the complete schema, including correction and translation entities that might be overlooked.
12. Implement required database indexes, including article listing indexes, translation slug uniqueness, full-text search indexes, ticker indexes, and comment indexes.
13. Configure Laravel queues and failed-job persistence consistently with the specified Redis architecture.
14. Create idempotent seeders for the two supported languages, essential settings, default article labels, initial roles and permissions, and documented starter configuration.
15. Ensure the seeders do not create duplicate roles, settings, languages, or records when rerun.
16. Review the complete schema for foreign-key ordering and circular dependencies.
17. If the existing schema conflicts with the PRD, report the discrepancy and implement the PRD-compatible result using a safe migration.

### Acceptance criteria

- All required database entities exist.
- Relationships and constraints are valid.
- Fresh test installations can migrate safely.
- Existing data is not destroyed.
- Seeders are repeatable.
- Migration and model relationship tests pass.

Do not continue until the database foundation is verified.

---

# PROMPT 3 — Staff Authentication, Roles, Permissions and Audit Logging

Implement the complete staff access-control system without mixing it with reader accounts.

### Staff roles

Implement the documented roles:

- Super Admin
- Admin
- Editor-in-Chief
- Section Editor
- Reporter / Writer
- Contributor
- Translator
- Photographer / Media Manager
- Ad Manager
- SEO Manager
- Moderator
- Analyst (read-only)

Support custom roles created by authorized Super Admin users.

### Tasks

1. Implement staff login, logout, password reset, invitations, account status management, and email verification where required.
2. Implement role-based permissions using Spatie Laravel Permission.
3. Create a permission matrix covering all protected resources and actions.
4. Enforce section-level restrictions for Section Editors and ownership restrictions for Writers.
5. Restrict Translators to English translations and prevent them from editing the Bangla source.
6. Restrict Media Managers, Ad Managers, SEO Managers, Moderators, and Analysts to their permitted modules.
7. Protect Super Admin accounts and critical system settings according to the PRD.
8. Implement two-factor authentication and enforcement settings as required by the staff security specification.
9. Record staff login history, failed login attempts, new-device alerts, and relevant audit events.
10. Implement force logout and account deactivation.
11. Build permission-aware admin navigation so users see only the modules they can access.
12. Protect backend routes with policies and middleware. Hiding a menu item is not authorization.
13. Ensure every sensitive mutation is authorized and audited.
14. Apply the documented secure session, password, CSRF, rate-limiting, and cookie rules.

### Acceptance criteria

Test both permitted and forbidden operations for every role, especially cross-section access, publishing restrictions, translator restrictions, and read-only reporting.

No staff role may gain access simply by entering an admin URL directly.

---

# PROMPT 4 — Core CMS: Articles, Editorial Workflow and Content Management

Build the complete editorial CMS, including the article editor and its related entities.

### Article editor

Implement the six editor tabs:

- Content
- Media
- Settings
- SEO
- Translation
- History

Use TipTap for article-body editing.

### Article fields and controls

Implement:

- Title, subtitle, summary and body.
- Section and additional sections.
- Tags, multiple authors and article labels.
- Featured image, focal-point controls, alt text, caption and image credit.
- Dateline and byline override.
- Featured and top/breaking flags.
- Comments on/off, ticker inclusion, most-read inclusion and ads on/off.
- Author visibility, share visibility and related-article mode.
- Scheduled publishing, expiration and unpublishing.
- Per-language translation status and publication time.
- SEO metadata and social-sharing metadata.
- Reading-time calculation.
- Autosave, revisions, comparison and restoration.
- Soft edit locks when another editor is working on the same article.

### Editorial workflow

Implement the documented article statuses:

- Draft
- In review
- Approved
- Scheduled
- Published
- Unpublished
- Archived

Support Writer submission, editorial review, reviewer comments, approval, requested changes, scheduling, publishing and unpublishing.

Apply the correct role permissions at every transition.

### Related CMS modules

Implement complete CRUD for:

- Sections and one-level subsections.
- Tags, including merge functionality.
- Authors and author biographies.
- Article labels and their translations.
- Static pages and their translations.

All relevant entities must support search, filters, pagination, validation, authorization, audit logging, and soft deletion where specified.

### Acceptance criteria

A Writer can create and submit a Bangla draft. An Editor can review it, request changes, approve it, schedule it, and publish it. The public site must not expose drafts, expired articles, unpublished translations, or content that has been soft-deleted.

Verify revisions, edit locks, relationships, scheduled publishing, and cache-purge hooks.

---

# PROMPT 5 — Media Library and Image Processing Pipeline

Implement the production media library before building image-dependent public layouts.

### Tasks

1. Implement media folders, upload, search, filtering, pagination, tagging, replacement, deletion, and Trash management.
2. Support image metadata, alt text, captions, credits, focal points, and translations wherever specified.
3. Implement secure uploads with file-type validation, size limits, MIME verification, safe storage, and SVG sanitization.
4. Generate responsive image variants through queued jobs.
5. Generate WebP and AVIF variants, with JPEG fallback and appropriate browser support.
6. Generate only the required image sizes from the specification, including responsive widths from 160 through 1600 pixels where needed.
7. Configure `srcset`, `sizes`, image dimensions, aspect ratios, lazy loading, asynchronous decoding, and high priority for the lead image.
8. Preserve references when a media file is replaced.
9. Track media usage so editors can identify where a file is used.
10. Prevent deletion of files that are still referenced unless a safe replacement or cleanup workflow is used.
11. Support light and dark logos, per-language logo configuration, favicon, share image, and footer logo.
12. Ensure missing logo assets fall back to the text masthead.
13. Make image-processing failures visible in the admin and retryable through queues.
14. Integrate media permissions and audit logging.

### Acceptance criteria

Uploaded media can be attached to articles, pages, galleries, and branding. Responsive variants load correctly, image replacements preserve references, and image failures do not break page layouts.

---

# PROMPT 6 — Public Newspaper Website and Design Implementation

Build the public-facing website according to the complete Design & UX Guide.

Do not redesign the guide into a generic modern blog template.

### Global design

- Newspaper-style content density.
- Restrained neutral palette and accessible accent colours.
- Serif headlines and readable sans-serif body text.
- Thin borders, ruled section headers and compact spacing.
- No decorative shadows.
- Exactly light and dark themes.
- Equal quality in Bangla and English.

Follow the specified responsive grid, typography scale, spacing, image ratios, card types and breakpoints.

### Header and navigation

Implement:

- Optional utility bar with date, social links, language switcher, theme toggle and login.
- Brand row supporting logo only, logo + text, and text only.
- Search.
- Main navigation and overflow menu.
- Subsection dropdowns and accessible mega-menu behaviour.
- Sticky desktop navigation.
- Mobile menu drawer and horizontally scrollable section chips.
- Language switching without requiring JavaScript.
- Footer with editable text, copyright, social links and navigation groups.

All brand, menu, footer and layout settings must be managed through the CMS.

### Homepage

Implement the documented desktop, tablet and mobile layouts.

The desktop homepage must include:

- Left column with latest-news headlines, timestamps, advertising and popular tags.
- Centre column with the lead story, secondary stories and side-thumbnail lists.
- Right column with most-read stories, advertising, opinion, polls and newsletter.
- Leaderboard advertising area.
- Configurable section blocks in paired columns.
- Photo and video strip.
- Additional news list with a functional Load more control.
- Footer.

On tablet, use the specified two-column layout.

On mobile, implement the lead story, four side-thumbnail stories, ad slot, latest/most-read tabs, section blocks, media strip, opinion strip, poll/newsletter, additional stories and compact footer.

### Important hero restriction

Do not implement a heavyweight autoplay slideshow or image carousel. The design specification calls for a single server-rendered hero image as the LCP element. Keep the hero image optimized and load it at high priority.

### Other public pages

Implement:

- Section and subsection pages.
- Article detail pages.
- Tag pages.
- Author pages.
- Search results.
- Date archives.
- Static content pages.
- Gallery pages.
- Video pages.
- Poll interfaces.
- Contact page.
- RSS feeds.
- Custom 404 and lightweight 500 pages.
- Maintenance page with staff bypass.
- Reader-facing account entry points.

### Article detail page

Implement the article title, subtitle, author information, publication and update dates, reading time, translation link, text-size controls, image caption and credit, article body, share tools, tags, related stories, latest-news sidebar, most-read stories, ad slots, corrections, and comments area.

Article body text must meet the specified reading sizes and line heights.

### Acceptance criteria

Every required page renders correctly at the documented breakpoints, in both languages and both themes. No duplicated content blocks, broken image references, overlapping headers, missing styling, or placeholder navigation should remain.

---

# PROMPT 7 — Latest News Ticker and Bilingual Translation Workflow

Implement the ticker and translation system as separate, fully functional modules.

## Part A: Latest News ticker

Support manual, automatic and mixed source modes.

Implement:

- Ticker items and ticker tags.
- Bangla and English text.
- Article-linked and external-linked items.
- Pinning, scheduling, expiry, sorting and manual ordering.
- Automatic article sourcing and mixed sourcing.
- Configurable tag colours and accessible contrast.
- Per-page ticker visibility.
- Ticker API and caching.
- Admin controls for ticker settings.
- Pause/play controls and reduced-motion support.
- CSS-based seamless marquee.
- Keyboard and focus behaviour.
- Cache invalidation when ticker settings or items change.

The ticker must sit directly below the main menu wherever enabled.

Hide it when no valid items exist for the current language.

**Never add ticker click tracking.** This is an explicit project exclusion.

## Part B: Translation workflow

Implement:

- Side-by-side Bangla and English editors.
- Translation statuses: Missing, Draft, Needs review and Published.
- Translation assignment and queue filters.
- Translation coverage reports by section and month.
- Translator-specific permissions.
- Source-changed detection.
- Machine-assisted translation suggestions behind the documented admin toggle.
- Human review before publication when required.
- Translation provider configuration without exposing credentials.
- Interface string management with missing-translation highlighting.
- Correct language-switch behaviour for articles and other translated entities.

When Bangla source content changes after English approval, mark the English translation as Needs review according to the PRD.

The translation system must not silently publish machine-generated English text when human review is required.

### Acceptance criteria

Ticker changes become visible within the specified cache window. Language switching points to the corresponding translated content when available and falls back to the other-language homepage when necessary. Translation permissions and review requirements are enforced on the server.

---

# PROMPT 8 — Reader Accounts, Saved Articles and Comment Moderation

Implement reader functionality as a completely separate application area from staff administration.

### Authentication separation

Use:

- The `readers` table.
- The `reader` guard.
- Separate reader login and password-reset flows.
- Separate reader session handling.
- `/account` and `/en/account`.
- Appropriate `noindex` and private caching rules.

Staff and readers must never share authentication sessions, login pages, password-reset flows or account tables.

### Reader features

Implement:

- Registration with name, email, password and terms/privacy acceptance.
- Email verification.
- Login, logout, remember me and password reset.
- Profile editing, avatar upload, biography, preferred language and theme.
- Saved articles with pagination and remove controls.
- My Comments with status, edit and delete actions.
- Reply notification preference.
- Newsletter subscription preference.
- Email change with re-verification.
- Password change and other-session logout.
- Account deletion with confirmation and the documented anonymization behaviour.

### Reader dashboard

Create the Overview, Profile, Saved Articles, My Comments, Notifications and Security screens.

Use the UX Guide's compact, accessible form and card patterns.

### Comments

Implement:

- Verified-reader-only posting.
- Plain-text content, validation and length limits.
- One-level replies.
- Published, pending, rejected and removed states.
- Configurable moderation modes.
- Banned-word handling.
- Reports and report reasons.
- Automatic hiding at configured thresholds.
- Rate limiting.
- Comment edit windows and automatic closure.
- Reader suspensions with history.
- Moderation logs and bulk actions.
- Optional email notifications for replies.
- Staff moderation permissions.
- Anonymization or removal of comments when a reader account is deleted, according to settings.

Load comments lazily and keep them outside cached public HTML.

### Acceptance criteria

Test the complete lifecycle: registration, verification, login, comment submission, moderation, replies, reporting, suspension, saved articles, account deletion and anonymization.

Unauthenticated or unverified users must not bypass the applicable restrictions.

---

# PROMPT 9 — Administration Dashboard and Full CMS CRUD Coverage

Build the complete English-only admin interface.

Use a consistent sidebar, top bar, permission-aware navigation, dense tables, filter rows, bulk actions, pagination and sticky form save bars.

### Dashboard

Implement role-appropriate dashboard widgets, including relevant editorial queues, publishing activity, translation progress, comment moderation, ad performance, system health and reports.

Analysts must remain read-only.

### Required admin modules

Verify that the following are fully implemented:

1. Dashboard.
2. Articles and article labels.
3. Sections and subsections.
4. Tags.
5. Authors.
6. Media library and folders.
7. Pages.
8. Menus and menu items.
9. Homepage layout and blocks.
10. Ticker items, tags and settings.
11. Comments and moderation.
12. Readers.
13. Galleries.
14. Videos.
15. Polls.
16. Advertisers.
17. Campaigns.
18. Ads and placements.
19. SEO tools.
20. Translation queue and interface strings.
21. Staff users and roles.
22. General settings.
23. Brand settings.
24. Header and footer settings.
25. Appearance settings.
26. Performance settings.
27. Comment and reader settings.
28. Email and integration settings.
29. Maintenance tools.
30. Redirect manager and 404 log.
31. Newsletter subscribers.
32. Contact message inbox.
33. Search synonyms and search logs.
34. Audit log.
35. Trash and supported recovery operations.

### CRUD requirements

For every applicable module, implement:

- Create.
- Read/list.
- Search.
- Filtering.
- Pagination.
- View details.
- Update.
- Delete or soft-delete as specified.
- Validation.
- Authorization.
- Audit logging.
- Cache invalidation.
- Empty, loading and error states.

Implement drag ordering where specified, CSV import/export where required, bulk actions, and saved filters where applicable.

Do not count a module as complete simply because a navigation item or empty page exists.

### Acceptance criteria

Run the PRD's full CRUD coverage checklist against the actual application. Report each module's verified Create, Read, Update and Delete status.

---

# PROMPT 10 — Advertising, SEO, Search, RSS and Discovery

Implement the advertising and discoverability systems without compromising public performance.

## Advertising

Implement:

- Advertisers.
- Campaigns.
- Ads and creatives.
- Ad placements.
- Ad-to-placement assignments.
- Scheduling and expiration.
- Targeting rules.
- Direct-sold image/HTML ads and configured provider integration.
- Impression and click statistics.
- Daily reporting.
- Ad-expiry notifications.
- Global ads-off switch.
- Per-page-type switches.
- Mobile and desktop slot limits.
- Reserved dimensions to prevent layout shifts.

Ads must load asynchronously and lazily near the viewport. Do not load ads on account, login or 404 pages.

Never allow ad scripts to block initial content rendering.

## SEO

Implement:

- Page-specific title, meta description and canonical URL.
- Open Graph and Twitter metadata.
- Correct hreflang links.
- Structured data and schema types.
- NewsArticle schema and applicable organization metadata.
- XML sitemaps.
- Google News sitemap.
- Robots.txt management.
- Search Console and Bing verification settings.
- Redirect manager supporting 301 and 302.
- Slug-change redirects.
- Redirect hit counters and CSV import/export.
- 404 logging and one-click redirect creation.
- SEO audit hints.
- RSS feeds at `/feed` and `/en/feed`, plus per-section feeds.
- Pagination with crawlable links.
- Correct noindex behaviour for private pages.

## Search

Implement MySQL FULLTEXT search with the specified Bangla-compatible ngram approach where supported by the deployment environment.

Add:

- Language-aware search.
- Configurable minimum query length.
- Search synonyms.
- Pagination.
- Search logs when enabled.
- Zero-result reporting.
- Rate limiting.
- A path to Meilisearch through Laravel Scout if needed later.

### Acceptance criteria

Validate metadata and structured data across all page types. Verify sitemap generation after publishing, slug-change redirects, RSS output, ad placement behaviour, and the absence of layout shifts caused by advertisements.

---

# PROMPT 11 — Galleries, Videos, Polls, Newsletter, Contact and Corrections

Complete the remaining public-facing features and their administration.

## Galleries

Implement gallery CRUD, bilingual content, scheduling, publication status, image ordering, responsive thumbnails and the lightweight gallery lightbox.

## Videos

Implement video CRUD, bilingual metadata, scheduling, status management, thumbnails and click-to-load embeds.

Do not autoplay videos. Do not load heavy third-party embeds before user interaction.

## Polls

Implement:

- Question and 2–8 options.
- Open and close dates.
- One vote per browser and per IP window according to the PRD.
- Server-side validation and rate limiting.
- Results shown after voting.
- Cached results endpoint.
- CSV export.
- Homepage visibility control.
- Full admin CRUD.

## Newsletter

Implement email-only subscriptions, double opt-in, confirmation, unsubscribe, subscription status, CSV export, and administrative management.

Newsletter sending must use the configured external email provider. Do not build an unrequested newsletter delivery platform.

## Contact

Implement the contact form with name, email, subject, message, honeypot and rate limiting.

Store submissions in the admin inbox, send notification emails, and support read/unread, replied, archived, deleted and spam states as defined by the schema and PRD.

## Static pages and corrections

Ensure the required pages are available and editable, including About, Contact, Editorial Policy, Ethics, Privacy, Terms, Advertise, Corrections and Team.

Implement article corrections and correction translations according to the database schema and editorial requirements.

### Acceptance criteria

All features persist to the database, are permission-protected, work in both languages, and include appropriate validation, empty states and error handling.

---

# PROMPT 12 — Performance, Caching and High-Traffic Readiness

Audit the complete application against the performance requirements. Optimize measured problems rather than blindly rewriting the architecture.

### Required targets

- LCP below 2.5 seconds.
- INP below 200 milliseconds.
- CLS below 0.1.
- Lighthouse mobile score at least 85, with a goal of 90, on homepage and article pages.
- Cached CDN TTFB below 200 milliseconds.
- Origin TTFB below 80 milliseconds for cached-page requirements as specified.
- Compressed JavaScript, CSS, image and total page-weight budgets from the PRD.
- At least 95% CDN cache hits for anonymous traffic in the load-test scenario.
- Uncached article-page database query count no greater than 12.
- Publishing changes reflected publicly within 10 seconds.

These are targets to measure and verify, not numbers to claim without evidence.

### Tasks

1. Implement full-page caching for anonymous public pages.
2. Use locale and path in cache keys, not the user's theme or identity.
3. Keep private and user-specific information out of cached HTML.
4. Implement cache invalidation for article changes, translations, sections, tags, authors, menus, homepage blocks, ticker, ads and settings.
5. Implement stale-while-revalidate behaviour and cache-stampede protection where supported.
6. Buffer article views in Redis and flush counters asynchronously.
7. Flush denormalized counters through scheduled jobs instead of writing on every request.
8. Prevent N+1 queries through eager loading and query-count tests.
9. Queue media conversions, emails, sitemap generation, search indexing and other expensive operations.
10. Configure separate queue workloads as specified.
11. Ensure public JavaScript is loaded only for necessary interactions.
12. Lazy-load below-the-fold images and interactive components.
13. Optimize Bangla fonts using subsetted WOFF2 files and the documented family/weight limits.
14. Implement the configurable lightweight mode.
15. Enforce ad weight and slot limits.
16. Add bundle-size checks, Lighthouse CI, cache tests, query-count tests and load-test procedures.
17. Document CDN, Redis, worker, scheduler and production configuration requirements.

### Acceptance criteria

Produce actual test reports and identify any target that cannot yet be verified. Do not disable features or remove required UI elements just to improve benchmark scores.

---

# PROMPT 13 — Security, Accessibility, Emails, Monitoring and Production Readiness

Conduct a comprehensive security and launch-readiness audit.

## Security

Verify:

- HTTPS, HSTS and security headers.
- Secure cookies, CSRF protection and session handling.
- Server-side validation and authorization.
- Sanitized rich-text HTML.
- Escaped reader comments.
- Secure file uploads and sanitized SVG.
- Password hashing and rate limiting.
- Safe secret storage in environment variables.
- Protection against SQL injection and mass assignment.
- Staff/reader guard separation.
- Two-factor authentication and staff access restrictions.
- Audit logging.
- Privacy-safe IP logging and retention.
- Backup and restore procedures.

## Accessibility

Audit against WCAG 2.2 AA, including:

- Keyboard navigation.
- Focus visibility.
- Accessible labels and form errors.
- Contrast in both themes.
- Semantic headings and landmarks.
- Image alternatives.
- Reduced-motion behaviour.
- Screen-reader announcements.
- Responsive layouts and text resizing.

## Emails and notifications

Implement the documented queued email templates and events, including:

- Staff invites and password resets.
- New-device alerts.
- Reader verification, welcome, password reset, email change and deletion emails.
- Comment replies and moderation digests.
- Article review and approval notifications.
- Translation assignments and review notifications.
- Newsletter confirmation.
- Contact notifications.
- Ad-expiry reminders.
- Failed-job and queue-backlog alerts.

Use editable templates, including Bangla and English reader-facing templates, and provide both HTML and plain-text email versions.

## Monitoring and deployment

Configure and document:

- Error tracking.
- Uptime monitoring.
- Queue monitoring.
- Cache metrics.
- Slow-query logging.
- Production log management.
- Daily database backups and the specified retention periods.
- Media backups.
- Restore testing.
- Staging and production environment separation.
- Scheduled tasks.
- Zero-downtime deployment strategy.
- Migration execution and rollback considerations.
- Cache warming and CDN purging.
- Search Console and sitemap submission checklist.

### Acceptance criteria

Run automated and manual checks where possible. Fix critical and high-severity issues before launch. Report any external infrastructure configuration that requires the project owner's action.

---

# PROMPT 14 — Final End-to-End Audit and Missing-Feature Repair

Do not redesign the website or rebuild the project.

Perform a final requirements-driven audit against all three specification files and the feature inventory from Prompt 0.

### Phase A: Requirements audit

Check every requirement in:

- PRD sections 1–24.
- Database schema sections 1–16.
- Design & UX Guide sections 1–10.

Inspect the complete files, not only their summaries.

For each requirement, mark:

- PASS — implemented and verified.
- PARTIAL — implemented but incomplete.
- FAIL — missing or broken.
- BLOCKED — cannot be verified because of an external dependency.

### Phase B: Feature completeness

Verify:

- Every documented public page exists.
- Every required admin module works.
- Every applicable CRUD operation works.
- All roles enforce the correct permissions.
- Bangla and English behaviour is correct.
- Both themes work.
- Reader and staff authentication remain separate.
- All database relationships and translations work.
- Editorial workflows and scheduled publishing work.
- Media processing and replacement work.
- Ticker controls and cache invalidation work.
- Reader accounts, saved articles and comments work.
- Ads and SEO features work.
- Galleries, videos, polls, newsletter and contact work.
- Emails, notifications and audit logs work.
- Security, accessibility and performance checks are covered.

### Phase C: Regression tests

Run:

- Laravel tests.
- Frontend build and lint checks.
- Static analysis.
- Database migration checks on a disposable test database.
- Route and authorization checks.
- CRUD integration tests.
- Authentication and workflow tests.
- Both-language and both-theme UI checks.
- Responsive layout checks.
- Lighthouse and bundle-size checks.
- Cache invalidation tests.
- SEO and sitemap validation.
- Load tests where the environment supports them.

Check that the final changes have not removed, duplicated, or broken previously working features.

### Phase D: Repair

Fix verified missing or broken features in small, isolated changes.

After each repair, rerun relevant tests and inspect dependent modules for regressions.

Do not silently change the PRD, database schema or UX requirements to make a failed test pass.

### Final deliverable

Produce a final report containing:

1. A requirements coverage matrix.
2. All passing, partial, failed and blocked requirements.
3. The final database and migration status.
4. The final role/permission test results.
5. Performance measurements and outstanding targets.
6. Security and accessibility findings.
7. Outstanding deployment requirements.
8. A list of files changed during the final repair.
9. Confirmation of which features remain incomplete.

The project is ready for launch only when the mandatory acceptance criteria have been verified and no unresolved critical issues remain.
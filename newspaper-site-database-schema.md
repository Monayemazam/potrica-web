# Bilingual Online Newspaper — Database Schema

**Database:** MySQL 8 · **Charset/collation:** `utf8mb4` / `utf8mb4_unicode_ci` · **Engine:** InnoDB
**Aligned with:** `newspaper-site-prd.md` (final). Where this document and the PRD differ, the PRD wins; tell me and I will fix the schema.
**Naming:** English, snake_case, plural table names. Sample DDL is plain MySQL; build the real thing as Laravel migrations in the order given in section 14.

---

## 1. Conventions

1. **Primary keys:** `id BIGINT UNSIGNED AUTO_INCREMENT`. Public article URLs use this id (`/news/{id}/{slug}`), shared across languages.
2. **Timestamps:** `created_at`, `updated_at` (`TIMESTAMP NULL`) on every table unless stated. Stored in **UTC**; displayed in `Asia/Dhaka` (setting).
3. **Soft deletes:** `deleted_at TIMESTAMP NULL` on articles, pages, media, readers' comments (status instead), authors, tags, sections, galleries, videos, polls, ads. Hard delete only from the Trash by Admin/Super Admin or by retention jobs.
4. **Translations:** every translatable entity has `<entity>_translations` with `<entity>_id`, `locale CHAR(2)`, translated columns, and `UNIQUE (<entity>_id, locale)`. `bn` row is required for public entities; `en` is optional. Non-translatable fields stay on the base table.
5. **Locale values:** `bn`, `en` (FK-like to `languages.code`, not enforced by FK to keep rows light).
6. **Statuses:** `VARCHAR(20)` with a CHECK-style list documented per table (validated in PHP enums; MySQL `ENUM` avoided for easier change).
7. **JSON columns:** used only for config/targeting/snapshots, never for fields that must be filtered or joined.
8. **Foreign keys:** declared with explicit `ON DELETE` behaviour. Hot read tables (`articles`, `comments`, `article_views_daily`, `ad_stats_daily`) use FKs but avoid cascading deletes of large sets in request time (deletes run in queued jobs).
9. **Counters** (`view_count`, `comment_count`) are denormalised and updated by scheduled flush jobs, not per request.
10. **Sanitised HTML** only in `article_translations.body` and `page_translations.body`.
11. **Never store:** plain-text passwords, card data, full IP addresses longer than needed (IPs in logs are stored as anonymised/hashed where noted).

---

## 2. Entity overview

```
languages ─ settings ─ ui_translations
users(staff) ─ roles/permissions (spatie) ─ section_user ─ staff_login_history
readers ─ saved_articles ─ reader_email_changes
sections ─┬─ section_translations
articles ─┬─ article_translations ─ article_revisions
          ├─ article_section (extra sections) ─ article_tag ─ article_author
          ├─ article_labels(+translations) ─ article_corrections(+translations)
          ├─ article_review_comments ─ article_related
          └─ comments ─ comment_reports ─ comment_moderation_logs
tags ─ authors ─ media_folders ─ media(+translations) ─ media_usages
pages ─ menus ─ menu_items ─ homepage_blocks
ticker_tags ─ ticker_items (+translations)
galleries ─ gallery_items; videos; polls ─ poll_options ─ poll_votes
advertisers ─ campaigns ─ ads ─ ad_placements ─ ad_placement_ad ─ ad_stats_daily
redirects ─ not_found_logs ─ search_logs ─ search_synonyms ─ article_views_daily
subscribers ─ contact_messages ─ banned_words ─ reader_suspensions
audit_logs ─ notifications ─ jobs/failed_jobs (Laravel)
```

---

## 3. Core and configuration

```sql
CREATE TABLE languages (
  id TINYINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  code CHAR(2) NOT NULL UNIQUE,              -- bn, en
  name VARCHAR(50) NOT NULL,                 -- Bangla
  native_name VARCHAR(50) NOT NULL,          -- বাংলা
  url_prefix VARCHAR(10) NULL,               -- NULL for default (bn), 'en' for English
  is_default TINYINT(1) NOT NULL DEFAULT 0,
  is_active TINYINT(1) NOT NULL DEFAULT 1,
  sort SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL
);

CREATE TABLE settings (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `group` VARCHAR(40) NOT NULL,              -- general, brand, header, performance, comments, ticker...
  `key` VARCHAR(100) NOT NULL,
  locale CHAR(2) NULL,                       -- NULL = applies to all languages
  value JSON NULL,
  is_public TINYINT(1) NOT NULL DEFAULT 0,   -- may be exposed to front-end config
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  UNIQUE KEY uq_settings (`group`,`key`,locale)
);

CREATE TABLE ui_translations (               -- interface strings (buttons, labels, emails snippets)
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  `key` VARCHAR(150) NOT NULL,               -- e.g. 'article.read_more'
  locale CHAR(2) NOT NULL,
  value TEXT NOT NULL,
  group_name VARCHAR(50) NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  UNIQUE KEY uq_ui (`key`,locale),
  KEY idx_ui_group (group_name)
);
```

Settings keys are listed in section 12.

---

## 4. Staff users, roles, permissions

```sql
CREATE TABLE users (                         -- STAFF ONLY (guard: web)
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(120) NOT NULL,
  email VARCHAR(190) NOT NULL UNIQUE,
  email_verified_at TIMESTAMP NULL,
  password VARCHAR(255) NOT NULL,
  avatar_media_id BIGINT UNSIGNED NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'active',   -- active, inactive, invited
  two_factor_secret TEXT NULL,
  two_factor_recovery_codes TEXT NULL,
  two_factor_confirmed_at TIMESTAMP NULL,
  admin_theme VARCHAR(10) NOT NULL DEFAULT 'system', -- light, dark, system
  last_login_at TIMESTAMP NULL,
  last_login_ip VARBINARY(16) NULL,
  remember_token VARCHAR(100) NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL,
  KEY idx_users_status (status)
);

CREATE TABLE password_reset_tokens (         -- staff
  email VARCHAR(190) PRIMARY KEY,
  token VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NULL
);

-- spatie/laravel-permission standard tables (guard_name = 'web'):
-- permissions, roles, model_has_permissions, model_has_roles, role_has_permissions
-- Permission names: resource.action, e.g. articles.create, articles.publish, ticker.manage, comments.moderate

CREATE TABLE section_user (                  -- sections a Section Editor / Writer may work in
  user_id BIGINT UNSIGNED NOT NULL,
  section_id BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (user_id, section_id),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE staff_login_history (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL,
  ip_hash CHAR(64) NOT NULL,
  user_agent VARCHAR(255) NULL,
  device_hash CHAR(64) NULL,                 -- for 'new device' alerts
  succeeded TINYINT(1) NOT NULL DEFAULT 1,
  created_at TIMESTAMP NULL,
  KEY idx_slh_user (user_id, created_at),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE notifications (                 -- in-admin bell (Laravel database notifications)
  id CHAR(36) PRIMARY KEY,
  type VARCHAR(190) NOT NULL,
  notifiable_type VARCHAR(190) NOT NULL,
  notifiable_id BIGINT UNSIGNED NOT NULL,
  data JSON NOT NULL,
  read_at TIMESTAMP NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  KEY idx_notif (notifiable_type, notifiable_id, read_at)
);
```
Sessions, cache and queues use **Redis** (no `sessions`/`cache` tables). Laravel `jobs` is not used; `failed_jobs` table is kept for failure history.

---

## 5. Readers (separate from staff)

```sql
CREATE TABLE readers (                       -- guard: reader. No roles.
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(80) NOT NULL,                 -- display name shown on comments
  email VARCHAR(190) NOT NULL UNIQUE,
  email_verified_at TIMESTAMP NULL,
  password VARCHAR(255) NOT NULL,
  avatar_path VARCHAR(255) NULL,             -- own small uploads, not the staff media library
  bio VARCHAR(160) NULL,
  locale CHAR(2) NOT NULL DEFAULT 'bn',
  theme_pref VARCHAR(10) NOT NULL DEFAULT 'system',  -- light, dark, system
  notify_replies TINYINT(1) NOT NULL DEFAULT 1,
  newsletter_opt_in TINYINT(1) NOT NULL DEFAULT 0,
  status VARCHAR(20) NOT NULL DEFAULT 'active',      -- active, suspended
  comment_trust TINYINT UNSIGNED NOT NULL DEFAULT 0, -- approved comments count (capped) for auto-publish rule
  last_login_at TIMESTAMP NULL,
  remember_token VARCHAR(100) NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  deleted_at TIMESTAMP NULL,
  KEY idx_readers_status (status),
  KEY idx_readers_created (created_at)
);

CREATE TABLE reader_password_reset_tokens (
  email VARCHAR(190) PRIMARY KEY,
  token VARCHAR(255) NOT NULL,
  created_at TIMESTAMP NULL
);

CREATE TABLE reader_email_changes (          -- pending email change (confirm via link)
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  reader_id BIGINT UNSIGNED NOT NULL,
  new_email VARCHAR(190) NOT NULL,
  token_hash CHAR(64) NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP NULL,
  KEY idx_rec_reader (reader_id),
  FOREIGN KEY (reader_id) REFERENCES readers(id) ON DELETE CASCADE
);

CREATE TABLE reader_suspensions (            -- history of comment suspensions
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  reader_id BIGINT UNSIGNED NOT NULL,
  suspended_by BIGINT UNSIGNED NOT NULL,     -- users.id
  reason VARCHAR(255) NOT NULL,
  starts_at TIMESTAMP NOT NULL,
  ends_at TIMESTAMP NULL,                    -- NULL = permanent
  lifted_at TIMESTAMP NULL,
  lifted_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL,
  KEY idx_rs_reader (reader_id, ends_at),
  FOREIGN KEY (reader_id) REFERENCES readers(id) ON DELETE CASCADE
);

CREATE TABLE saved_articles (
  reader_id BIGINT UNSIGNED NOT NULL,
  article_id BIGINT UNSIGNED NOT NULL,
  created_at TIMESTAMP NULL,
  PRIMARY KEY (reader_id, article_id),
  KEY idx_sa_article (article_id),
  FOREIGN KEY (reader_id) REFERENCES readers(id) ON DELETE CASCADE
);
```
Account deletion: reader row is soft-deleted then anonymised by a job (name → "Deleted user", email → random hash, avatar removed); comments remain anonymised or are removed per the `comments.on_account_delete` setting.

---

## 6. Content structure

### 6.1 Sections
```sql
CREATE TABLE sections (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  parent_id BIGINT UNSIGNED NULL,            -- one level only (enforced in PHP)
  accent_key VARCHAR(20) NULL,               -- fixed palette key that passes contrast
  icon VARCHAR(40) NULL,
  sort SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  show_in_menu TINYINT(1) NOT NULL DEFAULT 1,
  show_on_home TINYINT(1) NOT NULL DEFAULT 1,
  status VARCHAR(20) NOT NULL DEFAULT 'active',     -- active, hidden
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL,
  KEY idx_sections_parent (parent_id, sort),
  FOREIGN KEY (parent_id) REFERENCES sections(id) ON DELETE SET NULL
);

CREATE TABLE section_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  section_id BIGINT UNSIGNED NOT NULL,
  locale CHAR(2) NOT NULL,
  name VARCHAR(120) NOT NULL,
  slug VARCHAR(160) NOT NULL,
  description TEXT NULL,
  meta_title VARCHAR(190) NULL,
  meta_description VARCHAR(320) NULL,
  og_image_media_id BIGINT UNSIGNED NULL,
  UNIQUE KEY uq_st_section (section_id, locale),
  UNIQUE KEY uq_st_slug (locale, slug),
  FOREIGN KEY (section_id) REFERENCES sections(id) ON DELETE CASCADE
);
```

### 6.2 Articles
```sql
CREATE TABLE article_labels (                -- Opinion, Exclusive, Analysis, Sponsored, Fact-check
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  code VARCHAR(40) NOT NULL UNIQUE,
  color_bg VARCHAR(7) NULL, color_text VARCHAR(7) NULL,
  schema_type VARCHAR(40) NULL,              -- e.g. OpinionNewsArticle
  is_active TINYINT(1) NOT NULL DEFAULT 1,
  sort SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL
);
CREATE TABLE article_label_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  label_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL, name VARCHAR(60) NOT NULL,
  UNIQUE KEY uq_alt (label_id, locale),
  FOREIGN KEY (label_id) REFERENCES article_labels(id) ON DELETE CASCADE
);

CREATE TABLE articles (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  section_id BIGINT UNSIGNED NOT NULL,                 -- primary section
  label_id BIGINT UNSIGNED NULL,
  main_media_id BIGINT UNSIGNED NULL,
  focal_x TINYINT UNSIGNED NULL, focal_y TINYINT UNSIGNED NULL,  -- 0-100 override of media focal point
  status VARCHAR(20) NOT NULL DEFAULT 'draft',         -- draft, in_review, approved, scheduled, published, unpublished, archived
  published_at TIMESTAMP NULL,                         -- scheduled or actual
  first_published_at TIMESTAMP NULL,
  expires_at TIMESTAMP NULL,
  dateline VARCHAR(120) NULL,
  is_featured TINYINT(1) NOT NULL DEFAULT 0,           -- hero candidate
  is_top TINYINT(1) NOT NULL DEFAULT 0,                -- "top/breaking" flag
  allow_comments TINYINT(1) NOT NULL DEFAULT 1,
  add_to_ticker TINYINT(1) NOT NULL DEFAULT 0,
  include_in_most_read TINYINT(1) NOT NULL DEFAULT 1,
  no_ads TINYINT(1) NOT NULL DEFAULT 0,
  hide_author TINYINT(1) NOT NULL DEFAULT 0,
  hide_share TINYINT(1) NOT NULL DEFAULT 0,
  byline_override VARCHAR(190) NULL,
  related_mode VARCHAR(10) NOT NULL DEFAULT 'auto',    -- auto, manual
  view_count BIGINT UNSIGNED NOT NULL DEFAULT 0,       -- flushed from Redis
  comment_count INT UNSIGNED NOT NULL DEFAULT 0,       -- published comments only
  locked_by BIGINT UNSIGNED NULL,                      -- soft edit lock
  locked_at TIMESTAMP NULL,
  created_by BIGINT UNSIGNED NOT NULL,
  updated_by BIGINT UNSIGNED NULL,
  published_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL,
  KEY idx_art_status_pub (status, published_at),
  KEY idx_art_section_pub (section_id, status, published_at),
  KEY idx_art_featured (is_featured, status, published_at),
  KEY idx_art_ticker (add_to_ticker, status, published_at),
  KEY idx_art_creator (created_by, status),
  KEY idx_art_views (status, view_count),
  FOREIGN KEY (section_id) REFERENCES sections(id),
  FOREIGN KEY (label_id) REFERENCES article_labels(id) ON DELETE SET NULL,
  FOREIGN KEY (created_by) REFERENCES users(id)
);

CREATE TABLE article_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  article_id BIGINT UNSIGNED NOT NULL,
  locale CHAR(2) NOT NULL,
  title VARCHAR(190) NOT NULL,
  subtitle VARCHAR(255) NULL,
  summary VARCHAR(400) NULL,
  body MEDIUMTEXT NULL,                                -- sanitised HTML
  slug VARCHAR(190) NOT NULL,
  ticker_text VARCHAR(140) NULL,                       -- optional short text for the ticker
  image_alt VARCHAR(255) NULL, image_caption VARCHAR(500) NULL, image_credit VARCHAR(190) NULL,
  reading_minutes TINYINT UNSIGNED NULL,
  -- SEO
  meta_title VARCHAR(190) NULL, meta_description VARCHAR(320) NULL,
  canonical_url VARCHAR(500) NULL,
  robots_noindex TINYINT(1) NOT NULL DEFAULT 0, robots_nofollow TINYINT(1) NOT NULL DEFAULT 0,
  og_title VARCHAR(190) NULL, og_description VARCHAR(320) NULL, og_image_media_id BIGINT UNSIGNED NULL,
  twitter_card VARCHAR(30) NULL,
  schema_type VARCHAR(40) NOT NULL DEFAULT 'NewsArticle',
  news_keywords VARCHAR(255) NULL,
  -- Translation workflow
  translation_status VARCHAR(20) NOT NULL DEFAULT 'draft',  -- missing, draft, needs_review, published
  is_machine_assisted TINYINT(1) NOT NULL DEFAULT 0,
  machine_reviewed_by BIGINT UNSIGNED NULL, machine_reviewed_at TIMESTAMP NULL,
  source_changed_at TIMESTAMP NULL,                          -- Bangla changed after this version was approved
  assigned_translator_id BIGINT UNSIGNED NULL,
  translated_by BIGINT UNSIGNED NULL,
  published_at TIMESTAMP NULL,                               -- per-language publish time (defaults to article)
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  UNIQUE KEY uq_at_article (article_id, locale),
  UNIQUE KEY uq_at_slug (locale, slug),
  KEY idx_at_status (locale, translation_status),
  KEY idx_at_assignee (assigned_translator_id, translation_status),
  FULLTEXT KEY ft_at_search (title, subtitle, summary, body) WITH PARSER ngram,
  FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE
);
```
**Public visibility rule:** an article appears in language L when `articles.status='published' AND articles.published_at<=NOW() AND (expires_at IS NULL OR expires_at>NOW()) AND article_translations(locale=L).translation_status='published'` and a translation row exists. For `bn` the row is always required.

```sql
CREATE TABLE article_section (               -- additional sections
  article_id BIGINT UNSIGNED NOT NULL, section_id BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (article_id, section_id), KEY idx_as_section (section_id, article_id),
  FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE
);
CREATE TABLE article_tag (
  article_id BIGINT UNSIGNED NOT NULL, tag_id BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (article_id, tag_id), KEY idx_atg_tag (tag_id, article_id),
  FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE
);
CREATE TABLE article_author (
  article_id BIGINT UNSIGNED NOT NULL, author_id BIGINT UNSIGNED NOT NULL,
  sort TINYINT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (article_id, author_id), KEY idx_aa_author (author_id, article_id),
  FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE
);
CREATE TABLE article_related (               -- manual related picks
  article_id BIGINT UNSIGNED NOT NULL, related_id BIGINT UNSIGNED NOT NULL,
  sort TINYINT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (article_id, related_id),
  FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE
);

CREATE TABLE article_revisions (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  article_id BIGINT UNSIGNED NOT NULL,
  locale CHAR(2) NOT NULL,
  snapshot JSON NOT NULL,                    -- title, subtitle, summary, body, slug, seo
  is_autosave TINYINT(1) NOT NULL DEFAULT 0,
  user_id BIGINT UNSIGNED NOT NULL,
  created_at TIMESTAMP NULL,
  KEY idx_ar_article (article_id, locale, created_at),
  FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE
);                                           -- keep last 50 per article+locale (pruned by job)

CREATE TABLE article_review_comments (       -- internal editorial thread
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  article_id BIGINT UNSIGNED NOT NULL, user_id BIGINT UNSIGNED NOT NULL,
  body TEXT NOT NULL, kind VARCHAR(20) NOT NULL DEFAULT 'note',   -- note, change_request, approval
  is_resolved TINYINT(1) NOT NULL DEFAULT 0,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  KEY idx_arc_article (article_id, created_at),
  FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE
);

CREATE TABLE article_corrections (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  article_id BIGINT UNSIGNED NOT NULL, position VARCHAR(10) NOT NULL DEFAULT 'top', -- top, bottom
  user_id BIGINT UNSIGNED NOT NULL, created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  KEY idx_ac_article (article_id),
  FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE
);
CREATE TABLE article_correction_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  correction_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL, body VARCHAR(1000) NOT NULL,
  UNIQUE KEY uq_act (correction_id, locale),
  FOREIGN KEY (correction_id) REFERENCES article_corrections(id) ON DELETE CASCADE
);
```

### 6.3 Tags and authors
```sql
CREATE TABLE tags (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  article_count INT UNSIGNED NOT NULL DEFAULT 0,
  status VARCHAR(20) NOT NULL DEFAULT 'active',
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL
);
CREATE TABLE tag_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  tag_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL,
  name VARCHAR(100) NOT NULL, slug VARCHAR(140) NOT NULL, description TEXT NULL,
  meta_title VARCHAR(190) NULL, meta_description VARCHAR(320) NULL,
  UNIQUE KEY uq_tt (tag_id, locale), UNIQUE KEY uq_tt_slug (locale, slug),
  KEY idx_tt_name (locale, name),
  FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE
);

CREATE TABLE authors (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT UNSIGNED NULL,              -- optional link to staff user
  photo_media_id BIGINT UNSIGNED NULL,
  email VARCHAR(190) NULL,                   -- private
  social_links JSON NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'active',
  article_count INT UNSIGNED NOT NULL DEFAULT 0,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL,
  KEY idx_authors_user (user_id)
);
CREATE TABLE author_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  author_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL,
  name VARCHAR(120) NOT NULL, slug VARCHAR(160) NOT NULL,
  title VARCHAR(120) NULL, bio TEXT NULL,
  UNIQUE KEY uq_aut (author_id, locale), UNIQUE KEY uq_aut_slug (locale, slug),
  FOREIGN KEY (author_id) REFERENCES authors(id) ON DELETE CASCADE
);
```

---

## 7. Media

```sql
CREATE TABLE media_folders (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  parent_id BIGINT UNSIGNED NULL, name VARCHAR(120) NOT NULL,
  created_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  KEY idx_mf_parent (parent_id),
  FOREIGN KEY (parent_id) REFERENCES media_folders(id) ON DELETE CASCADE
);

CREATE TABLE media (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  folder_id BIGINT UNSIGNED NULL,
  type VARCHAR(10) NOT NULL,                 -- image, video, audio, document
  disk VARCHAR(20) NOT NULL DEFAULT 'public',
  path VARCHAR(500) NOT NULL,                -- original file path
  file_name VARCHAR(255) NOT NULL,
  mime VARCHAR(100) NOT NULL,
  size_bytes BIGINT UNSIGNED NOT NULL,
  width SMALLINT UNSIGNED NULL, height SMALLINT UNSIGNED NULL,
  duration_seconds INT UNSIGNED NULL,
  hash CHAR(64) NOT NULL,                    -- sha256, duplicate detection
  variants JSON NULL,                        -- {"webp":{"320":"path",...},"avif":{...},"jpg":{...}}
  dominant_color CHAR(7) NULL,
  focal_x TINYINT UNSIGNED NULL, focal_y TINYINT UNSIGNED NULL,   -- 0-100
  version SMALLINT UNSIGNED NOT NULL DEFAULT 1,                   -- bumps on replace (cache-busting)
  processing_status VARCHAR(15) NOT NULL DEFAULT 'pending',       -- pending, done, failed
  uploaded_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL,
  KEY idx_media_folder (folder_id, type),
  KEY idx_media_hash (hash),
  KEY idx_media_uploader (uploaded_by, created_at),
  FULLTEXT KEY ft_media_name (file_name),
  FOREIGN KEY (folder_id) REFERENCES media_folders(id) ON DELETE SET NULL
);

CREATE TABLE media_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  media_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL,
  title VARCHAR(190) NULL, alt VARCHAR(255) NULL, caption VARCHAR(500) NULL,
  credit VARCHAR(190) NULL,
  UNIQUE KEY uq_mt (media_id, locale),
  FOREIGN KEY (media_id) REFERENCES media(id) ON DELETE CASCADE
);

CREATE TABLE media_usages (                  -- "used in N places" and delete protection
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  media_id BIGINT UNSIGNED NOT NULL,
  usable_type VARCHAR(60) NOT NULL,          -- article, page, gallery, author, ad, setting...
  usable_id BIGINT UNSIGNED NOT NULL,
  role VARCHAR(40) NULL,                     -- main_image, inline, og_image, logo_light...
  UNIQUE KEY uq_mu (media_id, usable_type, usable_id, role),
  KEY idx_mu_usable (usable_type, usable_id),
  FOREIGN KEY (media_id) REFERENCES media(id) ON DELETE CASCADE
);

CREATE TABLE media_tags (                    -- simple labels for library search
  media_id BIGINT UNSIGNED NOT NULL, tag VARCHAR(60) NOT NULL,
  PRIMARY KEY (media_id, tag), KEY idx_mtag (tag),
  FOREIGN KEY (media_id) REFERENCES media(id) ON DELETE CASCADE
);
```
Add the missing foreign keys after `media` exists: `users.avatar_media_id`, `articles.main_media_id`, `authors.photo_media_id`, `*_translations.og_image_media_id` → `media(id) ON DELETE SET NULL`.

---

## 8. Pages, menus, homepage

```sql
CREATE TABLE pages (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  template VARCHAR(20) NOT NULL DEFAULT 'default',   -- default, wide, contact
  status VARCHAR(20) NOT NULL DEFAULT 'draft',       -- draft, published
  show_in_sitemap TINYINT(1) NOT NULL DEFAULT 1,
  system_key VARCHAR(40) NULL UNIQUE,                -- about, contact, privacy, terms, comment_policy...
  created_by BIGINT UNSIGNED NULL, updated_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL
);
CREATE TABLE page_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  page_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL,
  title VARCHAR(190) NOT NULL, slug VARCHAR(190) NOT NULL, body MEDIUMTEXT NULL,
  meta_title VARCHAR(190) NULL, meta_description VARCHAR(320) NULL,
  og_image_media_id BIGINT UNSIGNED NULL, robots_noindex TINYINT(1) NOT NULL DEFAULT 0,
  UNIQUE KEY uq_pt (page_id, locale), UNIQUE KEY uq_pt_slug (locale, slug),
  FOREIGN KEY (page_id) REFERENCES pages(id) ON DELETE CASCADE
);

CREATE TABLE menus (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  location VARCHAR(30) NOT NULL UNIQUE,      -- main, footer_1..footer_4, utility, mobile_extra
  name VARCHAR(80) NOT NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL
);
CREATE TABLE menu_items (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  menu_id BIGINT UNSIGNED NOT NULL, parent_id BIGINT UNSIGNED NULL,
  link_type VARCHAR(15) NOT NULL,            -- section, page, tag, article, url, archive
  linkable_id BIGINT UNSIGNED NULL, url VARCHAR(500) NULL,
  open_new_tab TINYINT(1) NOT NULL DEFAULT 0, icon VARCHAR(40) NULL,
  is_mega TINYINT(1) NOT NULL DEFAULT 0,
  starts_at TIMESTAMP NULL, ends_at TIMESTAMP NULL,
  sort SMALLINT UNSIGNED NOT NULL DEFAULT 0, is_active TINYINT(1) NOT NULL DEFAULT 1,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  KEY idx_mi_menu (menu_id, parent_id, sort),
  FOREIGN KEY (menu_id) REFERENCES menus(id) ON DELETE CASCADE,
  FOREIGN KEY (parent_id) REFERENCES menu_items(id) ON DELETE CASCADE
);
CREATE TABLE menu_item_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  menu_item_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL, label VARCHAR(100) NOT NULL,
  UNIQUE KEY uq_mit (menu_item_id, locale),
  FOREIGN KEY (menu_item_id) REFERENCES menu_items(id) ON DELETE CASCADE
);

CREATE TABLE homepage_blocks (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  type VARCHAR(30) NOT NULL,                 -- hero, top_list, most_read, section_grid, opinion, gallery, video, poll, newsletter, latest, ad
  config JSON NOT NULL,                      -- source (manual ids / section ids / tag / most_read), count, layout variant, link
  starts_at TIMESTAMP NULL, ends_at TIMESTAMP NULL,
  show_desktop TINYINT(1) NOT NULL DEFAULT 1, show_phone TINYINT(1) NOT NULL DEFAULT 1,
  sort SMALLINT UNSIGNED NOT NULL DEFAULT 0, is_active TINYINT(1) NOT NULL DEFAULT 1,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  KEY idx_hb_sort (is_active, sort)
);
CREATE TABLE homepage_block_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  block_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL,
  title VARCHAR(120) NULL, link_label VARCHAR(80) NULL,
  UNIQUE KEY uq_hbt (block_id, locale),
  FOREIGN KEY (block_id) REFERENCES homepage_blocks(id) ON DELETE CASCADE
);
```

---

## 9. Latest News ticker (no click tracking)

```sql
CREATE TABLE ticker_tags (                   -- colour-coded tags
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  bg_color CHAR(7) NOT NULL, text_color CHAR(7) NOT NULL,
  is_active TINYINT(1) NOT NULL DEFAULT 1, sort SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL
);
CREATE TABLE ticker_tag_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  ticker_tag_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL, name VARCHAR(40) NOT NULL,
  UNIQUE KEY uq_ttt (ticker_tag_id, locale),
  FOREIGN KEY (ticker_tag_id) REFERENCES ticker_tags(id) ON DELETE CASCADE
);

CREATE TABLE ticker_items (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  tag_id BIGINT UNSIGNED NULL,
  source VARCHAR(10) NOT NULL DEFAULT 'manual',      -- manual, article
  link_type VARCHAR(10) NOT NULL DEFAULT 'none',     -- none, article, url
  article_id BIGINT UNSIGNED NULL,
  url VARCHAR(500) NULL, open_new_tab TINYINT(1) NOT NULL DEFAULT 0,
  status VARCHAR(10) NOT NULL DEFAULT 'active',      -- active, inactive
  is_pinned TINYINT(1) NOT NULL DEFAULT 0,
  starts_at TIMESTAMP NULL, ends_at TIMESTAMP NULL,
  sort INT UNSIGNED NOT NULL DEFAULT 0,
  created_by BIGINT UNSIGNED NULL, updated_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  KEY idx_ti_active (status, is_pinned, starts_at, ends_at, sort),
  KEY idx_ti_article (article_id),
  FOREIGN KEY (tag_id) REFERENCES ticker_tags(id) ON DELETE SET NULL,
  FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE
);
CREATE TABLE ticker_item_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  ticker_item_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL,
  text VARCHAR(140) NULL,                    -- NULL for article-linked items = use article headline
  UNIQUE KEY uq_tit (ticker_item_id, locale),
  FOREIGN KEY (ticker_item_id) REFERENCES ticker_items(id) ON DELETE CASCADE
);
```
Ticker **settings** (master switch, source mode `manual|automatic|mixed`, sort mode `manual|newest|oldest`, automatic rules, max items, speed, label text, per-page-type switches, device switches) live in `settings` group `ticker` (section 12). No click or impression table exists for the ticker.

**Items returned for locale L** (cached in Redis 30–60 s):
1. Manual items: `status='active'`, window valid, translation text present for L (or article-linked with published translation in L).
2. Automatic items (if mode is automatic/mixed): latest published articles by rules (sections, `add_to_ticker`, max age) with a published translation in L.
3. Pinned first (by `sort`), then others by the chosen sort mode; cut to max items.

---

## 10. Galleries, videos, polls

```sql
CREATE TABLE galleries (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  cover_media_id BIGINT UNSIGNED NULL, status VARCHAR(20) NOT NULL DEFAULT 'draft',
  published_at TIMESTAMP NULL, show_on_home TINYINT(1) NOT NULL DEFAULT 0,
  created_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL,
  KEY idx_gal_status (status, published_at)
);
CREATE TABLE gallery_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  gallery_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL,
  title VARCHAR(190) NOT NULL, slug VARCHAR(190) NOT NULL, description TEXT NULL,
  meta_title VARCHAR(190) NULL, meta_description VARCHAR(320) NULL,
  UNIQUE KEY uq_galt (gallery_id, locale), UNIQUE KEY uq_galt_slug (locale, slug),
  FOREIGN KEY (gallery_id) REFERENCES galleries(id) ON DELETE CASCADE
);
CREATE TABLE gallery_items (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  gallery_id BIGINT UNSIGNED NOT NULL, media_id BIGINT UNSIGNED NOT NULL,
  sort SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  KEY idx_gi (gallery_id, sort),
  FOREIGN KEY (gallery_id) REFERENCES galleries(id) ON DELETE CASCADE
);                                           -- caption/alt/credit come from media_translations (optional per-item override via gallery_item_translations)
CREATE TABLE gallery_item_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  gallery_item_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL,
  caption VARCHAR(500) NULL, alt VARCHAR(255) NULL,
  UNIQUE KEY uq_git (gallery_item_id, locale),
  FOREIGN KEY (gallery_item_id) REFERENCES gallery_items(id) ON DELETE CASCADE
);

CREATE TABLE videos (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  provider VARCHAR(15) NOT NULL,             -- youtube, vimeo, upload
  provider_video_id VARCHAR(60) NULL, media_id BIGINT UNSIGNED NULL,
  thumbnail_media_id BIGINT UNSIGNED NULL, duration_seconds INT UNSIGNED NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'draft', published_at TIMESTAMP NULL,
  show_on_home TINYINT(1) NOT NULL DEFAULT 0, created_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL,
  KEY idx_vid_status (status, published_at)
);
CREATE TABLE video_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  video_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL,
  title VARCHAR(190) NOT NULL, slug VARCHAR(190) NOT NULL, description TEXT NULL, transcript MEDIUMTEXT NULL,
  meta_title VARCHAR(190) NULL, meta_description VARCHAR(320) NULL,
  UNIQUE KEY uq_vt (video_id, locale), UNIQUE KEY uq_vt_slug (locale, slug),
  FOREIGN KEY (video_id) REFERENCES videos(id) ON DELETE CASCADE
);

CREATE TABLE polls (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  status VARCHAR(20) NOT NULL DEFAULT 'draft',    -- draft, open, closed
  opens_at TIMESTAMP NULL, closes_at TIMESTAMP NULL,
  show_on_home TINYINT(1) NOT NULL DEFAULT 0, show_results_before_vote TINYINT(1) NOT NULL DEFAULT 0,
  total_votes INT UNSIGNED NOT NULL DEFAULT 0, created_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL,
  KEY idx_poll_status (status, opens_at, closes_at)
);
CREATE TABLE poll_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  poll_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL, question VARCHAR(255) NOT NULL,
  UNIQUE KEY uq_plt (poll_id, locale),
  FOREIGN KEY (poll_id) REFERENCES polls(id) ON DELETE CASCADE
);
CREATE TABLE poll_options (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  poll_id BIGINT UNSIGNED NOT NULL, sort TINYINT UNSIGNED NOT NULL DEFAULT 0,
  vote_count INT UNSIGNED NOT NULL DEFAULT 0,
  KEY idx_po (poll_id, sort),
  FOREIGN KEY (poll_id) REFERENCES polls(id) ON DELETE CASCADE
);
CREATE TABLE poll_option_translations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  option_id BIGINT UNSIGNED NOT NULL, locale CHAR(2) NOT NULL, label VARCHAR(160) NOT NULL,
  UNIQUE KEY uq_pot (option_id, locale),
  FOREIGN KEY (option_id) REFERENCES poll_options(id) ON DELETE CASCADE
);
CREATE TABLE poll_votes (                    -- one vote per browser/IP window
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  poll_id BIGINT UNSIGNED NOT NULL, option_id BIGINT UNSIGNED NOT NULL,
  voter_hash CHAR(64) NOT NULL,              -- hash(cookie id + IP window + salt)
  created_at TIMESTAMP NULL,
  UNIQUE KEY uq_pv (poll_id, voter_hash),
  FOREIGN KEY (poll_id) REFERENCES polls(id) ON DELETE CASCADE
);
```

---

## 11. Comments and moderation

```sql
CREATE TABLE comments (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  article_id BIGINT UNSIGNED NOT NULL,
  reader_id BIGINT UNSIGNED NULL,            -- NULL after hard anonymisation
  parent_id BIGINT UNSIGNED NULL,            -- one level: parent must have parent_id NULL
  body VARCHAR(1000) NOT NULL,               -- plain text
  locale CHAR(2) NOT NULL,                   -- language of the page it was written on
  status VARCHAR(15) NOT NULL DEFAULT 'pending',   -- pending, published, rejected, spam, removed, hidden, deleted_by_reader
  hold_reason VARCHAR(30) NULL,              -- first_comment, banned_word, link, duplicate, caps, reported, reader_flag, premod
  report_count SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  reply_count SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  edited_at TIMESTAMP NULL, edited_by_moderator TINYINT(1) NOT NULL DEFAULT 0,
  moderated_by BIGINT UNSIGNED NULL, moderated_at TIMESTAMP NULL, moderation_reason VARCHAR(255) NULL,
  ip_hash CHAR(64) NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  KEY idx_c_article (article_id, status, parent_id, created_at),
  KEY idx_c_reader (reader_id, created_at),
  KEY idx_c_queue (status, created_at),
  KEY idx_c_parent (parent_id),
  FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE,
  FOREIGN KEY (reader_id) REFERENCES readers(id) ON DELETE SET NULL,
  FOREIGN KEY (parent_id) REFERENCES comments(id) ON DELETE CASCADE
);

CREATE TABLE comment_reports (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  comment_id BIGINT UNSIGNED NOT NULL, reader_id BIGINT UNSIGNED NOT NULL,
  reason VARCHAR(20) NOT NULL,               -- abuse, spam, hate, misinformation, other
  note VARCHAR(255) NULL, status VARCHAR(10) NOT NULL DEFAULT 'open',   -- open, dismissed, actioned
  created_at TIMESTAMP NULL,
  UNIQUE KEY uq_cr (comment_id, reader_id),
  KEY idx_cr_status (status, created_at),
  FOREIGN KEY (comment_id) REFERENCES comments(id) ON DELETE CASCADE,
  FOREIGN KEY (reader_id) REFERENCES readers(id) ON DELETE CASCADE
);

CREATE TABLE comment_moderation_logs (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  comment_id BIGINT UNSIGNED NOT NULL, user_id BIGINT UNSIGNED NULL,   -- NULL = system (auto-hold/auto-hide)
  action VARCHAR(20) NOT NULL,               -- approve, reject, spam, remove, edit, hide, restore, auto_hold
  reason VARCHAR(255) NULL, old_body VARCHAR(1000) NULL,
  created_at TIMESTAMP NULL,
  KEY idx_cml_comment (comment_id, created_at),
  FOREIGN KEY (comment_id) REFERENCES comments(id) ON DELETE CASCADE
);

CREATE TABLE banned_words (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  word VARCHAR(100) NOT NULL, locale CHAR(2) NULL,        -- NULL = both
  action VARCHAR(10) NOT NULL DEFAULT 'hold',             -- hold, reject
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  UNIQUE KEY uq_bw (word, locale)
);
```
Rules enforced in PHP: only verified, active readers; reader must not be suspended; article `allow_comments=1` and not auto-closed; rate limits via Redis; `comment_count` on `articles` counts `status='published'` (top-level + replies) and is recalculated by job. Comments are **not** filtered by locale when displayed (shared across both language versions).

---

## 12. Settings keys (table `settings`)

| Group | Keys (value type) |
|---|---|
| `general` | `site_name` (per locale), `tagline` (per locale), `default_locale`, `timezone`, `digits_bn` (`bangla`/`western`), `contact_email`, `show_bangla_calendar` (bool) |
| `brand` | `display_mode` (`logo_only`/`logo_text`/`text_only`), `logo_light_media_id`, `logo_dark_media_id`, `logo_light_media_id_en`, `logo_dark_media_id_en`, `logo_alt` (per locale), `logo_height_desktop` (40–56), `logo_height_phone` (32–40), `brand_align` (`left`/`center`), `favicon_media_id`, `share_image_media_id`, `footer_logo_media_id` |
| `header` | `utility_bar` (bool), `weather` (bool), `sticky_header` (bool), `social_links` (JSON) |
| `appearance` | `default_theme` (`system`/`light`/`dark`), `article_text_size_default` (100/115/130) — **no other theme keys exist** |
| `performance` | `cache_ttl_home/section/article/tag/author/page/ticker` (sec), `lightweight_mode` (bool), `image_quality_webp`, `image_quality_avif`, `max_ads_mobile`, `max_ads_desktop`, `stale_max_seconds` |
| `comments` | `enabled`, `moderation_mode` (`pre`/`post`), `hold_first_comment` (bool), `report_threshold` (3), `edit_window_minutes` (15), `auto_close_days` (14), `max_length` (1000), `rate_limit_seconds` (20), `rate_limit_per_hour` (10), `new_account_per_hour` (3), `sort_default`, `links_clickable` (bool), `on_account_delete` (`anonymise`/`remove`) |
| `readers` | `registration_enabled`, `email_verification_required` (true), `min_password_length` (8) |
| `translation` | `machine_enabled`, `machine_provider`, `machine_api_key` (encrypted), `review_required` (true) |
| `ticker` | `enabled`, `source_mode`, `sort_mode`, `max_items`, `speed`, `label` (per locale), `auto_sections_include` (JSON), `auto_sections_exclude` (JSON), `auto_only_flagged` (bool), `auto_max_age_hours` (24), `pages` (JSON: home, section, article, tag, author, search, static, account → bool), `devices` (JSON: desktop, phone) |
| `search` | `min_chars`, `log_queries`, `suggestions_enabled` |
| `seo` | `title_pattern` (per locale), `default_description` (per locale), `robots_txt`, `verify_google`, `verify_bing`, `indexnow_key`, `tag_noindex_below` (3), `org_name`, `org_social` (JSON) |
| `email` | `from_name`, `from_address`, provider credentials (encrypted), `moderator_digest_time` |
| `integrations` | `analytics_id`, `sentry_dsn`, `ad_provider_scripts` (JSON) |
| `ads` | `enabled`, `pages` (JSON per page type), `article_inline_after_paragraph` (1: 3, 2: 8) |
| `legal` | `cookie_notice` (per locale), `privacy_page_id`, `terms_page_id`, `comment_policy_page_id` |
| `maintenance` | `enabled`, `bypass_key` |

Secrets (`*_api_key`, credentials) are stored encrypted (Laravel `Crypt`) and never exposed in `is_public` config.

---

## 13. Ads, SEO, analytics, engagement, audit

### 13.1 Ads
```sql
CREATE TABLE advertisers (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(150) NOT NULL, contact_name VARCHAR(120) NULL, email VARCHAR(190) NULL,
  phone VARCHAR(40) NULL, notes TEXT NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL
);
CREATE TABLE campaigns (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  advertiser_id BIGINT UNSIGNED NOT NULL, name VARCHAR(150) NOT NULL,
  starts_at TIMESTAMP NULL, ends_at TIMESTAMP NULL, budget_note VARCHAR(255) NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  FOREIGN KEY (advertiser_id) REFERENCES advertisers(id) ON DELETE CASCADE
);
CREATE TABLE ad_placements (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  code VARCHAR(40) NOT NULL UNIQUE,          -- header_leaderboard, sidebar_sticky, article_inline_1...
  name VARCHAR(100) NOT NULL,
  desktop_width SMALLINT UNSIGNED NULL, desktop_height SMALLINT UNSIGNED NULL,
  phone_width SMALLINT UNSIGNED NULL, phone_height SMALLINT UNSIGNED NULL,
  is_active TINYINT(1) NOT NULL DEFAULT 1, sort SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL
);
CREATE TABLE ads (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  advertiser_id BIGINT UNSIGNED NULL, campaign_id BIGINT UNSIGNED NULL,
  title VARCHAR(150) NOT NULL,
  type VARCHAR(10) NOT NULL,                 -- image, code, text
  image_desktop_media_id BIGINT UNSIGNED NULL, image_phone_media_id BIGINT UNSIGNED NULL,
  code TEXT NULL,                            -- trusted HTML/JS (Admin/Ad Manager only)
  text_label VARCHAR(190) NULL,
  destination_url VARCHAR(500) NULL, utm VARCHAR(255) NULL, alt_text VARCHAR(255) NULL,
  status VARCHAR(10) NOT NULL DEFAULT 'draft',     -- draft, active, paused, expired
  starts_at TIMESTAMP NULL, ends_at TIMESTAMP NULL,
  weight TINYINT UNSIGNED NOT NULL DEFAULT 1,
  targeting JSON NULL,                       -- {"locales":["bn"],"section_ids":[],"label_ids":[],"devices":["desktop"],"page_types":["article"]}
  frequency_cap_per_day TINYINT UNSIGNED NULL,
  notes TEXT NULL, created_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, deleted_at TIMESTAMP NULL,
  KEY idx_ads_active (status, starts_at, ends_at),
  FOREIGN KEY (advertiser_id) REFERENCES advertisers(id) ON DELETE SET NULL,
  FOREIGN KEY (campaign_id) REFERENCES campaigns(id) ON DELETE SET NULL
);
CREATE TABLE ad_placement_ad (
  placement_id BIGINT UNSIGNED NOT NULL, ad_id BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (placement_id, ad_id), KEY idx_apa_ad (ad_id),
  FOREIGN KEY (placement_id) REFERENCES ad_placements(id) ON DELETE CASCADE,
  FOREIGN KEY (ad_id) REFERENCES ads(id) ON DELETE CASCADE
);
CREATE TABLE ad_stats_daily (                -- flushed from Redis once a minute
  ad_id BIGINT UNSIGNED NOT NULL, placement_id BIGINT UNSIGNED NOT NULL, stat_date DATE NOT NULL,
  impressions INT UNSIGNED NOT NULL DEFAULT 0, clicks INT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (ad_id, placement_id, stat_date), KEY idx_asd_date (stat_date)
);
```

### 13.2 SEO tools
```sql
CREATE TABLE redirects (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  from_path VARCHAR(500) NOT NULL, to_path VARCHAR(500) NOT NULL,
  code SMALLINT UNSIGNED NOT NULL DEFAULT 301,      -- 301, 302, 410
  is_auto TINYINT(1) NOT NULL DEFAULT 0,            -- created by slug change
  hits INT UNSIGNED NOT NULL DEFAULT 0, last_hit_at TIMESTAMP NULL,
  created_by BIGINT UNSIGNED NULL, created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL,
  from_hash CHAR(32) NOT NULL,                      -- md5(from_path) for fast unique lookup
  UNIQUE KEY uq_red (from_hash)
);
CREATE TABLE not_found_logs (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  path_hash CHAR(32) NOT NULL UNIQUE, path VARCHAR(500) NOT NULL,
  hits INT UNSIGNED NOT NULL DEFAULT 1, last_referrer VARCHAR(255) NULL,
  first_seen_at TIMESTAMP NULL, last_seen_at TIMESTAMP NULL,
  is_resolved TINYINT(1) NOT NULL DEFAULT 0, KEY idx_nf_hits (is_resolved, hits)
);
CREATE TABLE search_synonyms (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  locale CHAR(2) NULL, term VARCHAR(100) NOT NULL, synonyms VARCHAR(500) NOT NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, KEY idx_ss_term (term)
);
CREATE TABLE search_logs (                   -- anonymous
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  term VARCHAR(150) NOT NULL, locale CHAR(2) NOT NULL, results_count INT UNSIGNED NOT NULL DEFAULT 0,
  created_at TIMESTAMP NULL, KEY idx_sl (created_at), KEY idx_sl_term (term, locale)
);
```

### 13.3 Analytics counters
```sql
CREATE TABLE article_views_daily (           -- flushed from Redis; powers Most Read and reports
  article_id BIGINT UNSIGNED NOT NULL, view_date DATE NOT NULL, views INT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (article_id, view_date), KEY idx_avd_date (view_date, views)
);
```
Most read (24 h / 7 d) = `SUM(views) GROUP BY article_id` over the window, cached 5 minutes. Rows older than 400 days are rolled up/archived.

### 13.4 Engagement
```sql
CREATE TABLE subscribers (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(190) NOT NULL UNIQUE, locale CHAR(2) NOT NULL DEFAULT 'bn',
  status VARCHAR(15) NOT NULL DEFAULT 'pending',   -- pending, confirmed, unsubscribed
  confirm_token_hash CHAR(64) NULL, confirmed_at TIMESTAMP NULL, unsubscribed_at TIMESTAMP NULL,
  source VARCHAR(30) NULL, reader_id BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, KEY idx_sub_status (status)
);
CREATE TABLE contact_messages (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(120) NOT NULL, email VARCHAR(190) NOT NULL, subject VARCHAR(190) NOT NULL,
  message TEXT NOT NULL, locale CHAR(2) NOT NULL DEFAULT 'bn',
  status VARCHAR(10) NOT NULL DEFAULT 'unread',    -- unread, read, replied, archived, spam
  ip_hash CHAR(64) NULL, handled_by BIGINT UNSIGNED NULL,
  created_at TIMESTAMP NULL, updated_at TIMESTAMP NULL, KEY idx_cm_status (status, created_at)
);
```

### 13.5 Audit
```sql
CREATE TABLE audit_logs (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT UNSIGNED NULL, action VARCHAR(40) NOT NULL,   -- created, updated, deleted, published, login, permission_changed...
  entity_type VARCHAR(60) NOT NULL, entity_id BIGINT UNSIGNED NULL,
  changes JSON NULL, ip_hash CHAR(64) NULL, user_agent VARCHAR(255) NULL,
  created_at TIMESTAMP NULL,
  KEY idx_al_entity (entity_type, entity_id), KEY idx_al_user (user_id, created_at), KEY idx_al_time (created_at)
);
```
Retention: 12 months (setting), pruned by a scheduled job.

---

## 14. Migration order (Laravel)

1. `languages`, `settings`, `ui_translations`
2. `users`, `password_reset_tokens`, spatie permission tables, `staff_login_history`, `notifications`, `failed_jobs`
3. `media_folders`, `media`, `media_translations`, `media_tags` (then add `users.avatar_media_id` FK)
4. `sections`, `section_translations`, `section_user`
5. `tags`, `tag_translations`, `authors`, `author_translations`, `article_labels`, `article_label_translations`
6. `articles`, `article_translations`, pivots (`article_section`, `article_tag`, `article_author`, `article_related`), `article_revisions`, `article_review_comments`, `article_corrections(+translations)`, `media_usages`
7. `pages`, `page_translations`, `menus`, `menu_items`, `menu_item_translations`, `homepage_blocks(+translations)`
8. `ticker_tags(+translations)`, `ticker_items(+translations)`
9. `readers`, reader token/email-change/suspension tables, `saved_articles`
10. `comments`, `comment_reports`, `comment_moderation_logs`, `banned_words`
11. `galleries…`, `videos…`, `polls…`
12. `advertisers`, `campaigns`, `ad_placements`, `ads`, `ad_placement_ad`, `ad_stats_daily`
13. `redirects`, `not_found_logs`, `search_synonyms`, `search_logs`, `article_views_daily`
14. `subscribers`, `contact_messages`, `audit_logs`
15. Seeders (section 16)

---

## 15. Key queries and index use

```sql
-- Home / section list (language L, page)
SELECT a.id, t.title, t.slug, t.summary, a.published_at, a.main_media_id
FROM articles a JOIN article_translations t ON t.article_id=a.id AND t.locale=:L AND t.translation_status='published'
WHERE a.status='published' AND a.published_at<=NOW() AND (a.expires_at IS NULL OR a.expires_at>NOW())
  AND a.section_id=:sid AND a.deleted_at IS NULL
ORDER BY a.published_at DESC LIMIT 20 OFFSET :o;           -- idx_art_section_pub

-- Article by id + locale
SELECT ... FROM articles a JOIN article_translations t ON t.article_id=a.id AND t.locale=:L WHERE a.id=:id;   -- PK + uq_at_article

-- Ticker (cached 30–60 s) – manual items for L
SELECT i.id, COALESCE(it.text, at.title) text, i.link_type, i.url, i.is_pinned, i.tag_id
FROM ticker_items i
LEFT JOIN ticker_item_translations it ON it.ticker_item_id=i.id AND it.locale=:L
LEFT JOIN article_translations at ON at.article_id=i.article_id AND at.locale=:L AND at.translation_status='published'
WHERE i.status='active' AND (i.starts_at IS NULL OR i.starts_at<=NOW()) AND (i.ends_at IS NULL OR i.ends_at>NOW())
  AND COALESCE(it.text, at.title) IS NOT NULL
ORDER BY i.is_pinned DESC, i.sort ASC;                      -- idx_ti_active

-- Comments for an article (first page of top-level)
SELECT * FROM comments WHERE article_id=:id AND status='published' AND parent_id IS NULL
ORDER BY created_at DESC LIMIT 10;                          -- idx_c_article

-- Translation queue
SELECT a.id, t.title FROM articles a JOIN article_translations t ON t.article_id=a.id AND t.locale='en'
WHERE t.translation_status IN ('draft','needs_review') AND t.assigned_translator_id=:u;   -- idx_at_assignee

-- Articles missing English
SELECT a.id FROM articles a LEFT JOIN article_translations t ON t.article_id=a.id AND t.locale='en'
WHERE a.status='published' AND t.id IS NULL;

-- Search (Bangla, ngram)
SELECT article_id, MATCH(title,subtitle,summary,body) AGAINST (:q IN NATURAL LANGUAGE MODE) score
FROM article_translations WHERE locale=:L AND MATCH(...) AGAINST (:q IN NATURAL LANGUAGE MODE) ORDER BY score DESC LIMIT 20;
```
Set `ngram_token_size=2` (MySQL config) for Bangla. Verify with `EXPLAIN` that each public query uses an index; add the query-count guard test from the PRD.

---

## 16. Seed data

| Table | Seed |
|---|---|
| `languages` | `bn` (default, no prefix), `en` (prefix `en`) |
| roles | Super Admin, Admin, Editor-in-Chief, Section Editor, Reporter/Writer, Contributor, Translator, Photographer/Media Manager, Ad Manager, SEO Manager, Moderator, Analyst (with permissions from the PRD matrix) |
| `users` | One Super Admin (email from env) |
| `sections` | বাংলাদেশ/Bangladesh, আন্তর্জাতিক/International, রাজনীতি/Politics, অর্থনীতি/Economy, খেলা/Sports, বিনোদন/Entertainment, প্রযুক্তি/Technology, মতামত/Opinion, শিক্ষা/Education, স্বাস্থ্য/Health, জীবনযাপন/Lifestyle |
| `article_labels` | Opinion, Exclusive, Analysis, Sponsored, Fact-check |
| `ticker_tags` | জরুরি/Urgent (red), আপডেট/Update (blue), খেলা/Sports (green) |
| `menus` | main, footer_1–footer_4, utility, mobile_extra |
| `pages` | About, Contact, Editorial policy, Privacy, Terms, Comment policy, Advertise, Corrections (placeholder text, `system_key` set) |
| `ad_placements` | All placements from the PRD section 13.2 with sizes |
| `settings` | Defaults from section 12 (ticker on, `source_mode=manual`, `sort_mode=manual`, comments `post` moderation, `appearance.default_theme=system`, `brand.display_mode=text_only`) |
| `ui_translations` | All interface strings in `bn` and `en` |
| `homepage_blocks` | Default home structure from PRD 8.4 |

---

## 17. Maintenance jobs (scheduler)

| Job | Frequency | What |
|---|---|---|
| Flush view and ad counters | every minute | Redis → `article_views_daily`, `articles.view_count`, `ad_stats_daily`, `poll_options.vote_count` |
| Publish scheduled articles | every minute | Set `status='published'` where `scheduled` and time reached; purge caches; update sitemaps |
| Expire articles/ads/ticker | every minute | Unpublish at `expires_at`; ads → `expired` |
| Auto-close comments | hourly | Per setting; sets a flag or is computed at read time |
| Recalculate counters | nightly | `comment_count`, `tags.article_count`, `authors.article_count` |
| Prune revisions | nightly | Keep last 50 per article+locale |
| Prune logs | nightly | `search_logs`, `not_found_logs`, `staff_login_history`, `audit_logs` per retention |
| Source-changed check | on save | Mark English `needs_review` when Bangla body/title changes after approval |
| Purge soft-deleted | daily | Hard delete Trash items older than 30 days |
| Anonymise deleted readers | hourly | Process queued account deletions |
| Backups | daily | Database dump + media sync |

---

## 18. Notes for the build

- Use Eloquent models with a `Translatable` trait (or `spatie/laravel-translatable` for simple cases); keep `article_translations` as a real table for FULLTEXT and slug uniqueness.
- Add database-level uniqueness for `(locale, slug)` per entity so URLs cannot collide.
- Do not use the `sessions`/`cache` tables; use Redis.
- All writes that affect public pages dispatch a cache-purge event (home, section, tag, author, article, ticker, menu, sitemap).
- Keep `body` HTML sanitised on save (allow-list) and cache the rendered HTML fragment if rendering is heavy.
- Reader and staff models are separate Authenticatable models with separate guards and password brokers.
- Run `php artisan schema:dump` after migrations stabilise to speed up CI.

*End of document.*

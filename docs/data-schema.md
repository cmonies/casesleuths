# CaseSleuths — Data Schema

_v0.3 · 2026-02-27 — Incorporates all P0–P5 review fixes + legal framework + realtime upvotes_

---

## Architecture Notes

### SSG / ISR vs. Client-Side Boundary

| Entity | Rendering | Revalidation Trigger |
|--------|-----------|----------------------|
| `cases` | SSG/ISR | DB webhook on `cases` row change |
| `timeline_events` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `people` (via `case_people`) | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `agencies` (via `case_agencies`) | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `evidence` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `interviews` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `case_faqs` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `related_cases` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `tags` / `case_tags` | SSG/ISR | DB webhook → tag listing pages |
| `podcast_episodes` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `tv_shows` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `news_articles` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `case_subreddits` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `case_reddit_posts` | **Client-side** | Weekly cron, displayed via Supabase client |
| `community_notes` | **Client-side** | Supabase Realtime subscription |
| `community_corrections` | **Client-side** | Supabase Realtime subscription |
| `content_reports` | **Admin only** | Admin dashboard |
| `upvotes` | **Client-side** | Supabase Realtime subscription (see below) |
| `user_media_progress` | **Client-side** | Local optimistic update |

**ISR revalidation intervals:**
- Case pages: `revalidate: 3600` (1 hour) — content changes infrequently
- Tag / listing pages: `revalidate: 1800` (30 min) — new cases may appear
- Person / agency pages: `revalidate: 3600` (1 hour)

**Revalidation implementation:** Supabase DB webhooks → Next.js `/api/revalidate?secret=TOKEN&path=/cases/[slug]`. Webhooks needed on: `cases`, `timeline_events`, `case_people`, `evidence`, `interviews`, `case_faqs`, `podcast_episodes`, `tv_shows`, `news_articles`, `case_subreddits`, `case_tags`, `related_cases`.

### Upvotes — Realtime Strategy

Reddit's model (and ours): **optimistic updates + Supabase Realtime push**.

1. User clicks upvote → UI updates count immediately (no server round-trip) via local state
2. Write goes to `upvotes` table in Supabase
3. Supabase Realtime broadcasts the insert/delete to all subscribers on that entity
4. All connected clients update their counts from the broadcast — no polling

**Count queries:** Do NOT use a `SELECT COUNT(*)` view for hot paths. Use a trigger-maintained counter cache:
- Each upvotable entity (e.g., `community_notes`) has an `upvote_count int NOT NULL DEFAULT 0` column
- A Postgres trigger on `INSERT/DELETE` on `upvotes` increments/decrements the parent row's counter
- Realtime broadcasts both the `upvotes` insert and the parent row's updated `upvote_count`

This is consistent (trigger = atomic), fast (integer read, no GROUP BY), and realtime-compatible.

```sql
-- Example trigger
CREATE OR REPLACE FUNCTION update_upvote_count() RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE community_notes SET upvote_count = upvote_count + 1 WHERE id = NEW.entity_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE community_notes SET upvote_count = upvote_count - 1 WHERE id = OLD.entity_id;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;
-- (Repeat per upvotable entity type or use a CASE statement)
```

### Legal Framework

See `docs/legal-safety-framework.md` for full policy. Key constraints on this schema:

- **No `perpetrator` label anywhere.** Person legal status in a case uses the controlled vocabulary below.
- **Convicted ≠ guilty for display purposes** — always use `convicted` not `guilty`, `alleged` not `accused` for uncharged individuals.
- **Content warnings are required fields** — stored in `cases.content_warnings`, surfaced before case content is shown.
- **Legal sensitivity flag** — cases with living named suspects who are uncharged require `legal_sensitivity: high` and human review before publish.
- **A legal review skill/filter** must be run against every case page before it goes live. (See: planned `legal-review` skill that evaluates page content against the framework.)

**Person legal status vocabulary (use these exact values everywhere):**
- `victim`
- `convicted` — conviction stands (note appeals status if applicable)
- `acquitted` — charged, not convicted
- `exonerated` — conviction overturned
- `alleged` — charged, trial pending
- `person_of_interest` — named by investigators, not charged
- `no_charges_filed` — named in media/podcasts, never formally accused
- `witness`
- `detective` — investigator on the case
- `attorney` — defense, prosecution, civil
- `judge`
- `other`

### Rendering
- `cases.body` is plain **Markdown** (not MDX). Rendered with `react-markdown`. MDX is too slow at SSG build time for hundreds of cases.

### Structured Data / SEO
- Every case page outputs: `Article`, `Person`, `Event`, `BreadcrumbList`, `FAQPage` JSON-LD
- `og_image_url` is distinct from `cover_image_url` — OG images = 1200×630

---

## Core Entity Map

```
Case
 ├── Timeline (events)
 ├── Knowledge Graph (derived from existing joins at MVP — no dedicated tables)
 ├── Tags (normalized, browsable tag pages)
 ├── Content Warnings
 ├── People (legal-status labeled)
 ├── Agencies
 ├── Evidence
 ├── Interviews
 ├── Media Coverage
 │    ├── Podcast Episodes (RSS auto-ingest)
 │    ├── TV Shows / Documentaries (TMDB)
 │    └── News Articles
 ├── Reddit (subreddits + top posts)
 ├── Related Cases
 ├── FAQs
 └── Community Layer (client-side, Realtime)
      ├── Upvotes (trigger-maintained count + Realtime push)
      ├── Notes (with soft delete + optional threading)
      ├── Corrections
      └── Reports
```

---

## Tables

> **Convention:** All tables have `created_at timestamptz NOT NULL DEFAULT now()` and `updated_at timestamptz NOT NULL DEFAULT now()` unless explicitly noted. Pure join tables (e.g. `case_tags`) omit both timestamps.

---

### `profiles`

Mirror of `auth.users`. Created via Supabase trigger on signup. Minimum age 18 (enforced at signup).

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK, FK → auth.users |
| `display_name` | text | |
| `avatar_url` | text | |
| `app_role` | enum | `user` · `moderator` · `admin` |
| `is_banned` | bool | default false |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `cases`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | URL-safe, unique e.g. `zodiac-killer` |
| `title` | text | |
| `alternate_names` | text[] | AKAs, nicknames |
| `status` | enum | `unsolved` · `solved_convicted` · `solved_acquitted` · `solved_exonerated` · `ongoing_trial` · `cold_case` · `civil_resolution` |
| `legal_sensitivity` | enum | `standard` · `elevated` · `high` — `high` blocks auto-publish, requires human review |
| `content_warnings` | text[] | From controlled list: `child_victim` · `sexual_violence` · `infant_victim` · `family_violence` · `mass_casualty` · `graphic_imagery_risk` |
| `location_city` | text | |
| `location_state` | text | |
| `location_country` | text | default `US` |
| `location_coords` | point | lat/lng |
| `date_start` | date | |
| `date_end` | date | nullable |
| `summary` | text | 1–2 paragraph overview |
| `body` | text | Full writeup — plain Markdown |
| `disclaimer` | text | Legal/editorial note shown prominently on page e.g. "No one has been charged in this case." |
| `seo_title` | text | |
| `seo_description` | text | |
| `cover_image_url` | text | |
| `og_image_url` | text | 1200×630 for Open Graph |
| `published_at` | timestamptz | null = draft. Never auto-set for `legal_sensitivity: high` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `slug_redirects`

Preserves SEO value and backlinks when a case slug changes.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `old_slug` | text | UNIQUE |
| `new_slug` | text | FK → cases.slug |
| `created_at` | timestamptz | |

---

### `tags`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | UNIQUE e.g. `serial-killer` |
| `label` | text | |
| `description` | text | For tag pages |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_tags` (join — no timestamps)

| Field | Type |
|-------|------|
| `case_id` | uuid |
| `tag_id` | uuid |

PK: `(case_id, tag_id)`

---

### `related_cases` (no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id_a` | uuid | |
| `case_id_b` | uuid | |
| `relationship_type` | text | e.g. `same_perpetrator_alleged` · `same_jurisdiction` · `connected_victim` |
| `notes` | text | |

PK: `(case_id_a, case_id_b)`
CONSTRAINT: `CHECK (case_id_a < case_id_b)` — prevents `(A,B)` and `(B,A)` duplicates. Queries must use `WHERE case_id_a = X OR case_id_b = X`.

---

### `case_faqs`

Q&A pairs → `FAQPage` JSON-LD → People Also Ask traffic.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `question` | text | |
| `answer` | text | |
| `sort_order` | int | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `timeline_events`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK → cases |
| `date` | date | |
| `date_precision` | enum | `exact` · `approximate` · `year_only` |
| `title` | text | |
| `description` | text | |
| `source_url` | text | Citation |
| `media_url` | text | Optional image/doc |
| `event_tags` | text[] | Intentionally denormalized — local to timeline filtering only. Not browsable. e.g. `arrest` · `discovery` · `trial` |
| `sort_order` | int | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `people`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | UNIQUE, URL-safe — for person SEO pages |
| `name` | text | |
| `aliases` | text[] | |
| `primary_known_role` | enum | Display hint only. Source of truth is `case_people.legal_status`. Values: see legal vocabulary above. |
| `dob` | date | nullable |
| `dod` | date | nullable |
| `bio` | text | |
| `photo_url` | text | |
| `links` | jsonb | `{ "wikipedia": "...", "news": [...] }` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_people`

A person can have multiple rows per case (e.g., witness at time of arrest, later charged).

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK — surrogate key to allow multi-role |
| `case_id` | uuid | |
| `person_id` | uuid | |
| `legal_status` | enum | **Source of truth.** Values: `victim` · `convicted` · `acquitted` · `exonerated` · `alleged` · `person_of_interest` · `no_charges_filed` · `witness` · `detective` · `attorney` · `judge` · `other` |
| `notes` | text | Case-specific context. e.g. "Charged 2023, trial pending" |
| `is_primary` | bool | Flag the most prominent role if multiple rows exist |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

UNIQUE: `(case_id, person_id, legal_status)` — prevents exact duplicate role rows.

---

### `agencies`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | UNIQUE, URL-safe |
| `name` | text | |
| `type` | enum | `local_pd` · `sheriff` · `fbi` · `da` · `state_police` · `other` _(US-centric v1 — known tech debt)_ |
| `jurisdiction` | text | |
| `contact_url` | text | Tip line, if public |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_agencies` (join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `agency_id` | uuid | |
| `role` | text | e.g. `lead investigator` · `supporting` |

PK: `(case_id, agency_id)`

---

### `evidence`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `type` | enum | `photo` · `audio` · `document` · `video` · `physical` · `other` |
| `title` | text | |
| `description` | text | |
| `file_url` | text | Supabase Storage |
| `source` | text | Attribution |
| `is_public` | bool | Some evidence is sensitive |
| `sort_order` | int | |
| `evidence_tags` | text[] | Intentionally denormalized — local filtering only. Not browsable. |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `interviews`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `type` | enum | `police` · `media` · `court` · `documentary` |
| `interviewer` | text | Name or org |
| `date` | date | |
| `summary` | text | |
| `duration_seconds` | int | For in-app player |
| `transcript_url` | text | |
| `media_url` | text | Audio/video |
| `source_url` | text | Citation |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `interview_people` (join — replaces single person_id FK)

Supports group interviews.

| Field | Type | Notes |
|-------|------|-------|
| `interview_id` | uuid | |
| `person_id` | uuid | |
| `role` | text | e.g. `interviewee` · `interviewer` |

PK: `(interview_id, person_id)`

---

### `podcasts`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `title` | text | |
| `host` | text | |
| `rss_url` | text | |
| `cover_image_url` | text | |
| `platforms` | jsonb | `{ "spotify": "...", "apple": "...", "youtube": "..." }` |
| `description` | text | |
| `auto_ingest` | bool | |
| `last_synced_at` | timestamptz | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `podcast_episodes`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `podcast_id` | uuid | FK → podcasts |
| `title` | text | |
| `description` | text | |
| `published_at` | timestamptz | Full timestamp from RSS |
| `duration_seconds` | int | |
| `audio_url` | text | Direct URL for in-app player |
| `external_only` | bool | true = link out only (Spotify-locked) |
| `episode_url` | text | UNIQUE — canonical link, used for deduplication |
| `transcript` | text | nullable — for Typesense indexing |
| `source` | enum | `rss` · `manual` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_podcast_episodes` (join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `episode_id` | uuid | |
| `relevance` | enum | `primary` · `mentions` |

PK: `(case_id, episode_id)`

---

### `tv_shows`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `title` | text | |
| `type` | enum | `documentary` · `docu_series` · `drama` · `news_special` |
| `network` | text | |
| `year_start` | int | |
| `year_end` | int | nullable — ongoing series |
| `seasons` | int | nullable |
| `description` | text | |
| `cover_image_url` | text | |
| `tmdb_id` | int | TMDB API |
| `imdb_id` | text | |
| `streaming_links` | jsonb | `{ "netflix": "...", "hulu": "..." }` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_tv_shows` (join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `show_id` | uuid | |
| `episode_refs` | jsonb | Array of `{ "season": 2, "episode": 5 }` objects |
| `notes` | text | |

PK: `(case_id, show_id)`

---

### `news_articles`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `title` | text | |
| `publication` | text | |
| `author` | text | |
| `published_at` | timestamptz | |
| `url` | text | UNIQUE |
| `summary` | text | |
| `source` | enum | `manual` · `rss` · `google_news` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_news_articles` (join — no timestamps)

| Field | Type |
|-------|------|
| `case_id` | uuid |
| `article_id` | uuid |

PK: `(case_id, article_id)`

---

### `case_subreddits`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `subreddit` | text | e.g. `r/JonBenetRamsey` — UNIQUE per case |
| `description` | text | |
| `member_count` | int | Cached, refreshed weekly |
| `is_primary` | bool | |
| `last_synced_at` | timestamptz | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_reddit_posts`

Rendered client-side (not SSG). Retention: keep top 50 posts per case by score; older posts replaced on sync.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `subreddit_id` | uuid | FK → case_subreddits — null if post comes from a non-curated subreddit |
| `post_id` | text | UNIQUE — Reddit's base36 ID, deduplication key |
| `title` | text | |
| `subreddit` | text | Denormalized for display |
| `url` | text | |
| `score` | int | |
| `comment_count` | int | |
| `fetched_at` | timestamptz | Updated on each sync |
| `created_at` | timestamptz | |

**Reddit API note:** Register an OAuth app from day one (free, 100 req/min). Unauthenticated `.json` endpoints are rate-limited to ~10 req/min and have been increasingly unreliable since Reddit's 2023 API changes. Do not rely on them for production syncs. If Reddit API becomes untenable, fallback: Pushshift archive (if restored) or manual curation.

---

## Community Layer

> All community data is **client-side fetched**. Never SSG. Use Supabase Realtime subscriptions.

### `upvotes`

| Field | Type | Notes |
|-------|------|-------|
| `user_id` | uuid | FK → profiles |
| `entity_type` | text | Table name e.g. `community_notes` |
| `entity_id` | uuid | No Postgres FK enforcement (polymorphic) — cleanup handled by trigger |
| `created_at` | timestamptz | |

PK: `(user_id, entity_type, entity_id)`

**Count strategy:** Trigger-maintained `upvote_count` int on each upvotable entity row. See Architecture Notes. Do NOT use GROUP BY views on hot paths.

### `community_notes`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `parent_id` | uuid | FK → community_notes (self-ref) — null for top-level. Enables threading later. |
| `user_id` | uuid | FK → profiles |
| `body` | text | |
| `upvote_count` | int | Trigger-maintained. NOT NULL DEFAULT 0. |
| `is_pinned` | bool | Moderator pin |
| `deleted_at` | timestamptz | Soft delete — null = active. Content replaced with "[removed]" on UI. |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `community_corrections`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `user_id` | uuid | FK → profiles |
| `field_path` | text | e.g. `timeline_events.{id}.date` |
| `current_value` | text | |
| `suggested_value` | text | |
| `source_url` | text | Required |
| `status` | enum | `pending` · `accepted` · `rejected` |
| `reviewed_by` | uuid | FK → profiles (admin) |
| `deleted_at` | timestamptz | Soft delete |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `content_reports`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `reporter_id` | uuid | FK → profiles |
| `entity_type` | text | |
| `entity_id` | uuid | |
| `reason` | enum | `inaccurate` · `inappropriate` · `spam` · `copyright` · `other` |
| `notes` | text | |
| `status` | enum | `pending` · `reviewed` · `resolved` |
| `deleted_at` | timestamptz | Soft delete |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `user_media_progress`

| Field | Type | Notes |
|-------|------|-------|
| `user_id` | uuid | FK → profiles |
| `media_type` | enum | `podcast_episode` · `tv_show` · `documentary` |
| `media_id` | uuid | |
| `status` | enum | `watched` · `listening` · `want_to_watch` |
| `updated_at` | timestamptz | |

PK: `(user_id, media_type, media_id)`

---

## Required Indexes

> Postgres does NOT auto-create indexes on FK columns (unlike MySQL). These must be explicitly created.

| Table | Column(s) | Type | Why |
|-------|-----------|------|-----|
| `cases` | `slug` | UNIQUE | URL lookups |
| `cases` | `status` | btree | Filter pages |
| `cases` | `published_at` | partial (`WHERE published_at IS NOT NULL`) | Published listing pages |
| `cases` | `legal_sensitivity` | btree | Admin review queue |
| `tags` | `slug` | UNIQUE | Tag page lookups |
| `people` | `slug` | UNIQUE | Person page lookups |
| `agencies` | `slug` | UNIQUE | Agency page lookups |
| `timeline_events` | `case_id` | btree | FK join |
| `evidence` | `case_id` | btree | FK join |
| `interviews` | `case_id` | btree | FK join |
| `case_faqs` | `case_id` | btree | FK join |
| `community_notes` | `case_id` | btree | Load notes per case |
| `community_notes` | `user_id` | btree | "My notes" page |
| `community_notes` | `deleted_at` | partial (`WHERE deleted_at IS NULL`) | Filter soft-deleted |
| `community_corrections` | `status` | btree | Admin review queue |
| `content_reports` | `status` | btree | Admin review queue |
| `upvotes` | `(entity_type, entity_id)` | btree | Count / exist checks |
| `case_reddit_posts` | `case_id` | btree | FK join |
| `case_reddit_posts` | `post_id` | UNIQUE | Deduplication |
| `podcast_episodes` | `podcast_id` | btree | FK join |
| `podcast_episodes` | `episode_url` | UNIQUE | RSS deduplication |
| `news_articles` | `url` | UNIQUE | News deduplication |
| `news_articles` | `published_at` | btree | Recent articles listing |
| `slug_redirects` | `old_slug` | UNIQUE | Redirect lookups |

---

## Automation Paths

### Podcast Ingestion (RSS)
1. Fetch RSS for podcasts where `auto_ingest = true`
2. Upsert `podcast_episodes` — deduplicate on `episode_url` (UNIQUE constraint)
3. NLP/keyword match against `cases` → suggest `case_podcast_episodes` links
4. Human review queue before publishing
5. `external_only = true` for Spotify-locked shows

### TV/Documentary Ingestion (TMDB + JustWatch)
- TMDB API → show metadata, posters, seasons
- JustWatch API → streaming availability per region
- Manual case linking (doc titles often don't name the case directly)

### Reddit Sync (weekly cron — requires OAuth)
- Register Reddit OAuth app first (100 req/min free)
- `GET /search?q=CASE_NAME&type=sr` → subreddit discovery
- `GET /search?q=CASE_NAME&sort=top&t=all` → top posts per case
- Retention: keep top 50 posts per case by score; upsert on `post_id` UNIQUE

### News Ingestion
- Google News RSS per case: `https://news.google.com/rss/search?q=CASE_NAME`
- NewsAPI.org fallback (free tier 100 req/day)
- Upsert on `url` UNIQUE

### Legal Review Skill (planned)
- Every case page content runs through a `legal-review` skill before `published_at` is set
- Skill checks: legal status vocabulary compliance, disclaimer presence for `legal_sensitivity: high` cases, content warning completeness, no `perpetrator` language
- Cases fail review → flagged for human editorial review, not auto-published

### SEO Content Generation (Top 20 Cases)
- Target per case: ~2,000 words body, 5+ timeline events, person stubs with correct legal status, 3+ FAQs, 3+ media links
- Top 20 cases = automation + legal review test bed before scaling

---

## Top 20 Priority Cases

_Draft — validate against search volume. Legal sensitivity flagged per Opus review._

| # | Case | Legal Notes |
|---|------|-------------|
| 1 | JonBenét Ramsey | No conviction — all persons are `no_charges_filed` or `person_of_interest` |
| 2 | Zodiac Killer | Unsolved — no named charged individuals |
| 3 | Gabby Petito | Brian Laundrie deceased — `alleged` |
| 4 | OJ Simpson | `acquitted` (criminal) / `civil_resolution`. Simpson deceased June 2024. Goldman/Brown estates protective. |
| 5 | Laci Peterson | Scott Peterson `convicted` — appeals ongoing, note status |
| 6 | Ted Bundy | `convicted`, deceased |
| 7 | BTK Killer (Dennis Rader) | `convicted` — guilty plea |
| 8 | Golden State Killer (Joseph DeAngelo) | `convicted` — guilty plea |
| 9 | Emmett Till | Historically resolved. Requires highest editorial care — civil rights touchstone. |
| 10 | Caylee Anthony / Casey Anthony | Casey Anthony `acquitted` — never call her `perpetrator` or imply guilt |
| 11 | Tupac Shakur | Duane "Keffe D" Davis charged 2023 — `alleged`, active proceedings. Flag `ongoing_trial`, legal review required before publish. |
| 12 | Biggie Smalls | Unsolved — all persons are `person_of_interest` |
| 13 | Trayvon Martin | George Zimmerman `acquitted` — actively litigious, has sued media orgs. HIGH legal risk. Must use `acquitted` status with prominent disclaimer. |
| 14 | Chandra Levy | Conviction vacated, no one currently convicted. Gary Condit sued multiple outlets. All persons `no_charges_filed`. |
| 15 | Elizabeth Short (Black Dahlia) | Unsolved, all deceased |
| 16 | Natalee Holloway | Van der Sloot `convicted` of different crime (extortion), never convicted of her murder — `person_of_interest` for this case |
| 17 | Etan Patz | Pedro Hernandez `convicted` |
| 18 | DB Cooper | Unsolved — no named charged individuals |
| 19 | Elisa Lam | Ruled accidental death — no suspects charged |
| 20 | Chris Watts | `convicted` — guilty plea |

---

## Open Questions

- **Audio player**: Howler.js or native HTML5 `<audio>`. Spotify embed API for linked episodes.
- **Knowledge graph viz**: Deferred to post-MVP. D3.js or Reagraph. No dedicated schema tables needed — derive from `case_people`, `timeline_events`, `evidence` joins.
- **Supabase RLS**: Define row-level security — anon read for published cases, auth required for community writes. `legal_sensitivity: high` cases may warrant additional RLS.
- **`agencies.type` enum**: US-centric in v1 — flagged as known tech debt.
- **TMDB API key**: Required for TV/doc enrichment. Register at themoviedb.org.
- **Legal review skill**: Plan and build before any case content goes live. See `docs/legal-safety-framework.md`.

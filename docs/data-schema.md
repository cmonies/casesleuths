# CaseSleuths — Data Schema

_v0.2 · 2026-02-27 — Incorporates Opus P0–P4 review + Reddit + community fixes_

This document defines the full data model for CaseSleuths. The schema is **case-centric**: every other entity (people, media, evidence, community contributions) hangs off a `Case`.

---

## Architecture Notes

### SSG vs. Client-Side Boundary
- **Case content** (cases, timeline, people, evidence, media) → SSG/ISR via Next.js. Built at deploy time, revalidated on content change.
- **Community layer** (notes, upvotes, corrections, reports, `user_media_progress`) → **client-side fetched** via Supabase JS client. Never baked into SSG — or every upvote triggers a rebuild.
- **On-demand revalidation**: Supabase database webhook → Next.js `/api/revalidate` when a case row changes.

### Rendering
- `body` field uses plain **Markdown** (not MDX). Rendered with `react-markdown`. MDX compilation is too slow at build time for hundreds of cases.

### Structured Data / SEO
- Every case page outputs `Article`, `Person`, `Event`, `BreadcrumbList`, and `FAQPage` JSON-LD.
- `og_image_url` is distinct from `cover_image_url` — OG images use 1200×630, cover images are flexible.

---

## Core Entity Map

```
Case
 ├── Timeline (events)
 ├── Knowledge Graph (derived from existing joins — no dedicated tables at MVP)
 ├── Tags (normalized, browsable tag pages)
 ├── People
 │    ├── Victims
 │    ├── Suspects / Perpetrators
 │    ├── Detectives / Investigators
 │    └── Witnesses
 ├── Agencies (police dept, FBI, DA, etc.)
 ├── Evidence
 │    ├── Case Photos
 │    ├── Police Call Recordings
 │    └── Documents / Reports
 ├── Interviews
 ├── Media Coverage
 │    ├── Podcast Episodes
 │    ├── TV Shows / Documentaries
 │    └── News Articles
 ├── Reddit (subreddits + top posts)
 ├── Related Cases
 ├── FAQs
 └── Community Layer
      ├── Upvotes (via view — no denormalized int)
      ├── Notes
      ├── Corrections
      └── Reports
```

---

## Tables / Collections

> **Convention:** Every table has `created_at timestamptz NOT NULL DEFAULT now()` and `updated_at timestamptz NOT NULL DEFAULT now()` unless noted.

---

### `profiles`

Mirror of `auth.users` for public profile data. Created via Supabase trigger on signup.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK, FK → auth.users |
| `display_name` | text | |
| `avatar_url` | text | |
| `role` | enum | `user` · `moderator` · `admin` |
| `is_banned` | bool | default false |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `cases`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | URL-safe, unique e.g. `zodiac-killer` |
| `title` | text | Display name |
| `alternate_names` | text[] | AKAs, nicknames |
| `status` | enum | `unsolved` · `solved` · `cold` · `closed` · `ongoing` |
| `location_city` | text | |
| `location_state` | text | |
| `location_country` | text | default `US` |
| `location_coords` | point | lat/lng |
| `date_start` | date | When crime(s) began |
| `date_end` | date | nullable |
| `summary` | text | 1–2 paragraph overview |
| `body` | text | Full case writeup — plain Markdown |
| `seo_title` | text | |
| `seo_description` | text | |
| `cover_image_url` | text | |
| `og_image_url` | text | 1200×630 for Open Graph |
| `published_at` | timestamptz | null = draft |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

_Note: `search_rank` removed — content completeness + traffic-derived ranking TBD. `crime_type` removed — replaced by `tags` / `case_tags`._

---

### `tags`

Normalized tags for cases (replaces `crime_type text[]`).

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | e.g. `serial-killer` |
| `label` | text | Display name |
| `description` | text | For tag pages |
| `created_at` | timestamptz | |

### `case_tags`

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `tag_id` | uuid | |

PK: `(case_id, tag_id)`

---

### `related_cases`

Self-referential join for "Related Cases" and internal SEO linking.

| Field | Type | Notes |
|-------|------|-------|
| `case_id_a` | uuid | |
| `case_id_b` | uuid | |
| `relationship_type` | text | e.g. `same_perpetrator` · `same_jurisdiction` · `connected_victim` |
| `notes` | text | |

PK: `(case_id_a, case_id_b)`

---

### `case_faqs`

Q&A pairs per case → `FAQPage` JSON-LD → People Also Ask traffic.

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
| `tags` | text[] | e.g. `arrest` · `discovery` · `trial` |
| `sort_order` | int | Manual sort override |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `people`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | URL-safe, for person pages |
| `name` | text | |
| `aliases` | text[] | |
| `role` | enum | `victim` · `suspect` · `perpetrator` · `detective` · `witness` · `attorney` · `judge` · `other` |
| `dob` | date | nullable |
| `dod` | date | nullable |
| `bio` | text | |
| `photo_url` | text | |
| `links` | jsonb | `{ "wikipedia": "...", "news": [...] }` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_people`

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `person_id` | uuid | |
| `role_in_case` | text | Overrides person.role if needed |
| `notes` | text | |

PK: `(case_id, person_id)`

---

### `agencies`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | URL-safe, for agency pages |
| `name` | text | |
| `type` | enum | `local_pd` · `sheriff` · `fbi` · `da` · `state_police` · `other` (US-centric v1) |
| `jurisdiction` | text | |
| `contact_url` | text | Tip line, if public |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_agencies`

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
| `sort_order` | int | Display order on case page |
| `tags` | text[] | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `interviews`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `person_id` | uuid | FK → people (interviewee) |
| `interviewer` | text | Name or org |
| `date` | date | |
| `type` | enum | `police` · `media` · `court` · `documentary` |
| `summary` | text | |
| `duration_seconds` | int | For in-app player |
| `transcript_url` | text | |
| `media_url` | text | Audio/video |
| `source_url` | text | Citation |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `podcasts`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `title` | text | |
| `host` | text | |
| `rss_url` | text | For automated ingestion |
| `cover_image_url` | text | |
| `platforms` | jsonb | `{ "spotify": "...", "apple": "...", "youtube": "..." }` |
| `description` | text | |
| `auto_ingest` | bool | RSS sync enabled |
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
| `published_at` | timestamptz | Full timestamp from RSS feed |
| `duration_seconds` | int | |
| `audio_url` | text | Direct file URL (in-app player) |
| `external_only` | bool | true = link out only (Spotify-locked) |
| `episode_url` | text | Canonical episode / platform link |
| `transcript` | text | nullable, for search indexing |
| `source` | enum | `rss` · `manual` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_podcast_episodes`

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `episode_id` | uuid | |
| `relevance` | enum | `primary` · `mentions` |

PK: `(case_id, episode_id)`
_Note: `community_score` removed — compute from `upvotes` table via view._

---

### `tv_shows`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `title` | text | |
| `type` | enum | `documentary` · `docu_series` · `drama` · `news_special` |
| `network` | text | Netflix, HBO, ID, etc. |
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

### `case_tv_shows`

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `show_id` | uuid | |
| `episode_refs` | jsonb | Array of `{ season, episode }` objects |
| `notes` | text | |

PK: `(case_id, show_id)`

---

### `news_articles`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `title` | text | |
| `publication` | text | NYT, WaPo, etc. |
| `author` | text | |
| `published_at` | timestamptz | |
| `url` | text | |
| `summary` | text | |
| `source` | enum | `manual` · `rss` · `google_news` |
| `created_at` | timestamptz | |

### `case_news_articles`

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `article_id` | uuid | |

PK: `(case_id, article_id)`

---

### `case_subreddits`

Curated subreddit links per case.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `subreddit` | text | e.g. `r/JonBenetRamsey` |
| `description` | text | Short label |
| `member_count` | int | Cached, refreshed weekly |
| `is_primary` | bool | Pin the most relevant one |
| `last_synced_at` | timestamptz | |
| `created_at` | timestamptz | |

### `case_reddit_posts`

Top/recent Reddit posts auto-fetched per case.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `post_id` | text | Reddit post ID (deduplicate) |
| `title` | text | |
| `subreddit` | text | |
| `url` | text | |
| `score` | int | Upvotes |
| `comment_count` | int | |
| `fetched_at` | timestamptz | |

---

## Community Layer

> Community data is **always client-side fetched** — never SSG.

### `upvotes`

Generic upvote table. `community_notes.upvotes` int column removed — compute counts with a Postgres view.

| Field | Type | Notes |
|-------|------|-------|
| `user_id` | uuid | FK → profiles |
| `entity_type` | text | Table name |
| `entity_id` | uuid | FK (no Postgres enforcement — use cleanup job) |
| `created_at` | timestamptz | |

PK: `(user_id, entity_type, entity_id)`

**View:** `upvote_counts` — `SELECT entity_type, entity_id, COUNT(*) as count FROM upvotes GROUP BY entity_type, entity_id`

### `community_notes`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `user_id` | uuid | FK → profiles |
| `body` | text | |
| `is_pinned` | bool | Moderator pin |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

_Note: `upvotes int` removed — use `upvote_counts` view._

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
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `content_reports`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `reporter_id` | uuid | FK → profiles |
| `entity_type` | text | Any table name |
| `entity_id` | uuid | |
| `reason` | enum | `inaccurate` · `inappropriate` · `spam` · `copyright` · `other` |
| `notes` | text | |
| `status` | enum | `pending` · `reviewed` · `resolved` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `user_media_progress`

Client-side only. Tracks watched/listened status per user.

| Field | Type | Notes |
|-------|------|-------|
| `user_id` | uuid | FK → profiles |
| `media_type` | enum | `podcast_episode` · `tv_show` · `documentary` |
| `media_id` | uuid | |
| `status` | enum | `watched` · `listening` · `want_to_watch` |
| `updated_at` | timestamptz | |

PK: `(user_id, media_type, media_id)`

---

## Automation Paths

### Podcast Ingestion (RSS)
1. For each `podcast` where `auto_ingest = true`: fetch RSS feed
2. Parse → upsert `podcast_episodes` (deduplicate on `episode_url`)
3. NLP/keyword match against `cases` → suggest `case_podcast_episodes` links
4. Human review queue before publishing
5. Set `external_only = true` when no direct audio URL (Spotify-locked)

### TV/Documentary Ingestion (TMDB + JustWatch)
- **TMDB API** → show metadata, posters, seasons
- **JustWatch API** → streaming availability by region
- Manual case linking (documentary titles often don't name the case directly)

### Reddit Sync (weekly cron)
- `https://www.reddit.com/search.json?q=CASE_NAME&type=sr` → subreddit discovery
- `https://www.reddit.com/search.json?q=CASE_NAME&sort=top&t=all` → top posts
- No API key needed (use `User-Agent` header). Register Reddit OAuth app before scaling beyond top 20.

### News Article Ingestion
- Google News RSS per case (free, no key): `https://news.google.com/rss/search?q=CASE_NAME`
- Or NewsAPI.org (free tier 100 req/day)

### SEO Content Generation (Top 20 Cases)
- Each case needs: summary, body (~2,000 words), timeline (5+ events), person stubs, 3+ media links, 3+ FAQs
- Top 20 cases = automation test bed before scaling

---

## Top 20 Priority Cases (Initial Seed)

Draft — validate against search volume before final selection:

1. JonBenét Ramsey
2. Zodiac Killer
3. Gabby Petito
4. OJ Simpson
5. Laci Peterson
6. Ted Bundy
7. BTK Killer
8. Golden State Killer
9. Emmett Till
10. Caylee Anthony (Casey Anthony)
11. Tupac Shakur
12. Biggie Smalls
13. Trayvon Martin
14. Chandra Levy
15. Elizabeth Short (Black Dahlia)
16. Natalee Holloway
17. Etan Patz
18. DB Cooper
19. Elisa Lam
20. Chris Watts

---

## Open Questions

- **Audio player**: Howler.js or native HTML5 `<audio>` for in-app playback. Spotify embed API possible for linked episodes.
- **Knowledge graph viz**: Deferred to post-MVP. Use D3.js or Reagraph (React). Derive node/edge data from existing join tables — no dedicated schema needed at v1.
- **TMDB API key**: Required for TV/documentary enrichment. Register at themoviedb.org.
- **Supabase RLS**: Define row-level security policies — anon read for published cases, auth required for community writes.
- **`agencies.type` enum**: US-centric in v1. International cases will need `federal_intl`, `military`, etc. — flag as known tech debt.

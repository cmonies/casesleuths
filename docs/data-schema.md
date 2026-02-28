# CaseSleuths — Data Schema

_v0.1 · 2026-02-27_

This document defines the full data model for CaseSleuths. The schema is **case-centric**: every other entity (people, media, evidence, community contributions) hangs off a `Case`.

---

## Core Entity Map

```
Case
 ├── Timeline (events)
 ├── Knowledge Graph (nodes + edges → Obsidian-style viz)
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
 └── Community Layer
      ├── Upvotes
      ├── Notes
      ├── Corrections
      └── Reports
```

---

## Tables / Collections

### `cases`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | URL-safe identifier e.g. `zodiac-killer` |
| `title` | text | Display name |
| `alternate_names` | text[] | AKAs, nicknames |
| `status` | enum | `unsolved` · `solved` · `cold` · `closed` · `ongoing` |
| `crime_type` | text[] | `murder` · `disappearance` · `serial` · etc. |
| `location_city` | text | |
| `location_state` | text | |
| `location_country` | text | default `US` |
| `location_coords` | point | lat/lng for map pins |
| `date_start` | date | When crime(s) began |
| `date_end` | date | nullable — ongoing cases |
| `summary` | text | 1–2 paragraph overview |
| `body` | text/mdx | Full article-style case writeup (SEO content) |
| `seo_title` | text | |
| `seo_description` | text | |
| `cover_image_url` | text | |
| `search_rank` | int | 1–20 for initial top-20 priority tier |
| `published_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `timeline_events`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK → cases |
| `date` | date | |
| `date_precision` | enum | `exact` · `approximate` · `year_only` |
| `title` | text | Short label |
| `description` | text | |
| `source_url` | text | Citation |
| `media_url` | text | Optional image/doc |
| `tags` | text[] | e.g. `arrest` · `discovery` · `trial` |
| `order` | int | Manual sort override |

---

### `people`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `name` | text | |
| `aliases` | text[] | |
| `role` | enum | `victim` · `suspect` · `perpetrator` · `detective` · `witness` · `attorney` · `judge` · `other` |
| `dob` | date | nullable |
| `dod` | date | nullable |
| `bio` | text | |
| `photo_url` | text | |
| `links` | jsonb | `{ "wikipedia": "...", "news": [...] }` |

### `case_people` (join)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `person_id` | uuid | |
| `role_in_case` | text | Overrides person.role if needed |
| `notes` | text | Case-specific context |

---

### `agencies`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `name` | text | e.g. `LAPD`, `FBI Field Office - LA` |
| `type` | enum | `local_pd` · `sheriff` · `fbi` · `da` · `state_police` · `other` |
| `jurisdiction` | text | |
| `contact_url` | text | Tip line, if public |

### `case_agencies` (join)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `agency_id` | uuid | |
| `role` | text | e.g. `lead investigator` · `supporting` |

---

### `evidence`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `type` | enum | `photo` · `audio` · `document` · `video` · `physical` · `other` |
| `title` | text | |
| `description` | text | |
| `file_url` | text | Hosted asset (Supabase Storage) |
| `source` | text | Attribution |
| `is_public` | bool | Some evidence may be sensitive |
| `tags` | text[] | |
| `created_at` | timestamptz | |

_Audio player content (police calls, recordings) lives here with `type = audio`._

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
| `transcript_url` | text | |
| `media_url` | text | Audio/video |
| `source_url` | text | Citation |

---

### `podcasts` (shows)

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `title` | text | |
| `host` | text | |
| `rss_url` | text | For automated ingestion |
| `cover_image_url` | text | |
| `platforms` | jsonb | `{ "spotify": "...", "apple": "...", "youtube": "..." }` |
| `description` | text | |
| `auto_ingest` | bool | Whether RSS sync is enabled |
| `last_synced_at` | timestamptz | |

### `podcast_episodes`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `podcast_id` | uuid | FK → podcasts |
| `title` | text | |
| `description` | text | |
| `published_at` | date | |
| `duration_seconds` | int | |
| `audio_url` | text | Direct file URL if available (for in-app player) |
| `external_only` | bool | true = link out, no in-app player |
| `episode_url` | text | Canonical episode page / platform link |
| `transcript` | text | nullable — for search indexing |
| `source` | enum | `rss` · `manual` |

### `case_podcast_episodes` (join)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `episode_id` | uuid | |
| `relevance` | enum | `primary` · `mentions` |
| `community_score` | int | Upvote count |

---

### `tv_shows`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `title` | text | |
| `type` | enum | `documentary` · `docu_series` · `drama` · `news_special` |
| `network` | text | Netflix, HBO, ID, etc. |
| `year` | int | |
| `seasons` | int | nullable |
| `description` | text | |
| `cover_image_url` | text | |
| `tmdb_id` | int | For TMDB API integration |
| `imdb_id` | text | |
| `streaming_links` | jsonb | `{ "netflix": "...", "hulu": "..." }` |

### `case_tv_shows` (join)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | |
| `show_id` | uuid | |
| `episode_refs` | text | Optional: specific season/episode |
| `notes` | text | |

---

### `knowledge_graph_nodes`

For the Obsidian-style visual graph — each case gets its own node graph.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `entity_type` | enum | `person` · `place` · `event` · `evidence` · `organization` · `concept` |
| `entity_id` | uuid | FK to person/agency/evidence/etc (nullable if freeform) |
| `label` | text | Display label on graph |
| `metadata` | jsonb | Any extra context |
| `x` | float | Saved layout position |
| `y` | float | Saved layout position |

### `knowledge_graph_edges`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `source_node_id` | uuid | |
| `target_node_id` | uuid | |
| `relationship` | text | e.g. `knew`, `was_suspect_in`, `discovered`, `worked_with` |
| `weight` | float | Edge strength / confidence 0–1 |
| `notes` | text | |

---

## Community Layer

### `user_media_progress`

Tracks what a user has watched/listened to.

| Field | Type | Notes |
|-------|------|-------|
| `user_id` | uuid | FK → auth.users |
| `media_type` | enum | `podcast_episode` · `tv_show` · `documentary` |
| `media_id` | uuid | FK to relevant table |
| `status` | enum | `watched` · `listening` · `want_to_watch` |
| `updated_at` | timestamptz | |

### `community_notes`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `user_id` | uuid | |
| `body` | text | |
| `upvotes` | int | |
| `created_at` | timestamptz | |
| `is_pinned` | bool | Moderator can pin |

### `community_corrections`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK |
| `user_id` | uuid | |
| `field_path` | text | e.g. `timeline_events.123.date` |
| `current_value` | text | |
| `suggested_value` | text | |
| `source_url` | text | Required for corrections |
| `status` | enum | `pending` · `accepted` · `rejected` |
| `reviewed_by` | uuid | admin user |
| `created_at` | timestamptz | |

### `content_reports`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `reporter_id` | uuid | |
| `entity_type` | text | Any table name |
| `entity_id` | uuid | |
| `reason` | enum | `inaccurate` · `inappropriate` · `spam` · `copyright` · `other` |
| `notes` | text | |
| `status` | enum | `pending` · `reviewed` · `resolved` |
| `created_at` | timestamptz | |

### `upvotes`

Generic upvote table (polymorphic).

| Field | Type | Notes |
|-------|------|-------|
| `user_id` | uuid | |
| `entity_type` | text | |
| `entity_id` | uuid | |
| `created_at` | timestamptz | |

PK: `(user_id, entity_type, entity_id)`

---

## Automation Paths

### Podcast Ingestion (Phase 1 — RSS)
1. For each `podcast` where `auto_ingest = true`: fetch RSS feed
2. Parse episodes → upsert into `podcast_episodes` (deduplicate on `episode_url`)
3. Run NLP/keyword match against `cases` → suggest `case_podcast_episodes` links
4. Human review queue for suggested links before publishing
5. Set `external_only = true` when no direct audio URL (Spotify-locked shows)

### TV/Documentary Ingestion (Phase 2 — TMDB + JustWatch)
- **TMDB API** → fetch show metadata, posters, seasons
- **JustWatch API / Unofficial** → streaming availability per region
- Manual case linking (harder to automate — documentary titles often don't name the case)

### SEO Content Generation (Top 20 Cases)
- Each case needs: summary, body, timeline, person stubs, 3+ media links
- Use top 20 search-volume cases as the automation test bed
- Target: fully populated case page = ~2,000 words of indexable content

---

## Top 20 Priority Cases (Initial Seed)

To be confirmed — draft based on search volume:

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
16. Natalie Holloway
17. Etan Patz
18. DB Cooper
19. Elisa Lam
20. Chris Watts

---

## Notes / Open Questions

- **Audio player**: For `podcast_episodes` with a direct `audio_url` — serve in-app player. For `external_only = true` — link to platform. Spotify embeds may be possible via their embed API.
- **Photos / evidence**: Store in Supabase Storage, serve via signed URLs for anything sensitive.
- **Knowledge graph viz**: D3.js or Reagraph (React-based, handles large graphs well). Layout positions saved back to DB.
- **TMDB API key** needed for TV/documentary enrichment.
- **User auth**: Supabase Auth — anonymous read, account required to contribute notes/corrections/upvotes.

# CaseSleuths — Data Schema

_v0.9 · 2026-02-28 — P0 fixes: added qa_review→legal_review transition, fixed upvote trigger GREATEST clamp. P1 fixes: dropped redundant case_id/person_id from charges/appeals (join through case_people only), moderation_actions mutual non-null CHECK, Agent D stale pipeline + auto-suppress P0 policy, charges/appeals case_person_id indexes, sources.url UNIQUE, minimum source coverage publish gate, pipeline publish flow for standard/elevated cases. P2 fixes: charges_dropped in legal framework, suppressed_entities.entity_type CHECK, agent model specs, Typesense suppression note, pipeline diagram updated_

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
| `case_reddit_posts` | **Client-side** | Weekly cron; no `updated_at` — uses `fetched_at` instead (explicit exception) |
| `community_notes` | **Client-side** | Supabase Realtime subscription |
| `community_corrections` | **Client-side** | Supabase Realtime subscription |
| `content_reports` | **Admin only** | Admin dashboard |
| `charges` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `appeals` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `case_series` | SSG/ISR | DB webhook → series listing pages |
| `books` / `case_books` | SSG/ISR | DB webhook → resolve parent `case.slug` |
| `jurisdictions` | SSG/ISR | DB webhook → jurisdiction listing pages |
| `content_takedown_requests` | **Admin only** | Admin dashboard |
| `suppressed_entities` | **Admin only** | Admin dashboard — changes trigger ISR revalidation on affected pages |
| `moderation_actions` | **Admin only** | Admin dashboard |
| `upvotes` | **Client-side** | Supabase Realtime subscription |
| `user_media_progress` | **Client-side** | Local optimistic update; no `created_at` (explicit exception — only tracks current state) |

**ISR revalidation intervals:**
- Case pages: `revalidate: 3600` (1 hour)
- Tag / listing pages: `revalidate: 1800` (30 min)
- Person / agency pages: `revalidate: 3600` (1 hour)

**Revalidation implementation:** Supabase DB webhooks → Next.js `/api/revalidate?secret=TOKEN&path=/cases/[slug]`. Webhooks required on: `cases`, `timeline_events`, `case_people`, `evidence`, `interviews`, `case_faqs`, `podcast_episodes`, `tv_shows`, `news_articles`, `case_subreddits`, `case_tags`, `related_cases`.

---

### Minor Name & Photo Display Rules

Enforced in the app layer. The `people.name_display_policy` and `photo_display_policy` fields drive rendering.

| Scenario | Name | Photo |
|---|---|---|
| Minor victim (deceased/missing) | Full name ✅ | ✅ |
| Minor POI/suspect named by LE | Initials only | ❌ |
| Minor charged as adult | Full name ✅ | ✅ |
| Living minor (any other role) | ❌ blocked | ❌ |
| Minor who aged out (now adult) | Human review required | Human review required |
| Suppressed entity (any) | "[Name withheld]" | ❌ |
| Living person, not convicted | Full name + aggressive disclaimer ✅ | ✅ with review |

`name_display_policy: auto` = system evaluates the above rules at render time based on `is_minor_at_time_of_crime`, `is_minor_currently`, `is_living`, and `case_people.legal_status`. Override with explicit policy values when automated logic needs to be overridden by an editor.

---

### Upvotes — Realtime Strategy

Reddit's model (and ours): **optimistic updates + Supabase Realtime push**.

1. User clicks upvote → UI updates immediately via local state (no server round-trip)
2. Write INSERT (upvote) or DELETE (un-upvote) to `upvotes` table
3. Postgres trigger fires → increments/decrements `upvote_count` on parent entity row (atomic, no race condition)
4. Supabase Realtime broadcasts the change to all subscribed clients — everyone sees it instantly, no polling

**Count strategy:** Trigger-maintained `upvote_count int NOT NULL DEFAULT 0` on each upvotable entity. Do NOT use `SELECT COUNT(*) GROUP BY` views on hot paths.

**Soft delete policy:** When an entity is soft-deleted (`deleted_at` set), RLS blocks new upvotes (`WHERE deleted_at IS NULL`). **Do NOT zero `upvote_count` on soft delete** — historical count is preserved so restored entities recover their signal.

**Ongoing trial lock:** Cases with `status = 'ongoing_trial'` have community contributions (notes, corrections, upvotes) blocked via RLS. No annotations on active criminal proceedings.

**Minimum source coverage gate:** A publish trigger blocks `review_status = 'published'` unless the case has at least: 1 source in `entity_sources` linked to `cases`, and 1 source linked to each `timeline_event` in the case. Enforced at DB level — not process-only.

**Typesense indexing + suppression:** All Typesense indexing jobs must JOIN against `suppressed_entities WHERE lifted_at IS NULL` and redact suppressed entities before indexing. Suppressed persons must not appear in search results even if the case page shows "[Name withheld]". Indexing queries should use the `public_people` view (TODO: define RLS views).

**Drift insurance:** Nightly reconciliation cron recomputes all `upvote_count` values from the `upvotes` table directly — cheap insurance against trigger edge cases (manual SQL, pg_restore, bulk imports).

```sql
-- updated_at auto-trigger template (apply to every table with updated_at)
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER set_updated_at BEFORE UPDATE ON <table>
--   FOR EACH ROW EXECUTE FUNCTION set_updated_at();
-- Apply to: cases, tags, people, agencies, evidence, interviews, podcasts,
--   podcast_episodes, tv_shows, news_articles, case_faqs, case_subreddits,
--   profiles, community_notes, community_corrections, content_reports,
--   slug_redirects, timeline_events, case_people, charges, appeals,
--   jurisdictions, case_series, books, content_takedown_requests, sources

-- Upvote count trigger (dispatches dynamically on entity_type)
CREATE OR REPLACE FUNCTION update_upvote_count() RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    EXECUTE format('UPDATE %I SET upvote_count = upvote_count + 1 WHERE id = $1', NEW.entity_type)
      USING NEW.entity_id;
  ELSIF TG_OP = 'DELETE' THEN
    -- GREATEST prevents negative counts on duplicate deletes or manual cleanup
    EXECUTE format('UPDATE %I SET upvote_count = GREATEST(upvote_count - 1, 0) WHERE id = $1', OLD.entity_type)
      USING OLD.entity_id;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;
-- Note: entity_type values MUST match actual table names for dynamic dispatch
-- CREATE TRIGGER update_upvote_count AFTER INSERT OR DELETE ON upvotes
--   FOR EACH ROW EXECUTE FUNCTION update_upvote_count();
```

```sql
-- Source coverage publish gate trigger
-- Blocks review_status being set to 'published' unless minimum source coverage is met
CREATE OR REPLACE FUNCTION check_source_coverage() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.review_status = 'published' AND OLD.review_status != 'published' THEN
    -- Case must have at least 1 source linked directly
    IF NOT EXISTS (
      SELECT 1 FROM entity_sources
      WHERE entity_type = 'cases' AND entity_id = NEW.id
    ) THEN
      RAISE EXCEPTION 'Cannot publish case %: no sources linked to case record', NEW.id;
    END IF;
    -- Every timeline_event must have at least 1 source
    IF EXISTS (
      SELECT 1 FROM timeline_events te
      WHERE te.case_id = NEW.id
      AND NOT EXISTS (
        SELECT 1 FROM entity_sources es
        WHERE es.entity_type = 'timeline_events' AND es.entity_id = te.id
      )
    ) THEN
      RAISE EXCEPTION 'Cannot publish case %: one or more timeline_events have no source', NEW.id;
    END IF;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER check_source_coverage BEFORE UPDATE ON cases
--   FOR EACH ROW EXECUTE FUNCTION check_source_coverage();

-- review_status state machine trigger
-- Enforces allowed transitions; blocks illegal jumps (e.g. draft → published)
CREATE OR REPLACE FUNCTION enforce_review_status_transition() RETURNS TRIGGER AS $$
DECLARE
  allowed boolean := false;
BEGIN
  IF OLD.review_status = NEW.review_status THEN RETURN NEW; END IF; -- no-op
  -- Define allowed transitions
  IF (OLD.review_status = 'draft'          AND NEW.review_status = 'legal_review')  THEN allowed := true; END IF;
  IF (OLD.review_status = 'legal_review'   AND NEW.review_status IN ('qa_review', 'draft')) THEN allowed := true; END IF;
  IF (OLD.review_status = 'qa_review'      AND NEW.review_status IN ('approved', 'legal_review', 'draft')) THEN allowed := true; END IF;
  IF (OLD.review_status = 'approved'       AND NEW.review_status IN ('published', 'legal_review')) THEN allowed := true; END IF;
  IF (NEW.review_status = 'rejected')      THEN allowed := true; END IF; -- admin-only, any→rejected
  IF (NEW.review_status = 'suppressed')    THEN allowed := true; END IF; -- any→suppressed (takedown)
  IF (OLD.review_status = 'suppressed'     AND NEW.review_status = 'legal_review') THEN allowed := true; END IF; -- admin unsuppression
  IF NOT allowed THEN
    RAISE EXCEPTION 'Invalid review_status transition: % → %', OLD.review_status, NEW.review_status;
  END IF;
  -- Log transition to moderation_actions (agent_source set by caller via session variable or default)
  INSERT INTO moderation_actions (entity_type, entity_id, action_type, agent_source, metadata)
  VALUES ('cases', NEW.id, 'edited', current_setting('app.agent_source', true),
    jsonb_build_object('field', 'review_status', 'from', OLD.review_status, 'to', NEW.review_status));
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER enforce_review_status_transition BEFORE UPDATE ON cases
--   FOR EACH ROW WHEN (OLD.review_status IS DISTINCT FROM NEW.review_status)
--   EXECUTE FUNCTION enforce_review_status_transition();

-- slug_redirects sync trigger
-- When cases.slug changes, updates denormalized new_slug on all slug_redirects rows pointing to this case
CREATE OR REPLACE FUNCTION sync_slug_redirect() RETURNS TRIGGER AS $$
BEGIN
  IF OLD.slug IS DISTINCT FROM NEW.slug THEN
    UPDATE slug_redirects SET new_slug = NEW.slug WHERE new_case_id = NEW.id;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER sync_slug_redirect AFTER UPDATE ON cases
--   FOR EACH ROW WHEN (OLD.slug IS DISTINCT FROM NEW.slug)
--   EXECUTE FUNCTION sync_slug_redirect();

-- cases.year sync trigger (application-managed — set from date_start on insert/update)
CREATE OR REPLACE FUNCTION sync_case_year() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.date_start IS NOT NULL THEN
    NEW.year := EXTRACT(YEAR FROM NEW.date_start)::int;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER sync_case_year BEFORE INSERT OR UPDATE ON cases
--   FOR EACH ROW EXECUTE FUNCTION sync_case_year();
```

**Security note:** The dynamic SQL in the trigger uses `entity_type` as a table name. The `CHECK` constraint on `entity_type` is the sole injection guard — never remove it or allow unvalidated values in this column. For maximum safety at scale, replace with per-entity triggers (one dedicated trigger per upvotable table).

**Hot-row contention note:** At high traffic, constant `upvote_count` increments on a single row can become a write bottleneck. Mitigation at scale: sharded counter tables or batched aggregation. For MVP (top 20 cases), trigger approach is fine.

**Client implementation note:** Supabase Realtime channels must be explicitly unsubscribed when the user navigates away from a case page. Failing to do so causes memory leaks as channels accumulate. Scope each subscription to the page lifecycle (mount → subscribe, unmount → unsubscribe).

---

### FK Cascade Policy

**Default: `ON DELETE CASCADE` for all child rows** (timeline_events, case_people, evidence, etc. are deleted when their parent case is deleted). Pure join tables always cascade. **Exception:** `people` and `agencies` use `ON DELETE RESTRICT` — deleting a person or agency that appears in multiple cases requires explicit cleanup. Document any exceptions per-table where they deviate.

---

### Legal Framework

See `docs/legal-safety-framework.md` for full policy. Key constraints:

- **No `perpetrator` label anywhere in this schema.** Word is banned.
- **Controlled legal vocabulary:** see person legal status enum below. These are the only valid values.
- **`legal_sensitivity: high`** cases block auto-publish; human review required before `published_at` is set.
- **`disclaimer` field** on cases surfaces "no charges filed" or equivalent prominently on the page.
- **Content warnings** are required fields — stored in `cases.content_warnings`, shown before content.
- **Ongoing trial lock**: `status = 'ongoing_trial'` blocks community contributions via RLS.
- **Legal review skill** (planned): evaluates every case page against this framework before publish. Checks vocabulary compliance, disclaimer presence, `legal_sensitivity` classification, content warning completeness.

**Person legal status — controlled vocabulary (use these exact values, no others):**

| Value | When to use |
|-------|-------------|
| `victim` | Victim of the crime |
| `alleged` | Formally charged, trial pending |
| `convicted` | Conviction currently stands (note if appealing) |
| `acquitted` | Charged, found not guilty |
| `exonerated` | Previously convicted, conviction overturned |
| `person_of_interest` | Named by investigators, not formally charged |
| `charges_dropped` | Was formally charged; charges subsequently dropped or dismissed before trial |
| `no_charges_filed` | Named in media or podcasts, never formally accused |
| `witness` | Witness to events |
| `detective` | Investigator on the case |
| `attorney` | Defense, prosecution, or civil attorney |
| `judge` | Presiding judge |
| `other` | Any other role |

---

### Enum Definitions

> All enums must be created with `CREATE TYPE ... AS ENUM (...)` before any table that references them. Supabase migration order: enums → tables → FKs → triggers → RLS policies → indexes.

| Enum name | Values |
|-----------|--------|
| `case_status` | `unsolved` · `solved_convicted` · `solved_acquitted` · `solved_exonerated` · `ongoing_trial` · `cold_case` · `civil_resolution` · `closed_no_crime` · `closed_suspect_deceased` |
| `legal_sensitivity` | `standard` · `elevated` · `high` |
| `review_status` | `draft` · `legal_review` · `qa_review` · `approved` · `published` · `rejected` · `suppressed` |
| `person_legal_status` | `victim` · `alleged` · `convicted` · `acquitted` · `exonerated` · `charges_dropped` · `person_of_interest` · `no_charges_filed` · `witness` · `detective` · `attorney` · `judge` · `other` |
| `name_display_policy` | `full` · `initials_only` · `redacted` · `auto` |
| `photo_display_policy` | `allowed` · `blocked` · `requires_review` |
| `app_role` | `user` · `moderator` · `admin` |
| `event_type` | `crime_event` · `discovery` · `arrest` · `indictment` · `trial_start` · `verdict` · `sentencing` · `appeal_filed` · `appeal_ruling` · `release` · `death` · `media_coverage` · `other` |
| `date_precision` | `exact` · `approximate` · `year_only` |
| `evidence_type` | `photo` · `audio` · `document` · `video` · `physical` · `other` |
| `interview_type` | `police` · `media` · `court` · `documentary` |
| `tv_show_type` | `documentary` · `docu_series` · `drama` · `news_special` |
| `media_source` | `rss` · `manual` |
| `news_source` | `manual` · `rss` · `google_news` |
| `relevance` | `primary` · `mentions` |
| `media_type` | `podcast_episode` · `tv_show` |
| `media_progress_status` | `completed` · `in_progress` · `want_to_consume` — **canonical values** (overrides any table definition using `watched`/`listening`/`want_to_watch`) |
| `correction_status` | `pending` · `accepted` · `rejected` |
| `report_reason` | `inaccurate` · `inappropriate` · `spam` · `copyright` · `other` |
| `report_status` | `pending` · `reviewed` · `resolved` |
| `charge_disposition` | `pending` · `dismissed` · `convicted` · `acquitted` · `plea_deal` · `mistrial` · `charges_dropped` |
| `appeal_type` | `direct_appeal` · `habeas_corpus` · `post_conviction_relief` · `civil` |
| `appeal_status` | `pending` · `granted` · `denied` · `withdrawn` |
| `source_type` | `court_document` · `news_article` · `official_statement` · `book` · `documentary` · `podcast_episode` · `other` |
| `claim_type` | `allegation` · `court_finding` · `media_report` · `official_statement` |
| `jurisdiction_type` | `federal` · `state` · `county` · `city` · `international` |
| `takedown_request_type` | `legal_demand` · `family_request` · `subject_request` · `minor_aged_out` · `victim_privacy` · `court_order` · `dmca` |
| `takedown_status` | `received` · `under_review` · `complied` · `partially_complied` · `denied` · `escalated` |
| `suppression_reason` | `legal_demand` · `court_order` · `minor_privacy` · `victim_request` · `family_request` · `editorial` |
| `moderation_action_type` | `approved` · `rejected` · `edited` · `deleted` · `restored` · `suppressed` · `unsuppressed` · `locked` · `unlocked` |
| `agency_type` | `local_pd` · `sheriff` · `fbi` · `da` · `state_police` · `other` |

### Agent Model Specs

| Agent | Role | Recommended Model | Notes |
|-------|------|-------------------|-------|
| Agent A | Researcher / content seeder | `claude-sonnet` | Broad web research, structured data extraction |
| Agent B | Legal reviewer | `claude-opus` (minimum) | Defamation defense gate — high accuracy required, especially for `legal_sensitivity: high` cases |
| Agent C | QA / editorial reviewer | `claude-sonnet` | Framing and completeness check |
| Agent D | Compliance monitor | `claude-haiku` for SQL/pattern checks, `claude-sonnet` for random sample legal framing audit | Haiku for structured DB queries; upgrade to Sonnet for the 5-case/night framing scan |

---

**Controlled text fields** (stored as `text` or `text[]` with CHECK constraints, not enums):

```sql
-- content_warnings array
CHECK (content_warnings <@ ARRAY['child_victim','sexual_violence','infant_victim',
  'family_violence','mass_casualty','graphic_imagery_risk']::text[])

-- primary_weapon
CHECK (primary_weapon IN ('firearm','knife','blunt_object','poison',
  'strangulation','unknown','other'))

-- entity_sources.entity_type
CHECK (entity_type IN ('cases','people','timeline_events','evidence','charges','appeals'))

-- is_living consistency with dod
CHECK (dod IS NULL OR is_living IS NOT TRUE)
```

---

### Rendering
- `cases.body` uses plain **Markdown**, not MDX. Rendered with `react-markdown`.

### Structured Data / SEO
- Every case page outputs: `Article`, `Person`, `Event`, `BreadcrumbList`, `FAQPage` JSON-LD
- `og_image_url` is distinct from `cover_image_url` — OG images = 1200×630

---

## Core Entity Map

```
Case
 ├── Timeline (events with event_type enum)
 ├── Knowledge Graph (derived from existing joins at MVP — no dedicated tables)
 ├── Tags (normalized, browsable tag pages)
 ├── Content Warnings
 ├── People (legal-status labeled, minor/privacy flags)
 │    └── Charges (structured legal proceedings per person)
 │         └── Appeals (per charge or per case)
 ├── Sources (citation trail — defamation defense)
 ├── Jurisdictions (state/country — programmatic SEO)
 ├── Case Series (serial killer / connected case grouping)
 ├── Agencies
 ├── Evidence
 ├── Interviews (with interview_people join for group interviews)
 ├── Media Coverage
 │    ├── Podcast Episodes (RSS auto-ingest)
 │    ├── TV Shows / Documentaries (TMDB)
 │    ├── News Articles
 │    └── Books
 ├── Reddit (subreddits + top posts)
 ├── Related Cases
 ├── FAQs
 ├── Community Layer (client-side, Realtime)
 │    ├── Upvotes (trigger count + Realtime push)
 │    ├── Notes (soft delete, optional threading via parent_id)
 │    ├── Corrections
 │    └── Reports
 └── Admin / Legal Layer
      ├── Content Takedown Requests
      ├── Suppressed Entities (kill switch)
      └── Moderation Actions (immutable audit log)
```

---

## Tables

> **Convention:** Every table has `created_at timestamptz NOT NULL DEFAULT now()` and `updated_at timestamptz NOT NULL DEFAULT now()` with a `set_updated_at` trigger, unless explicitly noted as an exception. Pure join tables (e.g. `case_tags`) omit both.

---

### `profiles`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK, FK → auth.users (ON DELETE CASCADE) |
| `display_name` | text | |
| `avatar_url` | text | |
| `app_role` | enum | NOT NULL DEFAULT `user` — `user` · `moderator` · `admin` |
| `is_banned` | bool | default false |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `cases`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | NOT NULL, UNIQUE, URL-safe |
| `title` | text | NOT NULL |
| `alternate_names` | text[] | NOT NULL DEFAULT '{}' |
| `status` | enum | NOT NULL — `unsolved` · `solved_convicted` · `solved_acquitted` · `solved_exonerated` · `ongoing_trial` · `cold_case` · `civil_resolution` · `closed_no_crime` · `closed_suspect_deceased` |
| `legal_sensitivity` | enum | NOT NULL DEFAULT `standard` — `high` blocks auto-publish |
| `content_warnings` | text[] | NOT NULL DEFAULT '{}' — Controlled list: `child_victim` · `sexual_violence` · `infant_victim` · `family_violence` · `mass_casualty` · `graphic_imagery_risk` |
| `jurisdiction_id` | uuid | FK → jurisdictions (SET NULL) — nullable |
| `location_city` | text | |
| `location_state` | text | |
| `location_country` | text | default `US` |
| `state_code` | text | 2-letter US state code — for /state/california SEO pages |
| `country_code` | text | NOT NULL default `US` — for international filtering |
| `location_coords` | point | lat/lng |
| `date_start` | date | |
| `date_end` | date | nullable |
| `year` | int | **Application-managed** (not a Postgres GENERATED ALWAYS AS column — Supabase does not reliably support generated columns on all plan tiers). Set by Agent A on insert from `EXTRACT(YEAR FROM date_start)`. Update trigger recommended: `BEFORE UPDATE ON cases` → `NEW.year = EXTRACT(YEAR FROM NEW.date_start) IF NEW.year IS NULL`. Nullable — some cold cases lack a precise date. Used for /year/1994 SEO pages. |
| `summary` | text | |
| `body` | text | Plain Markdown |
| `disclaimer` | text | Shown prominently — e.g. "No one has been charged in this case." |
| `seo_title` | text | |
| `seo_description` | text | |
| `cover_image_url` | text | |
| `og_image_url` | text | 1200×630 for Open Graph |
| `number_of_victims` | int | nullable — for programmatic SEO stats pages |
| `primary_weapon` | text | nullable — controlled: `firearm` · `knife` · `blunt_object` · `poison` · `strangulation` · `unknown` · `other` |
| `review_status` | enum | NOT NULL DEFAULT `draft` — pipeline gate, see `docs/data-pipeline.md`. **Valid transitions only:** `draft→legal_review`, `legal_review→qa_review`, `legal_review→draft` (Agent B FAIL), `qa_review→approved`, `qa_review→legal_review` (Agent C vocab-only fix), `qa_review→draft` (Agent C body-rewrite needed), `approved→published`, `approved→legal_review` (source coverage gate fail — system job), `any→rejected` (admin only), `any→suppressed` (takedown), `suppressed→legal_review` (admin-initiated unsuppression only). Enforced by app layer + state machine trigger (SQL below). `rejected` = permanently blocked (admin decision, not Agent B/C FAIL). |
| `revision_count` | int | NOT NULL DEFAULT 0 — incremented on each Agent B/C FAIL → draft cycle; escalates to human review after 3 |
| `published_at` | timestamptz | null = draft. Never auto-set for `legal_sensitivity: high`. Must be accompanied by `review_status: published`. CHECK: `(published_at IS NULL OR review_status = 'published') AND (review_status = 'published' OR published_at IS NULL)` — prevents `published_at` being set on a draft case and prevents `review_status: published` without a timestamp. |
| `deleted_at` | timestamptz | Soft delete. Cases are NEVER hard-deleted — destroys audit trail. Set `deleted_at` instead. RLS hides from all public queries. |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `slug_redirects`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `old_slug` | text | UNIQUE |
| `new_case_id` | uuid | FK → cases.id (ON DELETE CASCADE) — canonical reference, immune to slug changes |
| `new_slug` | text | Denormalized cache of the current slug — maintained by trigger, never set manually |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `tags`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | NOT NULL, UNIQUE |
| `label` | text | NOT NULL |
| `description` | text | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_tags` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
| `tag_id` | uuid | FK → tags (ON DELETE CASCADE) |

PK: `(case_id, tag_id)`

---

### `related_cases` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id_a` | uuid | FK → cases (ON DELETE CASCADE) |
| `case_id_b` | uuid | FK → cases (ON DELETE CASCADE) |
| `relationship_type` | text | e.g. `same_alleged_person` · `same_jurisdiction` · `connected_victim` · `linked_investigation` |
| `notes` | text | |

PK: `(case_id_a, case_id_b)`
CONSTRAINT: `CHECK (case_id_a < case_id_b)` — prevents `(A,B)` and `(B,A)` duplicates. Queries: `WHERE case_id_a = X OR case_id_b = X`.

---

### `case_faqs`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
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
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
| `date` | date | |
| `date_precision` | enum | `exact` · `approximate` · `year_only` |
| `event_type` | enum | `crime_event` · `discovery` · `arrest` · `indictment` · `trial_start` · `verdict` · `sentencing` · `appeal_filed` · `appeal_ruling` · `release` · `death` · `media_coverage` · `other` |
| `title` | text | |
| `description` | text | |
| `source_url` | text | Citation |
| `media_url` | text | Optional image/doc |
| `event_tags` | text[] | **Intentionally denormalized** — local timeline filtering only, not browsable on tag pages |
| `sort_order` | int | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `people`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | NOT NULL, UNIQUE, URL-safe — for person SEO pages |
| `name` | text | NOT NULL |
| `aliases` | text[] | NOT NULL DEFAULT '{}' |
| `primary_known_role` | `person_legal_status` enum | **Display hint only.** Source of truth is `case_people.legal_status`. Same enum type as `case_people.legal_status`. |
| `dob` | date | nullable |
| `dod` | date | nullable |
| `bio` | text | |
| `photo_url` | text | |
| `is_living` | bool | nullable — null = unknown. `true` + not convicted → aggressive disclaimer required. CHECK: `(dod IS NULL OR is_living IS NOT TRUE)` — person cannot have a date of death and be marked living. |
| `is_minor_at_time_of_crime` | bool | nullable — under 18 when the crime occurred |
| `is_minor_currently` | bool | nullable — still a minor today. **Auto-update mechanism:** Agent D nightly cron checks all `people` rows where `is_minor_currently = true` and `dob IS NOT NULL`; if `CURRENT_DATE - dob >= 18 years`, sets `is_minor_currently = false` and logs a `moderation_actions` record (`action_type: edited`, `agent_source: agent_d`, `metadata: { "field": "is_minor_currently", "from": true, "to": false }`). Editors can override by setting explicit `name_display_policy` + `photo_display_policy` values (not `auto`). |
| `name_display_policy` | enum | `full` · `initials_only` · `redacted` · `auto` — default `auto` (enforces minor display rules) |
| `photo_display_policy` | enum | `allowed` · `blocked` · `requires_review` — default `requires_review` |
| `links` | jsonb | `{ "wikipedia": "...", "news": [...] }` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_people`

Surrogate PK supports multi-role per case (e.g. initially `witness`, later `alleged`).

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK (surrogate) |
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
| `person_id` | uuid | FK → people (ON DELETE RESTRICT) |
| `legal_status` | enum | NOT NULL — **Source of truth.** Values: `victim` · `alleged` · `convicted` · `acquitted` · `exonerated` · `charges_dropped` · `person_of_interest` · `no_charges_filed` · `witness` · `detective` · `attorney` · `judge` · `other` |
| `notes` | text | Case-specific context |
| `is_primary` | bool | Flag most prominent role when multiple rows exist per person |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

UNIQUE: `(case_id, person_id)` — **one row per person per case**. `legal_status` is updated in place as the investigation evolves (e.g. `witness` → `alleged`). Status change history is tracked in `moderation_actions` (before/after in `metadata`). The surrogate PK `id` is kept for stable FK references from `charges` and `appeals`.

**When to update vs insert:** Always update the existing row. Never insert a second row for the same person-case pair. If a person's role in a case changes, update `legal_status` and log the change in `moderation_actions`.

**FK cascade policies:**
- `case_people.case_id` → ON DELETE CASCADE
- `case_people.person_id` → ON DELETE RESTRICT

---

### `agencies`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | NOT NULL, UNIQUE, URL-safe |
| `name` | text | NOT NULL |
| `type` | enum | `local_pd` · `sheriff` · `fbi` · `da` · `state_police` · `other` _(US-centric v1 — known tech debt)_ |
| `jurisdiction` | text | |
| `contact_url` | text | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_agencies` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | FK → cases (CASCADE) |
| `agency_id` | uuid | FK → agencies (RESTRICT) |
| `role` | text | e.g. `lead investigator` · `supporting` |

PK: `(case_id, agency_id)`

---

### `evidence`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
| `type` | enum | `photo` · `audio` · `document` · `video` · `physical` · `other` |
| `title` | text | |
| `description` | text | |
| `file_url` | text | Supabase Storage |
| `source` | text | Attribution |
| `is_public` | bool | |
| `sort_order` | int | |
| `evidence_tags` | text[] | **Intentionally denormalized** — local filtering only, not browsable on tag pages |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `interviews`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
| `type` | enum | `police` · `media` · `court` · `documentary` |
| `interviewer` | text | |
| `date` | date | |
| `summary` | text | |
| `duration_seconds` | int | For in-app player |
| `transcript_url` | text | |
| `media_url` | text | |
| `source_url` | text | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `interview_people` (pure join — no timestamps)

Supports group interviews.

| Field | Type | Notes |
|-------|------|-------|
| `interview_id` | uuid | FK → interviews (CASCADE) |
| `person_id` | uuid | FK → people (RESTRICT) |
| `role` | text | `interviewee` · `interviewer` |

PK: `(interview_id, person_id)`

---

### `podcasts`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `title` | text | |
| `host` | text | |
| `rss_url` | text | UNIQUE |
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
| `podcast_id` | uuid | FK → podcasts (ON DELETE CASCADE) |
| `title` | text | |
| `description` | text | |
| `published_at` | timestamptz | Full timestamp from RSS |
| `duration_seconds` | int | |
| `audio_url` | text | Direct URL for in-app player |
| `external_only` | bool | true = link out only (Spotify-locked) |
| `episode_url` | text | UNIQUE — canonical link, RSS deduplication key |
| `transcript` | text | nullable — Typesense indexing |
| `source` | enum | `rss` · `manual` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_podcast_episodes` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | FK → cases (CASCADE) |
| `episode_id` | uuid | FK → podcast_episodes (CASCADE) |
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
| `year_end` | int | nullable |
| `seasons` | int | nullable |
| `description` | text | |
| `cover_image_url` | text | |
| `tmdb_id` | int | UNIQUE |
| `imdb_id` | text | UNIQUE |
| `streaming_links` | jsonb | `{ "netflix": "...", "hulu": "..." }` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_tv_shows` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | FK → cases (CASCADE) |
| `show_id` | uuid | FK → tv_shows (ON DELETE CASCADE) |
| `episode_refs` | jsonb | Array of `{ "season": 2, "episode": 5 }` |
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
| `url` | text | UNIQUE — deduplication key |
| `summary` | text | |
| `source` | enum | `manual` · `rss` · `google_news` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_news_articles` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
| `article_id` | uuid | FK → news_articles (ON DELETE CASCADE) |

PK: `(case_id, article_id)`

---

### `case_subreddits`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK → cases (CASCADE) |
| `subreddit` | text | e.g. `r/JonBenetRamsey` |
| `description` | text | |
| `member_count` | int | Cached, refreshed weekly |
| `is_primary` | bool | |
| `last_synced_at` | timestamptz | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

UNIQUE: `(case_id, subreddit)` — r/TrueCrime can appear on multiple cases.

### `case_reddit_posts`

_Exception: uses `fetched_at` instead of `updated_at`. Scores are updated in-place on each sync._

Retention: keep top 50 posts per case by score. Upsert on `post_id` UNIQUE on each sync.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK → cases (CASCADE) |
| `subreddit_id` | uuid | FK → case_subreddits (SET NULL on delete) — null if post from non-curated subreddit |
| `post_id` | text | UNIQUE — Reddit base36 ID, deduplication key |
| `title` | text | |
| `subreddit` | text | Denormalized for display |
| `url` | text | |
| `score` | int | |
| `comment_count` | int | |
| `fetched_at` | timestamptz | Updated on each sync |
| `created_at` | timestamptz | |

**Reddit API note:** Register Reddit OAuth app from day one (100 req/min free). Do not rely on unauthenticated `.json` endpoints for production — rate-limited to ~10 req/min and have been breaking since Reddit's 2023 API changes.

---

### `jurisdictions`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `name` | text | e.g. `Los Angeles County, CA` |
| `state_code` | text | nullable — 2-letter US state |
| `country_code` | text | NOT NULL default `US` |
| `jurisdiction_type` | enum | `federal` · `state` · `county` · `city` · `international` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `charges`

Structured legal proceedings per person per case. `case_people.legal_status` is the display shorthand; `charges` is the source-of-truth legal record.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_person_id` | uuid | NOT NULL, FK → case_people.id (ON DELETE CASCADE) — single FK; derive `case_id` and `person_id` by joining `case_people`. Eliminates the cross-field consistency problem from having all three columns. |
| `charge_label` | text | NOT NULL — e.g. `First-degree murder` |
| `statute_citation` | text | nullable — e.g. `Cal. Penal Code § 187` |
| `filed_date` | date | nullable |
| `disposition` | enum | `pending` · `dismissed` · `convicted` · `acquitted` · `plea_deal` · `mistrial` · `charges_dropped` |
| `disposition_date` | date | nullable |
| `sentence_summary` | text | nullable — e.g. `Life without parole` |
| `jurisdiction_id` | uuid | FK → jurisdictions (SET NULL) |
| `court_name` | text | nullable |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

**Display note:** If a person has charges with `disposition: pending`, show "Convicted (appealing)" or "Charged (trial pending)" in the UI — not just the flat `legal_status` value.

---

### `appeals`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_person_id` | uuid | NOT NULL, FK → case_people.id (ON DELETE CASCADE) — derive case_id/person_id by joining case_people |
| `charge_id` | uuid | FK → charges (nullable — ON DELETE SET NULL) |
| `appeal_type` | enum | `direct_appeal` · `habeas_corpus` · `post_conviction_relief` · `civil` |
| `filed_date` | date | nullable |
| `status` | enum | `pending` · `granted` · `denied` · `withdrawn` |
| `outcome_summary` | text | nullable |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

**Display note:** Active appeal → show "Convicted (appealing)" rather than "Convicted" on case pages.

---

### `sources`

Citation trail — defamation defense. Every factual claim should have a source record.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `source_type` | enum | `court_document` · `news_article` · `official_statement` · `book` · `documentary` · `podcast_episode` · `other` |
| `title` | text | |
| `publisher` | text | nullable |
| `url` | text | NOT NULL, UNIQUE — deduplication key for citation audits |
| `archived_url` | text | nullable — Wayback Machine fallback |
| `published_at` | timestamptz | nullable |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `entity_sources` (pure join — no timestamps)

Links sources to any entity (case, person, timeline event, evidence, etc.) with claim type tagging.

| Field | Type | Notes |
|-------|------|-------|
| `source_id` | uuid | FK → sources (ON DELETE CASCADE) |
| `entity_type` | text | CHECK: `IN ('cases','people','timeline_events','evidence','charges','appeals')` |
| `entity_id` | uuid | |
| `claim_type` | enum | `allegation` · `court_finding` · `media_report` · `official_statement` — prevents editors from implying guilt |
| `notes` | text | nullable |

PK: `(source_id, entity_type, entity_id)`

---

### `case_series`

Groups serial killer cases or connected case clusters.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `name` | text | e.g. `Ted Bundy Murders` |
| `slug` | text | UNIQUE |
| `description` | text | nullable |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_series_members` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `series_id` | uuid | FK → case_series (ON DELETE CASCADE) |
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
| `sort_order` | int | |

PK: `(series_id, case_id)`

---

### `books`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `title` | text | |
| `author` | text | |
| `isbn` | text | UNIQUE, nullable |
| `published_year` | int | nullable |
| `publisher` | text | nullable |
| `cover_image_url` | text | nullable |
| `description` | text | nullable |
| `buy_url` | text | nullable |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_books` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
| `book_id` | uuid | FK → books (ON DELETE CASCADE) |

PK: `(case_id, book_id)`

---

## Community Layer

> All community data is **client-side fetched**. Never SSG. Use Supabase Realtime.
> Cases with `status = 'ongoing_trial'` have community features locked via RLS.

### `upvotes`

| Field | Type | Notes |
|-------|------|-------|
| `user_id` | uuid | FK → profiles (ON DELETE CASCADE) |
| `entity_type` | text | Table name — must match an actual table for trigger dispatch |
| `entity_id` | uuid | No Postgres FK (polymorphic) — orphan cleanup via trigger on soft delete |
| `created_at` | timestamptz | |

PK: `(user_id, entity_type, entity_id)` — naturally prevents double-upvotes.

CONSTRAINT: `CHECK (entity_type IN ('community_notes'))` — extend this list as more entities become upvotable. Prevents trigger runtime errors from bad entity_type values.

### `community_notes`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
| `parent_id` | uuid | FK → community_notes (self-ref, nullable) ON DELETE SET NULL — if parent is moderated/removed, replies are not deleted; they become top-level |
| `user_id` | uuid | FK → profiles (ON DELETE SET NULL) |
| `body` | text | |
| `upvote_count` | int | NOT NULL DEFAULT 0, CHECK (upvote_count >= 0) — trigger-maintained; trigger uses `GREATEST(upvote_count - 1, 0)` to prevent negative counts on duplicate deletes |
| `is_pinned` | bool | NOT NULL DEFAULT false |
| `deleted_at` | timestamptz | Soft delete. UI shows "[removed]". RLS blocks new upvotes (`WHERE deleted_at IS NULL`). `upvote_count` preserved (not zeroed) — restored notes recover their signal. |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `community_corrections`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | FK → cases (ON DELETE CASCADE) |
| `user_id` | uuid | FK → profiles (ON DELETE SET NULL) |
| `field_path` | text | e.g. `timeline_events.{id}.date` |
| `current_value` | text | |
| `suggested_value` | text | |
| `source_url` | text | NOT NULL — source required, enforced at DB level |
| `status` | enum | `pending` · `accepted` · `rejected` |
| `reviewed_by` | uuid | FK → profiles (nullable) |
| `deleted_at` | timestamptz | Soft delete |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `content_reports`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `reporter_id` | uuid | FK → profiles (ON DELETE SET NULL) |
| `entity_type` | text | NOT NULL, CHECK (`entity_type IN ('cases','community_notes','community_corrections','people','timeline_events','evidence')`) |
| `entity_id` | uuid | NOT NULL |
| `reason` | enum | `inaccurate` · `inappropriate` · `spam` · `copyright` · `other` |
| `notes` | text | |
| `status` | enum | `pending` · `reviewed` · `resolved` |
| `reviewed_by` | uuid | FK → profiles (nullable) — audit trail parity with corrections |
| `deleted_at` | timestamptz | Soft delete |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

## Admin / Legal Layer

> These tables are admin-only. Never exposed via public API or client-side Supabase queries. RLS: `admin` role only.

### `content_takedown_requests`

All takedown/erasure/legal requests must be logged here — never handle informally.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `request_type` | enum | `legal_demand` · `family_request` · `subject_request` · `minor_aged_out` · `victim_privacy` · `court_order` · `dmca` |
| `entity_type` | text | CHECK (`entity_type IN ('cases','people','timeline_events','evidence','community_notes','community_corrections')`) — what is being requested for removal |
| `entity_id` | uuid | |
| `requester_name` | text | nullable |
| `requester_email` | text | nullable |
| `requester_organization` | text | nullable — law firm, family org, etc. |
| `details` | text | Full description of request |
| `supporting_document_url` | text | nullable — uploaded letter, court order, etc. |
| `status` | enum | `received` · `under_review` · `complied` · `partially_complied` · `denied` · `escalated` |
| `assigned_to` | uuid | FK → profiles (nullable) |
| `response_notes` | text | nullable |
| `response_deadline` | timestamptz | nullable — 24h for `court_order`, 72h for `legal_demand`, 7 days for personal requests |
| `resolved_at` | timestamptz | nullable |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

**Response SLAs:** `court_order` = 24h · `legal_demand` = 72h · `minor_aged_out` = 48h · all others = 7 days. After complying: purge Typesense index entry, submit Google Search Console removal request, check OG cache.

### `suppressed_entities`

Kill switch. Suppressed person → renders as "[Name withheld]" site-wide. Suppressed case → renders as "[Case removed]". Data is preserved for audit/legal; only display is blocked.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `entity_type` | text | NOT NULL, CHECK (`entity_type IN ('people', 'cases')`) |
| `entity_id` | uuid | NOT NULL |
| `reason` | enum | NOT NULL — `legal_demand` · `court_order` · `minor_privacy` · `victim_request` · `family_request` · `editorial` |
| `takedown_request_id` | uuid | FK → content_takedown_requests (nullable) |
| `suppressed_by` | uuid | FK → profiles (SET NULL) — nullable; null when suppression is agent-initiated (e.g. Agent D auto-suppress) |
| `agent_source` | text | nullable — `agent_a` · `agent_b` · `agent_c` · `agent_d` · `system` · `human`. CHECK: `(suppressed_by IS NOT NULL OR agent_source IS NOT NULL)` — same mutual non-null pattern as `moderation_actions`. |
| `suppressed_at` | timestamptz | NOT NULL DEFAULT now() |
| `lifted_at` | timestamptz | nullable — if suppression was later reversed |
| `lift_reason` | text | nullable |
| `created_at` | timestamptz | |

PARTIAL UNIQUE INDEX: `CREATE UNIQUE INDEX ON suppressed_entities (entity_type, entity_id) WHERE lifted_at IS NULL` — allows re-suppression after a suppression has been lifted. No full `UNIQUE` constraint (would block INSERT after lift). No `updated_at` — immutable; use `lifted_at` to record reversal.

**Unsuppression flow:** To lift suppression: (1) set `lifted_at = now()` and `lift_reason` on the existing row — **never delete** it (audit trail); (2) log a `moderation_actions` record (`action_type: unsuppressed`, `agent_source: human`, `metadata: { reason }`); (3) trigger ISR revalidation on affected case pages; (4) re-index in Typesense. Entity can be re-suppressed after a lift by inserting a new `suppressed_entities` row (partial index allows this). **Path back from `suppressed` review_status:** admin manually sets `review_status → legal_review` and logs the action. Only admins can un-suppress cases. `suppressed → any` transition is admin-only, not pipeline-accessible.

### `moderation_actions`

Immutable audit log of every moderation decision. Required for legal defense.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `moderator_id` | uuid | FK → profiles (SET NULL) — nullable; null when action is from an automated agent |
| `agent_source` | text | nullable — `agent_a` · `agent_b` · `agent_c` · `agent_d` · `human` · `system` (automated cron/trigger jobs). Set when `moderator_id` is null. CHECK: `(moderator_id IS NOT NULL OR agent_source IS NOT NULL)` — at least one attribution required. Prevents orphaned audit records. |
| `entity_type` | text | NOT NULL |
| `entity_id` | uuid | NOT NULL |
| `action_type` | enum | NOT NULL — `approved` · `rejected` · `edited` · `deleted` · `restored` · `suppressed` · `unsuppressed` · `locked` · `unlocked` |
| `reason` | text | nullable |
| `metadata` | jsonb | nullable — before/after diffs, violation lists from Agent B/C, relevant context |
| `created_at` | timestamptz | NOT NULL DEFAULT now() — **no `updated_at`**: immutable audit log |

---

### `user_media_progress`

_Exception: no `created_at` — only tracks current state, not history._

| Field | Type | Notes |
|-------|------|-------|
| `user_id` | uuid | FK → profiles (ON DELETE CASCADE) |
| `media_type` | enum | `podcast_episode` · `tv_show` — `tv_show` covers all rows in `tv_shows` including documentaries |
| `media_id` | uuid | |
| `status` | enum | `completed` · `in_progress` · `want_to_consume` |
| `updated_at` | timestamptz | |

PK: `(user_id, media_type, media_id)`

---

## Required Indexes

> Postgres does NOT auto-create indexes on FK columns (unlike MySQL). All must be explicit.

| Table | Column(s) | Type | Reason |
|-------|-----------|------|--------|
| `cases` | `slug` | UNIQUE | URL lookups |
| `cases` | `status` | btree | Filter pages |
| `cases` | `published_at` | partial (`WHERE published_at IS NOT NULL`) | Published listings |
| `cases` | `legal_sensitivity` | btree | Admin review queue |
| `tags` | `slug` | UNIQUE | Tag page lookups |
| `people` | `slug` | UNIQUE | Person page lookups |
| `agencies` | `slug` | UNIQUE | Agency page lookups |
| `timeline_events` | `case_id` | btree | FK join |
| `evidence` | `case_id` | btree | FK join |
| `interviews` | `case_id` | btree | FK join |
| `case_faqs` | `case_id` | btree | FK join |
| `case_tags` | `tag_id` | btree | "Cases with this tag" reverse lookup |
| `case_people` | `case_id` | btree | People per case |
| `case_people` | `person_id` | btree | Cases per person (surrogate PK means neither gets a free index) |
| `case_podcast_episodes` | `episode_id` | btree | Episode → case reverse lookup |
| `case_news_articles` | `article_id` | btree | Article → case reverse lookup |
| `case_tv_shows` | `show_id` | btree | Show → case reverse lookup |
| `interview_people` | `person_id` | btree | Person → interview reverse lookup |
| `community_notes` | `case_id` | btree | Load notes per case |
| `community_notes` | `user_id` | btree | "My notes" page |
| `community_notes` | `deleted_at` | partial (`WHERE deleted_at IS NULL`) | Filter soft-deleted |
| `community_notes` | `parent_id` | partial (`WHERE parent_id IS NOT NULL`) | Thread reply loading |
| `community_corrections` | `case_id` | btree | Corrections per case |
| `community_corrections` | `status` | btree | Admin review queue |
| `content_reports` | `status` | btree | Admin review queue |
| `upvotes` | `(entity_type, entity_id)` | btree | Count / existence checks |
| `case_subreddits` | `case_id` | btree | Subreddits per case |
| `case_reddit_posts` | `case_id` | btree | Posts per case |
| `case_reddit_posts` | `post_id` | UNIQUE | Deduplication |
| `podcast_episodes` | `podcast_id` | btree | FK join |
| `podcast_episodes` | `episode_url` | UNIQUE | RSS deduplication |
| `podcasts` | `rss_url` | UNIQUE | Feed deduplication |
| `tv_shows` | `tmdb_id` | UNIQUE | TMDB deduplication |
| `tv_shows` | `imdb_id` | UNIQUE | IMDB deduplication |
| `news_articles` | `url` | UNIQUE | News deduplication |
| `sources` | `url` | UNIQUE | Citation deduplication |
| `news_articles` | `published_at` | btree | Recent articles listing |
| `slug_redirects` | `old_slug` | UNIQUE | Redirect lookups |
| `jurisdictions` | `state_code` | btree | State SEO pages |
| `cases` | `state_code` | btree | /state/california pages |
| `cases` | `year` | btree | /year/1994 pages |
| `cases` | `jurisdiction_id` | btree | FK join |
| `charges` | `disposition` | btree | Filter by outcome |
| `appeals` | `status` | partial (`WHERE status = 'pending'`) | Active appeals |
| `entity_sources` | `(entity_type, entity_id)` | btree | Sources for an entity |
| `case_series` | `slug` | UNIQUE | Series page lookups |
| `case_series_members` | `case_id` | btree | Series a case belongs to |
| `books` | `isbn` | UNIQUE | Deduplication |
| `case_books` | `book_id` | btree | Books reverse lookup |
| `content_takedown_requests` | `status` | btree | Open requests queue |
| `content_takedown_requests` | `response_deadline` | partial (`WHERE resolved_at IS NULL`) | Overdue SLA alerts |
| `suppressed_entities` | `(entity_type, entity_id)` | partial UNIQUE (`WHERE lifted_at IS NULL`) | One active suppression per entity; allows re-suppression after lift |
| `moderation_actions` | `(entity_type, entity_id)` | btree | Audit trail per entity |
| `moderation_actions` | `moderator_id` | btree | Actions per moderator |
| `case_agencies` | `agency_id` | btree | "Cases per agency" reverse lookup |
| `charges` | `case_person_id` | btree | FK join — primary access path |
| `charges` | `jurisdiction_id` | btree | FK join |
| `appeals` | `case_person_id` | btree | FK join — primary access path |
| `appeals` | `charge_id` | btree | FK join |
| `community_corrections` | `user_id` | btree | "My corrections" page |
| `content_reports` | `(entity_type, entity_id)` | btree | Reports on a specific entity |
| `case_reddit_posts` | `subreddit_id` | btree | Posts grouped by subreddit |
| `cases` | `review_status` | btree | Pipeline admin queue |
| `cases` | `deleted_at` | partial (`WHERE deleted_at IS NULL`) | Filter soft-deleted cases |

---

## Automation Paths

### Podcast Ingestion (RSS)
1. Fetch RSS for `auto_ingest = true` podcasts
2. Upsert `podcast_episodes` — deduplicate on `episode_url` UNIQUE
3. NLP/keyword match against `cases` → suggest `case_podcast_episodes`
4. Human review queue before publishing
5. `external_only = true` for Spotify-locked shows

### TV/Documentary Ingestion (TMDB + JustWatch)
- TMDB API → metadata, posters, seasons (deduplicate on `tmdb_id` UNIQUE)
- JustWatch API → streaming availability per region
- Manual case linking required (doc titles often don't name the case)

### Reddit Sync (weekly cron — OAuth required)
1. Register Reddit OAuth app (100 req/min free — do this before first sync)
2. `GET /search?q=CASE_NAME&type=sr` → subreddit discovery → upsert `case_subreddits`
3. `GET /search?q=CASE_NAME&sort=top&t=all` → top posts → upsert `case_reddit_posts` on `post_id` UNIQUE
4. Retention: keep top 50 posts per case by score; delete lower-ranked on each sync

### News Ingestion
- Google News RSS: `https://news.google.com/rss/search?q=CASE_NAME`
- Upsert on `url` UNIQUE

### Legal Review Skill (planned — required before any case goes live)
- Validates: no banned vocabulary (perpetrator, killer, murderer as labels), disclaimer present for `legal_sensitivity: high` cases, content warnings populated, `legal_status` on all persons, `status` accurately set
- Cases that fail review → flagged, `published_at` withheld

### Upvote Reconciliation (nightly cron)
```sql
UPDATE community_notes cn
SET upvote_count = (
  SELECT COUNT(*) FROM upvotes
  WHERE entity_type = 'community_notes' AND entity_id = cn.id
);
-- repeat for other upvotable entities
```

### SEO Content Generation (Top 20)
- Target per case: ~2,000 word body, 5+ timeline events, person stubs with correct `legal_status`, 3+ FAQs, 3+ media links
- Top 20 cases = automation + legal review test bed

---

## Top 20 Priority Cases

| # | Case | Status | Legal Sensitivity | Notes |
|---|------|--------|-------------------|-------|
| 1 | JonBenét Ramsey | `unsolved` | `high` | No charges ever filed. All persons = `no_charges_filed` or `person_of_interest` |
| 2 | Zodiac Killer | `unsolved` | `standard` | No named charged individuals |
| 3 | Gabby Petito | `closed_suspect_deceased` | `standard` | Brian Laundrie = `person_of_interest` — was named as POI, **never formally charged** with Petito's murder before his death. `alleged` requires formal charges; this is a POI case only. |
| 4 | OJ Simpson | `civil_resolution` | `elevated` | Simpson `acquitted` (criminal), `civil_resolution` (civil). Simpson deceased June 2024. Goldman/Brown estates protective. |
| 5 | Laci Peterson | `solved_convicted` | `standard` | Scott Peterson = `convicted`. Note: appealing — add note field. |
| 6 | Ted Bundy | `solved_convicted` | `standard` | `convicted`, deceased |
| 7 | BTK Killer | `solved_convicted` | `standard` | Dennis Rader = `convicted` (guilty plea) |
| 8 | Golden State Killer | `solved_convicted` | `standard` | Joseph DeAngelo = `convicted` (guilty plea) |
| 9 | Emmett Till | `solved_acquitted` | `elevated` | Civil rights touchstone — highest editorial bar. Till's killers = `acquitted`. Requires careful framing. |
| 10 | Caylee Anthony | `solved_acquitted` | `high` | Casey Anthony = `acquitted`. Never imply guilt. Actively litigious. |
| 11 | Tupac Shakur | `ongoing_trial` | `high` | Duane Davis = `alleged`, active proceedings. Community features locked. Human review required. |
| 12 | Biggie Smalls | `unsolved` | `standard` | All persons = `person_of_interest` |
| 13 | Trayvon Martin | `solved_acquitted` | `high` | George Zimmerman = `acquitted`. Actively litigious — has sued multiple media orgs. Prominent disclaimer required. |
| 14 | Chandra Levy | `unsolved` | `elevated` | Conviction vacated, no one currently convicted. Gary Condit sued multiple outlets. All persons = `no_charges_filed`. |
| 15 | Elizabeth Short (Black Dahlia) | `unsolved` | `standard` | All deceased, no named charged individuals |
| 16 | Natalee Holloway | `unsolved` | `elevated` | Van der Sloot = `person_of_interest` for this case (convicted of unrelated crime only) |
| 17 | Etan Patz | `solved_convicted` | `standard` | Pedro Hernandez = `convicted` |
| 18 | DB Cooper | `unsolved` | `standard` | No named charged individuals |
| 19 | Elisa Lam | `closed_no_crime` | `standard` | Ruled accidental death. No suspects. |
| 20 | Chris Watts | `solved_convicted` | `standard` | `convicted` (guilty plea) |

---

## Open Questions

- **Audio player**: Howler.js or native HTML5 `<audio>`. Spotify embed API for linked episodes.
- **Knowledge graph viz**: Deferred post-MVP. D3.js or Reagraph. Derive from `case_people`, `timeline_events`, `evidence` joins — no dedicated schema tables needed at v1.
- **Supabase RLS**: Anon read for published cases; auth required for community writes; `ongoing_trial` cases block all community writes.
- **`agencies.type` enum**: US-centric in v1 — known tech debt.
- **TMDB API key**: Register at themoviedb.org.
- **Legal review skill**: Build and validate against top 20 before any page goes live. See `docs/legal-safety-framework.md`.

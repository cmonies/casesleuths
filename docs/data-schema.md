# CaseSleuths — Data Schema

_v0.20 · 2026-02-27 — P0 fixes: replaced dynamic SQL in upvote trigger with static IF/ELSIF dispatch (eliminates SQL injection vector), added trigger ordering prefix (aaa_enforce_review_status_transition fires first). P1 fixes: elevated sensitivity advisory flag in publish gate, country_code + moderation_actions covering indexes added, case_podcast_episodes added to webhook list, people name/photo policy explicit NOT NULL DEFAULT, case_reddit_posts upsert conflict target documented, suppressed_entities scope + redundancy notes, entity_sources CHECK rationale note, case_people legal_status+role combo documentation. P2 fixes: case_agencies.role convention note, interview_people.role CHECK constraint, sources URL normalization note, media_progress_status enum parenthetical moved to note._

_v0.16 · 2026-02-28 — P0 fixes: added qa_review→legal_review transition, fixed upvote trigger GREATEST clamp. P1 fixes: dropped redundant case_id/person_id from charges/appeals (join through case_people only), moderation_actions at-least-one-non-null CHECK, Agent D stale pipeline + auto-suppress P0 policy, charges/appeals case_person_id indexes, sources.url UNIQUE, minimum source coverage publish gate, pipeline publish flow for standard/elevated cases. P2 fixes: charges_dropped in legal framework, suppressed_entities.entity_type CHECK, agent model specs, Typesense suppression note, pipeline diagram updated_

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

**Revalidation implementation:** Supabase DB webhooks → Next.js `/api/revalidate?secret=TOKEN&path=/cases/[slug]`. Webhooks required on: `cases`, `timeline_events`, `case_people`, `evidence`, `interviews`, `case_faqs`, `podcast_episodes`, `case_podcast_episodes`, `tv_shows`, `news_articles`, `case_subreddits`, `case_tags`, `related_cases`, `charges`, `appeals`, `case_agencies`, `books`, `case_books`, `jurisdictions`, `case_series`, `case_series_members`, `suppressed_entities` (suppressing a person or case must trigger immediate ISR revalidation — content must not remain visible after suppression). Note: `case_podcast_episodes` (the join table) must be included — it changes whenever an episode is linked to a case, which is the ISR-triggering event. `podcast_episodes` itself rarely changes after initial RSS ingestion.

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

**Typesense indexing + suppression:** All Typesense indexing jobs must use the safe views — never query `people`, `cases`, or `case_people` base tables directly. Routing:
- Case index: use `public_cases` view (filters suppressed + unpublished)
- Person name resolution in search snippets: use `public_case_people` view (case-scoped, full auto-policy resolution)
- Person details (admin/editorial only): use `public_people` view
Suppressed persons must not appear in search results. The `public_case_people` view handles suppression via LEFT JOIN on `suppressed_entities`. Typesense indexing jobs must also pass results through this view — not apply suppression logic independently.

**Suppression-safe views (define in migration before any RLS policies):**
```sql
-- public_people: masks suppressed persons and enforces name_display_policy
CREATE OR REPLACE VIEW public_people WITH (security_barrier = true) AS
SELECT
  p.id,
  CASE
    WHEN se.id IS NOT NULL THEN '[Name withheld]'
    WHEN p.name_display_policy IN ('redacted', 'requires_human_review') THEN '[Name withheld]'
    WHEN p.name_display_policy = 'initials_only' THEN
      array_to_string(ARRAY(
        SELECT left(word, 1) || '.'
        FROM unnest(string_to_array(p.name, ' ')) AS word WHERE word <> ''  -- guard: skip empty tokens from double spaces
      ), ' ')
    WHEN p.name_display_policy = 'auto' THEN '[Review required]'
    -- 'auto': cannot safely resolve without case context; use public_case_people instead
    WHEN p.name_display_policy = 'full' THEN p.name
    ELSE '[Name withheld]'
    -- ELSE is fail-closed: unknown/future policy values return '[Name withheld]' not the real name.
    -- Admin tools needing the real name must query base tables via service-role.
  END AS display_name, -- comma required: SQL SELECT list
  -- Note: public_people is ADMIN/EDITORIAL only. 'auto' returns '[Review required]' — admin tools
  -- should show the real name via a service-role query when needed for editorial review.
  CASE
    WHEN se.id IS NOT NULL THEN NULL
    WHEN p.photo_display_policy IN ('blocked', 'requires_review') THEN NULL
    -- INTENTIONAL DIVERGENCE from public_case_people:
    -- public_people is admin-only. For 'auto', we can't resolve without case context here,
    -- so we fail-closed: block photo until admin resolves the policy.
    -- public_case_people resolves 'auto' with case context and only blocks photo if
    -- name would actually be withheld (minor/charged-as-adult check). This is correct.
    -- An adult with 'auto' policy: photo visible in public (correct name shown),
    -- photo blocked in admin view (unresolved — admin must confirm policy). Intentional.
    WHEN p.name_display_policy IN ('auto', 'requires_human_review') THEN NULL
    WHEN p.photo_display_policy = 'allowed' THEN p.photo_url
    ELSE NULL  -- fail-closed: unknown future photo_display_policy values block photo
  END AS display_photo_url,
  p.slug,
  -- Suppress identifying fields when name is withheld — indirect identification via bio/dob/aliases
  -- is a real privacy risk. Admin tools needing full data must query base table via service-role.
  CASE WHEN se.id IS NOT NULL OR p.name_display_policy IN ('redacted','requires_human_review','initials_only')
       OR p.name_display_policy = 'auto'
       THEN NULL ELSE p.aliases END AS aliases,
  p.primary_known_role,
  CASE WHEN se.id IS NOT NULL OR p.name_display_policy IN ('redacted','requires_human_review','initials_only')
       OR p.name_display_policy = 'auto'
       THEN NULL ELSE p.dob END AS dob,
  CASE WHEN se.id IS NOT NULL OR p.name_display_policy IN ('redacted','requires_human_review','initials_only')
       OR p.name_display_policy = 'auto'
       THEN NULL ELSE p.dod END AS dod,
  CASE WHEN se.id IS NOT NULL OR p.name_display_policy IN ('redacted','requires_human_review','initials_only')
       OR p.name_display_policy = 'auto'
       THEN NULL ELSE p.bio END AS bio,
  p.is_living,
  -- Suppress minor flags for suppressed persons (combined with primary_known_role, is_living,
  -- these can narrow identification of a suppressed minor even without name/photo)
  CASE WHEN se.id IS NOT NULL THEN NULL ELSE p.is_minor_at_time_of_crime END AS is_minor_at_time_of_crime,
  CASE WHEN se.id IS NOT NULL THEN NULL ELSE p.is_minor_currently END AS is_minor_currently,
  p.name_display_policy,
  p.photo_display_policy,
  CASE WHEN se.id IS NOT NULL OR p.name_display_policy IN ('redacted','requires_human_review','initials_only')
       OR p.name_display_policy = 'auto'
       THEN NULL ELSE p.links END AS links,
  p.created_at,
  p.updated_at
FROM people p
LEFT JOIN suppressed_entities se
  ON se.entity_type = 'people' AND se.entity_id = p.id AND se.lifted_at IS NULL;
-- NOTE: public_people is ADMIN/EDITORIAL only. All identifying fields suppressed when name is withheld.
-- Admin tools needing full data on suppressed/review persons must use service-role queries on base table.

-- public_case_tombstones: returns suppressed cases with redacted fields for tombstone pages
-- Required for rendering "[Case removed]" pages at known URLs after takedowns.
-- public_cases EXCLUDES suppressed cases — if a route exists, the app must check
-- public_case_tombstones to serve a proper removal notice instead of a 404.
CREATE OR REPLACE VIEW public_case_tombstones WITH (security_barrier = true) AS
SELECT
  c.id,
  c.slug,
  '[Case removed]'::text AS title,
  NULL::text AS body,
  NULL::text AS summary,
  c.review_status,
  c.deleted_at,
  c.published_at,
  se.created_at AS suppressed_at
FROM cases c
JOIN suppressed_entities se
  ON se.entity_type = 'cases' AND se.entity_id = c.id AND se.lifted_at IS NULL
WHERE c.deleted_at IS NULL;
-- Usage: app layer checks public_case_tombstones before 404ing on a known slug.
-- Routing logic (both views are anon-accessible via RLS policies):
--   SELECT * FROM public_cases WHERE slug = $slug → found = 200 OK
--   SELECT * FROM public_cases WHERE slug = $slug → not found:
--     SELECT * FROM public_case_tombstones WHERE slug = $slug → found = 410 Gone
--     SELECT * FROM public_case_tombstones WHERE slug = $slug → not found = 404 Not Found
-- Anon never queries base `cases` table — routing is entirely via these two views.
-- Note: slug_redirects handles renamed slugs; check slug_redirects BEFORE tombstones
-- if you want clean redirect chains (old_slug → new_slug before checking suppression).

-- public_cases: hides soft-deleted and suppressed cases from public queries
-- KNOWN LIMITATION: cases.body/title/summary/seo_* may still name suppressed persons or
-- persons whose name should be withheld. DB cannot scan free-text for names at publish time.
-- Mitigation: Agent B/D must scan body text for suppressed person names before publish,
-- and re-scan when a person is suppressed post-publish (triggering ISR revalidation).
-- This is a product-layer responsibility, not a schema constraint. Accept the limitation.
CREATE OR REPLACE VIEW public_cases WITH (security_barrier = true) AS
SELECT c.*
FROM cases c
LEFT JOIN suppressed_entities se
  ON se.entity_type = 'cases' AND se.entity_id = c.id AND se.lifted_at IS NULL
WHERE c.deleted_at IS NULL
  AND se.id IS NULL
  AND c.review_status = 'published';
```
All views must be created with `WITH (security_barrier = true)` to prevent predicate pushdown attacks.

**Required RLS policies (migration must include these):**

**Design:** anon/public clients query ONLY via views (which apply security_barrier + masking). Direct base table reads by anon are DENIED — no policies on base tables that allow anon SELECT. Service role bypasses RLS (used by agents + server-side rendering).

```sql
-- Enable RLS on all sensitive tables — deny by default (no policy = no access)
ALTER TABLE people ENABLE ROW LEVEL SECURITY;
ALTER TABLE cases ENABLE ROW LEVEL SECURITY;
ALTER TABLE case_people ENABLE ROW LEVEL SECURITY;
ALTER TABLE suppressed_entities ENABLE ROW LEVEL SECURITY;

-- NO anon SELECT policies on base tables — views handle masking
-- Anon access is via Supabase auto-generated APIs on the views themselves,
-- OR via server-side rendering with service_role (which bypasses RLS).

-- Authenticated users (editors): can read their own moderation_actions, reports
CREATE POLICY "auth_read_own_reports" ON content_reports FOR SELECT TO authenticated
  USING (reporter_id = auth.uid());

-- Service role: bypasses all RLS automatically (Supabase service_role key)
-- Use service_role for: all agent writes, admin dashboard, SSR data fetching

-- Upvote write: authenticated users only, one per entity
CREATE POLICY "auth_insert_upvote" ON upvotes FOR INSERT TO authenticated
  WITH CHECK (user_id = auth.uid());
CREATE POLICY "auth_delete_own_upvote" ON upvotes FOR DELETE TO authenticated
  USING (user_id = auth.uid());
```
-- REQUIRED GRANTS: anon must be able to SELECT from public views (views don't inherit
-- base table grants; explicit GRANT is required in Supabase for anon reads to work)
GRANT SELECT ON public_cases TO anon, authenticated;
GRANT SELECT ON public_case_people TO anon, authenticated;
GRANT SELECT ON public_case_tombstones TO anon, authenticated;
-- public_people is ADMIN/EDITORIAL only — do NOT grant to anon
GRANT SELECT ON public_people TO authenticated;  -- editorial/admin tools only

-- REQUIRED: admin-table RLS policies — authenticated editorial/admin users need access
-- to suppressed_entities and moderation_actions for dashboard tooling.
-- Without these, authenticated (non-service-role) admin users are hard-blocked.
-- Note: service_role always bypasses RLS (for agents + server-side code).
CREATE POLICY "admin_read_suppressed_entities" ON suppressed_entities FOR SELECT TO authenticated
  USING (auth.jwt() ->> 'role' IN ('admin', 'editorial'));
-- Split per-command: FOR ALL with USING only ignores USING on INSERT, allowing non-admin inserts
CREATE POLICY "admin_insert_suppressed_entities" ON suppressed_entities FOR INSERT TO authenticated
  WITH CHECK (auth.jwt() ->> 'role' IN ('admin', 'editorial'));
CREATE POLICY "admin_update_suppressed_entities" ON suppressed_entities FOR UPDATE TO authenticated
  USING (auth.jwt() ->> 'role' IN ('admin', 'editorial'))
  WITH CHECK (auth.jwt() ->> 'role' IN ('admin', 'editorial'));
CREATE POLICY "admin_delete_suppressed_entities" ON suppressed_entities FOR DELETE TO authenticated
  USING (auth.jwt() ->> 'role' IN ('admin', 'editorial'));
CREATE POLICY "admin_read_moderation_actions" ON moderation_actions FOR SELECT TO authenticated
  USING (auth.jwt() ->> 'role' IN ('admin', 'editorial'));
CREATE POLICY "admin_write_moderation_actions" ON moderation_actions FOR INSERT TO authenticated
  WITH CHECK (auth.jwt() ->> 'role' IN ('admin', 'editorial'));
-- Note: 'role' claim must be set in Supabase auth.users metadata and included in JWT.
-- The migration must configure this claim; it is not automatic.

-- REQUIRED: upvotes immutability — prevent count drift from UPDATE operations
CREATE OR REPLACE FUNCTION block_upvote_update() RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'upvotes rows are immutable — delete and re-insert instead of updating';
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER block_upvote_update BEFORE UPDATE ON upvotes
--   FOR EACH ROW EXECUTE FUNCTION block_upvote_update();

-- REQUIRED: ongoing_trial community lock — these policies MUST be in the migration
-- community_notes INSERT: block on ongoing_trial cases
CREATE POLICY "block_ongoing_trial_notes" ON community_notes FOR INSERT TO authenticated
  WITH CHECK (
    NOT EXISTS (
      SELECT 1 FROM cases WHERE id = community_notes.case_id AND status = 'ongoing_trial'
    )
  );
-- community_corrections INSERT: same gate
CREATE POLICY "block_ongoing_trial_corrections" ON community_corrections FOR INSERT TO authenticated
  WITH CHECK (
    NOT EXISTS (
      SELECT 1 FROM cases WHERE id = community_corrections.case_id AND status = 'ongoing_trial'
    )
  );
-- upvotes INSERT: block on ongoing_trial cases (entity_type='community_notes' only in MVP;
-- join through community_notes.case_id to reach cases.status)
CREATE POLICY "block_ongoing_trial_upvotes" ON upvotes FOR INSERT TO authenticated
  WITH CHECK (
    NOT EXISTS (
      SELECT 1 FROM community_notes cn
      JOIN cases c ON c.id = cn.case_id
      WHERE cn.id = upvotes.entity_id
        AND upvotes.entity_type = 'community_notes'
        AND c.status = 'ongoing_trial'
    )
  );
-- UPDATE policies: also block editing community_notes/corrections on ongoing_trial cases
-- (INSERT-only lock is insufficient — existing notes could be edited to add content)
CREATE POLICY "block_ongoing_trial_notes_update" ON community_notes FOR UPDATE TO authenticated
  USING (
    NOT EXISTS (SELECT 1 FROM cases WHERE id = community_notes.case_id AND status = 'ongoing_trial')
  );
CREATE POLICY "block_ongoing_trial_corrections_update" ON community_corrections FOR UPDATE TO authenticated
  USING (
    NOT EXISTS (SELECT 1 FROM cases WHERE id = community_corrections.case_id AND status = 'ongoing_trial')
  );
-- DELETE policies: also block deletion (removing content during active trial = mutation)
CREATE POLICY "block_ongoing_trial_notes_delete" ON community_notes FOR DELETE TO authenticated
  USING (
    NOT EXISTS (SELECT 1 FROM cases WHERE id = community_notes.case_id AND status = 'ongoing_trial')
  );
CREATE POLICY "block_ongoing_trial_corrections_delete" ON community_corrections FOR DELETE TO authenticated
  USING (
    NOT EXISTS (SELECT 1 FROM cases WHERE id = community_corrections.case_id AND status = 'ongoing_trial')
  );
CREATE POLICY "block_ongoing_trial_upvotes_delete" ON upvotes FOR DELETE TO authenticated
  USING (
    NOT EXISTS (
      SELECT 1 FROM community_notes cn
      JOIN cases c ON c.id = cn.case_id
      WHERE cn.id = upvotes.entity_id
        AND upvotes.entity_type = 'community_notes'
        AND c.status = 'ongoing_trial'
    )
  );

Note: The above ongoing_trial lock policies are REQUIRED in the migration — this is an explicit legal safety rule. The remaining full RLS policy set (auth reads for community tables, content_reports) is written in the migration file. The key rule is NO direct anon access to base tables.

**`public_people` is ADMIN/EDITORIAL USE ONLY — not for public pages.** It returns `'[Review required]'` for `auto` policy persons, which would create confusing UX if used on public-facing routes. Public search, Typesense indexing, and all user-visible pages MUST use `public_case_people` (case-scoped view below). `public_people` is safe for admin dashboards and editorial review tools only.

**`name_display_policy: auto` limitation:** `public_people` resolves `full`, `initials_only`, and `redacted` policies. For `auto`, it returns `'[Review required]'` as a safe sentinel (no name leak). `auto` requires case context (`case_people.legal_status`) to evaluate correctly; use `public_case_people` for full auto-resolution.

```sql
-- public_case_people: case-scoped person view that resolves 'auto' name_display_policy
-- Use for Typesense indexing and any query that renders person names in case context
CREATE OR REPLACE VIEW public_case_people WITH (security_barrier = true) AS
SELECT
  cp.id AS case_person_id,
  cp.case_id,
  p.id AS person_id,
  CASE
    WHEN se_person.id IS NOT NULL THEN '[Name withheld]'
    WHEN p.name_display_policy IN ('redacted', 'requires_human_review') THEN '[Name withheld]'
    WHEN p.name_display_policy = 'initials_only' THEN
      array_to_string(ARRAY(
        SELECT left(word, 1) || '.'
        FROM unnest(string_to_array(p.name, ' ')) AS word WHERE word <> ''  -- guard: skip empty tokens from double spaces
      ), ' ')
    WHEN p.name_display_policy = 'auto' THEN
      CASE
        -- 1. Minor charged as adult → full name (check FIRST — overrides all other minor rules)
        WHEN p.is_minor_at_time_of_crime = true AND cp.charged_as_adult IS TRUE THEN p.name
        -- 2. Currently a minor and not convicted/acquitted/exonerated → withhold name
        WHEN p.is_minor_currently = true
             AND (cp.legal_status IS NULL OR cp.legal_status NOT IN ('convicted','acquitted','exonerated'))
             THEN '[Name withheld]'
        -- 3. Minor at time + formally charged (alleged) but NOT as adult → initials only
        WHEN p.is_minor_at_time_of_crime = true AND cp.legal_status = 'alleged' THEN
          array_to_string(ARRAY(
            SELECT left(word, 1) || '.'
            FROM unnest(string_to_array(p.name, ' ')) AS word WHERE word <> ''  -- guard: skip empty tokens from double spaces
          ), ' ')
        -- 4. Minor at time + POI/no charges/witness → initials only
        WHEN p.is_minor_at_time_of_crime = true
             AND (cp.legal_status IN ('person_of_interest','no_charges_filed') OR cp.case_role = 'witness')
             THEN
          array_to_string(ARRAY(
            SELECT left(word, 1) || '.'
            FROM unnest(string_to_array(p.name, ' ')) AS word WHERE word <> ''  -- guard: skip empty tokens from double spaces
          ), ' ')
        -- 5. Aged-out minor (was minor at time, now adult, still living) → human review required
        WHEN p.is_minor_at_time_of_crime = true
             AND p.is_minor_currently = false
             AND p.is_living = true
             THEN '[Review required]'
        -- 6. Fallthrough (adult, deceased, etc.) → full name
        ELSE p.name
      END
    -- Fail-closed: unknown policy values → withhold (never fail-open on a privacy-critical view)
    ELSE '[Name withheld]' 
  END AS display_name,
  CASE
    WHEN se_person.id IS NOT NULL THEN NULL
    WHEN p.photo_display_policy IN ('blocked', 'requires_review') THEN NULL
    WHEN p.name_display_policy IN ('redacted', 'requires_human_review') THEN NULL
    -- initials_only = name is intentionally obscured; photo must also be blocked
    -- (Minor POI rule: initials only → no photo. Consistent with privacy policy table.)
    WHEN p.name_display_policy = 'initials_only' THEN NULL
    -- Block photo whenever name is being withheld by auto-policy (minor protection)
    WHEN p.name_display_policy = 'auto' AND (
      p.is_minor_currently IS TRUE
      OR (p.is_minor_at_time_of_crime IS TRUE AND cp.charged_as_adult IS NOT TRUE)
    ) THEN NULL
    WHEN p.photo_display_policy = 'allowed' THEN p.photo_url
    ELSE NULL  -- fail-closed: unknown photo_display_policy values block photo
  END AS display_photo_url,
  cp.legal_status,
  cp.case_role,
  -- cp.notes EXCLUDED: editorial/internal notes; may contain unvetted allegations.
  -- Not safe for public-facing views (platform-owned publication surface risk).
  -- Admin tools must query case_people base table via service-role to access notes.
  cp.is_primary,
  p.slug,
  p.primary_known_role,
  p.is_living,
  p.is_minor_at_time_of_crime,
  p.is_minor_currently,
  p.name_display_policy,
  p.photo_display_policy
FROM case_people cp
JOIN people p ON p.id = cp.person_id
-- Only include people on published, non-deleted, non-suppressed cases
JOIN cases c ON c.id = cp.case_id
  AND c.review_status = 'published'
  AND c.deleted_at IS NULL
LEFT JOIN suppressed_entities se_person
  ON se_person.entity_type = 'people' AND se_person.entity_id = p.id AND se_person.lifted_at IS NULL
LEFT JOIN suppressed_entities se_case
  ON se_case.entity_type = 'cases' AND se_case.entity_id = c.id AND se_case.lifted_at IS NULL
WHERE se_case.id IS NULL; -- exclude suppressed cases entirely
```


**Indexing rule:** Typesense jobs use `public_case_people` for person name resolution in search. `public_people` is for admin/editorial UIs that need the full person record without case context.

**Drift insurance:** Nightly reconciliation cron recomputes all `upvote_count` values from the `upvotes` table directly — cheap insurance against trigger edge cases (manual SQL, pg_restore, bulk imports).

⚠️ **All `-- CREATE TRIGGER` statements below are REQUIRED in the migration.** They are commented out in this spec to prevent confusion with Postgres syntax highlighting, but every one must be uncommented and executed in the migration SQL. Missing any trigger silently breaks the corresponding safety guarantee (source coverage, high-sensitivity gate, insert guard, etc.). See migration order note at top of Enum Definitions section.

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
--   jurisdictions, case_series, books, content_takedown_requests, sources,
--   user_media_progress (has updated_at; no created_at — see exception note below)
-- Intentional exceptions (NO updated_at trigger):
--   case_reddit_posts → uses fetched_at instead (scores updated in-place on each sync)
--   All pure join tables (case_tags, case_agencies, case_books, etc.) → no timestamps

-- Upvote count trigger (static dispatch — no dynamic SQL, no injection risk)
-- Future: add ELSIF branches here when upvotes.entity_type CHECK is extended to new tables.
CREATE OR REPLACE FUNCTION update_upvote_count() RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    IF NEW.entity_type = 'community_notes' THEN
      UPDATE community_notes SET upvote_count = GREATEST(upvote_count + 1, 0) WHERE id = NEW.entity_id;
      -- GREATEST on INSERT: defends against corrupt negative baseline (bad import, manual SQL)
    -- community_corrections is NOT in MVP upvote scope — no upvote_count column on that table yet.
    -- To extend: (1) add upvote_count to community_corrections, (2) add to entity_type CHECK,
    -- (3) add ELSIF branch here for both INSERT and DELETE cases.
    END IF;
  ELSIF TG_OP = 'DELETE' THEN
    IF OLD.entity_type = 'community_notes' THEN
      -- GREATEST prevents negative counts on duplicate deletes or manual cleanup
      UPDATE community_notes SET upvote_count = GREATEST(upvote_count - 1, 0) WHERE id = OLD.entity_id;
    END IF;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;
-- Upvote soft-delete strategy: do NOT delete upvote rows on soft delete.
-- Deleting rows would fire update_upvote_count AFTER DELETE, decrementing count to 0,
-- directly violating "Do NOT zero upvote_count on soft delete" policy.
-- Correct approach: rely on RLS to block new votes (WHERE deleted_at IS NULL),
-- and preserve existing upvote rows + upvote_count for historical signal.
-- Orphan cleanup: upvote rows for soft-deleted entities remain in DB. Agent D nightly
-- reconciliation cron verifies counts; upvotes for hard-deleted entities are cleaned
-- up via app-layer DELETE before hard delete (hard deletes are rare).

-- Soft-delete guard: RLS on upvotes prevents new inserts when parent deleted_at IS NOT NULL.
-- Service-role bypass: if a service-role client inserts an upvote for a soft-deleted entity,
-- the trigger will still fire. Accept this edge case at MVP; mitigate at scale by adding
-- a guard in the trigger body: check parent deleted_at before updating count.
-- CREATE TRIGGER update_upvote_count AFTER INSERT OR DELETE ON upvotes
--   FOR EACH ROW EXECUTE FUNCTION update_upvote_count();
```

```sql
-- legal_sensitivity publish gate trigger
-- Distinction: 'high' = HARD BLOCK (no auto-publish without human approval).
--              'elevated' = ADVISORY FLAG (auto-publish allowed, but human should review).
--              'standard' = no gate.
-- MIGRATION DEPENDENCY: this function must be created AFTER the moderation_actions table exists.
-- Migration order: CREATE TABLE moderation_actions → CREATE FUNCTION block_high_sensitivity_autopublish
-- (plpgsql resolves table names at runtime, not compile time, but the SELECT inside will error
-- on first publish attempt if moderation_actions doesn't exist — so enforce order explicitly.)
CREATE OR REPLACE FUNCTION block_high_sensitivity_autopublish() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.review_status = 'published'
     AND OLD.review_status != 'published' THEN

    IF NEW.legal_sensitivity = 'high' THEN
      -- HARD BLOCK: high-sensitivity cases require a human-authored 'approved' record.
      -- IMPORTANT: the state machine trigger logs action_type='edited' for all transitions.
      -- App layer MUST create a separate moderation_actions row with action_type='approved',
      -- agent_source='human' before setting review_status='published' on high-sensitivity cases.
      -- This is a two-step process: (1) human creates approved record, (2) system sets published.
      IF NOT EXISTS (
        SELECT 1 FROM moderation_actions
        WHERE entity_type = 'cases'
          AND entity_id = NEW.id
          AND action_type = 'approved'
          AND agent_source = 'human'
      ) THEN
        RAISE EXCEPTION 'Cannot auto-publish case % with legal_sensitivity=high: human approval required', NEW.id;
      END IF;

    ELSIF NEW.legal_sensitivity = 'elevated' THEN
      -- ADVISORY FLAG: elevated cases may auto-publish, but Agent B must flag for human review.
      -- Log a WARNING to moderation_actions and allow publish to proceed.
      -- 'elevated' = advisory, not a hard stop. Human should review but system will not block.
      INSERT INTO moderation_actions (entity_type, entity_id, action_type, agent_source, metadata)
      VALUES ('cases', NEW.id, 'flagged', 'system',
        jsonb_build_object(
          'reason', 'elevated_sensitivity_autopublish',
          'message', 'Case published with legal_sensitivity=elevated. Human review recommended.'
        ));
    END IF;

  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER bbb_block_high_sensitivity_autopublish BEFORE UPDATE ON cases
--   FOR EACH ROW EXECUTE FUNCTION block_high_sensitivity_autopublish();

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
    -- Case must have at least one timeline event before publishing
    IF NOT EXISTS (SELECT 1 FROM timeline_events WHERE case_id = NEW.id) THEN
      RAISE EXCEPTION 'Cannot publish case %: no timeline_events exist', NEW.id;
    END IF;
    -- Every timeline_event must have at least 1 source
    -- Note: vacuous satisfaction (no timeline_events) is now blocked by the check above
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
-- CREATE TRIGGER ddd_check_source_coverage BEFORE UPDATE ON cases
--   FOR EACH ROW EXECUTE FUNCTION check_source_coverage();

-- cases DELETE block: cases are never hard-deleted; enforces soft-delete-only policy
CREATE OR REPLACE FUNCTION block_case_hard_delete() RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Hard delete on cases is forbidden. Set deleted_at instead. Case id: %', OLD.id;
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER block_case_hard_delete BEFORE DELETE ON cases
--   FOR EACH ROW EXECUTE FUNCTION block_case_hard_delete();

-- cases INSERT guard: new cases must start as 'draft'; blocks INSERT bypass of pipeline
CREATE OR REPLACE FUNCTION enforce_cases_insert_draft() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.review_status IS DISTINCT FROM 'draft' THEN
    RAISE EXCEPTION 'New cases must be inserted with review_status = draft; got: %', NEW.review_status;
  END IF;
  IF NEW.published_at IS NOT NULL THEN
    RAISE EXCEPTION 'New cases cannot have published_at set on INSERT';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER enforce_cases_insert_draft BEFORE INSERT ON cases
--   FOR EACH ROW EXECUTE FUNCTION enforce_cases_insert_draft();

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
  -- agent_source contract: callers MUST SET LOCAL app.agent_source = 'agent_a' (or valid value)
  -- before updating review_status. Trigger defaults to 'system' if not set.
  -- Valid values: 'agent_a','agent_b','agent_c','agent_d','human','system'
  INSERT INTO moderation_actions (entity_type, entity_id, action_type, agent_source, metadata)
  VALUES ('cases', NEW.id, 'edited',
    COALESCE(NULLIF(current_setting('app.agent_source', true), ''), 'system'),
    jsonb_build_object('field', 'review_status', 'from', OLD.review_status, 'to', NEW.review_status));
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- TRIGGER ORDERING NOTE: Postgres fires BEFORE UPDATE triggers alphabetically by trigger name.
-- The four BEFORE UPDATE triggers on `cases` must fire in this order:
--   1. enforce_review_status_transition → prefix: aaa_ (cheapest gate, validate transition first)
--   2. block_high_sensitivity_autopublish → prefix: bbb_ (check moderation_actions)
--   3. check_living_person_disclaimer → prefix: ccc_ (scan case_people)
--   4. check_source_coverage → prefix: ddd_ (scan entity_sources + timeline_events — most expensive)
-- Trigger names below MUST use these prefixes. Do not rename without adjusting all four.
-- CREATE TRIGGER aaa_enforce_review_status_transition BEFORE UPDATE ON cases
--   FOR EACH ROW WHEN (OLD.review_status IS DISTINCT FROM NEW.review_status)
--   EXECUTE FUNCTION enforce_review_status_transition();

-- Disclaimer gate: block publishing if any living unconvicted person is linked without disclaimer
CREATE OR REPLACE FUNCTION check_living_person_disclaimer() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.review_status = 'published' AND OLD.review_status IS DISTINCT FROM 'published' THEN
    IF EXISTS (
      SELECT 1 FROM case_people cp
      JOIN people p ON p.id = cp.person_id
      WHERE cp.case_id = NEW.id
        AND p.is_living = true
        -- Exclude purely institutional roles only: detectives, attorneys, judges act in official
        -- institutional capacity. Their involvement is public record; defamation risk is minimal.
        -- Witnesses ARE included — they are private individuals whose reputation can be harmed by
        -- association with a case. A living witness with no conviction needs the same disclaimer
        -- protection as any other living unconvicted person.
        -- Victims with NULL legal_status ARE also checked (framing can be defamatory).
        AND cp.case_role NOT IN ('detective', 'attorney', 'judge')
        AND (cp.legal_status IS NULL OR cp.legal_status NOT IN ('convicted','acquitted','exonerated'))
    ) AND (NEW.disclaimer IS NULL OR trim(NEW.disclaimer) = '') THEN
      RAISE EXCEPTION
        'Cannot publish case %: living unconvicted person(s) linked but disclaimer is empty. '
        'Set cases.disclaimer before publishing.', NEW.id;
    END IF;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER ccc_check_living_person_disclaimer BEFORE UPDATE ON cases
--   FOR EACH ROW EXECUTE FUNCTION check_living_person_disclaimer();

-- community_notes soft-delete: null out parent_id on children when parent is soft-deleted
-- (ON DELETE SET NULL only fires on hard DELETE; soft delete needs explicit trigger)
CREATE OR REPLACE FUNCTION null_parent_on_note_soft_delete() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.deleted_at IS NOT NULL AND OLD.deleted_at IS NULL THEN
    -- Null out parent_id on all existing replies atomically within the same transaction.
    -- Race condition mitigation: new replies to a being-deleted parent are prevented by
    -- the FK constraint (parent row still exists) PLUS application-layer checks on deleted_at.
    -- App layer must refuse new replies to notes where deleted_at IS NOT NULL before insert.
    -- This trigger handles existing replies at the moment of soft-delete.
    UPDATE community_notes SET parent_id = NULL WHERE parent_id = NEW.id;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER null_parent_on_note_soft_delete AFTER UPDATE ON community_notes
--   FOR EACH ROW WHEN (OLD.deleted_at IS DISTINCT FROM NEW.deleted_at)
--   EXECUTE FUNCTION null_parent_on_note_soft_delete();

-- slug_redirects INSERT trigger: auto-populate new_slug from cases.slug on INSERT
-- Eliminates app-layer initialization requirement (was easy to miss)
CREATE OR REPLACE FUNCTION init_slug_redirect_new_slug() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.new_slug IS NULL THEN
    SELECT slug INTO NEW.new_slug FROM cases WHERE id = NEW.new_case_id;
    IF NEW.new_slug IS NULL THEN
      RAISE EXCEPTION 'slug_redirects.new_case_id % not found or has no slug', NEW.new_case_id;
    END IF;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- CREATE TRIGGER init_slug_redirect_new_slug BEFORE INSERT ON slug_redirects
--   FOR EACH ROW EXECUTE FUNCTION init_slug_redirect_new_slug();

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

**Hot-row contention note:** At high traffic, constant `upvote_count` increments on a single row can become a write bottleneck. Mitigation at scale: sharded counter tables or batched aggregation. For MVP (top 20 cases), trigger approach is fine.

**Client implementation note:** Supabase Realtime channels must be explicitly unsubscribed when the user navigates away from a case page. Failing to do so causes memory leaks as channels accumulate. Scope each subscription to the page lifecycle (mount → subscribe, unmount → unsubscribe).

---

### FK Cascade Policy

**Default: `ON DELETE CASCADE` for all child rows** (timeline_events, case_people, evidence, etc. cascade when their parent case row is deleted). Pure join tables always cascade. **Exception:** `people` and `agencies` use `ON DELETE RESTRICT`. **Note on `cases` hard-delete:** the `block_case_hard_delete` trigger prevents any DELETE on the `cases` table — child cascades therefore can never fire in practice. They exist in the schema for DB integrity consistency. If the `block_case_hard_delete` trigger is ever removed, cascades will activate and all child rows (timeline_events, case_people, charges, appeals, evidence, etc.) will be deleted. The operational path to removing cases is always soft-delete via `deleted_at` — never a hard DELETE. |

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

**Person controlled vocabulary — two separate enums (v0.16 split):**

`case_people.case_role` (`case_role` enum) — functional role in the case:

| Value | When to use |
|-------|-------------|
| `victim` | Victim of the crime |
| `witness` | Witness to events |
| `detective` | Investigator on the case |
| `attorney` | Defense, prosecution, or civil attorney |
| `judge` | Presiding judge |
| `other` | Any other functional role |

`case_people.legal_status` (`person_legal_status` enum) — legal standing; **nullable** for role-only persons:

| Value | When to use |
|-------|-------------|
| `alleged` | Formally charged, trial pending |
| `convicted` | Conviction currently stands (note if appealing) |
| `acquitted` | Charged, found not guilty |
| `exonerated` | Previously convicted, conviction overturned |
| `person_of_interest` | Named by investigators, not formally charged |
| `charges_dropped` | Was formally charged; charges subsequently dropped or dismissed before trial |
| `no_charges_filed` | Named in media or podcasts, never formally accused |

**Rule:** `legal_status` is NULL for persons whose only relationship is a functional role (detective, attorney, judge, victim with no charges). Victims who were also charged (e.g. self-defense cases) can have both a `case_role = 'victim'` and a `legal_status = 'alleged'`.

---

### Enum Definitions

> All enums must be created with `CREATE TYPE ... AS ENUM (...)` before any table that references them. Supabase migration order: enums → tables → FKs → triggers → RLS policies → indexes.

| Enum name | Values |
|-----------|--------|
| `case_status` | `unsolved` · `solved_convicted` · `solved_acquitted` · `solved_exonerated` · `ongoing_trial` · `cold_case` · `civil_resolution` · `closed_no_crime` · `closed_suspect_deceased` |
| `legal_sensitivity` | `standard` · `elevated` · `high` |
| `review_status` | `draft` · `legal_review` · `qa_review` · `approved` · `published` · `rejected` · `suppressed` |
| `case_role` | `victim` · `witness` · `detective` · `attorney` · `judge` · `other` — functional role in the case (who they ARE in relation to it) |
| `person_legal_status` | `alleged` · `convicted` · `acquitted` · `exonerated` · `charges_dropped` · `person_of_interest` · `no_charges_filed` — legal standing (what the legal system has determined). Nullable: `NULL` for persons whose only relationship is a functional role (detective, attorney, judge, victim with no charges). |
| `name_display_policy` | `full` · `initials_only` · `redacted` · `auto` · `requires_human_review` |
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
| `media_progress_status` | `completed` · `in_progress` · `want_to_consume` — **canonical values**. Note (v0.20): earlier drafts used `watched`/`listening`/`want_to_watch` — these are superseded. Any migration from old values must map to these three. |
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
| `moderation_action_type` | `approved` · `rejected` · `edited` · `deleted` · `restored` · `suppressed` · `unsuppressed` · `locked` · `unlocked` · `flagged` — `flagged` is used by Agent D to log compliance alerts (e.g. minor_aged_out, is_living/dod mismatch) that require human review before action is taken |
| `agency_type` | `local_pd` · `sheriff` · `fbi` · `da` · `state_police` · `other` |

### Agent Model Specs

| Agent | Role | Recommended Model | Notes |
|-------|------|-------------------|-------|
| Agent A | Researcher / content seeder | `claude-sonnet` | Broad web research, structured data extraction |
| Agent B | Legal reviewer | `claude-opus` (minimum) | Defamation defense gate — high accuracy required, especially for `legal_sensitivity: high` cases |
| Agent C | QA / editorial reviewer | `claude-sonnet` | Framing and completeness check |
| Agent D | Compliance monitor | `claude-haiku` for SQL/pattern checks, `claude-sonnet` for random sample legal framing audit | Haiku for structured DB queries; upgrade to Sonnet for the 5-case/night framing scan |

---

**Controlled text fields** (stored as `text` or `text[]` with DB-level CHECK constraints):
These are enforced at the Postgres level — they are NOT soft guidance. The migration SQL must
include these as `ADD CONSTRAINT` statements on the tables specified below:

```sql
-- cases.content_warnings (text[]) — applied on CREATE TABLE cases
ALTER TABLE cases ADD CONSTRAINT cases_content_warnings_check
  CHECK (content_warnings <@ ARRAY['child_victim','sexual_violence','infant_victim',
    'family_violence','mass_casualty','graphic_imagery_risk']::text[]);

-- cases.primary_weapon (text, nullable) — applied on CREATE TABLE cases
ALTER TABLE cases ADD CONSTRAINT cases_primary_weapon_check
  CHECK (primary_weapon IN ('firearm','knife','blunt_object','poison',
    'strangulation','unknown','other'));

-- entity_sources.entity_type (text) — applied on CREATE TABLE entity_sources
ALTER TABLE entity_sources ADD CONSTRAINT entity_sources_entity_type_check
  CHECK (entity_type IN ('cases','people','timeline_events','evidence','charges','appeals'));
  -- INTENTIONALLY excludes community_notes, community_corrections, content_reports, moderation_actions.
  -- Rationale: entity_sources is the citation trail for factual claims made BY the platform.
  -- community_notes/corrections are user-generated and cite existing sources — they don't GET cited.
  -- See rationale note in entity_sources table definition for full explanation.
  -- Do not extend without evaluating whether the new entity type makes platform-owned factual claims.

-- people.is_living consistency with dod — applied on CREATE TABLE people
ALTER TABLE people ADD CONSTRAINT people_is_living_dod_check
  CHECK (dod IS NULL OR is_living = false);
  -- IS NOT TRUE would allow NULL (unknown living status) with a dod set, which is contradictory.
  -- is_living = false is the correct constraint: person with dod must be marked not-living.
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
| `slug` | text | NOT NULL, UNIQUE, URL-safe. CHECK (`slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'`) — lowercase alphanumeric, hyphens only, no leading/trailing hyphens. UNIQUE is case-sensitive in Postgres (fine — slugs are always lowercase by this CHECK). |
| `title` | text | NOT NULL |
| `alternate_names` | text[] | NOT NULL DEFAULT '{}' |
| `status` | enum | NOT NULL — `unsolved` · `solved_convicted` · `solved_acquitted` · `solved_exonerated` · `ongoing_trial` · `cold_case` · `civil_resolution` · `closed_no_crime` · `closed_suspect_deceased` |
| `legal_sensitivity` | enum | NOT NULL DEFAULT `standard` — `high` blocks auto-publish |
| `content_warnings` | text[] | NOT NULL DEFAULT '{}' — Controlled list: `child_victim` · `sexual_violence` · `infant_victim` · `family_violence` · `mass_casualty` · `graphic_imagery_risk` |
| `jurisdiction_id` | uuid | FK → jurisdictions (SET NULL) — nullable |
| `location_city` | text | |
| `location_state` | text | **Display field** — full state name in English, title case e.g. `California`. Not used for filtering. Normalized value expected; no CHECK (too complex). |
| `location_country` | text | **Display field** — full country name in English, title case e.g. `United States`. Not used for filtering. Use standard English country names (not native-language names). |
| `state_code` | text | **Filter/SEO field** — 2-letter US state code e.g. `CA`. CHECK (`state_code ~ '^[A-Z]{2}$'`) when NOT NULL. Used for /state/california pages. NULL for non-US cases. |
| `country_code` | text | NOT NULL default `US` — **Filter/SEO field** — ISO 3166-1 alpha-2 e.g. `US`, `GB`. CHECK (`country_code ~ '^[A-Z]{2}$'`). Used for international filtering. CHECK: `(country_code = 'US' OR state_code IS NULL)` — enforces that non-US cases cannot have a `state_code` set. |
| `location_coords` | point | Postgres `point(x, y)` where **x = longitude, y = latitude** (standard GIS convention). e.g. `point(-122.4194, 37.7749)` for San Francisco. Note: Postgres `point` is `(x,y)`, NOT `(lat,lng)` — inverted from common API conventions. |
| `date_start` | date | |
| `date_end` | date | nullable |
| `year` | int | **Trigger-maintained** (not GENERATED ALWAYS AS — Supabase doesn't support it reliably). The `sync_case_year` trigger unconditionally sets `year = EXTRACT(YEAR FROM date_start)` on every INSERT and UPDATE where `date_start IS NOT NULL`. If `date_start IS NULL`, `year` remains NULL. **Manual year overrides are not supported** — to set a custom year, you must set `date_start` accordingly or patch via a direct admin UPDATE that bypasses the trigger (service role only, log in `moderation_actions`). Nullable — cold cases without a known date have `year = NULL`. Used for /year/1994 SEO pages. |
| `summary` | text | |
| `body` | text | NOT NULL. CHECK (`length(trim(body)) > 0`) — empty body forbidden; even draft stubs must have at least placeholder text. Agent A writes ≥ 500 words minimum; Agent B enforces word count (too complex for a DB CHECK). **Draft minimum:** a single-sentence placeholder is sufficient for `review_status = 'draft'` — the DB only enforces non-empty, not minimum length. |
| `disclaimer` | text | Shown prominently — e.g. "No one has been charged in this case." CHECK (`disclaimer IS NULL OR length(trim(disclaimer)) > 0`) — if set, cannot be blank string. |
| `seo_title` | text | |
| `seo_description` | text | |
| `cover_image_url` | text | |
| `og_image_url` | text | 1200×630 for Open Graph |
| `number_of_victims` | int | nullable — for programmatic SEO stats pages |
| `primary_weapon` | text | nullable — controlled: `firearm` · `knife` · `blunt_object` · `poison` · `strangulation` · `unknown` · `other` |
| `review_status` | enum | NOT NULL DEFAULT `draft` — pipeline gate, see `docs/data-pipeline.md`. **Valid transitions only:** `draft→legal_review`, `legal_review→qa_review`, `legal_review→draft` (Agent B FAIL), `qa_review→approved`, `qa_review→legal_review` (Agent C vocab-only fix — does NOT increment revision_count), `qa_review→draft` (Agent C body-rewrite needed — increments revision_count), `approved→published`, `approved→legal_review` (source coverage gate fail — system job), `any→rejected` (admin only), `any→suppressed` (takedown), `suppressed→legal_review` (admin-initiated unsuppression only). To prevent infinite `qa_review↔legal_review` loops: after 3 back-and-forth cycles (tracked via `moderation_actions` count), escalate to human. Enforced by app layer + state machine trigger (SQL below). `rejected` = permanently blocked (admin decision, not Agent B/C FAIL). |
| `revision_count` | int | NOT NULL DEFAULT 0 — incremented on each Agent B/C FAIL → draft cycle; escalates to human review after 3 |
| `published_at` | timestamptz | null = draft. Never auto-set for `legal_sensitivity: high`. CHECK: `(review_status != 'published' OR published_at IS NOT NULL)` — one-directional: if `review_status = 'published'` then `published_at` must be set. The reverse is NOT enforced — suppressed or rejected cases may retain `published_at` as an audit timestamp of when they were live. Do NOT clear `published_at` when suppressing; it is a permanent audit record. |
| `deleted_at` | timestamptz | Soft delete. Cases are NEVER hard-deleted — destroys audit trail. Set `deleted_at` instead. RLS hides from all public queries. A `BEFORE DELETE` trigger (see trigger SQL section) raises an exception on any DELETE attempt. Child FK cascades (timeline_events, case_people, etc.) exist but can never fire because the parent cases row cannot be deleted. This resolves the apparent contradiction between "never hard-delete" and `ON DELETE CASCADE` on child tables. |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `slug_redirects`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `old_slug` | text | UNIQUE |
| `new_case_id` | uuid | FK → cases.id (ON DELETE CASCADE) — canonical reference, immune to slug changes |
| `new_slug` | text | NOT NULL — denormalized cache of the current `cases.slug`. **Initialization:** automatically set by `init_slug_redirect_new_slug` BEFORE INSERT trigger — fetches `cases.slug` for `new_case_id` and raises an exception if case not found. `sync_slug_redirect` trigger keeps it current on `cases.slug` UPDATE. App layer does not need to set this explicitly. |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `tags`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | NOT NULL, UNIQUE. CHECK (`slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'`). |
| `label` | text | NOT NULL |
| `description` | text | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_tags` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `tag_id` | uuid | FK → tags (ON DELETE CASCADE) |

PK: `(case_id, tag_id)`

---

### `related_cases` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id_a` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `case_id_b` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `relationship_type` | text | **Intentionally free-form** — not a CHECK/enum. **Canonicalization rule: lowercase snake_case only.** CHECK (`relationship_type ~ '^[a-z_]+$'`). Common values: `same_alleged_person` · `same_jurisdiction` · `connected_victim` · `linked_investigation` · `same_series` · `cold_case_reopened`. Free-form allows novel relationship types without a migration, but the snake_case CHECK prevents drift. Document new values in `docs/taxonomy.md` (to be created) when introduced. |
| `notes` | text | |

PK: `(case_id_a, case_id_b)`
CONSTRAINT: `CHECK (case_id_a < case_id_b)` — prevents `(A,B)` and `(B,A)` duplicates.
**Required write contract:** Always canonicalize before insert: `case_id_a = LEAST(uuid1, uuid2)`, `case_id_b = GREATEST(uuid1, uuid2)`. PostgreSQL UUID comparison is lexicographic. Any service writing relationships must sort the two UUIDs before insert — failure to do so causes intermittent constraint violations. Recommend wrapping in a helper function `upsert_related_case(uuid, uuid, relationship_type)` that canonicalizes automatically.
Query pattern: `WHERE case_id_a = X OR case_id_b = X`.

---

### `case_faqs`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `question` | text | NOT NULL. CHECK (`length(trim(question)) > 0`). |
| `answer` | text | NOT NULL. CHECK (`length(trim(answer)) > 0`). |
| `sort_order` | int | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

### `timeline_events`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `date` | date | |
| `date_precision` | enum | NOT NULL DEFAULT `approximate` — `exact` · `approximate` · `year_only`. CHECKs: `(date IS NOT NULL OR date_precision = 'approximate')` — when `date` is NULL, precision must be `approximate`. Additionally: `(date_precision != 'year_only' OR EXTRACT(MONTH FROM date) = 1 AND EXTRACT(DAY FROM date) = 1)` — year_only dates must use Jan 1 placeholder; enforces the encoding convention at DB level. Semantics: `exact` = confirmed date; `approximate` = estimated (UI: "circa [date]"); `year_only` = only year is known — store as `YYYY-01-01` (January 1 of the year as a placeholder), UI renders as "[year]" only. A real date with `approximate` precision is valid — means "approximately this date." For `year_only`, set `date = '[year]-01-01'::date` so the NOT NULL constraint is satisfied; the precision field signals to UI not to display month/day. Agent A must always set this explicitly; never leave at default without intention. |
| `event_type` | enum | `crime_event` · `discovery` · `arrest` · `indictment` · `trial_start` · `verdict` · `sentencing` · `appeal_filed` · `appeal_ruling` · `release` · `death` · `media_coverage` · `other` |
| `title` | text | |
| `description` | text | |
| `source_url` | text | **Legacy citation shortcut** — quick single-source field for Agent A data entry. **Not DB-enforced against `sources.url`** (no FK possible — `source_url` is free text, `sources.url` is UNIQUE but separate table). Agent B must verify: if `source_url` is set, a matching `sources.url` row must exist AND a corresponding `entity_sources` record must link it. Mismatches are flagged as P1 in the Agent B violation log. The source coverage publish gate checks `entity_sources` counts only — `source_url` has no role in the gate. This field is a data-entry convenience, not part of the canonical citation trail. |
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
| `slug` | text | NOT NULL, UNIQUE, URL-safe. CHECK (`slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'`) — same pattern as cases.slug. |
| `name` | text | NOT NULL |
| `aliases` | text[] | NOT NULL DEFAULT '{}' |
| `primary_known_role` | `case_role` enum | **Display hint only.** Captures the person's most common role across cases (e.g., `detective`, `attorney`). Not a legal status. |
| `dob` | date | nullable |
| `dod` | date | nullable |
| `bio` | text | |
| `photo_url` | text | |
| `is_living` | bool | nullable — null = unknown. `true` + not convicted → aggressive disclaimer required. CHECK: `(dod IS NULL OR is_living = false)` — person cannot have a date of death and be marked living. |
| `is_minor_at_time_of_crime` | bool | nullable — under 18 when the crime occurred |
| `is_minor_currently` | bool | nullable — still a minor today. **Auto-update mechanism:** Agent D nightly cron detects when `is_minor_currently = true` persons turn 18. On detection, Agent D does NOT auto-flip the flag. Instead: (1) Agent D sets `name_display_policy = 'requires_human_review'` (sentinel value — prevents display), (2) logs a `moderation_actions` record (`action_type: 'flagged'`, `agent_source: 'agent_d'`, `metadata: { "reason": "minor_aged_out", "dob": "...", "age": 18 }`), (3) alerts Carmen directly. Carmen reviews the case and either sets `name_display_policy = 'full'` (clear for display) or `name_display_policy = 'redacted'` (keep protected). Only after Carmen acts does `is_minor_currently` get set to false. **No auto-flip without human review.** Add `'requires_human_review'` to `name_display_policy` enum. The `public_case_people` view treats `requires_human_review` as `'[Review required]'` (same as auto sentinel). |
| `name_display_policy` | enum | `NOT NULL DEFAULT 'auto'` — `full` · `initials_only` · `redacted` · `auto` · `requires_human_review`. `auto` = system resolves based on minor flags + case role. `requires_human_review` = sentinel set by Agent D when a minor ages out — blocks display until human clears it. |
| `photo_display_policy` | enum | `NOT NULL DEFAULT 'requires_review'` — `allowed` · `blocked` · `requires_review`. New persons default to requiring review before photos are shown. |
| `links` | jsonb | `{ "wikipedia": "...", "news": [...] }` |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_people`

Surrogate PK provides a stable FK target for `charges` and `appeals` (they reference `case_people.id`, not the composite key). One row per person per case — `legal_status` is updated in place as status changes (e.g. `witness` → `person_of_interest`). History tracked in `moderation_actions`. Do not insert a second row for a status change.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK (surrogate) |
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `person_id` | uuid | FK → people (ON DELETE RESTRICT) |
| `case_role` | `case_role` enum | NOT NULL — functional role in this case: `victim` · `witness` · `detective` · `attorney` · `judge` · `other`. Who this person IS in relation to the case. |
| `legal_status` | `person_legal_status` enum | **Nullable** — legal standing: `alleged` · `convicted` · `acquitted` · `exonerated` · `charges_dropped` · `person_of_interest` · `no_charges_filed`. NULL for persons whose only relationship is a functional role (detective, attorney, judge, victim with no charges). **Display label** (denormalized from `charges`). The `charges` table is the structured source of truth; this field is the UI shorthand. Agent B must verify this label is consistent with `charges.disposition`. CHECK: `(case_role NOT IN ('detective','attorney','judge') OR legal_status IS NULL)` — detectives/attorneys/judges do not have legal standing in the case. |
| `notes` | text | Case-specific context |
| `charged_as_adult` | bool | NOT NULL DEFAULT false — true if this minor was tried in adult court. Required for minor display rules: `is_minor_at_time_of_crime = true` + `charged_as_adult = true` → `name_display_policy` may be `full`. The `public_case_people` view uses this field to resolve `name_display_policy: auto` for minors charged as adults. |
| `is_primary` | bool | NOT NULL DEFAULT false — flags the most prominent person(s) for a case. **Multiple `is_primary = true` rows per case are intentionally allowed** (e.g. a case with two co-defendants can have both flagged). No UNIQUE constraint. UI should display all primary-flagged persons prominently. If editorial intent is "exactly one primary", enforce at application layer, not DB. |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

UNIQUE: `(case_id, person_id)` — **one row per person per case**. `legal_status` is updated in place as the investigation evolves (e.g. `witness` → `alleged`). Status change history is tracked in `moderation_actions` (before/after in `metadata`). The surrogate PK `id` is kept for stable FK references from `charges` and `appeals`.

**When to update vs insert:** Always update the existing row. Never insert a second row for the same person-case pair. If a person's role in a case changes, update `legal_status` and log the change in `moderation_actions`.

**NOTE — legal_status + case_role valid/invalid combinations:**
The CHECK constraint blocks `legal_status` for `detective`/`attorney`/`judge` only. Additional combos require Agent B validation on every pipeline pass:
- `victim` + `legal_status`: ALLOWED only for self-defense cases where the victim was also charged (e.g. `victim` + `alleged`). Agent B must verify this combo is supported by the case facts.
- `witness` + `legal_status`: ALLOWED — witnesses can be formally charged.
- `victim` + `person_of_interest`: technically not blocked by CHECK but is semantically nonsensical; Agent B must flag.
- Any `legal_status` value inconsistent with `charges.disposition` for the same `case_person_id`: Agent B must flag. The DB does not auto-sync these.
Agent B must verify `legal_status` is consistent with `case_role` and `charges.disposition` on every pipeline pass.

**FK cascade policies:**
- `case_people.case_id` → ON DELETE CASCADE
- `case_people.person_id` → ON DELETE RESTRICT

---

### `agencies`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `slug` | text | NOT NULL, UNIQUE, URL-safe. CHECK (`slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'`). |
| `name` | text | NOT NULL |
| `type` | `agency_type` enum | `local_pd` · `sheriff` · `fbi` · `da` · `state_police` · `other` _(US-centric v1 — known tech debt)_ |
| `jurisdiction` | text | |
| `contact_url` | text | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_agencies` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `agency_id` | uuid | FK → agencies (RESTRICT) |
| `role` | text | e.g. `lead_investigator` · `supporting` · `forensics`. **Intentionally free-form** (agency roles are more varied than can be captured in an enum). Convention: lowercase snake_case. No CHECK constraint — document new values in `docs/taxonomy.md` when introduced. |

PK: `(case_id, agency_id)`

---

### `evidence`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
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
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
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
| `interview_id` | uuid | FK → interviews (ON DELETE CASCADE) |
| `person_id` | uuid | FK → people (RESTRICT) |
| `role` | text | CHECK (`role IN ('interviewee', 'interviewer')`). Only two valid values — a CHECK is appropriate here. |

PK: `(interview_id, person_id, role)` — role included in PK to allow same person as both interviewer and interviewee in edge cases (Q&A formats, host-turned-subject). UNIQUE(interview_id, person_id) without role would prevent this legitimate modeling.

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
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `episode_id` | uuid | FK → podcast_episodes (ON DELETE CASCADE) |
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
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
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
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `article_id` | uuid | FK → news_articles (ON DELETE CASCADE) |

PK: `(case_id, article_id)`

---

### `case_subreddits`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
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
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `subreddit_id` | uuid | FK → case_subreddits (SET NULL on delete) — null if post from non-curated subreddit. **Cross-case constraint:** migration MUST add a BEFORE INSERT OR UPDATE trigger: `RAISE EXCEPTION` if `subreddit_id IS NOT NULL AND NOT EXISTS (SELECT 1 FROM case_subreddits WHERE id = NEW.subreddit_id AND case_id = NEW.case_id)`. Not expressible as a simple CHECK (requires subquery). Trigger is required — app-layer-only is insufficient given service-role writes by agents. |
| `post_id` | text | UNIQUE — Reddit base36 ID, deduplication key |
| `title` | text | |
| `subreddit` | text | Denormalized for display |
| `url` | text | |
| `score` | int | |
| `comment_count` | int | |
| `fetched_at` | timestamptz | Updated on each sync |
| `created_at` | timestamptz | |

**Upsert pattern:** `INSERT INTO case_reddit_posts (...) ON CONFLICT (post_id) DO UPDATE SET score = EXCLUDED.score, comment_count = EXCLUDED.comment_count, fetched_at = now()`. The conflict target is `post_id` UNIQUE, not the uuid PK. Do not attempt to upsert on `id` — the uuid PK is not the deduplication key.

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

Structured legal proceedings per person per case. **`charges` is the source-of-truth legal record**; `case_people.legal_status` is the derived display label. Agent B must verify consistency on every pipeline pass. No DB trigger syncs them automatically — sync is pipeline-enforced (see `docs/data-pipeline.md` Agent B checks).

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

**Display note:** `disposition: pending` = pre-trial (charges filed, trial not yet concluded) → show "Charged (trial pending)". "Convicted (appealing)" is driven by `appeals.status = 'pending'` on a person with `legal_status: convicted` — see `appeals` display note below.

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

**Cross-person integrity:** If `charge_id` is set, it must belong to the same `case_person_id` as the appeal — `charges.case_person_id = appeals.case_person_id`. Enforced via trigger:
```sql
CREATE OR REPLACE FUNCTION check_appeal_charge_consistency() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.charge_id IS NOT NULL THEN
    IF NOT EXISTS (
      SELECT 1 FROM charges
      WHERE id = NEW.charge_id AND case_person_id = NEW.case_person_id
    ) THEN
      RAISE EXCEPTION 'appeals.charge_id % does not belong to case_person_id %', NEW.charge_id, NEW.case_person_id;
    END IF;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
-- REQUIRED: uncomment in migration — this is a legal integrity constraint, not optional
-- CREATE TRIGGER check_appeal_charge_consistency BEFORE INSERT OR UPDATE ON appeals
--   FOR EACH ROW EXECUTE FUNCTION check_appeal_charge_consistency();
```

---

### `sources`

Citation trail — defamation defense. Every factual claim should have a source record.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `source_type` | enum | `court_document` · `news_article` · `official_statement` · `book` · `documentary` · `podcast_episode` · `other` |
| `title` | text | |
| `publisher` | text | nullable |
| `url` | text | NOT NULL — deduplication key for citation audits. **No plain UNIQUE constraint** — use functional index `CREATE UNIQUE INDEX sources_url_lower_unique ON sources (lower(url))` (case-insensitive). See canonicalization note below. |
| `archived_url` | text | nullable — Wayback Machine fallback |
| `published_at` | timestamptz | nullable |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

**URL canonicalization required before INSERT:** strip UTM params, normalize trailing slash, lowercase scheme+host. Use a helper function in the app layer — do NOT rely on the DB to normalize. `sources.url UNIQUE` deduplication is case-sensitive in Postgres; inconsistent normalization will cause duplicate rows. **Migration MUST: (1) NOT add a plain UNIQUE constraint on `url`; (2) add `CREATE UNIQUE INDEX sources_url_lower_unique ON sources (lower(url));` instead.** This is the canonical approach — do not use a plain UNIQUE + normalization trigger (two mechanisms can diverge). The functional index handles deduplication; app layer handles canonicalization before insert. Canonical form: `https://host.com/path` (no trailing slash, no query params except content-relevant ones).

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

**Why entity_sources CHECK includes exactly these entity types:** These are the primary citation-bearing entities — the things that make factual claims requiring sourcing. `community_notes`, `community_corrections`, and `content_reports` are user-generated and cite existing sources; they don't themselves GET cited. `moderation_actions` are internal audit records, not citable claims. This CHECK is intentionally narrower than `content_reports.entity_type` or `moderation_actions.entity_type`. Do not extend without evaluating whether the new entity type is genuinely citation-bearing.

**Orphan cleanup policy (all polymorphic tables):** The following tables have no Postgres FK because they reference multiple parent tables: `entity_sources`, `upvotes`, `content_reports`, `content_takedown_requests`, `suppressed_entities`. Orphan cleanup applies to all of them:
- **entity_sources:** Agent D nightly scan flags rows where `entity_id` no longer exists in the target table. Hard deletes from parent tables: app layer must DELETE from entity_sources before (or in the same transaction as) any hard delete of a referenced entity.
- **upvotes:** Rows for soft-deleted entities are intentionally preserved (historical count). Rows for hard-deleted entities: app layer must DELETE from upvotes before hard-deleting the parent. The nightly reconciliation cron verifies counts match surviving rows.
- **content_reports, content_takedown_requests:** App layer must clean up when referenced entity is hard-deleted. Agent D nightly scan flags orphans.
- **suppressed_entities:** Do NOT delete records for suppressed entities — suppression is display-only, not data deletion. Only delete the `suppressed_entities` row when `lifted_at` is set and Carmen confirms removal.
Hard deletes are rare (cases are never hard-deleted; block_case_hard_delete trigger enforces this). The highest-risk path is timeline_events: cascade from cases.deleted_at does NOT propagate to entity_sources (no FK). App layer must handle this.

---

### `case_series`

Groups serial killer cases or connected case clusters.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `name` | text | NOT NULL — e.g. `Ted Bundy Murders` |
| `slug` | text | NOT NULL, UNIQUE. CHECK (`slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'`) — same pattern as all other slug fields. |
| `description` | text | nullable |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `case_series_members` (pure join — no timestamps)

| Field | Type | Notes |
|-------|------|-------|
| `series_id` | uuid | FK → case_series (ON DELETE CASCADE) |
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
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
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
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
| `entity_id` | uuid | No Postgres FK (polymorphic) — **no automatic orphan cleanup trigger.** Nightly reconciliation cron recomputes `upvote_count` from live rows only; orphaned upvote rows (parent soft-deleted) do not affect count accuracy since the reconciliation query JOINs to the parent table. Agent D flags orphan count mismatches. App layer should block new upvotes on soft-deleted entities (check `deleted_at` before insert). |
| `created_at` | timestamptz | |

PK: `(user_id, entity_type, entity_id)` — naturally prevents double-upvotes.

CONSTRAINT: `CHECK (entity_type IN ('community_notes'))` — **MVP scope: only `community_notes` are upvotable.** Tables with `upvote_count int NOT NULL DEFAULT 0 CHECK (upvote_count >= 0)` in v1: `community_notes` only. Future candidates: `community_corrections`, `evidence`, `sources` — extend this CHECK + add the column when ready. Do not use the dynamic trigger for any other entity type until this CHECK is extended. The CHECK and the trigger scope MUST stay in sync — adding a table to the trigger without adding to this CHECK causes immediate write failures (no rows can be inserted). Extension procedure: (1) add `upvote_count int NOT NULL DEFAULT 0 CHECK (upvote_count >= 0)` to the new table, (2) add the new entity_type to this CHECK (migration required), (3) add a nightly reconciliation query to `docs/data-pipeline.md`.

### `community_notes`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
| `parent_id` | uuid | `uuid NULL REFERENCES community_notes(id) ON DELETE SET NULL` — self-referencing FK for threaded replies. When a parent note is deleted/moderated, replies are preserved with `parent_id = NULL` (surfaced as top-level). |
| `user_id` | uuid | FK → profiles (ON DELETE SET NULL) |
| `body` | text | NOT NULL. CHECK (`length(trim(body)) > 0`). CHECK (`length(body) <= 50000`) — 50k char cap prevents runaway inserts. |
| `upvote_count` | int | NOT NULL DEFAULT 0, CHECK (upvote_count >= 0) — trigger-maintained; trigger uses `GREATEST(upvote_count - 1, 0)` to prevent negative counts on duplicate deletes |
| `is_pinned` | bool | NOT NULL DEFAULT false |
| `deleted_at` | timestamptz | Soft delete. UI shows "[removed]". RLS blocks new upvotes (`WHERE deleted_at IS NULL`). `upvote_count` preserved (not zeroed) — restored notes recover their signal. |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

### `community_corrections`

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `case_id` | uuid | NOT NULL, FK → cases (ON DELETE CASCADE) |
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

**Scope note — intentional narrowing:** Suppression is case-level or person-level only. The CHECK (`entity_type IN ('people', 'cases')`) is intentionally narrow. For sub-entity takedowns (timeline events, evidence, community_notes, community_corrections), use soft-delete via `moderation_actions` + app-layer removal rather than `suppressed_entities`. Do not extend this CHECK to sub-entity types without evaluating the implications for `public_case_tombstones` (which currently only renders suppressed cases, not suppressed sub-entities).
| `takedown_request_id` | uuid | FK → content_takedown_requests (nullable) |
| `suppressed_by` | uuid | FK → profiles (SET NULL) — nullable; null when suppression is agent-initiated (e.g. Agent D auto-suppress) |
| `agent_source` | text | nullable — CHECK (`agent_source IN ('agent_a','agent_b','agent_c','agent_d','human','system')`). Also subject to: CHECK `(suppressed_by IS NOT NULL OR agent_source IS NOT NULL)` — same at-least-one-non-null pattern as `moderation_actions`. |
| `suppressed_at` | timestamptz | NOT NULL DEFAULT now() |
| `lifted_at` | timestamptz | nullable — if suppression was later reversed |
| `lift_reason` | text | nullable |
| `created_at` | timestamptz | NOT NULL DEFAULT now() |

**Note on suppressed_at vs created_at:** Both default to `now()` on INSERT and should never diverge in practice. The redundancy is intentional for audit clarity: `created_at` is the universal row-creation convention present on all tables; `suppressed_at` is the human-readable semantic field ("when suppression took effect"). Migration: both must be `NOT NULL DEFAULT now()`. If they ever diverge (e.g. due to a manual SQL fix), treat `suppressed_at` as authoritative for suppression logic and `created_at` as authoritative for row provenance.

PARTIAL UNIQUE INDEX: `CREATE UNIQUE INDEX ON suppressed_entities (entity_type, entity_id) WHERE lifted_at IS NULL` — allows re-suppression after a suppression has been lifted. No full `UNIQUE` constraint (would block INSERT after lift). No `updated_at` — immutable; use `lifted_at` to record reversal.

**Unsuppression flow:** To lift suppression: (1) set `lifted_at = now()` and `lift_reason` on the existing row — **never delete** it (audit trail); (2) log a `moderation_actions` record (`action_type: unsuppressed`, `agent_source: human`, `metadata: { reason }`); (3) trigger ISR revalidation on affected case pages; (4) re-index in Typesense. Entity can be re-suppressed after a lift by inserting a new `suppressed_entities` row (partial index allows this). **Path back from `suppressed` review_status:** admin manually sets `review_status → legal_review` and logs the action. Only admins can un-suppress cases. `suppressed → any` transition is admin-only, not pipeline-accessible.

### `moderation_actions`

Immutable audit log of every moderation decision. Required for legal defense.

| Field | Type | Notes |
|-------|------|-------|
| `id` | uuid | PK |
| `moderator_id` | uuid | FK → profiles (SET NULL) — nullable; null when action is from an automated agent |
| `agent_source` | text | nullable — CHECK (`agent_source IN ('agent_a','agent_b','agent_c','agent_d','human','system')`). `system` = automated cron/trigger jobs. Set when `moderator_id` is null. Also subject to: CHECK `(moderator_id IS NOT NULL OR agent_source IS NOT NULL)` — at least one attribution required. Prevents orphaned audit records. |
| `entity_type` | text | NOT NULL, CHECK (`entity_type IN ('cases','people','case_people','charges','appeals','community_notes','community_corrections','content_reports','suppressed_entities','content_takedown_requests')`) — constrained like all other entity_type columns; audit records with invalid entity_type are meaningless |
| `entity_id` | uuid | NOT NULL |
| `action_type` | enum | NOT NULL — `approved` · `rejected` · `edited` · `deleted` · `restored` · `suppressed` · `unsuppressed` · `locked` · `unlocked` · `flagged` |
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
| `media_id` | uuid | No Postgres FK (polymorphic — references `case_podcast_episodes` or `tv_shows` depending on `media_type`). **Orphan policy:** no cleanup trigger. Accept orphans — if the media row is deleted, progress rows become stale but harmless (UI filters by joining to the media table). Nightly reconciliation not required; rows auto-orphan silently and are excluded from JOIN-based queries. |
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
| `sources` | `url` | `CREATE UNIQUE INDEX sources_url_lower_unique ON sources (lower(url))` | Citation deduplication (case-insensitive functional index — do NOT use plain UNIQUE) |
| `news_articles` | `published_at` | btree | Recent articles listing |
| `slug_redirects` | `old_slug` | UNIQUE | Redirect lookups |
| `slug_redirects` | `new_case_id` | btree | FK join — sync_slug_redirect trigger queries this on every case slug change |
| `jurisdictions` | `state_code` | btree | State SEO pages |
| `cases` | `country_code` | btree | International filtering / SEO pages |
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
| `moderation_actions` | `(entity_type, entity_id, action_type, agent_source)` | btree | Covering index for publish gate query (`WHERE entity_type='cases' AND entity_id=X AND action_type='approved' AND agent_source='human'`) |
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
| 3 | Gabby Petito | `closed_suspect_deceased` | `standard` | Brian Laundrie = `person_of_interest` for Petito's murder (never charged). Note: Laundrie WAS federally charged with unauthorized use of a debit card — that charge should appear as a separate `charges` row with `charge_label: 'Unauthorized use of debit card'`, `disposition: convicted` (posthumous). `person_of_interest` scopes only to Petito's homicide. |
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
| 14 | Chandra Levy | `unsolved` | `elevated` | Ingmar Guandique = `exonerated` (convicted 2010, conviction vacated 2016 — no retrial). Gary Condit = `no_charges_filed` (never formally accused, sued multiple outlets). High editorial care required. |
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

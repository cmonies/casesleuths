# CaseSleuths — Data Pipeline & Agent Protocol

_v1.0 · 2026-02-28_
_Status: Planned — implement before first case seeding run_

---

## Overview

CaseSleuths has two distinct data quality risks that a human editorial process alone cannot catch at scale:

1. **Legal/compliance drift** — a case published with wrong legal vocabulary, missing disclaimers, `legal_sensitivity: high` with no review, living unconvicted person with no aggressive disclaimer
2. **Structural incompleteness** — case goes live with no sources, no timeline events, no content warnings, `case_people` rows without charge records

The solution is a **4-agent pipeline** modeled on the designjobs.cv validation approach: agents do work, other agents review that work, nightly crons audit for drift. Nothing gets published without passing multiple gates.

---

## Pipeline Overview

```
New Case Request
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 1: Content Seeding                               │
│  Agent A (claude-sonnet): Researcher                    │
│  → Seeds all tables, writes body + FAQs                 │
│  → review_status: draft → legal_review                  │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 2: Legal Review                                  │
│  Agent B (claude-opus): Legal Reviewer                  │
│  → Checks vocab, labels, disclaimers, sources           │
│  → PASS: review_status → qa_review                      │
│  → FAIL: review_status → draft, revision_count++        │
│  → revision_count >= 3: escalate to human               │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 3: QA / Editorial Review                         │
│  Agent C (claude-sonnet): QA Reviewer                   │
│  → Independent framing check, completeness score        │
│  → PASS: review_status → approved                       │
│  → FAIL (body rewrite): review_status → draft           │
│  → FAIL (vocab/label only): review_status → legal_review│
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 4: Publish Gate                                  │
│                                                         │
│  legal_sensitivity: HIGH → Carmen required              │
│    → Carmen reviews, sets published_at + status         │
│                                                         │
│  legal_sensitivity: standard/elevated → Auto-publish    │
│    → System job sets published_at + review_status:      │
│      published after approved; logs to moderation_      │
│      actions (agent_source: 'system')                   │
│                                                         │
│  DB trigger enforces minimum source coverage before     │
│  allowing published status to be set                    │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
                    Case is LIVE
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  STAGE 5: Nightly Compliance Audit (ongoing)            │
│  Agent D (haiku + sonnet): Compliance Monitor           │
│  → P0 violations → auto-suppress + DM Carmen           │
│  → Stale pipeline stages (> 24h) → alert               │
│  → Summary → #truecrime                                 │
└─────────────────────────────────────────────────────────┘
```

---

## Agent A: Researcher

**Trigger:** Manual (new case request) or scheduled batch  
**Input:** Case name, slug, initial seed data (Wikipedia URL, known court records)  
**Output:** Populated draft rows across all relevant tables, `review_status: legal_review`

### What it seeds:

| Table | What gets populated |
|-------|---------------------|
| `cases` | title, slug, status, location, dates, summary, body (markdown), content_warnings, disclaimer, legal_sensitivity |
| `timeline_events` | 5+ events minimum, each with event_type and source_url |
| `people` | name, aliases, dob/dod, is_living, is_minor_at_time_of_crime, name_display_policy |
| `case_people` | legal_status (from controlled vocabulary), notes |
| `charges` | charge_label, disposition, filed_date, court_name (for anyone with a formal charge) |
| `appeals` | if active appeal exists |
| `case_faqs` | 3+ Q&A pairs targeting People Also Ask queries |
| `sources` + `entity_sources` | At least 1 source per timeline_event, 1 per person legal_status claim |
| `tags` + `case_tags` | 3+ tags |
| `jurisdictions` | jurisdiction_id on case if applicable |

### Completion criteria (required before handoff to Agent B):

- [ ] `cases.body` ≥ 500 words
- [ ] ≥ 5 `timeline_events` rows
- [ ] All persons in `case_people` have `legal_status` set (no nulls)
- [ ] `content_warnings` populated (or explicitly empty with a note why)
- [ ] `disclaimer` set if `legal_sensitivity: high` or `status: unsolved`
- [ ] ≥ 1 source per timeline event
- [ ] ≥ 3 `case_faqs`
- [ ] `review_status` set to `legal_review`

---

## Agent B: Legal Reviewer

**Trigger:** `review_status = 'legal_review'`  
**Input:** Full case record (cases + all joined tables)  
**Output:** PASS (→ `qa_review`) or FAIL (→ `draft` with violation log)

### Checks performed:

#### Vocabulary scan (cases.body + all text fields)
- [ ] No banned words used as labels: `perpetrator`, `killer`, `murderer` (as applied to living/charged persons)
- [ ] Every named person in body text matches a `case_people` row
- [ ] Implied-guilt language check: phrases like "who killed," "the killer," "the murderer" without hedging → FLAG

#### Legal status consistency
- [ ] Every `case_people` row has `legal_status` set
- [ ] `case_people.legal_status = 'convicted'` → at least 1 `charges` row with `disposition: convicted`
- [ ] `case_people.legal_status = 'alleged'` → at least 1 `charges` row with `disposition: pending`
- [ ] `case_people.legal_status = 'acquitted'` → at least 1 `charges` row with `disposition: acquitted`
- [ ] If person has active appeal (`appeals.status = 'pending'`) → note must say "Convicted (appealing)"
- [ ] `person.is_living = true` + `legal_status != 'convicted'` → aggressive disclaimer present in `cases.disclaimer`

#### Minor privacy
- [ ] Any person with `is_minor_at_time_of_crime = true` → `name_display_policy` is not `full` (unless charged as adult)
- [ ] Any person with `is_minor_currently = true` → `name_display_policy: redacted` or `initials_only`
- [ ] Photo URLs present for minor POIs → `photo_display_policy: blocked`

#### Structural requirements
- [ ] `cases.content_warnings` is not null/empty (or explicitly documented as empty with justification)
- [ ] `cases.disclaimer` present if `status` is `unsolved`, `closed_no_crime`, or `solved_acquitted`
- [ ] `legal_sensitivity: high` cases → must have explicit justification note in `cases.disclaimer`
- [ ] Every `timeline_event` has at least 1 source in `entity_sources`

#### Source quality
- [ ] No `source.url` values that 404 (spot check 3 random sources)
- [ ] `entity_sources.claim_type` set correctly — no `court_finding` without a court document source
- [ ] **Living unconvicted persons**: any factual claim in `cases.body` about a living person who is not `convicted` must be traceable to a named primary source (court document, official LE statement, or verifiable news article). This includes: "was seen at scene," "failed polygraph," "owned the weapon," "was last to see victim." If the source is a podcast or speculation — flag for human review before approve.

**P0 auto-fail for living unconvicted persons:** Any claim that directly implies involvement without a court-document source → FAIL, route to `draft`.

### Output format:
```
LEGAL_REVIEW_RESULT: PASS | FAIL
violations: [
  { severity: P0|P1|P2, table: "case_people", row_id: "uuid", issue: "..." },
  ...
]
notes: "..."
```

PASS → set `review_status: qa_review`, increment no counters, log to `moderation_actions` (`action_type: approved`, `agent_source: agent_b`, `reason: "legal_review_pass"`)  
FAIL → set `review_status: draft`, increment `cases.revision_count`, log violations to `moderation_actions` (`action_type: rejected`, `agent_source: agent_b`, `metadata: { violations: [...] }`)

**P0 violations (any single one = auto-FAIL):**
- Banned vocabulary used as a label
- Living acquitted person with no disclaimer
- Minor with `name_display_policy: full` and not charged as adult
- `case_people.legal_status` directly contradicts `charges.disposition`

---

## Agent C: QA Reviewer

**Trigger:** `review_status = 'qa_review'`  
**Input:** Full case record  
**Output:** APPROVED (→ `approved`) or NEEDS_REVISION (→ `legal_review` with notes)

### Perspective: read as a first-time visitor

Agent C operates independently of Agent B's output — it re-reads the case as a reader, not as a schema validator.

#### Framing check
- [ ] Would a reasonable reader infer guilt from the case body, even if legal vocab is technically correct?
- [ ] Are hedge words ("allegedly," "according to court documents," "charged with") present wherever needed?
- [ ] Is the disclaimer visible and prominent, not buried?

#### Completeness score (must hit threshold before approve)

| Criterion | Weight | Min score |
|-----------|--------|-----------|
| Body completeness (length, key facts covered) | 20% | 7/10 |
| Timeline coverage (major events present, dates accurate) | 20% | 7/10 |
| Person records (all major figures, correct legal status) | 20% | 8/10 |
| Source quality (credible, not dead links) | 15% | 8/10 |
| FAQ coverage (People Also Ask targeting) | 10% | 6/10 |
| Legal framing (hedging, disclaimers, no implied guilt) | 15% | 9/10 |

**Threshold:** Overall weighted score ≥ 7.5/10, no single criterion below minimum.

#### Suppression check
- [ ] None of the persons in this case are in `suppressed_entities`
- [ ] None of the cases in `related_cases` are suppressed

#### Output format:
```
QA_REVIEW_RESULT: APPROVED | NEEDS_REVISION
scores: { body: X, timeline: X, persons: X, sources: X, faqs: X, legal: X, overall: X }
revision_notes: "..."
```

APPROVED → set `review_status: approved`, log to `moderation_actions`  
NEEDS_REVISION (editorial framing issue — body text needs rewriting) → set `review_status: draft`, increment `revision_count`. Agent A picks it up.  
NEEDS_REVISION (vocab/label issue only — no body rewrite needed) → set `review_status: legal_review`, Agent B re-checks without Agent A involvement.

**When `rejected` is used:** Admin-only. Set when a case will never be published (e.g. determined to be fabricated, subject of court order blocking publication, or out of scope). Not set by Agent B or C — their failures always route back to `draft` or `legal_review` for correction.

**For `legal_sensitivity: high` cases:** Agent C flags for human review instead of auto-approving. Sets `review_status: approved` but adds a `moderation_actions` record with `action_type: approved`, `reason: "requires_human_final_approval"`. Human must then explicitly set `published_at` and `review_status: published`.

---

## Human Approval Gate

**Required for:** All `legal_sensitivity: high` cases  
**Not required for:** `standard` and `elevated` cases — auto-published after Agent C approval

### High-sensitivity cases (human required)
Carmen (or trusted editor) reviews Agent C's score report, checks the case page live, then:
- Sets `published_at = now()`
- Sets `review_status: published`
- Logs to `moderation_actions` (`action_type: approved`, `agent_source: 'human'`, `reason: "human_final_approval"`)

### Standard/elevated cases (auto-publish)
After Agent C sets `review_status: approved`:
1. A system job (cron or DB trigger) checks the minimum source coverage gate
2. If gate passes: sets `published_at = now()`, `review_status: published`
3. Logs to `moderation_actions` (`action_type: approved`, `agent_source: 'system'`, `reason: "auto_publish_approved"`)
4. If gate fails: routes back to `legal_review`, increments `revision_count`

**Source coverage gate (enforced at DB level):**
- At least 1 source in `entity_sources` linked to the case itself
- At least 1 source linked to each `timeline_event` in the case
- Trigger blocks `INSERT/UPDATE SET review_status = 'published'` if not met

---

## Agent D: Compliance Monitor (Nightly Cron)

**Schedule:** Nightly at 2:00 AM PT  
**Model:** claude-haiku (lightweight — mostly SQL queries + pattern matching)  
**Output:** Summary to #truecrime · High-severity alerts → DM Carmen

### Checks performed on all published cases:

#### Critical (P0) — AUTO-SUPPRESS + DM Carmen immediately

For specific P0 categories, Agent D **auto-suppresses** the case (`review_status: suppressed`, logs to `moderation_actions` with `agent_source: agent_d`, `reason: 'auto_suppressed_p0'`) without waiting for human response. Carmen's DM includes the suppression action already taken and what she needs to review.

**Auto-suppress categories:**
- Any `cases` row with `legal_sensitivity: high` + `published_at IS NOT NULL` but no `moderation_actions` record confirming human approval → **published without human review** → AUTO-SUPPRESS
- Any person with `dod IS NOT NULL` + `is_living = true` → **data integrity error** → flag the case, DO NOT auto-suppress (may be a data entry error, not malicious)
- Any `suppressed_entities` record where the entity's Typesense index still returns the person's name → **suppression bypass** → trigger Typesense re-index immediately

**DM-only categories (no auto-suppress):**
- Any `case_people` row with `legal_status = 'convicted'` where `appeals.status = 'pending'` (display bug, not legal risk)
- Any `is_living = true` person with no disclaimer on their case (legal risk but needs editorial judgment)

#### Stale Pipeline Detection (P1) — Include in nightly summary
- Any case with `review_status IN ('legal_review', 'qa_review')` and `updated_at` older than 24 hours → **Agent B or C may have crashed** → flag for investigation
- Any case with `review_status = 'approved'` and `published_at IS NULL` for > 7 days → stale approval

#### High (P1) — Include in nightly summary, flag for next editorial pass
- Any `case_people` row with `legal_status = 'convicted'` where `appeals` table shows `status = 'pending'` → should display "Convicted (appealing)" (DM-only, not auto-suppress)
- Any person with `is_living = true` + `legal_status NOT IN ('convicted')` + no `disclaimer` text on their case pages
- Any published case with `content_warnings` empty that contains keywords: `child`, `minor`, `infant`, `sexual`, `rape` in `cases.body`
- Any `case_people` row added in last 24h with no corresponding `charges` row (for persons with legal_status `alleged`, `convicted`, `acquitted`)
- Any `source.url` in `entity_sources` returning 404 (spot check 10 random sources)

#### Medium (P2) — Weekly batch summary
- Cases with `review_status: approved` but `published_at IS NULL` for > 7 days (stale approvals)
- Cases with 0 `case_faqs` rows (missing SEO opportunity)
- Persons with `photo_url` set but `photo_display_policy: blocked` (photo stored but never displayed — clean up)

#### Random sample audit (nightly)
- Pick 5 random published cases
- Re-run Agent B legal vocab check on `cases.body`
- Any violations → P1 flag with case name and violation detail

### Output format:
```
NIGHTLY_COMPLIANCE: date
p0_alerts: 0  (or list)
p1_flags: X   (with details)
p2_batch: X   (summary only)
sample_audit: 5 cases checked, 0 violations
```

Post summary to #truecrime thread. If any P0 alerts: DM Carmen with full detail.

---

## Pipeline Schema Requirements

The following fields in `cases` support this pipeline:

| Field | Purpose |
|-------|---------|
| `review_status` | Current pipeline stage — the single source of truth for where a case is |
| `published_at` | Set only when `review_status: published`. Never set independently. |
| `deleted_at` | Soft delete — cases never hard-deleted |
| `legal_sensitivity` | Determines whether human approval gate is required |
| `disclaimer` | Required text for certain case types — validated by Agent B |
| `content_warnings` | Required field — empty array must be explicit, not absent |

`moderation_actions` is the audit trail for every stage gate. Every agent pass/fail **must** produce a `moderation_actions` record with:
- `entity_type: 'cases'`, `entity_id: [case uuid]`
- `action_type: 'approved'` or `'rejected'`
- `reason`: which agent + which checks passed/failed
- `metadata`: full violations JSON from Agent B/C output

---

## Iteration Loop

If Agent B or C fails a case, the loop is:

```
FAIL → review_status: draft
     → revision notes logged to moderation_actions
     → Agent A reads revision notes, corrects issues
     → review_status: legal_review
     → Agent B re-runs
     → ... until PASS
```

Maximum 3 revision loops (`cases.revision_count >= 3`) before escalating to human review. Log to `moderation_actions` with `reason: 'max_revisions_exceeded'`, `agent_source: agent_b`. Human must manually reset `revision_count` and set `review_status: legal_review` to re-enter the pipeline.

---

## Rollout Plan

| Phase | What gets built |
|-------|----------------|
| Pre-launch | Agent B (legal reviewer) — run manually on every case before publish |
| Launch | Agent D (nightly cron) — compliance monitoring starts day 1 |
| Post-launch week 1 | Agent A (researcher) — automate seeding for cases 11–20 in Top 20 |
| Post-launch week 2 | Agent C (QA reviewer) — formalize the completeness scoring |
| Month 2 | Full pipeline automated for new case additions |

Cases 1–10 (highest legal sensitivity) are seeded and reviewed manually using the Agent B checklist as a human editorial guide before automation is ready.

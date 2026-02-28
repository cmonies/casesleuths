# CaseLog — Legal & Safety Framework
_Last updated: 2026-02-26_
_Status: Draft — review with a lawyer before launch_

---

## Overview

True crime is a uniquely sensitive content category. Unlike most apps, CaseLog touches:
- Living people accused of crimes (not always convicted)
- Victims and their families, who may not want coverage
- Ongoing investigations and trials
- Minors as victims or witnesses
- Unresolved cases where facts are disputed

Getting this wrong isn't just a PR problem — it's a **defamation liability** and potentially a **privacy law violation**. This framework defines the policies, data model requirements, and moderation approach to build this responsibly.

---

## 1. Defamation Risk

### The Core Risk
CaseLog aggregates and surfaces claims about real people. If we label a living person a "killer" or "suspect" and they were acquitted, never charged, or wrongfully convicted — that's defamation exposure.

### High-Risk Scenarios
- **Acquitted suspects** — if our UI implies guilt, that's actionable
- **Wrongful conviction cases** — people who were convicted but later exonerated (e.g., Brendan Dassey, Steven Avery's ongoing case)
- **Persons of interest who were never charged** — named by media/podcasts but never formally accused
- **Living family members** — podcast speculation about relatives of victims or suspects
- **Unsolved cases** — fan theories that point to real living people

### Mitigation: Status Labels (Required in Data Model)

Every case and every person in the case must carry a legal status label. This is not cosmetic — it's structural.

**Case status:**
- `unsolved` — no conviction, no definitive perpetrator
- `solved_convicted` — conviction stands
- `solved_exonerated` — original conviction overturned
- `solved_acquitted` — charged, acquitted
- `ongoing_trial` — active criminal proceedings
- `cold_case` — officially inactive, no resolution
- `civil_resolution` — no criminal conviction but civil judgment (e.g., O.J. Simpson)
- `closed_no_crime` — ruled accidental, suicide, or natural causes (e.g., Elisa Lam)
- `closed_suspect_deceased` — primary suspect died before trial or charges (e.g., Gabby Petito case)

**Person status (within a case):**
- `victim`
- `convicted` (with caveat: appeals ongoing if applicable)
- `acquitted`
- `exonerated`
- `alleged` (charged but not convicted)
- `person_of_interest` (named by investigators, not formally charged — if truly unnamed, omit from person records)
- `witness`
- `no_charges_filed` (named in media/podcasts, never charged)

**UI requirement:** Person status must be visible on case pages. Never just "killer" — always "convicted of murder" or "alleged to have..." Language matters legally.

### What We Don't Do
- No user-generated accusations against real people
- No "who do you think did it?" polls for unsolved cases with living named suspects
- No case pages that present speculation as fact
- No episode linking that implies a podcast's speculation = established fact

---

## 2. Ongoing Cases & Active Trials

### The Risk
Linking to podcast content about an ongoing trial could theoretically be raised as influencing proceedings. More practically: facts change rapidly, and we could be presenting outdated or prejudicial framing.

### Policy
- Cases with `status: ongoing_trial` get a prominent banner: *"This case involves active legal proceedings. Information may change."*
- Episode index for ongoing cases is manually reviewed before display (not auto-published from classifier)
- No community ratings/annotations on ongoing cases until post-verdict (ratings lock)
- Case data updates reviewed by admin before going live (not community edits)

---

## 3. Victim Privacy

### The Risk
- Some jurisdictions restrict naming underage victims (varies by country and state)
- Some families have explicitly asked media to stop covering their cases
- Some victims have living relatives who find the coverage retraumatizing

### Policy

**Minors as victims:**
- Default to initials or "unnamed minor victim" for cases involving child victims where the minor's identity is legally protected (e.g., many UK cases)
- US law is less restrictive but apply conservatism — check jurisdiction
- Never display ages or identifying details of minor victims beyond what's necessary for case context

**Family requests:**
- If a victim's family contacts us asking for case removal or amendment, review within 48 hours
- Removal is a last resort — we can anonymize, reduce detail, or add a family statement
- Document all such requests and responses

**Content warnings (required at case level):**
These must be stored in the case schema and surfaced in UI before the user sees case details:
- `child_victim` — case involves a minor victim
- `sexual_violence` — case involves rape or sexual assault
- `infant_victim` — case involves a baby or infant
- `family_violence` — domestic violence / intimate partner violence
- `mass_casualty` — multiple victims
- `graphic_imagery_risk` — associated media contains graphic content

Users must be able to **filter out content warning categories** from browse/discovery feeds. This is a required V1 feature, not optional.

---

## 4. Privacy Laws

### Jurisdictions That Matter
- **GDPR (EU/UK):** User data, right to erasure, explicit consent. If we have EU users, we need a privacy policy, cookie consent, and a data deletion mechanism.
- **CCPA (California):** Right to know, opt out of data selling (we don't sell data, but must state so). Required for California users — which is basically everyone.
- **Children's privacy (COPPA, US):** CaseLog is not for users under 13 (content is adult). Must have age gate at signup.

### Required at Launch
- Privacy policy (covers: what we collect, how we use it, your rights, deletion)
- Terms of service (covers: content policy, liability, acceptable use)
- Age verification gate at signup (13+ minimum; recommend 18+ given content)
- Cookie consent banner if we use tracking (even basic analytics)
- Data deletion mechanism (user can delete account + all data)

---

## 5. Community Moderation Policy

### Phase 2 Problem: When Users Can Contribute
Once we allow ratings, corrections, or annotations, we're a UGC platform. True crime communities are passionate and can be harmful — speculation, doxxing, harassment of families and suspects.

### What Users Can Do (Phase 2)
- Rate episode quality (1-5 stars) — anonymous, no text
- Flag a wrong episode-case mapping — explains why, admin reviews
- Submit a case status update (e.g., "case resolved Oct 2025") — admin approves before live

### What Users Cannot Do
- Post free-text comments on cases (too much moderation burden, too much defamation risk in V1)
- Name new suspects in unsolved cases
- Link to personal social media accounts of suspects/persons of interest
- Post contact information for any person connected to a case
- Upload images (not until we have a full moderation pipeline)

### When We Add Annotations / Community Notes (Phase 3+)
If we add community notes (Wikipedia-style or X-style), we need:
- **Pre-moderation for named living people** — no annotation goes live without admin review if it names a living person
- **Structured format only** — not freeform text. E.g., "Update: [person] was acquitted on [date]. Source: [link]." Templates reduce abuse.
- **Source required** — annotations must cite a verifiable source
- **Cooldown period** — no annotations on cases within 30 days of major media coverage (reduces pile-ons)

### Moderation Infrastructure (Phase 2 requirement)
- **Report queue** with < 24h SLA
- **Shadow-flagging** — flagged content hidden from discovery but visible to user until reviewed (reduces chilling effect)
- **Strike system** — 3 policy violations = account suspension
- **Appeals process** — email only, 48h response SLA

---

## 6. Podcast Content & Copyright

### What We're Doing
- Displaying episode metadata (title, description, date, artwork) — this is fair use / publicly available data
- Streaming audio from RSS URLs — legally the same as any podcast app (Overcast, Pocket Casts)
- Linking back to source show and publisher

### What We're Not Doing
- Re-hosting audio files (hosting = liability, cost, DMCA exposure)
- Transcribing and displaying full episode text (copyright issue)
- Stripping ad tracking from RSS URLs (against most publisher ToS)

### DMCA
- Register a DMCA agent with the US Copyright Office ($6/yr) — required for safe harbor protection
- Have a standard DMCA takedown process (receive notice → review → remove if valid → counter-notice option)
- Log all DMCA requests

### Publisher Relations
- Long-term: formal partnerships with major TC podcast networks (Audiochuck, Wondery, iHeart TC) for verified episode data and potentially preferred placement
- Short-term: be a good citizen — respect attribution, link to shows, don't block their analytics

---

## 7. Terms of Service Summary

Key provisions to include (work with a lawyer for actual drafting):

1. **Age requirement:** Users must be 18+ (recommended over 13+, given content)
2. **Content disclaimer:** Case information is for educational/entertainment purposes. We are not law enforcement. Do not use this information to contact or approach anyone.
3. **No harassment:** Using CaseLog to identify, locate, or contact victims, suspects, or their families is a permanent ban offense.
4. **No doxxing:** Posting personal information about real people is prohibited.
5. **No speculation as fact:** Community contributions may not present theories as established fact.
6. **Liability limitation:** We are not liable for inaccurate case information; this is a curated index, not legal record.
7. **Takedown rights:** We reserve the right to remove any case or episode listing at our discretion.
8. **Governing law:** California (where Floppy is registered).

---

## 8. What To Do Before Launch

### Before V1 (Web Launch)
- [ ] Draft privacy policy + terms of service (use a lawyer or at minimum a reviewed template)
- [ ] Register DMCA agent: https://www.copyright.gov/dmca-agent/
- [ ] Add content warning fields to case schema (required for data model)
- [ ] Add legal status labels to case schema (alleged/convicted/acquitted/etc.)
- [ ] Add age gate to signup
- [ ] Add case status banners for ongoing trials
- [ ] Data review: ensure top 200 cases use correct legal status language throughout

### Before Community Features (Phase 2)
- [ ] Publish community guidelines / content policy (public-facing)
- [ ] Build report queue + admin dashboard
- [ ] Establish moderation SLA (48h for reports)
- [ ] Legal review of community notes format before enabling

### Before Mobile Launch (Phase 4)
- [ ] App Store review guidelines compliance (content warnings, age rating)
- [ ] App should be rated 17+ (Apple) / Mature (Google Play) due to crime content
- [ ] Verify push notification consent flow is GDPR-compliant

---

## 9. Reference: How Others Handle This

**Websleuths (20+ years of TC moderation):**
- Heavily moderated forums — every post reviewed in high-profile ongoing cases
- Named suspects in unsolved cases are handled with extreme care
- Family members who request thread removal often get it
- https://www.websleuths.com/forums/threads/rules-and-guidelines.1/

**Wikipedia true crime articles:**
- Strict sourcing requirements — only cites established media, court records, official statements
- "Alleged," "convicted," "acquitted" used precisely
- BLP (Biographies of Living Persons) policy is extremely conservative

**Letterboxd (our model for the product):**
- No review moderation for film critiques (films aren't people with legal status)
- CaseLog is different — our "films" are real people and real crimes. Letterboxd's moderation model doesn't apply here.

---

## 10. Unsolved Cases with Living Named Suspects — Special Handling

This is the highest-risk category for CaseLog. An unsolved case where a living person has been named (by media, podcasts, or community speculation) as a suspect requires extra gates before publication.

### Policy
- **No auto-publish** for case pages in this category. Human review required before the case goes live.
- The AI classifier must NOT automatically link podcast episodes to a case if the episode names a living person as a suspect and that person has not been charged.
- Case page must carry "Unsolved — No charges filed" status prominently.
- Person entries for named-but-uncharged individuals must use `no_charges_filed` status with prominent display: *"[Name] has not been charged in connection with this case."*
- No "suspects" section on the case page — instead: "Persons named in media coverage" with legal status for each.
- Episode index for this case type: manual review only. No AI auto-approval.

### "Request Review" Contact
Every case page must include a visible link/email for removal or amendment requests. Families, victims, persons of interest, and their representatives must have a real path to reach us. Response SLA: 48 hours acknowledgment, 7 days decision.

Showing good faith in responding to these requests matters enormously in any future litigation. Document every request and response.

### Precedent: Reddit/Boston Marathon Bombing (2013)
Reddit's r/findbostonbombers misidentified Sunil Tripathi (a missing Brown student) as a bombing suspect. His family learned their missing son was being publicly named as a terrorist from news coverage. He had already died. Reddit issued a formal apology. The *New York Post* — which published a front page with photos of two innocent men — settled a defamation lawsuit for an undisclosed sum. Section 230 protected Reddit as a platform; it did not protect the Post, which created and published the content.

CaseLog, as a curator and organizer of podcast content, sits closer to the Post end than the Reddit end when it comes to case pages we write and maintain.

---

## Disclaimer

_This document is an internal planning framework. It is not legal advice. Before launch, have a licensed attorney review the privacy policy, terms of service, DMCA registration, and any content policies that reference specific legal standards. This is especially important for cases involving ongoing trials, living suspects, and international jurisdictions._

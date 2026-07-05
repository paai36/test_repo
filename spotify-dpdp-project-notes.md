# Spotify DPDP Consent Experience — Project Notes
Vedantu PM Intern Case Study | Focus: Children's Data & Consent

---

## 1. Problem Statement

- **Product**: Spotify (India)
- **Focus area**: Consent experience for minors — no age verification or parental consent mechanism exists today
- **Why it matters**:
  - Legal risk: DPDP mandates verifiable parental consent for under-18 data processing; penalties up to ₹250 crore
  - User trust: parents have zero visibility into what a minor's account collects
  - Business opportunity: first-mover trust differentiator in India's family-heavy household internet usage
- **Assumptions**:
  - "Child" = under 18 per DPDP (not under 13, the US COPPA standard)
  - No public data exists on Spotify India's under-18 user base, since no age-verification exists today — this absence of data is itself evidence of the gap
  - Estimate used: **10–15% of Spotify India's free-tier base are unverified minors**, based on India's youth population skew (not a sourced Spotify-specific stat — flagged explicitly as an estimate)
  - No existing consent-manager integration (greenfield build)
  - Parents have their own separate, verifiable contact info (email/phone)

---

## 2. User / Personas

**Primary**: Parent of a 14-year-old with an undetected Spotify account — worried about data collected on their minor, no visibility or control today.

**Secondary (added nuance)**: Teen living away from home (hostel/PG in a metro city for study) whose parent lives in a rural/low-connectivity area. This stress-tests the consent flow — the teen has full digital access, but the parent may not (basic phone, patchy internet, lower digital literacy).

**Why this matters for design**: Assuming "parent taps a link" as the default consent mechanism breaks for the second persona. This drives the SMS/voice-fallback design decision below.

---

## 3. Current Experience (assumptions stated)

**Collection**: email/phone/name, self-declared unverified DOB, listening history, search/skip patterns, device ID, approximate location, contacts (if Find Friends enabled), Facebook/Google login data, Ad ID/engagement.

**Use**: personalization (Discover Weekly), ad targeting (free tier), product analytics, location-based content.

**Sharing**: third-party ad partners (free-tier ad personalization), analytics/infra vendors (standard SaaS processors). Assumption: no direct data sale, but broad "service provider" sharing clauses typical of consumer apps.

**How consent is asked today**: one bundled "Agree to Terms & Privacy Policy" checkbox at signup — covers all data uses in one action. No itemisation.

**Where it falls short of DPDP**:
- Not "specific" consent (Section 6) — bundled, not itemised
- No age verification, so children's-data provisions never trigger
- No verifiable parental consent
- Not meaningfully "informed" — legal-language policy, not plain-language in-context notice
- No easy withdrawal mechanism (violates "withdrawal as easy as consent")
- No per-purpose data-use notice

**Key friction/risk points**:
- Self-declared age trivially bypassed
- No fallback flow even if a user honestly enters an under-18 DOB
- Bundled consent = every free-tier signup is technically non-compliant
- No consent audit trail for regulators

---

## 4. Data Handling for Minors — DPDP Section 9 Mapping

DPDP restricts some processing **structurally**, regardless of parental consent — parents cannot "consent away" certain protections for a child.

| Data Type | Adult Flow Today | DPDP Rule for Minors | Required Upgrade |
|---|---|---|---|
| Account/DOB | Self-declared, unverified | Must trigger age-gate + parental verification | Verified age-gate logic |
| Listening history | Used for personalization | OK, but can't feed targeted-ad profiling | Split: personalization only, blocked from ad pipeline |
| Location | Used for regional charts/ads | Restrict to coarse/regional only | Downgrade to city-level, excluded from ad data |
| Contacts (Find Friends) | Opt-in sync | High child-safety risk | Off by default; parent must explicitly opt in |
| Ad ID/targeting | Personalized ads | **Hard-blocked for children**, not consent-gated | Generic/contextual ads only, always, for minors |
| Third-party ad sharing | Shared with ad partners | Must not share minor data for targeting | Structurally exclude minor accounts from ad pipeline |
| Analytics | Individual + aggregate | Must be aggregated, not individually profiled | Aggregate-only for minor cohorts |

---

## 5. Proposed Solution

### A. Consent Architecture
- Age-gate at signup → branches into Guarded Signup for minors
- Layered consent for adults: mandatory core + itemised optional toggles (ads, contacts, location)
- Verifiable parental consent for minors: magic-link (+ SMS/voice fallback) verification, itemised toggles, parent dashboard
- Withdrawal parity: one-tap revoke, same ease as granting

### B. Safe Mode (headline feature)
Grace-period access instead of a hard block — teen gets immediate, limited access (core music only, no ads-targeting/contacts/precise location) while parental verification runs in the background. Full features unlock once parent consents.
- Avoids the "wall" that kills signups
- Ad-ID/targeted ads: permanently off for minors, even after consent (structural, not consent-gated)
- Contacts: off until parent opts in
- Location: always downgraded to city-level for minors

### C. Data Minimisation
- Ad-ID data purged automatically on consent withdrawal, not just flagged inactive
- Collect/retain only what's needed for the specific consented purpose

### D. User Rights Enablement
- Family Privacy Dashboard (parent-facing): view collected data, correct DOB, request erasure
- Self-service export/delete for adult users, defined SLA
- Erasure cascades correctly without breaking shared family accounts

### E. Accessibility / Equity Additions
- SMS-based verification (works on basic phones, no data needed)
- Voice-call OTP / IVR consent option for parents uncomfortable with digital links
- Rationale: verification shouldn't assume urban digital fluency — an email/app-link-only flow silently fails exactly the population DPDP is most trying to protect

### Why this over alternatives
| Alternative | Why rejected |
|---|---|
| Hard-block minors until parent consents | Kills signups, high abandonment |
| ID/document verification | More invasive than the problem it solves |
| One-time consent only, no dashboard | Fails "withdrawal as easy as consent" |
| Consent-gate everything incl. ads for minors | Legally wrong — DPDP restricts targeted ads to children structurally |

---

## 6. Metrics

**North Star Metric**: % of identified minor accounts with valid, current parental consent
`(Minor accounts with active, non-expired, itemised consent) / (Total identified minor accounts) × 100`
Target: 70% within 90 days, 90% within 6 months.

**Supporting Metrics**:
1. Parental verification completion rate = (Parents completing consent) / (Parents receiving link) × 100
2. Median time-to-consent (parent contact entered → consent completed)
3. Safe Mode → Full Access conversion rate
4. (Optional) Granular toggle opt-in rate

**Guardrail Metrics**:
1. Teen signup abandonment rate at/after age-gate
2. Support/complaint volume tied to consent flow

**Additional "impress" metrics considered**:
- Consent Health Score (composite of NSM + completion rate + time-to-consent)
- Voluntary self-disclosure rate (users correcting DOB later — trust signal)
- Cost-of-compliance transparency (ad-revenue impact shown alongside NSM)
- Re-engagement rate 30 days after Safe Mode entry (proves Safe Mode isn't just a holding pen)

---

## 7. Diagnostic Thinking — NSM Drops 15% in Week 3, No Release

**Hypothesis tree**:
- A. Denominator issue — more minors identified (age-gate logic change/reclassification)
- B. Numerator issue — fewer valid consents (verification completion drop, deliverability failure, UX break, time-to-consent increase)
- C. Decay issue — existing consents expiring faster than renewed (batch expiry effect, broken reminder flow)

**Most likely cause given "3 weeks, no release"**: consent expiry batch effect — if consent validity window = ~21 days and most Week-1 users consented simultaneously, they'd expire together in Week 3.

**Data to analyse**: funnel breakdown by week, consent expiry timestamp distribution, verification channel deliverability logs, geographic/platform segmentation of the drop.

**Root cause approach**: check expiry distribution first (cheapest, matches timing) → check infra/deliverability logs → segment by platform/geography → only then consider genuine behavioral shift.

**Next steps**: patch confirmed stage → stagger future consent expirations structurally → move from 3-week to weekly NSM monitoring.

**Diagnostic timeline**:
| Day | Action |
|---|---|
| Day 1 | Detect via dashboard alert |
| Day 1-2 | Check expiry distribution + infra logs |
| Day 3-4 | Segment analysis if unresolved |
| Day 5 | Root cause confirmed |
| Day 6-7 | Hotfix shipped |
| Week 2 | Monitor recovery; implement staggered-expiry fix |
| Week 3-4 | Confirm NSM back to baseline, postmortem |

---

## 8. Dashboard Design (mock data)

**Main KPI**: % Minors with Valid Consent — e.g. 68%, ▲3% vs last week

**Trend over time**: 12-week line chart; 0% pre-launch → ~45% by week 4 (dip at week 3) → recovers to ~68-70% by week 12. Markers at "Launch" and "Expiry batch fix deployed."

**Supporting metrics cards**: Verification completion 72%, median time-to-consent 4.2 hrs, Safe Mode→Full conversion 58%, teen abandonment 12%.

**Segmentation**:
1. Geography: Metro 74% vs Tier-2/3 58% — insight: lower digital literacy/connectivity in smaller cities
2. Platform: iOS 71%, Android 65%, Web 60% — insight: web verification may have more friction
3. (Added) Rural vs. urban parent-location — expect lower completion in rural-parent segments, directly tied to the hostel-teen/rural-parent persona

---

## 9. Rollout Plan

| Phase | Duration | What | Why |
|---|---|---|---|
| 1. Shadow mode | 2 weeks | Log only, no real restrictions | Validate detection logic risk-free |
| 2. Soft launch | 4 weeks | Live in Tier-1 metros only | Higher literacy, easier support, faster signal |
| 3. National rollout | Ongoing | Expand after Phase 2 stabilizes | Confidence before scale |
| 4. Retroactive sweep | 3+ months, staggered | Existing users re-confirm DOB gradually | Avoid mass-abandonment spike |

**Validation**: internal QA + small opt-in beta pre-launch; weekly NSM/guardrail dashboard review post-launch with defined rollback trigger (e.g. teen abandonment > 20% pauses expansion).

**A/B testing**:
- CAN test: verification channel (email/SMS/both), reminder timing, consent screen copy — UX layer only
- CANNOT test: whether parental consent is required at all, whether Safe Mode applies — unequal legal treatment across test groups is itself a compliance risk. These roll out sequentially by geography with kill-switches instead.

---

## 10. Risks & Trade-offs

**Engineering complexity**: partial-permission state (Safe Mode) needs feature flags per data category, not per account; consent expiry/renewal scheduling system; new parent-child account linking relationship; staggered retroactive-sweep sequencing.

**Business impact**: Safe Mode reduces ad-targeting data/revenue on minor segment (explicit cost of compliance); some irreducible signup loss from verification friction — needs an accepted "reasonable range," not zero-tolerance.

**User impact**: teens with unavailable/uncooperative parents may get stuck in Safe Mode long-term (equity concern — single-parent households, estranged families); parents get an added task even if minimized.

**Edge cases**: user declines consent entirely (stays in Safe Mode, clear non-punitive messaging); parent revokes consent (must downgrade immediately, not next login); teen turns 18 (auto-exit via birthday trigger); parent requests deletion (must cascade without breaking shared account).

**Trade-offs accepted**:
- Friction + trust over frictionless + risk
- Sequential/phased rollout over true A/B testing for compliance elements
- Safe Mode's engineering cost over a simpler hard-block, to preserve the minor user segment

---

## 11. Prototype Scope (Happy Path Only)

Tool: Figma (or web app) — clickable, end-to-end, happy path only, edge cases optional per brief.

**Screens**:
1. Signup — DOB entry → system detects under-18
2. "Ask a parent to continue" — teen enters parent's phone number
3. Safe Mode activated instantly — teen lands in limited app (core music only)
4. Parent side: SMS received → taps link → itemised consent screen (Core-mandatory, Ads, Contacts, Location toggles)
5. Parent taps "Approve"
6. Teen's app auto-upgrades from Safe Mode to Full Access per what parent approved
7. (Optional) Parent mini-dashboard: "what's approved, revoke anytime"

**Assumptions for prototype**: mock phone/OTP (no real SMS integration), mock data throughout, no real Spotify backend.

---

## 12. Interviewer Narrative — How I Approached This

- Started by asking: *"Where does Spotify's current experience most clearly violate a specific DPDP requirement, not just 'feel unsafe'?"* → age-gating/parental consent was the sharpest, legally-specific gap.
- Framed the core design tension as: any parental step causes some abandonment — the goal isn't to eliminate friction, it's to reduce it intelligently. Safe Mode is the answer: value now, compliance close behind.
- Used "withdrawal must be as easy as consent" as a hard design constraint, not just a compliance checkbox.
- Deliberately distinguished **hard-blocked vs. consent-gated** data categories for children — parents can't consent away a child's protection from targeted advertising under DPDP Section 9. This is the most legally precise point in the deck.
- Added the rural-parent/urban-teen persona specifically to stress-test the assumption that "parent access = teen's access" — led to the SMS/voice-fallback design decision, preventing the consent flow itself from becoming a digital-divide filter.
- Structured metrics as NSM (is the goal actually met) → Supporting (why it's moving) → Guardrails (are we losing the business/users to get there) — explicit framework, not ad hoc numbers.
- On A/B testing: explicitly separated what's safe to test (UX layer) from what's not (compliance-critical, unequal legal treatment risk) — chose sequential rollout with kill-switches for the latter.

---

## Evaluation Criteria Coverage

| What They Look For | Where It Shows Up |
|---|---|
| Strong user empathy | Two personas incl. rural-parent/urban-teen; Safe Mode avoids punishing teens; SMS/voice fallback |
| Clear, structured thinking | Full Problem→User→Experience→Solution→Metrics→Diagnostics→Dashboard→Rollout→Risks flow |
| Product intuition | Safe Mode as bridge not wall; hard-blocked vs. consent-gated distinction |
| Data-driven decision-making | NSM/Supporting/Guardrail structure; hypothesis tree; segmentation |
| Meaningful success metrics | NSM tied directly to compliance outcome |
| Sensible prioritization | Chose one high-impact area (minors' consent) over shallow full-DPDP coverage |
| Awareness of trade-offs incl. privacy/regulatory | Explicit trade-offs section; A/B-testable vs. not distinction |

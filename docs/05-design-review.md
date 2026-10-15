# 05 — Design Review of Documentation v0.1

Version 1.3 · 2026-10-15 · v1.1 added §9 (resolution of the pre-commit audit); v1.2 added §10 (focused security and distributed-systems review); v1.3 adds §11 (final pre-commit gate). Reviewed: [01](01-market-research.md), [02](02-product-requirements.md), [03](03-mvp-and-architecture.md), [04](04-open-decisions.md), [ADR-0001..0003](adr/)

This review records what was wrong, inconsistent, or missing in v0.1 and where each item is now addressed. Earlier documents were corrected in place, and each carries a changelog entry pointing back here.

---

## 1. Conflicting requirements

| # | Conflict | Where | Resolution |
|---|---|---|---|
| C-1 | `SKIPPED` is a queue **state** in FR-QUE-3, but only a **transition** (CALLED → WAITING) in the 03 state diagram. | 02 FR-QUE-3 vs 03 §4.3 | Skip is a transition with a counter, not a state. See [queue-lifecycle](domain/queue-lifecycle.md). |
| C-2 | Service progress (`IN_SERVICE`, `COMPLETED`) appears in **three** lifecycles: appointment, queue entry, and visit. The copies would drift apart. | 02 FR-QUE-3, 03 §4.3–4.4, BR-11 | Only **Visit** owns service progress. An appointment ends in `FULFILLED` and a queue entry ends in `SERVED`, each pointing to the visit. See [ADR-0004](adr/0004-separate-appointment-queue-entry-visit.md). |
| C-3 | BR-2 says a barber cannot have overlapping *appointment or in-service* blocks. 03 §4.5 says haircuts may run long, and the constraint in 03 §4.1 covers appointments only. | 02 BR-2 vs 03 | Split: **planned commitments** (appointments, time off) cannot overlap and the DB enforces it. **Actual service** may overrun; this is reality and is handled by ETA and notifications. |
| C-4 | The `staff_commitment` table had `appointment_id NOT NULL` but was also meant to store time off. | 03 §4.1 | Table renamed `scheduling.staff_commitment` with a `kind` column and an optional source. See [database-design](architecture/database-design.md). |
| C-5 | FR-CAT-5 snapshots price "at confirmation" for queue entries too. But with per-barber pricing (D-12) and "any barber", the price is **unknown until a barber is assigned**. | 02 FR-CAT-5 vs FR-CAT-3, FR-QUE-1 | A queue entry stores a price **estimate range** at join. The **binding snapshot** is taken at the visit start. Appointments snapshot at confirmation because the barber is known then (D-26). |
| C-6 | A remote entry in `WAITING` can transition to `CALLED`, even though the customer may not be in the shop yet. Presence and position were mixed into one state. | 03 §4.3 | **Presence** (`NOT_ARRIVED`/`ARRIVED`) becomes a separate attribute. Only arrived entries can be called. |
| C-7 | FR-REV-3 shows reviewers as "first name + last initial". Arabic names often don't split into first/last name, and users may not want their real name shown. | 02 FR-REV-3 | The reviewer's public display name is chosen by the user (default: their first word of name), and the user can edit it. |
| C-8 | The "one active remote entry per customer" index (BR-7) runs across tenants, while RLS isolates tenants. | 02 BR-7, ADR-0002 | This is acceptable: unique indexes are enforced below RLS. The resulting conflict is reported only to the customer, about their own entry. Documented in [security model](architecture/tenant-isolation-and-security.md). |

## 2. Inconsistent terminology

| Used interchangeably | Canonical term (see [glossary](domain/domain-glossary.md)) |
|---|---|
| booking / appointment | **Appointment** (entity). "Booking" means only the *act* or the module name. |
| waitlist / queue | **Queue** = live walk-in line. **Slot waitlist** = P1 "notify me when a slot opens". "Waitlist" alone is banned. |
| customer / customer record / customer relationship / CustomerVisit | **User** (platform identity), **Customer** (tenant-owned relationship record), **Visit** (service actually delivered) |
| staff / barber / StaffProfile / member | **Membership** (user ↔ tenant + role), **Staff member** (bookable service provider, may have no user account). "Barber" is the vertical-specific label for a staff member. |
| tenant / business | **Tenant** = isolation + billing unit. **Business** = public brand. 1:1 in the MVP. |
| branch / location / shop | **Branch** (physical place). "Shop" is used only in UI copy. |
| lane | **Lane** = a *computed view* of one staff member's upcoming work. It is not stored. |

## 3. Missing business rules (added)

| # | Gap | Now covered in |
|---|---|---|
| M-1 | **Shops open past midnight** (common in Egypt). There was no *business day* concept for queues, tickets, reports, or end-of-day expiry. | D-22; [queue-lifecycle §2](domain/queue-lifecycle.md) |
| M-2 | **Groups** (e.g., a parent with kids). | D-21 (resolved: separate entries, linked) |
| M-3 | **Minors' data.** Under Egypt's PDPL executive regulations, processing data of children under 15 requires the guardian's explicit written consent (reported Art. 15). | D-24 (legal) |
| M-4 | **Late arrival for an appointment** after the grace period: what happens? | D-27; [booking-lifecycle](domain/booking-lifecycle.md) |
| M-5 | Can a barber serve two customers at once? | D-28 |
| M-6 | **Staff without a smartphone/account** (the front desk operates for them). | D-29 |
| M-7 | **Phone number reuse and family phones.** Linking a walk-in record to a platform user by phone can expose another person's history. | D-25; [domain-model §5](domain/domain-model.md) |
| M-8 | **End of day**: what happens to entries still waiting at closing time? | [queue-lifecycle](domain/queue-lifecycle.md) |
| M-9 | A **direct serve** (barber takes a customer with no entry or appointment). | Modeled as a queue entry created and served in one step, so every visit has a source. |
| M-10 | **Reschedule semantics**: is it a new appointment or a modified one? Does the price change? | [booking-lifecycle §4](domain/booking-lifecycle.md) |
| M-11 | **Barber leaves the business**: what happens to their future appointments, reviews, and customer ownership? | D-5c resolved (business keeps customers); [domain-model §3](domain/domain-model.md) |
| M-12 | **Customer has an appointment *and* joins a queue** at the same or another shop. | [queue-lifecycle §6](domain/queue-lifecycle.md) |
| M-13 | **Informal Egyptian addresses** (landmarks, areas). Search "by area". | [domain-model §2](domain/domain-model.md) (branch address model) |
| M-14 | **Correcting a completed visit** (wrong price or service entered). | [queue-lifecycle §5](domain/queue-lifecycle.md) (visit corrections, audited) |

## 4. Assumptions presented as facts (now labeled)

| Statement (v0.1) | Issue | Fix |
|---|---|---|
| "Verified-visit reviews are the industry norm" (04 D-10) | Based on 2 vendors | Reworded to "Booksy and Fresha both require…" |
| "Google is usually strongest for MENA addresses" (04 D-14) | Unverified opinion | Labeled as hypothesis; added to the provider evaluation in the [validation plan](06-validation-plan.md) |
| "Thousands of shops fit one PostgreSQL instance" (03 §2) | No load model | Labeled ASSUMPTION-10; load test in the roadmap |
| "One barbershop queue sees a few writes per minute" (ADR-0003) | Unmeasured | Labeled ASSUMPTION-11; measure in the pilot |
| "Seniority-based pricing is common" (04 D-12) | Was already labeled | Kept; decision now "design for it, gate the UI" |
| "Prisma is awkward for RLS" (03 §3) | Opinion | Reworded as a trade-off |
| NFR targets | Already labeled | No change |

## 5. Architecture choices that depend on unresolved business requirements

| Architecture choice | Depends on | Status |
|---|---|---|
| Hosting region, managed services, observability vendors (03 §3) | **D-23**: PDPL cross-border transfer (needs a licence when hosting outside Egypt) | **Open, legal.** New [ADR-0005](adr/0005-hosting-and-data-residency.md) (Proposed). The design stays portable: no lock-in to proprietary managed services. |
| Push notifications via APNs/FCM (inherently cross-border) | D-23 | Payloads carry no personal data (a "you have an update" message plus an id). Content is fetched in-app. |
| WhatsApp/SMS provider | D-8, D-23 | Open |
| Shared-schema tenancy (ADR-0002) | D-5 (now resolved: employees-only) | The risk of cross-tenant queues is removed for the MVP. ADR-0002 stays Proposed pending technical sign-off. |
| Phase order: queue before booking (03 §1.3) | ASSUMPTION-1 | Still depends on validation (H1 in the [validation plan](06-validation-plan.md)). |
| Review aggregates per barber | D-10 (resolved: business + barber) | Covered: aggregates per branch and per staff member. |

## 6. Risks that could force significant database redesign

Ranked by cost of a late change.

| # | Risk | Mitigation in this design |
|---|---|---|
| RD-1 | Cross-tenant queues (chair-renters as their own tenant inside a host shop) | Ruled out for the MVP (D-5). A future solution would be a "host branch" agreement, not shared tables. |
| RD-2 | Data must live in Egypt | No cross-region features; one region per deployment. UUIDs and no cross-tenant FKs make relocation possible. |
| RD-3 | Children as first-class customers | Attendee labels are optional, no birth dates are stored, and guardian consent records are separate. Adding child profiles later would be additive. |
| RD-4 | Resources other than staff (chairs, rooms, bays) for future verticals | `staff_commitment` can be generalized to `resource_commitment` with a resource type. We keep the barber MVP staff-only, but the constraint is written on a `resource_id` column, so the change is a rename, not a redesign. |
| RD-5 | Multi-barber services or parallel services | One barber per appointment/visit in the MVP. Service lines carry their own staff id (nullable), leaving room for growth. |
| RD-6 | Tenant ↔ Business 1:n | Kept as separate tables now (1:1 enforced by a unique constraint that can be dropped later). |
| RD-7 | Multi-currency / multi-country tenants | Currency is stored on the tenant *and* on every money snapshot. Branch-level currency can be added later. |
| RD-8 | Composite tenant integrity | Every tenant-owned FK is `(tenant_id, id)`, so a child row cannot reference another tenant's parent, even through an application bug. |

## 7. Changes made in this review

- **Updated:** [02](02-product-requirements.md) (FR-QUE-3, FR-CAT-5, BR-2, FR-REV-3, new BR-12…BR-20, ASSUMPTION-10/11, Egypt specifics), [03](03-mvp-and-architecture.md) (module map adds `visit`; state machines now link to domain docs; hosting caveat), [04](04-open-decisions.md) (decision review, resolutions, new decisions D-21…D-29), [01](01-market-research.md) (wording fix only).
- **New:**
  - [06-validation-plan.md](06-validation-plan.md)
  - [domain/](domain/)
  - [architecture/](architecture/)
  - [ADR-0004](adr/0004-separate-appointment-queue-entry-visit.md)
  - [ADR-0005](adr/0005-hosting-and-data-residency.md)

## 8. Sources for Egypt-specific facts used

- PDPL executive regulations, Ministerial Decree 816/2025 (Nov 2025); compliance period reported to end 1 Nov 2026; cross-border transfer licence (Art. 16); breach notification to the regulator within 72 h (Art. 5); children under 15 need guardian's explicit written consent (Art. 15); direct-marketing licence (Art. 28):
  - Al Tamimi: https://www.tamimi.com/law_update_articles/from-policy-to-practice-egypt-issues-executive-regulations-of-the-personal-data-protection-law/
  - Chambers: https://chambers.com/articles/a-first-look-at-egypt-s-personal-data-protection-executive-regulations
  - DataGuidance: https://www.dataguidance.com/jurisdictions/egypt
  - Sources disagree on the decree number (816 vs 81) and on whether the regulator is operational. **Legal counsel must confirm.**
- DST: Law 34/2023, last Friday of April → last Thursday of October; 2026: 24 Apr → 29/30 Oct. https://www.timeanddate.com/time/change/egypt?year=2026 · https://www.egypttoday.com/Article/1/149478/et-guide-When-will-Egypt-switch-to-winter-time
- Consumer Protection Law 181/2018, Art. 9 (misleading price, including VAT): https://eg.andersen.com/wp-content/uploads/2025/07/Law-No.-181-of-2018.pdf (secondary summary; counsel to confirm display rules)

## 9. Pre-commit audit — resolutions (round 2, 2026-10-14)

The pre-commit audit of the uncommitted v0.1→v0.2 changes found blocking items B1–B6, smaller items N1–N6, and validation/privacy items. Each is classified below:
- **Fixed:** the documents now support the conclusion.
- **Superseded:** old text is kept but explicitly marked.
- **Open decision:** recorded in [04 §I](04-open-decisions.md).
- **Unresolved:** with a reason.

**No product decision was approved in this round.** Every new rule is proposed or provisional.

| Finding | Classification | Where |
|---|---|---|
| **B1** Remote rule rejected groups (I-5 vs D-21) | **Fixed.** Remote queue *session* (one ACTIVE per user, product rule P-1, partial unique index on `remote_session`); entries are bound to their session's queue by a composite FK; the session ends in the transaction of the last entry's terminal transition. The old index is withdrawn. Review item C-8 (§1) is superseded by P-1. | [queue-lifecycle §3](domain/queue-lifecycle.md), [database-design §5.2](architecture/database-design.md), [domain-model §10](domain/domain-model.md), PRD FR-QUE-9/BR-7, 04 D-3 |
| **B2** Review path vs D-25 history rule | **Fixed** (design). Per-visit claim invitation (single-use hashed token + same-phone OTP) grants ownership of *one* visit. Customer identity, visit ownership, and review eligibility are separated. No backfill on linking. Residual shared/recycled-phone risk → **Open D-30**. | [domain-model §7](domain/domain-model.md), [database-design §5.3](architecture/database-design.md), [security §5.2](architecture/tenant-isolation-and-security.md), PRD FR-REV-1/9, BR-18, AC-REV-claim |
| **B3** Stale lifecycle text | **Fixed / Superseded.** 03 §4 rewritten as a summary; v0.1 drafts moved into a marked historical block. ADR-0003 Decision updated. PRD AC-REV/AC-QUE fixed; FR-QUE-10 → EXPIRED only (enforced by a CHECK); FR-BKG-3 link fixed. | 03 §4, ADR-0003, PRD §5/§8 |
| **B4** Lane changes bypassed `seq` | **Fixed.** A single queue mutation protocol (lock + one `seq` per committed transaction + `(seq, idx)` events + outbox; rollback consumes nothing), with an authoritative list of mutations, retry rules, and client recovery. | [database-design §6.1](architecture/database-design.md), [queue-lifecycle §7, §9](domain/queue-lifecycle.md) |
| **B5** Closures could invalidate appointments | **Fixed** (workflow) with **Open D-31** for the policy choice. Reject-until-resolved, resolution set recomputed under a branch lock, audit, notifications, booking takes a branch share lock, daily integrity job. | [booking-lifecycle §5a](domain/booking-lifecycle.md) |
| **B6-1** RLS inferred from `tenant_id` | **Fixed.** Classification registry + CI check. | [database-design §3.0](architecture/database-design.md), [security §10](architecture/tenant-isolation-and-security.md) |
| **B6-2** Public reviews view vs view allowlist | **Fixed.** Replaced by the `discovery.published_review` projection table. Allowlist of definer views/functions is empty. | database-design §3.11, security §5.4 |
| **B6-3** Owner role at runtime | **Fixed.** `turnio_maint` (separate task) and `turnio_migrator`; `turnio_owner` is NOLOGIN. | database-design §9, security §3 |
| **B6-4** Policies without `TO` | **Fixed.** Every policy is `TO`-scoped; access matrix added. | security §3.1, §5 |
| **B6-5** Identity and notification tables unprotected | **Fixed.** PLATFORM_PERSONAL class with own-subject policies per role; separate `turnio_auth` role. | security §5.6–5.7 |
| **B6-6** Customer commands had full business context | **Fixed** (design change, **needs review**): separate `turnio_app_customer` role; queue-entry personal data split into `queue.queue_entry_party`; `staff_live_status.expected_free_at` so ETA needs no visit rows; own-row policies on `crm.customer`. | security §5.3, database-design §3.4/§3.6 |
| **B6-7** Admin grant "per session" wording | **Fixed.** Transaction-local `set_config(..., true)`. | security §4, §5.5 |
| Idempotency "in progress" branch unreachable (found while fixing) | **Fixed.** The real PostgreSQL blocking behavior is documented; only successful outcomes are replayed. | database-design §7 |
| **N1** Same-day slots during overrun | **Open D-32** | 04 §I, booking-lifecycle §4 |
| **N2** Direct appointment start | **Open D-33** | 04 §I, booking-lifecycle §2 |
| **N3** Converted placement / `arrive_by` movement / cooldown scope | **Open D-38 / D-34 / D-35** | 04 §I, queue-lifecycle |
| Self check-in P0 vs P1 | **Open D-36** (PROPOSED P0 for remote customers with staff fallback; arrival confirmer always recorded) | 04 §I, PRD FR-QUE-14, J1 |
| **N4** Portable-profile text | **Superseded** (04 D-5 row struck through; PRD P6 and 01 differentiator #6 corrected) | 04, 02, 01 |
| **N5** TV display / rollover home / D-28 wording / ADR-0004 claim | **Fixed** (ticket numbers only; `tenancy.branch.day_rollover_minute` is authoritative; D-28 marked provisional; ADR-0004 corrected) | PRD FR-QUE-11, queue-lifecycle §1/§10, 04 H, ADR-0004 |
| **N6** Future-vertical items | **Classified.** `PROVIDER` role code and `resource_id` = low-cost neutral naming now (D-37, proposed). One queue per branch, D-28, and visit source XOR = deliberate MVP limits. Resource types, multi-provider services, and category policy engines wait for a real second vertical. | [domain-model §11](domain/domain-model.md) |
| Validation: adults only, A1 Q2 leading, A4 legal gate, raw notes outside Git | **Fixed** in the plan. The `.gitignore` entry for raw notes is **recommended, not applied** (awaiting approval). | [06](06-validation-plan.md) |
| Legal/privacy decisions (D-17, D-23, D-24), launch city (D-1b), maps (D-14), notifications (D-8) | **Still open**, unchanged. No compliance is claimed anywhere. | 04 §G |

**Known remaining limitations (unresolved, with reasons):**
- **Cross-table rules held by the application, not the DB.** A remote entry needs a party row with a `customer_id`, and a visit's source must be FULFILLED/SERVED. PostgreSQL CHECKs can't span tables; nightly integrity jobs and tests cover them.
- ~~`turnio_auth` has broad `user_phone` read access.~~ *Superseded by §10 F-1:* access is now one OTP-verified phone per transaction.
- **The self check-in presence proof (D-36 sub-question) has no chosen mechanism.**
- **ASSUMPTION-10/11 (volume, queue write rate) are unmeasured.**

## 10. Focused security and distributed-systems review (2026-10-15)

**Scope:** login and phone access, queue personal-data separation, transactional events, idempotency, and visit claims, followed by a consistency check.

**Classification:**
- **Fixed:** the existing design already covers it.
- **Clarified:** documentation clarification made in this round.
- **Open:** product decision.
- **Specialist:** requires legal or specialist security review.

| ID | Finding | Classification | Resolution / where |
|---|---|---|---|
| F-1 | `turnio_auth` could read every `user_phone` row, more than login needs | **Clarified** (design corrected) | OTP request never reads `user_phone`. After verification, policies expose only that phone (a verified, unconsumed challenge from the last 5 minutes); there are no plaintext-phone column grants and no privileged function or view. [security §5.7](architecture/tenant-isolation-and-security.md) |
| F-2 | `otp_challenge` stored plaintext phones | **Clarified** | `phone_hash` only. [database-design §3.1](architecture/database-design.md) |
| F-3 | OTP delivery was described as a notification, but the notification CHECK requires a user or customer recipient, which an OTP to an unknown phone has neither of | **Clarified** | OTPs are sent directly by the auth service and never stored in `notification`. security §5.6 |
| F-4 | Enumeration and SMS-pumping controls were thin | **Clarified** + **Open D-39** | Limits per phone/IP/device, a global breaker, verify-attempt lockout, alerts. Restricting to Egyptian mobiles is open as D-39. |
| F-5 | `queue_entry_party` per-role CRUD, organizer access, and account-less customers were not specified | **Clarified** | security §5.3a. The organizer owns all companion rows. No DELETE; anonymization instead (database-design §10). Account-less customers are staff-mediated; a status link is **Open D-40**. |
| F-6 | Customer role needed writes the matrix didn't allow (`queue.seq` increment, `queue_event` insert, own `queue_entry` updates, commitments) | **Clarified** | Narrow column- and row-level writes listed in security §5.3a; matrix updated. |
| F-7 | Customer `queue_entry` updates had no ownership predicate after the PII split | **Clarified** | UPDATE `USING EXISTS (own party row)`, plus column grants. |
| F-8 | Delivery guarantees were blurred | **Clarified** | database-design §6.3 (worked "complete visit" transaction) and §8.1 (commit / persistence / delivery / receipt). The relay enqueues and marks published atomically; jobs run at-least-once with transactional checkpoints; external sends are at-least-once (possible duplicates unless the provider supports idempotency, D-8); heartbeats carry the committed `seq`. **No exactly-once external delivery is claimed.** |
| F-9 | Idempotency scope could replay across users; cross-tenant replay depended on undocumented hash contents | **Clarified** | Key = `(user_id, surface, key)` with RLS `user_id = app.user_id()`; the request hash covers method, route, tenant, resource ids, and body. Concurrent identical requests: the second waits, then replays or proceeds. database-design §7 |
| F-10 | Claim invitations: a user could see every invitation sent to their phone in that tenant (a family member's visits) | **Clarified** | The policy also requires `token_hash = app.claim_token_hash()`. Token: 128-bit CSPRNG, SHA-256 at rest, not in logged URL paths, 14-day expiry, atomic redemption, rate limits. security §5.3, §8 |
| F-11 | A claim exposes only one visit | **Fixed** (existing design) | Visit policy = recorded owner or claim row; no backfill on linking |
| F-12 | Shared or recycled phones in claims | **Open D-30** (unchanged) | — |
| F-13 | HMAC pepper management (rotation means rehashing `phone_hash` columns), token and OTP parameters, rate-limit thresholds | **Specialist** | Security review before Phase 0 ends |
| F-14 | PDPL treatment of phone hashes (pseudonymous personal data), OTP logs, and research data | **Specialist (legal)** | D-17, A6 |
| F-15 | Two simultaneous same-key requests | **Fixed** (existing design) | PostgreSQL unique-insert waits; P-1 index and exclusion constraint are backstops |

**Remaining risks:**
- A fully compromised auth service can still issue sessions; this is inherent to any login component.
- External notifications may be delivered twice.
- Shared-phone claims (D-30).
- Cross-table rules held by the application, with nightly integrity checks.
- Assumptions about load and per-queue write rate are unmeasured.
- None of the PostgreSQL policy snippets have been executed yet. They are illustrative and must be proven by the Phase 0 test suite (security §10) before any of this is relied on.

## 11. Final pre-commit gate (2026-10-15)

**Documentation defects found and corrected** (smallest changes; no decision status changed):

| ID | Defect | Correction |
|---|---|---|
| G-1 | Phone-hash key rotation and compromise recovery were not documented (previously only "specialist") | Versioned hashes, rehash procedure, and compromise response: [security §5.7](architecture/tenant-isolation-and-security.md). The procedure still **requires specialist review**. |
| G-2 | OTP code parameters and concurrent-verification behavior were unspecified | Code length, expiry, one live challenge per phone, atomic attempt counting, single-use verification: security §5.7; database-design §3.1 |
| G-3 | The customer role could read identifier columns of other customers' queue entries (`arrival_confirmed_by_id`, `remote_session_id`, …) and `appointment_id` on commitments | Column-level SELECT grants: security §5.3a |
| G-4 | The customer role could read the staff-typed `display_name` (and business-private fields) of an unlinked `crm.customer` row bearing its phone, which may be a family member's record | Column grants exclude it: security §5.3a |
| G-5 | The context for customer status and real-time reads was undefined | `customer_command`, with the tenant resolved from the caller's own row: security §5.3a |
| G-6 | The relay atomicity claim didn't state that it depends on pg-boss living in PostgreSQL | Stated, along with Redis's limited role: database-design §8.1 |

**Gate status:** the documentation is internally consistent. **No runtime property has been tested.** Every SQL fragment is illustrative, and every security claim depends on the Phase 0 test suite (security §10).

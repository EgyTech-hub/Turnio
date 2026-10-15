# 04 — Open Decisions (blocking database design & implementation)

Version 0.3 · 2026-10-14 · v0.2 added §G decision review, §H new decisions, and the decision log (see [05 design review](05-design-review.md)). v0.3 adds §I (decisions raised by the audit, round 2; [05 §9](05-design-review.md)) and corrects D-3, D-5, and D-28 wording. v0.3.1 (2026-10-15) adds D-39 and D-40 from the focused security review ([05 §10](05-design-review.md)). **No new decision is approved.**

Each decision lists the options, our recommendation **[R]**, and what it blocks. **Blocking** = the database schema cannot be finalized without it.

## A. Market & business model

| ID | Decision | Options | [R] Recommendation | Blocks |
|---|---|---|---|---|
| **D-1** | Launch country/city, currency, languages | e.g., Egypt (Cairo), KSA (Riyadh), other | Pick **one city** with dense barber supply. Prefer a market where walk-ins dominate (validates the differentiator). | Currency, tax display, languages, data residency, providers. **Blocking** |
| **D-9** | Monetization at launch | Free pilot → subscription · subscription + acquisition fee · commission per booking | Free for pilot shops; subscription per branch afterwards. Postpone acquisition fees until there is meaningful marketplace traffic. Capture `acquisition_source` on every customer relationship from day 1 anyway. | Entitlements model; attribution fields |
| **D-19** | Team size, budget, and target date for the pilot | — | Needed to size the phases in [03 §1.3](03-mvp-and-architecture.md) | Roadmap |

## B. Domain model (blocking)

| ID | Decision | Options | [R] Recommendation |
|---|---|---|---|
| **D-5** | How are independent / chair-renting barbers modeled? Can a barber work for several shops? | (a) Barber is only staff of a shop. (b) A barber can be their own tenant **and** staff elsewhere. (c) Portable professional profile owned by the user, linked to tenant staff profiles. | ~~**(c) light:** a user-owned `ProfessionalProfile` (portfolio, bio) linked to 0..n StaffProfiles in tenants.~~ **SUPERSEDED 2026-10-14:** the product owner approved the employees-only MVP (see the decision log). The portable profile is deferred and not part of the MVP model. The original recommendation continues: Bookings and customer data stay with the tenant. An independent barber = a single-person tenant. Clarify who owns the customer list when a barber leaves (default: the tenant). |
| **D-5b** | Tenant ↔ Business cardinality | 1:1 · 1:n (holding company with several brands) | **1:1 for the MVP**, kept as separate concepts in code so a 1:n change is a migration, not a rewrite. |
| **D-2** | Queue structure | One queue per branch with barber preference · one queue per barber · both | **One queue per branch; entries carry preferred barber or "any".** Per-barber "lanes" are a computed view. Matches how shops work and how Great Clips assigns the next available stylist. |
| **D-6** | Customers without the app | Staff can only enter name · name + optional phone · required phone | **Name required, phone optional.** If a phone is given, the customer gets SMS/WhatsApp updates, and the record links to their account when they sign up and verify that phone. |
| **D-12** | Pricing granularity in the MVP | Base only · base + branch · base + branch + staff | **All three levels.** Seniority-based barber pricing is common (ASSUMPTION — confirm). It is cheap now and costly to retrofit. Add-ons, promotions, and tax rules wait for P1. |
| **D-12b** | Multi-service bookings | Single service · multiple services, same barber · multiple barbers | **Multiple services, same barber, run in sequence.** |

## C. Queue & booking policies

| ID | Decision | Options | [R] Recommendation |
|---|---|---|---|
| **D-3** | Remote queue join rules | Allowed? Max distance? Arrival window? Limit per customer? | Per-branch toggle (default on). Optional max distance (default off). Arrival window = ETA − 10 min … ETA + 10 min. ~~One active remote entry per customer platform-wide.~~ **v0.3: one active remote *session* per customer platform-wide; a session holds several linked people at one branch** (corrects the conflict with D-21). Repeated no-shows cause a temporary remote-join cooldown (scope: D-35). |
| **D-4** | Appointments vs walk-ins priority | Strict appointment priority · first-come regardless · "fit-in" rule | **Appointment priority at its time (+10 min grace). Walk-ins start on a barber only if expected duration fits before that barber's next appointment.** The manager can override with a reason (audited). |
| **D-11** | Cancellation / no-show defaults and enforcement in the MVP (no payments) | Record only · block after N · reliability score visible to shops | Record + show the shop a private count ("2 no-shows in 90 days"). Remote-join cooldown. Manual block list. Fees wait for P1 payments. |
| **D-7** | Online payments / deposits in the MVP | None · deposits only · full prepay | **None in the MVP** (ASSUMPTION-4). Choose a local PSP in P1. The model should still keep `price_snapshot` and `amount_charged` + `payment_method` (cash/card/other) on Visit. |

## D. Trust, privacy, compliance

| ID | Decision | Options | [R] Recommendation |
|---|---|---|---|
| **D-10** | Review policy | Verified-only? Window? Edits? Pre- or post-moderation? Display name? | **Verified completed visits only** (Booksy and Fresha both require this). 14-day window. Customer may edit within 30 days. Automated pre-check, then publish; reported reviews are hidden pending review only for severe categories. ~~Display "first name + initial".~~ *(Superseded by review C-7: a user-chosen public display name, FR-REV-3.)* Owner gets one reply. No incentives and no review gating. |
| **D-13** | Business verification before listing | None · phone + location check · documents (commercial registration) | **Phone verification + map pin within a radius of the geocoded address + admin review of photos.** Ask for documents only if fraud appears or local law requires it. |
| **D-17** | Applicable privacy law & data residency | Depends on D-1 (e.g., Egypt PDPL 151/2020, KSA PDPL) | Get legal input before the schema is finalized: it drives retention fields, consent records, and hosting region. **Blocking** |
| **D-18** | Data retention periods | — | Proposal: queue events in raw form 13 months, then aggregated. Location search inputs are not persisted. Deleted accounts are anonymized within 30 days; tenant financial/visit records are kept as legally required, with personal data pseudonymized. |

## E. Product surfaces & providers

| ID | Decision | Options | [R] Recommendation |
|---|---|---|---|
| **D-8** | Notification channels and who pays | Push only · push + SMS · push + WhatsApp · all | Push first. **WhatsApp** for "you're next" and "about to expire" if ASSUMPTION-2 holds; SMS as fallback and for OTP. The platform absorbs costs within plan quotas. |
| **D-14** | Maps & geocoding provider | Google Maps Platform · Mapbox · MapLibre + OSM/self-hosted tiles | Evaluate address/POI quality in the launch city first. Hypothesis (unverified): Google has the best address/POI coverage in Egypt but costs the most. Keep provider calls behind an adapter. |
| **D-15** | Customer channel at launch | Native app + web · web/PWA only · app only | **Expo app + SEO web.** The web shows profiles and can book and join queues, but iOS web push is unreliable for queue alerts unless the PWA is installed. |
| **D-16** | In-shop hardware | Staff phones only · shared tablet · kiosk/TV (P1) | Design the staff board for **one shared tablet + barbers' phones**. Kiosk/TV in P1. |
| **D-20** | Tax display | Tax-inclusive prices · exclusive | Follow launch-market consumer law (often tax-inclusive). Store `tax_inclusive` + rate on the snapshot even if tax config is P1. |

## F. Validation work required before decisions are final

1. **15–20 owner/barber interviews** plus **3 site visits at peak hours** in the launch city. These check ASSUMPTION-1, 4, 6, and 8 and settle D-4, D-5, and D-12.
2. **20–30 customer interviews or a survey** on how they choose a barber, tolerance for waiting, WhatsApp vs app, and willingness to join a queue remotely. These check ASSUMPTION-2, 3, and 7.
3. **Legal consult** on privacy, consumer law (cancellations, pricing display), and review liability (D-17, D-20).
4. **Provider quotes**: WhatsApp/SMS costs per message in the launch country; map/geocode costs at projected volume.

## G. Decision review (2026-10-14)

Columns:
- **DM** = blocks domain modeling
- **DB** = blocks database design
- **Impl** = blocks implementation start
- **Prov.?** = is a reversible provisional default enough to proceed?

| ID | Status | DM | DB | Impl | Prov.? | Evidence that would resolve it | Default used now & rationale |
|---|---|---|---|---|---|---|---|
| D-1 | **Resolved** (Egypt) | ✔ | ✔ | ✔ | — | — | City (D-1b) is still open: pick by supply density from interviews. It doesn't affect the schema. |
| D-1b | Open | ✘ | ✘ | ✘ (blocks pilot) | ✔ | Owner interviews in 2–3 candidate areas | Design is city-agnostic |
| D-2 | **Resolved** | ✔ | ✔ | ✔ | — | — | Branch queue + preference |
| D-3 | Provisional | ✘ | ✔ (v0.3: the session rule needs the `remote_session` table and one partial unique index; replaceable without redesign) | ✘ | ✔ | Pilot: remote-join no-show rate; owner tolerance | Remote join on; arrival window ETA−10…ETA+10 min; **one ACTIVE remote session per user, with up to 4 people (product rule P-1)**; cooldown after 2 expired sessions in 30 days. Other values are per-branch settings with bounds. |
| D-4 | **Resolved** | ✔ | ✔ | ✔ | — | Pilot will tune grace values | Grace 10 min (configurable 0–30) |
| D-5 | **Resolved** | ✔ | ✔ | ✔ | — | — | See log |
| D-5b | Provisional | ✔ | ✔ | ✘ | ✔ | Do multi-brand owners exist in the target segment? | 1:1 enforced by a droppable unique constraint |
| D-5c | **Resolved** | ✔ | ✔ | ✔ | — | — | Business owns customers |
| D-6 | **Resolved** | ✔ | ✔ | ✔ | — | — | Name required, phone optional |
| D-7 | Provisional | ✘ | ✘ | ✘ | ✔ | Owner interviews (payment mix); PSP quotes | No online payments. Visits record `amount_charged` + `payment_method`. |
| D-8 | Open | ✘ | ✘ | ✘ (blocks notification build, phase 3) | ✔ | Provider quotes; customer interviews on channel preference | Push first. Channel-agnostic notification model. |
| D-9 | Provisional | ✘ | ✘ | ✘ | ✔ | Willingness-to-pay interviews | Free pilot; `acquisition_source` captured on every customer |
| D-10 | Partly resolved | ✔ (target resolved) | ✔ | ✘ | ✔ | Legal review of policy text | 14-day window, edits ≤30 days, automated pre-check |
| D-11 | Provisional | ✘ | ✘ | ✘ | ✔ | Pilot data | Record + private counts + manual block list |
| D-12 | **Resolved** | ✔ | ✔ | ✔ | — | Validation confirms whether to show the UI | — |
| D-12b | Provisional | ✔ | ✔ | ✘ | ✔ | Observation: how often a single customer gets several services | Many services, one barber, run in sequence |
| D-13 | Provisional | ✘ | ✘ | ✘ | ✔ | Fraud observed in pilot; legal | Phone + pin check + admin review |
| D-14 | Open | ✘ | ✘ | ✘ (blocks discovery UI, phase 3) | ✔ | Side-by-side geocoding test on 50 real Cairo shop addresses; cost model | Provider behind an adapter; the DB stores our own coordinates |
| D-15 | Provisional | ✘ | ✘ | ✘ | ✔ | Customer interviews | Expo app + SEO web |
| D-16 | Provisional | ✘ | ✘ | ✘ | ✔ | Site visits | Shared tablet + phones |
| D-17 | **Open — legal** | ✘ | Partly (retention, consent records) | ✔ (before processing real customer data) | ✔ for schema | Counsel: PDPL licences, DPO, transfer licence, retention | Consent and retention fields included; nothing irreversible |
| D-18 | Provisional | ✘ | Partly | ✘ | ✔ | Counsel | Table in [database-design §9](architecture/database-design.md) |
| D-19 | Open | ✘ | ✘ | ✔ (planning) | — | Team plan | — |
| D-20 | Provisional | ✘ | ✘ | ✘ | ✔ | Counsel (Law 181/2018 Art. 9) | Show the final price the customer pays; store `tax_inclusive` + rate on snapshots |

## H. New decisions found during the design review

| ID | Decision | DM | DB | Impl | Prov.? | Default used now & rationale | Evidence |
|---|---|---|---|---|---|---|---|
| D-21 | Groups | — | — | — | — | **Resolved**: separate linked entries | — |
| D-22 | **Business-day boundary** for shops open past midnight | ✔ | ✔ | ✔ | ✔ | Per-branch `day_rollover` local time (default 05:00). Tickets, queue reset, and daily reports use the business day, not the calendar date. | Site visits: closing times |
| D-23 | **Hosting location & cross-border transfer** (PDPL Art. 16 licence when data leaves Egypt) | ✘ | ✘ | ✔ (before production data) | ✔ for dev | Stay cloud-portable. Dev/staging use synthetic data only. See [ADR-0005](adr/0005-hosting-and-data-residency.md). | Counsel + hosting quotes (in-country vs abroad + licence) |
| D-24 | **Minors' data**: guardian consent for under-15s (PDPL Art. 15) | ✔ | ✔ | ✔ (before launch) | ✔ | A guardian books for an attendee with an optional label (e.g., a first name). No child accounts and no birth dates. A guardian-consent record is stored when a label is given for a minor. | Counsel |
| D-25 | **History visibility after linking** a walk-in record to a user by phone | ✔ | ✘ | ✘ | ✔ | Users see only visits made **after** linking. The business sees the full history. This protects against recycled or shared phones. | Counsel; UX test |
| D-26 | **Price for "any barber" queue entries** | ✔ | ✔ | ✘ | ✔ | Show an estimated range at join. Bind the price snapshot at visit start (when the barber is known). If the business disables per-barber pricing, the range collapses to one price. | Validation: do customers accept a range? |
| D-27 | **Late appointment** after grace | ✔ | ✔ | ✘ | ✔ | Staff choose **convert to walk-in** (new queue entry linked to the appointment; appointment ends `CONVERTED_TO_WALK_IN`) or **no-show**. There is no automatic no-show in the MVP; the system only prompts. | Site visits / owner interviews |
| D-28 | **One customer at a time per barber** | ✔ | ✔ | ✘ | ✔ | Yes for barbers, enforced by a partial unique index. Future verticals may relax it through a category policy, which would need a schema change. **Status: provisional default, not approved.** | Observation |
| D-29 | **Staff members without a user account** | ✔ | ✔ | ✘ | ✔ | Allowed. The front desk operates for them, and a user can be linked later. | Site visits |

## I. Decisions raised by the audit (round 2, 2026-10-14) — none approved

| ID | Question | Options | Recommended default (**PROPOSED, not approved**) & rationale | Blocks | Evidence |
|---|---|---|---|---|---|
| D-30 | **Identity assurance for visit claims** (shared family phones, recycled numbers, disputes) | (a) Token + same-phone OTP (current design). (b) Plus a staff-confirmed name match. (c) A stronger identity provider (none selected). (d) No claims: reviews only from visits recorded under an account. | (a) for the MVP. The blast radius is one visit and one review. Shared-phone risk is accepted and monitored (claim-dispute rate). Revisit if abuse appears. **No verification provider is assumed.** | Review flow for phone-only walk-ins (P0) | Legal (A6); pilot dispute rate |
| D-31 | **Branch closure over existing appointments** | (a) Reject until each affected appointment is resolved, with optional bulk resolution in the same request. (b) Auto-cancel on closure. (c) Allow the closure and leave appointments pending. | (a): reversible, never silent, audited, customers notified. (b) is rejected because it is silent; (c) because it creates invalid state. Customer-agreed reschedule offers in P1. | Closure UI (phase 1) | Owner interviews (A1) |
| D-32 | **Same-day availability while a barber is running late** | (a) Planned commitments only (current). (b) Also exclude today's slots before the barber's `expected_free_at` and any CALLED entry. (c) Disable same-day online booking within X min. | (b): reversible, avoids offering slots that are already late. | Availability engine (phase 4) | Pilot: late-start rate for same-day bookings |
| D-33 | **Can an appointment start without explicit check-in?** | (a) Require CHECKED_IN (current draft). (b) Allow CONFIRMED → FULFILLED with an implicit check-in recorded. | (b): matches "customer just sits down". The audit trail is kept via `check_in_method = IMPLICIT`. | Staff UX (phase 4) | Site visits (A2) |
| D-34 | **Does `arrive_by` move when the wait grows?** | (a) Fixed at join. (b) Extend-only, with notification. (c) Recompute both ways. | (b): customers are never told to come earlier than promised, and are not made to wait in the shop when the queue slows. | Remote flow (phase 3) | Concierge test (A4) |
| D-35 | **Remote-join cooldown scope** | (a) Per branch. (b) Per business (tenant). (c) Platform-wide. | (b): the data is tenant-owned (`crm.customer`). (c) would create a cross-business reputation record, which raises privacy questions (legal) and contradicts D-5c data ownership. | Remote flow (phase 3) | Legal (A6); pilot |
| D-36 | **Customer self check-in: P0 or P1?** | (a) P1: staff confirm every arrival. (b) **P0 for remote customers, with a staff-operated fallback.** | **PROPOSED (b):** a customer may confirm arrival only for their own eligible queue entry (own active session, WAITING, NOT_ARRIVED, within `arrive_by + tolerance`). The system records who or what confirmed arrival and when (`arrival_confirmed_by`, `_id`, `arrived_at`). How presence is proven (in-shop QR code, geofence, or trust plus staff override) is a sub-question; no mechanism is chosen. **If it stays P1:** staff must tap "arrived" for every remote joiner during peaks (H4 load). Missed taps let present customers be skipped as NOT_ARRIVED, or let them expire. The remote-queue value depends on staff discipline. | Remote flow (phase 3) | Concierge test (A4); site visits (A2) |
| D-37 | **Neutral role code** `PROVIDER` instead of `BARBER` | (a) Keep `BARBER`. (b) `PROVIDER`, labeled per category. | (b): a low-cost naming change now; it avoids a rename for later verticals. It is a technical naming choice, so it doesn't change any approved decision. | DB design | — |
| D-39 | **Restrict OTP sign-up/login to Egyptian mobile numbers (`+20 1…`) for the MVP** | (a) Egyptian mobiles only. (b) Any E.164 number. (c) An allowlist of countries. | (a): reduces SMS-pumping (toll-fraud) exposure and matches the launch market. Reversible by configuration. It excludes tourists and foreign numbers, which is the product trade-off. | Auth (phase 0) | Customer interviews; provider fraud data |
| D-40 | **Account-less status link for phone-only walk-ins** | (a) None in the MVP (current). (b) A single-entry capability link in the SMS/WhatsApp message. | (a) for the MVP. (b) would need its own token design (entropy, expiry at entry end, no PII on the page, rate limits) and security review. | None for the MVP | Concierge test (A4) |
| D-38 | **Where a converted late appointment enters the line** | (a) Back of the line. (b) At the original appointment start time (`sort_key = start`). (c) Front. | Late **customer** → (b). Barber **overrun** (staff-caused) → (c). | Dispatch (phase 4) | Owner interviews (A1) |

## Decision log

| ID | Decision | Date | Decided by | Notes |
|---|---|---|---|---|
| D-1 | Launch market: **Egypt** (city still open: D-1b) | 2026-10-14 | Product owner | Implies EGP, ar/en + RTL, Africa/Cairo (DST), PDPL 151/2020 |
| D-2 | **One queue per branch**; entries carry preferred barber or "any" | 2026-10-14 | Product owner | Lanes are computed views |
| D-4 | **Appointment priority at its time (+grace); walk-ins start only if they fit before the barber's next appointment** | 2026-10-14 | Product owner | Manager override with reason (audited) |
| D-5 | **Employees-only MVP.** Independent barber = one-person business. A person may be staff at several businesses. No cross-tenant queues. | 2026-10-14 | Product owner | Chair-renters are modeled operationally as the host shop's staff |
| D-5c | **The business keeps customer relationships** when a barber leaves | 2026-10-14 | Product owner | Barber keeps nothing tenant-owned; portable profile deferred |
| D-6 | Walk-in: **name/nickname required, phone optional** | 2026-10-14 | Product owner | Phone enables notifications and later linking |
| D-10 (part) | Review **rates the business (branch) and the serving barber**: one review, counted in both aggregates | 2026-10-14 | Product owner | Window/edit/moderation details remain provisional |
| D-12 | **Model per-barber price/duration overrides now; expose them in the UI only if validation confirms** | 2026-10-14 | Product owner | Resolution order: staff → branch → base |
| D-21 | Groups: **separate entries/appointments per person, linked by an optional group id** | 2026-10-14 | Product owner | Served in parallel; cancel-together action |

# 04 — Open Decisions (blocking database design & implementation)

Version 0.1 · 2026-10-14

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
| **D-5** | How are independent / chair-renting barbers modeled? Can a barber work for several shops? | (a) Barber is only staff of a shop. (b) A barber can be their own tenant **and** staff elsewhere. (c) Portable professional profile owned by the user, linked to tenant staff profiles. | **(c) light:** a user-owned `ProfessionalProfile` (portfolio, bio) linked to 0..n StaffProfiles in tenants. Bookings and customer data stay with the tenant. An independent barber = a single-person tenant. Clarify who owns the customer list when a barber leaves (default: the tenant). |
| **D-5b** | Tenant ↔ Business cardinality | 1:1 · 1:n (holding company with several brands) | **1:1 for the MVP**, kept as separate concepts in code so a 1:n change is a migration, not a rewrite. |
| **D-2** | Queue structure | One queue per branch with barber preference · one queue per barber · both | **One queue per branch; entries carry preferred barber or "any".** Per-barber "lanes" are a computed view. Matches how shops work and how Great Clips assigns the next available stylist. |
| **D-6** | Customers without the app | Staff can only enter name · name + optional phone · required phone | **Name required, phone optional.** If a phone is given, the customer gets SMS/WhatsApp updates, and the record links to their account when they sign up and verify that phone. |
| **D-12** | Pricing granularity in the MVP | Base only · base + branch · base + branch + staff | **All three levels.** Seniority-based barber pricing is common (ASSUMPTION — confirm). It is cheap now and costly to retrofit. Add-ons, promotions, and tax rules wait for P1. |
| **D-12b** | Multi-service bookings | Single service · multiple services, same barber · multiple barbers | **Multiple services, same barber, run in sequence.** |

## C. Queue & booking policies

| ID | Decision | Options | [R] Recommendation |
|---|---|---|---|
| **D-3** | Remote queue join rules | Allowed? Max distance? Arrival window? Limit per customer? | Per-branch toggle (default on). Optional max distance (default off). Arrival window = ETA − 10 min … ETA + 10 min. **One active remote entry per customer platform-wide.** Repeated no-shows cause a temporary remote-join cooldown. |
| **D-4** | Appointments vs walk-ins priority | Strict appointment priority · first-come regardless · "fit-in" rule | **Appointment priority at its time (+10 min grace). Walk-ins start on a barber only if expected duration fits before that barber's next appointment.** The manager can override with a reason (audited). |
| **D-11** | Cancellation / no-show defaults and enforcement in the MVP (no payments) | Record only · block after N · reliability score visible to shops | Record + show the shop a private count ("2 no-shows in 90 days"). Remote-join cooldown. Manual block list. Fees wait for P1 payments. |
| **D-7** | Online payments / deposits in the MVP | None · deposits only · full prepay | **None in the MVP** (ASSUMPTION-4). Choose a local PSP in P1. The model should still keep `price_snapshot` and `amount_charged` + `payment_method` (cash/card/other) on Visit. |

## D. Trust, privacy, compliance

| ID | Decision | Options | [R] Recommendation |
|---|---|---|---|
| **D-10** | Review policy | Verified-only? Window? Edits? Pre- or post-moderation? Display name? | **Verified completed visits only** (industry norm: Booksy, Fresha). 14-day window. Customer may edit within 30 days. Automated pre-check, then publish; reported reviews are hidden pending review only for severe categories. Display "first name + initial". Owner gets one reply. No incentives and no review gating. |
| **D-13** | Business verification before listing | None · phone + location check · documents (commercial registration) | **Phone verification + map pin within a radius of the geocoded address + admin review of photos.** Ask for documents only if fraud appears or local law requires it. |
| **D-17** | Applicable privacy law & data residency | Depends on D-1 (e.g., Egypt PDPL 151/2020, KSA PDPL) | Get legal input before the schema is finalized: it drives retention fields, consent records, and hosting region. **Blocking** |
| **D-18** | Data retention periods | — | Proposal: queue events in raw form 13 months, then aggregated. Location search inputs are not persisted. Deleted accounts are anonymized within 30 days; tenant financial/visit records are kept as legally required, with personal data pseudonymized. |

## E. Product surfaces & providers

| ID | Decision | Options | [R] Recommendation |
|---|---|---|---|
| **D-8** | Notification channels and who pays | Push only · push + SMS · push + WhatsApp · all | Push first. **WhatsApp** for "you're next" and "about to expire" if ASSUMPTION-2 holds; SMS as fallback and for OTP. The platform absorbs costs within plan quotas. |
| **D-14** | Maps & geocoding provider | Google Maps Platform · Mapbox · MapLibre + OSM/self-hosted tiles | Evaluate address/POI quality in the launch city first. Google is usually the strongest for MENA addresses but costs the most. Keep provider calls behind an adapter. |
| **D-15** | Customer channel at launch | Native app + web · web/PWA only · app only | **Expo app + SEO web.** The web shows profiles and can book and join queues, but iOS web push is unreliable for queue alerts unless the PWA is installed. |
| **D-16** | In-shop hardware | Staff phones only · shared tablet · kiosk/TV (P1) | Design the staff board for **one shared tablet + barbers' phones**. Kiosk/TV in P1. |
| **D-20** | Tax display | Tax-inclusive prices · exclusive | Follow launch-market consumer law (often tax-inclusive). Store `tax_inclusive` + rate on the snapshot even if tax config is P1. |

## F. Validation work required before decisions are final

1. **15–20 owner/barber interviews** plus **3 site visits at peak hours** in the launch city. These check ASSUMPTION-1, 4, 6, and 8 and settle D-4, D-5, and D-12.
2. **20–30 customer interviews or a survey** on how they choose a barber, tolerance for waiting, WhatsApp vs app, and willingness to join a queue remotely. These check ASSUMPTION-2, 3, and 7.
3. **Legal consult** on privacy, consumer law (cancellations, pricing display), and review liability (D-17, D-20).
4. **Provider quotes**: WhatsApp/SMS costs per message in the launch country; map/geocode costs at projected volume.

## Decision log

| ID | Decision | Date | Decided by | Notes |
|---|---|---|---|---|
| — | — | — | — | Nothing decided yet |

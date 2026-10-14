# 02 — Product Requirements Document (Initial)

Version 0.1 · 2026-10-14 · Status: Draft. Scope: the barber vertical MVP, with the platform foundations that later verticals need.

Related: [Market research](01-market-research.md) · [MVP & architecture](03-mvp-and-architecture.md) · [Open decisions](04-open-decisions.md)

---

## 1. Product goals

| ID | Goal | Measure (initial target, to be validated) |
|---|---|---|
| G1 | Customers spend less time waiting idle in shops | Median in-shop wait for remote-queue customers vs walk-ins |
| G2 | Barbershops fill more chair-hours with less admin | Utilization; share of visits entered through the platform |
| G3 | Customers find a suitable barber nearby quickly | Search → booking/queue-join conversion; time-to-decision |
| G4 | Trustworthy information | ETA accuracy (actual vs predicted wait within ±10 min for ≥70% of entries — ASSUMPTION-9); only verified reviews |
| G5 | A safe multi-tenant foundation | Zero cross-tenant data exposure incidents; audited admin access |

## 2. Assumptions requiring validation

| ID | Assumption | How to validate |
|---|---|---|
| ASSUMPTION-1 | The target market's barbershops are walk-in-heavy and also take some appointments. | 15–20 shop owner interviews; 2–3 days of observing peak hours |
| ASSUMPTION-2 | WhatsApp is the expected notification channel; SMS is the fallback. | Customer interviews; delivery cost model |
| ASSUMPTION-3 | Phone-number OTP is the preferred login. | Interviews; competitor norms in the launch market |
| ASSUMPTION-4 | Most service payments are cash at the shop; online payment is not required for the MVP. | Owner interviews |
| ASSUMPTION-5 | Arabic + English with RTL are required at launch. | Depends on D-1 |
| ASSUMPTION-6 | Shops have at least one smartphone or tablet that staff can operate during service. | Interviews / site visits |
| ASSUMPTION-7 | Customers will join a remote queue and arrive within a window, if they are reminded. | Pilot data (no-show rate of remote joins) |
| ASSUMPTION-8 | Staff will reliably tap "start" and "complete". Without this, ETAs degrade. | Pilot; UX must make it one tap |
| ASSUMPTION-9 | Rule-based ETAs (durations + live state) are accurate enough. | Pilot measurement |

## 3. Personas

Personas are roles, not demographic profiles.

| Persona | Context | Primary needs | Pain today |
|---|---|---|---|
| **P1 Walk-in regular** | Gets a cut every 2–3 weeks; decides same day | "Where can I get a cut *now* with the shortest wait?"; get told when to leave home | Arrives to a full shop; waits 60+ min with no information |
| **P2 Planner** | Busy schedule; wants a fixed time with a specific barber | Book a specific barber and time; reminders; easy reschedule | Calling or WhatsApping the shop; double-booked or forgotten |
| **P3 Barber (employee)** | Hands busy; uses a phone between clients | See who is next; one-tap call/start/finish; know about the next appointment | Paper lists, shouting names, disputes over who is next |
| **P4 Shop owner/manager (1 branch, 2–6 chairs)** | Works a chair and runs the shop | Fair queue, fewer no-shows, simple service/price setup, daily numbers | No data; no-shows; WhatsApp overload |
| **P5 Multi-branch owner** | 2–10 branches | Consistent catalog, per-branch prices/staff, cross-branch reports, delegation | Fragmented tools |
| **P6 Independent / chair-renting barber** | Rents a chair, or is mobile; owns their clients | Personal booking page and portfolio that moves with them | Losing clients when moving shops (scope depends on D-5) |
| **P7 Platform admin / moderator** | Internal | Approve businesses, moderate reviews/photos, handle abuse, support | — |

## 4. Core user journeys (MVP)

**J1 — Find and join a queue now (P1)**
1. Open the app or web. Allow location, or type an area.
2. See a list and map of nearby shops with distance, *open now*, estimated wait, price range, and rating.
3. Open a shop. Pick a service (e.g., cut + beard) and "any barber" or a specific barber.
4. See the ETA as a range and a "leave by" hint. Join the queue.
5. Get an update when the wait drops, a "you're next soon — head over" message, and a "please arrive by HH:MM" deadline.
6. Arrive and check in (staff tap or customer tap). Get called, served, and completed.
7. Receive a review invitation.

**J2 — Book an appointment (P2)**
1. Search, then pick a shop, service(s), and barber or "any".
2. See available slots (branch time zone).
3. Confirm. The booking is created atomically with a price snapshot. Get a confirmation and later a reminder.
4. Optionally reschedule or cancel within the policy.
5. Arrive, check in, and get served on time. Walk-ins do not delay a confirmed appointment beyond the grace period (D-4).

**J3 — Run the floor (P3/P4)**
1. Open the staff queue board, which shows each barber's lane: in-service, next appointment, and waiting walk-ins.
2. Add a walk-in (name, optional phone, service, preferred barber).
3. "Call next" → "Start" → "Complete" (record the price actually charged; cash/other).
4. Mark no-show, skip (with a reason), move to another barber, or pause a barber (break, unavailable).
5. All other devices and waiting customers update within seconds.

**J4 — Onboard a business (P4/P5)**
1. The owner signs up and creates a business, then branch(es) with an address and map pin.
2. Set opening hours, services, durations, prices, and queue/booking policies.
3. Invite staff by phone/email, assign roles, and pick which services each barber offers.
4. Submit for verification. An admin approves, and the shop is listed publicly.

**J5 — Moderate (P7)**
1. Review the queue of new businesses, reported reviews, and reported photos.
2. Approve, reject, hide, or suspend. Every action is audited.

## 5. Functional requirements

Priority: **P0** = secure, usable MVP · **P1** = after MVP · **P2** = advanced/future.

### 5.1 Identity & access (IAM)
| ID | Requirement | P |
|---|---|---|
| FR-IAM-1 | Customers sign up and log in with phone OTP (email optional). Sessions use short-lived access tokens plus rotating refresh tokens. | P0 |
| FR-IAM-2 | One platform identity per person across all businesses and verticals. | P0 |
| FR-IAM-3 | Staff log in with the same identity system. Being staff = a **membership** in a tenant with a role. A person can be both a customer and staff. | P0 |
| FR-IAM-4 | Tenant roles are fixed: Owner, Manager, Barber (Staff), Front-desk. Internally they map to permissions so custom roles can be added later. | P0 |
| FR-IAM-5 | Custom tenant roles. | P1 |
| FR-IAM-6 | Platform roles are separate from tenant roles: Admin, Moderator, Support. Each support access to tenant data is time-boxed and audited. | P0 |
| FR-IAM-7 | Account deletion and data export on request. | P0 |
| FR-IAM-8 | MFA for platform admins (mandatory) and owners (optional). | P0 admins / P1 owners |

### 5.2 Tenancy, business & branch (TEN)
| ID | Requirement | P |
|---|---|---|
| FR-TEN-1 | Owners create a business (tenant) with a legal/display name, category (barber), and contact details. | P0 |
| FR-TEN-2 | A business has 1..n branches. Each branch has an address, a verified map coordinate, an IANA time zone, a currency, and opening hours. | P0 (model and basic UI) |
| FR-TEN-3 | A business is not publicly listed until a platform admin verifies it. | P0 |
| FR-TEN-4 | Staff are invited to the tenant and assigned to one or more branches. | P0 |
| FR-TEN-5 | Cross-branch reports and catalog templates. | P1 |

### 5.3 Catalog & pricing (CAT)
| ID | Requirement | P |
|---|---|---|
| FR-CAT-1 | A service has a name (ar/en), description, category, default duration, and base price. | P0 |
| FR-CAT-2 | The branch offering sets whether the service is active at the branch, plus an optional branch price/duration override. | P0 |
| FR-CAT-3 | Staff eligibility says which barbers perform which services. An optional per-staff price/duration override is allowed when the business enables it. | P0 |
| FR-CAT-4 | Price resolution order: staff override → branch override → base. The result is deterministic, and the rule is documented and tested. | P0 |
| FR-CAT-5 | Bookings and queue entries store a **price and duration snapshot** at confirmation. Later price changes do not affect them. | P0 |
| FR-CAT-6 | Shown prices can be "from X" ranges where barber prices differ. | P0 |
| FR-CAT-7 | Add-ons, promotions/discounts, tax configuration, deposits. | P1 |
| FR-CAT-8 | Packages, memberships, gift cards. | P2 |

### 5.4 Scheduling & availability (SCH)
| ID | Requirement | P |
|---|---|---|
| FR-SCH-1 | Branch opening hours per weekday, with multiple intervals per day. | P0 |
| FR-SCH-2 | Staff working hours per branch, breaks, and time off. | P0 |
| FR-SCH-3 | Branch closures and exceptional hours (holidays). | P0 |
| FR-SCH-4 | Staff can toggle live status: available / on break / unavailable (with an expected return time). | P0 |
| FR-SCH-5 | All times are stored in UTC and computed in the branch's IANA zone. DST is handled correctly; Egypt, for example, reinstated DST in 2023. | P0 |

### 5.5 Booking (BKG)
| ID | Requirement | P |
|---|---|---|
| FR-BKG-1 | The availability query returns bookable slots for (branch, services[], staff or "any", date range). It respects hours, breaks, time off, existing bookings, buffers, minimum notice, and horizon. | P0 |
| FR-BKG-2 | Create a booking with 1..n services, performed sequentially by one barber. The request requires an `Idempotency-Key`. | P0 |
| FR-BKG-3 | **Double-booking is impossible** for one barber, enforced by a database constraint, not only by an application check (see [03 §5](03-mvp-and-architecture.md)). | P0 |
| FR-BKG-4 | "Any barber" assigns a staff member at creation using a deterministic rule (e.g., the least-loaded eligible barber). | P0 |
| FR-BKG-5 | Customers cancel or reschedule within business-configured windows. Late cancellations are recorded. | P0 |
| FR-BKG-6 | Staff can create, cancel, and reschedule on behalf of customers, including customers without an app (see FR-CRM-1). | P0 |
| FR-BKG-7 | Optional approval workflow (pending → confirmed/rejected). | P1 |
| FR-BKG-8 | Deposits / prepayment / no-show fees. | P1 (D-7) |
| FR-BKG-9 | Notify the customer when a slot opens (slot waitlist). | P1 |
| FR-BKG-10 | Recurring appointments. | P2 |

### 5.6 Queue (QUE)
| ID | Requirement | P |
|---|---|---|
| FR-QUE-1 | Each branch has a walk-in queue. Entries carry services, a preferred barber (or any), and a source (staff-entered / remote / kiosk). | P0 |
| FR-QUE-2 | Remote join is configurable per branch: on/off, maximum active remote entries, optional maximum join distance, and an arrival window. | P0 |
| FR-QUE-3 | Entry lifecycle: `WAITING → CALLED → IN_SERVICE → COMPLETED`, plus `CHECKED_IN` (remote arrivals), `SKIPPED` (returns to WAITING with a policy), `NO_SHOW`, `CANCELLED`, and `EXPIRED`. Only legal transitions are accepted (state machine in [03 §6](03-mvp-and-architecture.md)). | P0 |
| FR-QUE-4 | Appointment check-in turns the appointment into the barber's lane at its scheduled time. Walk-ins fill gaps when they fit before the next appointment (D-4). | P0 |
| FR-QUE-5 | A queue entry can be served by only one staff member. Two concurrent "call/start" actions → exactly one succeeds; the other gets a clear conflict. | P0 |
| FR-QUE-6 | The ETA per entry is computed from remaining service durations, eligible barbers' availability, in-progress services (elapsed vs expected), and upcoming appointments. It is shown as a range. | P0 |
| FR-QUE-7 | Pause/resume intake per branch. Queue capacity limit per branch. Intake auto-closes at a configured time before closing. | P0 |
| FR-QUE-8 | Authorized reordering (manager+), with a mandatory reason and an audit record. | P0 |
| FR-QUE-9 | One active remote queue entry per customer platform-wide (default; D-3). | P0 |
| FR-QUE-10 | A late remote arrival gets a grace period, then is EXPIRED/NO_SHOW. The customer is notified before expiry. | P0 |
| FR-QUE-11 | Public in-shop display (TV/tablet) shows first name + initial or a ticket number only. | P1 |
| FR-QUE-12 | Kiosk self check-in. | P1 |
| FR-QUE-13 | ML-based ETA. | P2 |

### 5.7 Real-time (RT)
| ID | Requirement | P |
|---|---|---|
| FR-RT-1 | Staff boards and customer status screens receive changes pushed by the server. | P0 |
| FR-RT-2 | Every queue change increments a per-queue sequence number. Clients detect gaps and fetch an authoritative snapshot over HTTP. | P0 |
| FR-RT-3 | Customer channels receive only that customer's entry state plus the aggregate position/ETA, never other customers' identities. | P0 |

### 5.8 Discovery (DIS)
| ID | Requirement | P |
|---|---|---|
| FR-DIS-1 | Search by device location *or* a manually chosen place/area, without requiring the location permission. | P0 |
| FR-DIS-2 | Radius search with distance sort. Filters: service, price range, open now, accepts walk-ins/remote queue, rating, has availability today. | P0 |
| FR-DIS-3 | List and map views with cursor pagination. | P0 |
| FR-DIS-4 | Show live status in results: open/closed, current estimated wait (bucketed), next available slot. | P0 |
| FR-DIS-5 | The user's search location is not stored with their identity beyond the request (except an opted-in "saved places" list) and is never shared with businesses. | P0 |
| FR-DIS-6 | SEO-indexable public business pages (web). | P0 |
| FR-DIS-7 | Text search by business/barber name. | P0 |
| FR-DIS-8 | Personalized ranking and recommendations. | P2 |

### 5.9 Reviews (REV)
| ID | Requirement | P |
|---|---|---|
| FR-REV-1 | Only a customer with a **COMPLETED** visit can review it. One review per visit, submitted within N days (D-10). | P0 |
| FR-REV-2 | A review has a 1–5 rating and optional text. The barber reviewed is the one who served the visit. | P0 |
| FR-REV-3 | Public display shows first name + last initial (or a chosen alias) and the visit month. It never shows a phone, email, or exact visit time. | P0 |
| FR-REV-4 | Owners/managers may post one public reply per review. | P0 |
| FR-REV-5 | Anyone can report a review. Moderators can hide or restore it, with an audited reason. Automated pre-checks (profanity, PII patterns). | P0 |
| FR-REV-6 | Businesses cannot edit or delete reviews, and cannot choose which customers get review invitations (no "review gating"). | P0 |
| FR-REV-7 | Aggregates (average, count, distribution) are recomputed on change. A minimum review count is required before a public average is shown (e.g., 3). | P0 |
| FR-REV-8 | Review photos. | P1 |

### 5.10 Notifications (NTF)
| ID | Requirement | P |
|---|---|---|
| FR-NTF-1 | Transactional events: booking confirmed/changed/cancelled, reminder, queue joined, "N min left", called, about to expire, review invitation. | P0 |
| FR-NTF-2 | Channels: in-app/push first. WhatsApp and/or SMS for critical events (D-8). Per-user language. | P0 |
| FR-NTF-3 | Delivery is asynchronous and retried, with deduplication. A notification failure never rolls back a booking or queue action. | P0 |
| FR-NTF-4 | Marketing messages require explicit opt-in. | P1 |

### 5.11 Customer relationship (CRM)
| ID | Requirement | P |
|---|---|---|
| FR-CRM-1 | A business can hold a **customer record** that is not linked to a platform user (walk-in with only a name and/or phone). Linking to a platform user requires the user to verify the phone. | P0 |
| FR-CRM-2 | A business sees only its own customer relationship data (visit history *at that business*). It never sees the customer's activity at other businesses. | P0 |
| FR-CRM-3 | Customer history, favorites, and preferences are visible in the customer app. | P0 |
| FR-CRM-4 | Business-private notes and a block list for a customer. | P1 (block list is a P0 candidate) |

### 5.12 Analytics (ANA)
| ID | Requirement | P |
|---|---|---|
| FR-ANA-1 | Daily/weekly per branch and per barber: visits, walk-in vs appointment vs remote, average wait, average service time vs configured, no-show rate, recorded revenue. | P0 basic |
| FR-ANA-2 | Cross-branch dashboards and exports. | P1 |
| FR-ANA-3 | Platform analytics on aggregated, de-identified data. | P1 |

### 5.13 Platform administration (ADM)
| ID | Requirement | P |
|---|---|---|
| FR-ADM-1 | Business verification queue (approve/reject/request info). | P0 |
| FR-ADM-2 | Suspend/restore a business or user. | P0 |
| FR-ADM-3 | Moderation queue for reviews and photos. | P0 |
| FR-ADM-4 | Manage business categories and platform settings. | P0 (seeded) |
| FR-ADM-5 | View audit logs. Support "view as tenant" only with a recorded reason and a time limit. | P0 |
| FR-ADM-6 | Subscription plans and feature entitlements. | P1 (D-9) |

## 6. Non-functional requirements

| ID | Requirement | Initial target (ASSUMPTION; revisit after pilot) |
|---|---|---|
| NFR-1 Availability | API + real-time | 99.5% monthly during the MVP; staff can still serve the next customer from the last snapshot during brief outages |
| NFR-2 Latency | API p95 | < 300 ms for reads, < 500 ms for booking/queue commands, measured at the server |
| NFR-3 Real-time propagation | Commit → client | p95 < 2 s |
| NFR-4 Consistency | Bookings & queue | Strongly consistent; PostgreSQL is the single source of truth |
| NFR-5 Tenant isolation | — | Enforced at the API (authorization) **and** the database (row-level security); automated cross-tenant tests in CI |
| NFR-6 Security | — | OWASP ASVS Level 2 as a baseline; rate limits on OTP, search, and queue join |
| NFR-7 Privacy | — | Data minimization; compliance with the launch market's law (e.g., Egypt PDPL Law 151/2020, KSA PDPL, GDPR if EU users) — confirm with counsel (D-17) |
| NFR-8 Backup/DR | — | Point-in-time recovery; RPO ≤ 5 min, RTO ≤ 4 h; restore tested quarterly |
| NFR-9 Accessibility | — | WCAG 2.2 AA for web; platform accessibility APIs on mobile |
| NFR-10 Localization | — | ar/en, RTL layouts, locale-aware numbers/dates, IANA time zones |
| NFR-11 Observability | — | Structured logs, traces, and metrics with tenant id; alerting on error rate, latency, and real-time lag |
| NFR-12 Mobile network tolerance | — | Commands are idempotent and retry-safe; clients recover from reconnects automatically |

## 7. Key business rules (initial; defaults are configurable per business within limits)

| ID | Rule |
|---|---|
| BR-1 | A confirmed booking's price, duration, and services are snapshotted. Changing the catalog does not change existing bookings. |
| BR-2 | A barber cannot have two overlapping committed time blocks (appointment or in-service). This covers buffers. |
| BR-3 | Booking minimum notice (default 30 min), horizon (default 30 days), and slot granularity (default 15 min) are configurable. |
| BR-4 | Default cancellation: free until X hours before the appointment; later = late cancellation (recorded, no fee in the MVP). |
| BR-5 | A confirmed appointment has priority at its start time (+ grace, default 10 min). A walk-in is offered to a barber only if its expected duration fits before the barber's next appointment, or no eligible barber has an appointment conflict (D-4). |
| BR-6 | A remote queue entry must check in within the arrival window. After a grace period it expires and the customer is notified. |
| BR-7 | One active remote queue entry per customer across the platform (default). |
| BR-8 | Only staff assigned to the branch can act on its queue. Reordering requires Manager+ and a reason. |
| BR-9 | Reviews only for COMPLETED visits, one per visit, within N days. |
| BR-10 | Public surfaces (profiles, reviews, displays, queue position) never expose customer phone, email, full name, or precise location. |
| BR-11 | A queue entry and an appointment both result in a **visit** when service starts. The visit is the record of what actually happened (who served, start/end, charged price). |

## 8. Acceptance criteria (selected P0)

- **AC-BKG-concurrency**
  - Given barber B is free 10:00–10:30.
  - When 50 concurrent requests try to book B at 10:00 for 30 min,
  - then exactly 1 succeeds and 49 receive `409 SLOT_UNAVAILABLE`, and the database holds exactly one booking.
- **AC-BKG-idempotency**
  - When the same `Idempotency-Key` + payload is retried 3 times,
  - then one booking exists and all 3 responses return it.
  - A different payload with the same key → `422 IDEMPOTENCY_KEY_REUSED`.
- **AC-QUE-double-serve**
  - Given entry E is WAITING.
  - When two staff press "start" on E simultaneously,
  - then exactly one transition succeeds, and the other receives `409 INVALID_STATE` with the current state.
- **AC-TEN-isolation**
  - Given a Manager of tenant A.
  - When they request any tenant-B resource by id (booking, customer, queue entry),
  - then the API returns `404`, not `403`, so ids are not confirmed to exist.
  - A direct SQL query under tenant A's context returns 0 rows of tenant B.
- **AC-REV-eligibility**
  - Given a visit in state CANCELLED or NO_SHOW, or one older than N days, review creation → `403 REVIEW_NOT_ELIGIBLE`.
  - A second review for the same visit → `409`.
- **AC-RT-recovery**
  - Given a client disconnected while 3 queue events occurred.
  - When it reconnects with `lastSeq`,
  - then it either receives the missed events in order or a full snapshot with the current `seq`. The UI then matches the server state.
- **AC-PRIV-location**
  - A business's API responses and exports never contain a customer's search coordinates.
  - The remote-join distance check stores only pass/fail, not the coordinates.

## 9. Dependencies

- A map/geocoding provider (D-14).
- An SMS / WhatsApp Business provider (D-8).
- Push: APNs/FCM (via Expo or direct).
- Object storage + CDN for images.
- Managed PostgreSQL with PostGIS.
- Legal: terms, privacy policy, review policy, data processing agreement for businesses.

## 10. Constraints

- The first release covers the barber vertical only. The domain model must not hard-code "haircut" assumptions into shared modules.
- Team size and timeline are not yet known (D-19). The architecture assumes a small team (2–6 engineers).
- No unrestricted configuration engine. Business policies are typed, validated settings with bounded ranges.

## 11. Risks (initial)

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-1 | Staff don't update states, so ETAs are wrong and customers lose trust | H | H | One-tap UX; auto-complete prompts; show ETA ranges; measure accuracy per shop |
| R-2 | Remote-queue no-shows anger shops | M | H | Arrival windows, limits, expiry, reliability signals, per-shop toggle |
| R-3 | Marketplace cold start | H | H | City-by-city launch; the business tool must be useful even without marketplace traffic |
| R-4 | Cross-tenant data leak | L | Critical | RLS + authorization + isolation test suite + audit |
| R-5 | Scope creep into POS/payments/marketing | H | M | P0 discipline; ADRs for any additions |
| R-6 | Messaging costs | M | M | Push-first, quotas, plan-based entitlements |
| R-7 | Over-engineering for future verticals | M | M | Modular monolith; vertical-specific behavior behind explicit policies; no other vertical until barber is validated |
| R-8 | Fake businesses / review fraud | M | H | Verification, verified-visit reviews, rate limits, moderation |

## 12. Out of scope for the MVP

- Online payments, deposits, refunds, payouts, POS, inventory, chair-rent collection.
- Other verticals (clinics, dental, automotive).
- Marketing campaigns, loyalty, memberships, gift cards.
- Public API / webhooks, SSO, Reserve with Google.
- ML-based ETA and personalized recommendations.
- Kiosk and TV display (P1).
- Separate databases per tenant.

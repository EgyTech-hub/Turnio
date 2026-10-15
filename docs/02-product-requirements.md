# 02 — Product Requirements Document (Initial)

Version 0.3 · 2026-10-14 · Status: Draft. Scope: the barber vertical MVP, with the platform foundations that later verticals need.

> **Changelog v0.2 (2026-10-14):**
> - Applied the [05 design review](05-design-review.md): C-1, C-3, C-5, C-6, C-7, plus missing rules M-1…M-14.
> - Recorded the approved decisions (Egypt; branch queue; appointment priority; employees-only; the business owns customers; name-only walk-ins allowed; linked groups; per-barber pricing designed in; reviews rate the business and the barber).
> - Added BR-12…BR-20 and ASSUMPTION-10/11.
> - Lifecycles now live in [domain/](domain/).
>
> **Changelog v0.3 (2026-10-14, audit round 2; see [05 §9](05-design-review.md)):**
> - Remote queue sessions (FR-QUE-9, BR-7).
> - Visit claim invitations (FR-REV-9, BR-18).
> - A late remote customer → EXPIRED (FR-QUE-10).
> - TV display: ticket numbers only (FR-QUE-11).
> - Self check-in recorded as open (FR-QUE-14, D-36).
> - Branch closures (BR-21).
> - FR-BKG-3 link fixed; acceptance criteria aligned with the visit lifecycle.
> - Neutral `PROVIDER` role code.
> - P6 persona aligned with D-5.

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
| ASSUMPTION-10 | One PostgreSQL primary handles year-1 volume (~500 branches; see [database design §11](architecture/database-design.md)). | Load test before launch |
| ASSUMPTION-11 | A branch queue sees at most about 1 command every few seconds at peak, so one lock per queue is not a bottleneck. | Pilot measurement |

**Resolved 2026-10-14:** ASSUMPTION-5 is confirmed by D-1 (Egypt): Arabic + English with RTL are required at launch.

## 3. Personas

Personas are roles, not demographic profiles.

| Persona | Context | Primary needs | Pain today |
|---|---|---|---|
| **P1 Walk-in regular** | Gets a cut every 2–3 weeks; decides same day | "Where can I get a cut *now* with the shortest wait?"; get told when to leave home | Arrives to a full shop; waits 60+ min with no information |
| **P2 Planner** | Busy schedule; wants a fixed time with a specific barber | Book a specific barber and time; reminders; easy reschedule | Calling or WhatsApping the shop; double-booked or forgotten |
| **P3 Barber (employee)** | Hands busy; uses a phone between clients | See who is next; one-tap call/start/finish; know about the next appointment | Paper lists, shouting names, disputes over who is next |
| **P4 Shop owner/manager (1 branch, 2–6 chairs)** | Works a chair and runs the shop | Fair queue, fewer no-shows, simple service/price setup, daily numbers | No data; no-shows; WhatsApp overload |
| **P5 Multi-branch owner** | 2–10 branches | Consistent catalog, per-branch prices/staff, cross-branch reports, delegation | Fragmented tools |
| **P6 Independent / chair-renting barber** | Rents a chair, or is mobile; owns their clients | MVP (D-5, approved): signs up as a one-person business with its own booking page. Chair-renters inside a shop are modeled as that shop's staff. A portfolio that *moves with the barber* across businesses is **deferred** (superseded v0.1 idea). Mobile/home service is out of scope. | Losing clients when moving shops (business keeps customer records, D-5c) |
| **P7 Platform admin / moderator** | Internal | Approve businesses, moderate reviews/photos, handle abuse, support | — |

## 4. Core user journeys (MVP)

**J1 — Find and join a queue now (P1)**
1. Open the app or web. Allow location, or type an area.
2. See a list and map of nearby shops with distance, *open now*, estimated wait, price range, and rating.
3. Open a shop. Pick a service (e.g., cut + beard) and "any barber" or a specific barber.
4. See the ETA as a range and a "leave by" hint. Join the queue.
5. Get an update when the wait drops, a "you're next soon — head over" message, and a "please arrive by HH:MM" deadline.
6. Arrive. Arrival is confirmed by staff (P0). Customer self-confirmation of arrival is **proposed as P0 but not approved (D-36)**. Get called, served, and completed.
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
| FR-IAM-4 | Tenant roles are fixed: Owner, Manager, Provider (shown as "Barber" for the barber category), Front-desk. The role code is `PROVIDER` (proposed neutral naming). Internally roles map to permissions so custom roles can be added later. | P0 |
| FR-IAM-5 | Custom tenant roles. | P1 |
| FR-IAM-6 | Platform roles are separate from tenant roles: Admin, Moderator, Support. Each support access to tenant data is time-boxed and audited. | P0 |
| FR-IAM-7 | Account deletion and data export on request. | P0 |
| FR-IAM-8 | MFA for platform admins (mandatory) and owners (optional). | P0 admins / P1 owners |

### 5.2 Tenancy, business & branch (TEN)
| ID | Requirement | P |
|---|---|---|
| FR-TEN-1 | Owners create a business (tenant) with a legal/display name, category (barber), and contact details. | P0 |
| FR-TEN-2 | A business has 1..n branches. Each branch has an informal address (governorate, city, area, street, landmark), a verified map pin, an IANA time zone (`Africa/Cairo`), a business-day rollover time (D-22), and opening hours. Currency is set per tenant (`EGP`) and copied onto every money snapshot. | P0 (model and basic UI) |
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
| FR-CAT-5 | Appointments store a binding **price and duration snapshot** at confirmation (the barber is known then). Queue entries store an **estimated price range** at join. The binding snapshot is taken on the **visit** at service start, when the barber is known (D-26). Later catalog changes never affect existing snapshots. | P0 |
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
| FR-BKG-3 | **Double-booking is impossible** for one barber, enforced by a database constraint, not only by an application check (see [database design §5.1](architecture/database-design.md) and [booking lifecycle §6](domain/booking-lifecycle.md)). | P0 |
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
| FR-QUE-3 | Entry lifecycle: `WAITING → CALLED → SERVED`, plus `CANCELLED`, `NO_SHOW`, and `EXPIRED`. **Presence** (`NOT_ARRIVED`/`ARRIVED`) is a separate attribute, and only arrived entries can be called. *Skip* is a transition (CALLED → WAITING, counted), not a state. Service progress lives on the **Visit** (`IN_SERVICE → COMPLETED`). See [queue-lifecycle](domain/queue-lifecycle.md). | P0 |
| FR-QUE-4 | Appointment check-in turns the appointment into the barber's lane at its scheduled time. Walk-ins fill gaps when they fit before the next appointment (D-4). | P0 |
| FR-QUE-5 | A queue entry can be served by only one staff member. Two concurrent "call/start" actions → exactly one succeeds; the other gets a clear conflict. | P0 |
| FR-QUE-6 | The ETA per entry is computed from remaining service durations, eligible barbers' availability, in-progress services (elapsed vs expected), and upcoming appointments. It is shown as a range. | P0 |
| FR-QUE-7 | Pause/resume intake per branch. Queue capacity limit per branch. Intake auto-closes at a configured time before closing. | P0 |
| FR-QUE-8 | Authorized reordering (manager+), with a mandatory reason and an audit record. | P0 |
| FR-QUE-9 | **Product rule (provisional D-3):** one active **remote queue session** per customer platform-wide. A session belongs to one branch and may contain several linked people (D-21), each with their own entry and position. The database enforces it race-free; it can change later without a domain redesign. See [queue-lifecycle §3](domain/queue-lifecycle.md). | P0 |
| FR-QUE-10 | A remote customer who hasn't arrived by `arrive_by` + tolerance becomes **EXPIRED** (never NO_SHOW, which is reserved for arrived customers who don't respond when called). The customer is warned before expiry. | P0 |
| FR-QUE-11 | Public in-shop display (TV/tablet) shows **ticket numbers and the called barber's display name only**. No customer names or initials. | P1 |
| FR-QUE-12 | Kiosk self check-in. | P1 |
| FR-QUE-13 | ML-based ETA. | P2 |
| FR-QUE-14 | Customer self-confirmation of arrival for their own remote entries, with a staff fallback. Every arrival records who or what confirmed it and when. | **Open (D-36)**: PROPOSED P0, not approved |

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
| FR-REV-1 | Only the **owner of a COMPLETED visit** can review it. The owner is the user recorded on the visit at creation, or the user who claimed that visit (FR-REV-9). One review per visit, submitted within N days (D-10). | P0 |
| FR-REV-2 | A review has a 1–5 rating and optional text. It counts toward **both** the branch rating and the serving barber's rating at that business (D-10, approved). | P0 |
| FR-REV-3 | Public display shows the reviewer's **public display name**, which the user chooses and can edit (Arabic names don't reliably split into first/last), plus the visit month. It never shows a phone, email, or exact visit time. | P0 |
| FR-REV-4 | Owners/managers may post one public reply per review. | P0 |
| FR-REV-5 | Anyone can report a review. Moderators can hide or restore it, with an audited reason. Automated pre-checks (profanity, PII patterns). | P0 |
| FR-REV-6 | Businesses cannot edit or delete reviews, and cannot choose which customers get review invitations (no "review gating"). | P0 |
| FR-REV-7 | Aggregates (average, count, distribution) are recomputed on change. A minimum review count is required before a public average is shown (e.g., 3). | P0 |
| FR-REV-8 | Review photos. | P1 |
| FR-REV-9 | **Visit claim invitation**: after a completed visit by a customer without a linked account (phone on record), a single-use link is sent. Claiming requires OTP sign-in with that same phone and grants access to **that visit only** (view + review). It does not link the customer record or expose other visits. Residual shared-phone risk is open (D-30). | P0 |

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
| FR-CRM-1 | A business can hold a **customer record** that is not linked to a platform user (walk-in with only a name and/or phone). Linking to a platform user requires the user to verify the phone **and** book or join at that business. Linking affects future visits only (D-25). | P0 |
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
| NFR-7 Privacy | — | Data minimization. Compliance with Egypt PDPL Law 151/2020 and its executive regulations (Nov 2025; compliance period reported to end 1 Nov 2026): consent, breach notification (reported 72 h to the regulator), a cross-border transfer licence if data leaves Egypt (D-23), guardian consent for under-15s (D-24). Confirm with counsel (D-17). |
| NFR-8 Backup/DR | — | Point-in-time recovery; RPO ≤ 5 min, RTO ≤ 4 h; restore tested quarterly |
| NFR-9 Accessibility | — | WCAG 2.2 AA for web; platform accessibility APIs on mobile |
| NFR-10 Localization | — | ar/en, RTL layouts, locale-aware numbers/dates, IANA time zones |
| NFR-11 Observability | — | Structured logs, traces, and metrics with tenant id; alerting on error rate, latency, and real-time lag |
| NFR-12 Mobile network tolerance | — | Commands are idempotent and retry-safe; clients recover from reconnects automatically |

## 7. Key business rules (initial; defaults are configurable per business within limits)

| ID | Rule |
|---|---|
| BR-1 | A confirmed booking's price, duration, and services are snapshotted. Changing the catalog does not change existing bookings. |
| BR-2 | A barber cannot have two overlapping **planned** blocks (appointments, time off), including buffers. The database enforces this. **Actual** service time may overrun and is handled by ETA and notifications (review C-3). |
| BR-3 | Booking minimum notice (default 30 min), horizon (default 30 days), and slot granularity (default 15 min) are configurable. |
| BR-4 | Default cancellation: free until X hours before the appointment; later = late cancellation (recorded, no fee in the MVP). |
| BR-5 | A confirmed appointment has priority at its start time (+ grace, default 10 min). A walk-in is offered to a barber only if its expected duration fits before the barber's next appointment, or no eligible barber has an appointment conflict (D-4). |
| BR-6 | A remote queue entry must check in within the arrival window. After a grace period it expires and the customer is notified. |
| BR-7 | **Product rule (provisional):** one active remote queue session per customer across the platform. A session covers one branch and may hold several linked people. It ends automatically when its last entry ends. |
| BR-8 | Only staff assigned to the branch can act on its queue. Reordering requires Manager+ and a reason. |
| BR-9 | Reviews only for COMPLETED visits, one per visit, within N days. |
| BR-10 | Public surfaces (profiles, reviews, displays, queue position) never expose customer phone, email, full name, or precise location. |
| BR-11 | A queue entry and an appointment both result in a **visit** when service starts. The visit is the record of what actually happened (who served, start/end, charged price). See [ADR-0004](adr/0004-separate-appointment-queue-entry-visit.md). |
| BR-12 | Queues, ticket numbers, and daily reports use the branch **business day** (rollover default 05:00 local, D-22). Non-terminal entries expire at rollover. |
| BR-13 | Rescheduling keeps the original price snapshot unless the services change. |
| BR-14 | A late appointment beyond grace may be **converted to a walk-in** (a new queue entry) or marked no-show by staff (D-27). |
| BR-15 | A barber serves one customer at a time (D-28). |
| BR-16 | Groups are separate entries/appointments linked by a group id. They can be cancelled together (D-21). |
| BR-17 | Name-only walk-ins create no customer record. A phone creates or reuses one (D-6). |
| BR-18 | Linking a walk-in record to a user never exposes earlier visits. A user sees a visit only if it was recorded under their account at creation, or if they claimed that specific visit through its invitation. Phone matching alone is never proof of ownership of past visits. The business sees all (D-25). |
| BR-19 | A guardian may book or queue for a child using an optional attendee label. A consent record is stored. No child accounts or birth dates (D-24, pending legal). |
| BR-20 | The business owns its customer records. A departing barber takes none (D-5c). Future appointments must be reassigned or cancelled before the barber is marked departed. |
| BR-21 | A branch closure or reduction of hours cannot be saved while it overlaps live appointments, unless each one has an explicit resolution (cancel by business with reason, or reassign). Appointments are never silently cancelled or rescheduled. Affected customers are notified. See [booking-lifecycle §5a](domain/booking-lifecycle.md) (D-31). |

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
  - Given entry E is CALLED for barber S.
  - When two "start" requests for E arrive simultaneously (e.g., S and the front desk),
  - then exactly one creates a visit, the other receives `409 INVALID_STATE` with the current state, and exactly one visit references E.
- **AC-TEN-isolation**
  - Given a Manager of tenant A.
  - When they request any tenant-B resource by id (booking, customer, queue entry),
  - then the API returns `404`, not `403`, so ids are not confirmed to exist.
  - A direct SQL query under tenant A's context returns 0 rows of tenant B.
- **AC-REV-eligibility**
  - Given a visit that is IN_SERVICE or VOIDED, or COMPLETED with `review_eligible_until` in the past, or a caller who doesn't own the visit, review creation → `403 REVIEW_NOT_ELIGIBLE`.
  - An appointment or queue entry that never produced a visit (cancelled, no-show, expired) cannot be reviewed, because there is no visit.
  - A second review for the same visit → `409`.
- **AC-REV-claim**
  - Given a completed phone-only walk-in visit V with an issued invitation.
  - When a user signed in with the same phone opens the link,
  - then they can see and review V only; their other visits at that business made before linking stay invisible.
  - A second claim, a wrong phone, an expired token, or a revoked token → identical `404 INVITATION_INVALID`.
- **AC-QUE-remote-session**
  - Given a user with an ACTIVE remote session at branch X containing 3 linked entries.
  - When they try to join branch Y → `409 ACTIVE_SESSION_EXISTS`.
  - When all 3 entries become terminal, the session ends in the same transaction, and joining Y then succeeds.
  - Two concurrent first joins at X and Y → exactly one session.
- **AC-BKG-closure**
  - A closure overlapping 2 confirmed appointments without resolutions → `409 CLOSURE_HAS_APPOINTMENTS`.
  - With resolutions for both → both are cancelled or reassigned in the same transaction, with audit and notifications.
  - With a stale resolution set (a booking arrived meanwhile) → `409 AFFECTED_SET_CHANGED`.
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

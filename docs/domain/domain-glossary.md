# Domain Glossary

Version 0.2 · 2026-10-14 (adds remote session, visit owner/claim, arrival confirmation, `PROVIDER`; audit round 2). These are the canonical terms. Code, API, database, and UI copy must use them. Arabic UI labels are listed where they matter for consistency; final copy comes from the UX writer.

| Term | Definition | Not to be confused with |
|---|---|---|
| **User** | A platform identity: one person, one account, verified phone. Shared across all businesses and future verticals. | Customer, Staff member |
| **Platform role** | A grant to a User for platform operations: `ADMIN`, `MODERATOR`, `SUPPORT`. | Tenant role |
| **Tenant** | The isolation, ownership, and billing unit. All operational data belongs to exactly one tenant. | Business (brand), Branch (place) |
| **Business** | The public brand of a tenant: name, category, profile. 1:1 with Tenant in the MVP (D-5b). | Tenant |
| **Business category** | Platform-defined vertical, e.g. `barber`. Selects vertical policies. | Service type |
| **Branch** | A physical place where services are delivered. It has coordinates, a time zone, hours, a business-day rollover, and exactly one queue. UI: فرع. | Business address (legal), user location |
| **Membership** | A User's role in a Tenant: `OWNER`, `MANAGER`, `FRONT_DESK`, `PROVIDER` (the code is vertical-neutral; shown as "Barber" for barbers, proposed D-37). Grants permissions. A user may hold memberships in several tenants. | Staff member |
| **Tenant role** | The role on a Membership. | Platform role |
| **Staff member** | A bookable service provider in a tenant (a barber). It may be linked to a Membership, or exist without any user account (D-29). Owns working hours and service capabilities. | User, Membership |
| **Independent barber** | In the MVP, an independent barber is a one-person Tenant whose owner is also its only Staff member (D-5). | — |
| **Service** | A tenant-level definition of what is sold: name, default duration, base price, optional service type. | Offering |
| **Service type** | Platform taxonomy used for cross-business search filters (e.g., `HAIRCUT`, `BEARD`, `KIDS_HAIRCUT`). | Service |
| **Branch offering** | Whether a Service is offered at a Branch, plus branch price/duration overrides. | Service |
| **Staff capability** | Which Staff member performs which Service, plus optional per-staff price/duration overrides (D-12). | — |
| **Price quote** | The result of price resolution: staff → branch → base. It is a value object and is not stored, except inside snapshots. | Price snapshot |
| **Price snapshot** | Prices and durations copied onto an Appointment (at confirmation) or a Visit (at start). Later catalog changes never alter it. | — |
| **Working hours** | A staff member's recurring weekly availability at a branch, in local time. | Opening hours |
| **Opening hours** | A branch's recurring weekly hours, in local time. Intervals may cross midnight. | — |
| **Schedule exception** | A dated override: branch closure, special hours, staff time off. | — |
| **Business day** | The operating day of a branch, from `day_rollover` local time to the next day's `day_rollover` (default 05:00, D-22). Tickets, queue resets, and daily reports use it. | Calendar date |
| **Staff commitment** | A **planned** block of a staff member's time (appointment or time off). Planned blocks may not overlap; the database enforces this. | Visit (actual time) |
| **Appointment** | A reservation of a specific staff member's time in the future. UI: حجز / موعد. | Queue entry, Visit |
| **Queue** | The single live walk-in line of a branch (D-2). UI: الدور. | Slot waitlist |
| **Queue entry** | A person's place in a branch queue (walk-in, remote join, or converted late appointment). | Appointment, Visit |
| **Ticket number** | A short per-branch, per-business-day number shown publicly instead of names. | Entry id |
| **Presence** | Whether a queue entry's person is physically at the branch: `NOT_ARRIVED` / `ARRIVED`. Kept separate from state. | Entry state |
| **Arrival confirmation** | Who or what set presence to ARRIVED (`STAFF`, `CUSTOMER_SELF`, `KIOSK`), by whom, and when. Customer self-confirmation is open (D-36). | Appointment check-in |
| **Remote queue session** | A customer's single active remote party at one branch. It holds one or more linked queue entries (companions, children). At most one ACTIVE session per user platform-wide is a **product rule** (P-1), not an invariant. | Group (staff-created) |
| **Arrival window** | The time by which a remote joiner must check in. | Grace period |
| **Grace period** | The tolerated lateness for an appointment start or a called entry. | — |
| **Lane** | A computed, unstored view of a staff member's current visit, checked-in appointments, and assigned or eligible queue entries. | Queue |
| **Dispatch** | Choosing which entry or appointment a staff member serves next, according to the branch policy (D-4). | — |
| **ETA** | Estimated time until a queue entry is called, shown as a range. | Appointment time |
| **Visit** | The service actually delivered: who served whom, real start/end, services, and amount charged. The single source for service history, analytics, and review eligibility. States: `IN_SERVICE`, `COMPLETED`, `VOIDED` only. UI: زيارة. | Appointment, Queue entry |
| **Visit owner** | The platform User allowed to see and review one visit: the user recorded on the visit at creation, or the user who claimed it. | Customer (identity), linking |
| **Visit claim invitation / claim** | A single-use, phone-bound link issued for one completed visit, and the resulting ownership record for **that visit only**. | Customer linking |
| **Customer** | A tenant-owned record of a person the business serves. It may be linked to a User or exist on its own (walk-in with name and optional phone). Owned by the business (D-5c). | User |
| **Attendee** | The person actually served when it differs from the booker (e.g., a child, D-24). Stored as an optional label, not a separate profile. | Customer |
| **Group** | A link between several appointments or entries made together (e.g., a parent and children, D-21). | Party size |
| **Review** | A rating (1–5) and optional text about one completed Visit. It counts toward both the branch rating and the serving staff member's rating (D-10). | Rating aggregate |
| **Rating aggregate** | Precomputed count, sum, and distribution for a branch or a staff member. | — |
| **Notification** | One message to one recipient on one channel about one domain event. | Domain event |
| **Domain event** | A fact recorded in the outbox (e.g., `queue.entry.called`). Notifications, projections, and real-time updates consume it. | Notification |
| **Audit event** | An append-only record of a security-relevant action, with actor, target, and reason. | Domain event, queue event |
| **Discovery listing** | A public, PII-free projection of a branch used for search. | Branch |
| **Slot waitlist** *(P1)* | Customers asking to be told when an appointment slot opens. | Queue |

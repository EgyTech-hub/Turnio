# ADR-0004: Appointment, queue entry, and visit are separate aggregates

- Status: **Proposed** · 2026-10-14
- Business inputs approved 2026-10-14: D-2 (branch queue), D-4 (appointment priority + fit rule)
- Context docs:
  - [domain-model §6](../domain/domain-model.md)
  - [booking-lifecycle](../domain/booking-lifecycle.md)
  - [queue-lifecycle](../domain/queue-lifecycle.md)
  - [05 review C-2](../05-design-review.md)

## Context
In v0.1 docs, `IN_SERVICE`/`COMPLETED` appeared on appointments, queue entries, *and* visits. Three copies of service progress would drift apart, double-count in analytics, and confuse review eligibility. The brief also requires that an appointment is not automatically a queue entry and that neither is automatically a completed visit.

## Decision
- **Appointment** = reservation of a staff member's time. It ends in `FULFILLED`, `CANCELLED`, `NO_SHOW`, `CONVERTED_TO_WALK_IN` (or `REJECTED`/`EXPIRED` with P1 approval).
- **Queue entry** = place and presence in a branch line. It ends in `SERVED`, `CANCELLED`, `NO_SHOW`, `EXPIRED`.
- **Visit** = the service actually delivered. It is the only aggregate with `IN_SERVICE → COMPLETED | VOIDED`. It is created when service starts and references exactly one source (DB CHECK); each source has at most one visit (unique indexes).
- A checked-in appointment is **not** copied into the queue. Lanes and dispatch query both kinds of items.
- A late appointment that becomes a walk-in creates a **new** queue entry with `origin_appointment_id`.
- Reviews attach to completed visits only. **Visit ownership** (who may see or review a visit) is separate from customer identity: it comes from the user recorded on the visit at creation, or from a per-visit claim (B2; [domain-model §7](../domain/domain-model.md)).
- Whether an appointment may start without an explicit check-in is open (D-33).

## Consequences
- ➕ One source of truth for service facts. Clean analytics (funnel vs delivery). Simple review eligibility. Database-enforced linkage.
- ➕ Future verticals (clinics) can reuse Visit and add a different source type (e.g., referral). *Corrected 2026-10-14 (audit N5):* this **does** require changing the source-integrity rule (the `num_nonnulls(...) = 1` CHECK) and adding a source column and unique index. It doesn't require changing booking or queue.
- ➖ The staff board must merge two sources into lanes (query-side complexity), and the start transaction touches two aggregates (acceptable in the monolith, ADR-0001).

## Alternatives
- **Appointments become queue entries on check-in:** rejected. It duplicates state and loses the reservation semantics needed for the D-4 priority rule.
- **A single "ticket" entity for everything:** rejected. It mixes the lifecycles, so the constraints become conditional on type.

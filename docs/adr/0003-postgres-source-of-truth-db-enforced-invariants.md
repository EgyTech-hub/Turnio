# ADR-0003: PostgreSQL is the source of truth; core invariants enforced in the database

- Status: **Proposed** · 2026-10-14
- Context docs: [03 §4](../03-mvp-and-architecture.md) · [database-design §5–8](../architecture/database-design.md)
- 2026-10-14 review: the commitment table is renamed `scheduling.staff_commitment` with `resource_id` and `kind` (appointment / time off). Only *planned* time is conflict-checked; actual service overruns are not (review C-3). Added invariants: one IN_SERVICE visit and one CALLED entry per staff member; a fixed lock order; READ COMMITTED with explicit locks. Writes under one queue lock remain an assumption to measure (ASSUMPTION-11).
- 2026-10-14 audit round 2 (B1, B3, B4): the Decision section below is updated in place for the visit-based service model, remote sessions, and the queue mutation protocol. Status is unchanged: **Proposed**.

## Context
- Checking availability and then inserting is unsafe under concurrency.
- Queue actions can race: two staff members may act on the same entry.
- Real-time delivery can drop messages.

## Decision
- **No double booking:** a GiST exclusion constraint on `scheduling.staff_commitment (resource_id, period)` for active commitments. These are planned time only: appointments and time off.
- **No double serving:**
  - conditional updates on `(state, version)`;
  - partial unique indexes: one CALLED queue entry per provider, and one IN_SERVICE **visit** per provider (service state lives on the visit, ADR-0004);
  - a visit has exactly one source, and each source has at most one visit.
- **Remote sessions:** one ACTIVE remote session per user platform-wide. This is a **product policy** (P-1, provisional D-3), enforced race-free by a partial unique index. It is not a domain invariant.
- **Ordered events:** every lane-affecting command follows the queue mutation protocol ([database-design §6.1](../architecture/database-design.md)):
  - one transaction;
  - `UPDATE queue SET seq = seq + 1 RETURNING` (lock + exactly one `seq`);
  - `queue_event (seq, idx)` rows plus one outbox row;
  - commit.
  A rollback consumes no `seq` and publishes nothing.
- **Redis or a broker never holds authoritative state.** Clients recover from gaps via `seq` replay or an HTTP snapshot.
- **Idempotency:** idempotency keys are stored transactionally for creation commands. Only successful outcomes are replayed. State transitions rely on version preconditions.

## Consequences
- ➕ Correct under concurrency by construction. Testable with real PostgreSQL in CI.
- ➖ Writes to a single queue are serialized. This is acceptable if a branch queue sees at most about one command every few seconds at peak (ASSUMPTION-11, to be measured in the pilot).
- ➖ More hand-written SQL. Mitigation: a SQL-first migration tool and query builder (Kysely/Drizzle).

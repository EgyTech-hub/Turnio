# ADR-0003: PostgreSQL is the source of truth; core invariants enforced in the database

- Status: **Proposed** · 2026-10-14
- Context docs: [03 §4](../03-mvp-and-architecture.md)

## Context
- Checking availability and then inserting is unsafe under concurrency.
- Queue actions can race: two staff members may act on the same entry.
- Real-time delivery can drop messages.

## Decision
- **No double booking:** a GiST exclusion constraint on `(staff_id, tstzrange)` for active commitments.
- **No double serving:** conditional updates on `(state, version)`, plus partial unique indexes (one in-service entry per staff member; one active remote entry per customer).
- **Ordered events:** a per-queue `seq` is incremented in the same transaction as the change. Events go to an append-only `queue.event` log and to an outbox.
- **Redis or a broker never holds authoritative state.** Clients recover from gaps via `seq` replay or an HTTP snapshot.
- **Idempotency:** idempotency keys are stored transactionally for booking and queue commands.

## Consequences
- ➕ Correct under concurrency by construction. Testable with real PostgreSQL in CI.
- ➖ Writes to a single queue are serialized. This is acceptable: one barbershop queue sees a few writes per minute.
- ➖ More hand-written SQL. Mitigation: a SQL-first migration tool and query builder (Kysely/Drizzle).

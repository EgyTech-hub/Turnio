# ADR-0001: Start with a modular monolith

- Status: **Proposed** · 2026-10-14
- Context docs: [03 §2](../03-mvp-and-architecture.md)

## Context
The hardest invariants touch appointments, queue entries, staff commitments, and visits together. They need to be enforced together at one shop at one moment. The team is expected to be small (D-19). Supporting future verticals is a goal but not an MVP requirement.

## Decision
- Build one backend codebase (NestJS) with modules as the unit of ownership.
- Each module owns a PostgreSQL schema.
- Modules talk to each other through public facades and outbox domain events.
- The codebase is deployed as three processes: `api`, `realtime`, and `worker`.

## Consequences
- ➕ ACID transactions across booking, queue, and visit; one pipeline; simple debugging and testing.
- ➖ Boundaries can erode over time. Mitigation: architecture lint (no cross-module imports except facades), and no cross-schema SQL outside the module.
- ➖ The whole app scales together. This is acceptable at the expected load. The extraction triggers are listed in 03 §2.

## Alternatives
- **A few services:** rejected for now; adds sagas to the core flow.
- **Microservices:** rejected; operational cost with no current benefit.

# Project Documentation

Multi-tenant service discovery, booking, and real-time queue platform.
Initial vertical: barbershops and independent barbers.

| # | Document | Status | Last updated |
|---|----------|--------|--------------|
| 01 | [Market research & competitor matrix](01-market-research.md) | Draft v0.1 (+ wording fixes) | 2026-10-14 |
| 02 | [Product requirements (initial PRD)](02-product-requirements.md) | Draft v0.3 | 2026-10-14 |
| 03 | [MVP & architecture recommendation](03-mvp-and-architecture.md) | Draft v0.3 | 2026-10-14 |
| 04 | [Decision register](04-open-decisions.md) | v0.3.1 (9 resolved; D-30…D-40 open) | 2026-10-15 |
| 05 | [Design review + audit resolutions](05-design-review.md) | v1.3 | 2026-10-15 |
| 06 | [Validation plan](06-validation-plan.md) | Draft v0.2 | 2026-10-14 |
| D1 | [Domain glossary](domain/domain-glossary.md) | Proposed v0.2 | 2026-10-14 |
| D2 | [Domain model](domain/domain-model.md) | Proposed v0.2 | 2026-10-14 |
| D3 | [Booking lifecycle](domain/booking-lifecycle.md) | Proposed v0.2 | 2026-10-14 |
| D4 | [Queue, remote session & visit lifecycle](domain/queue-lifecycle.md) | Proposed v0.2 | 2026-10-14 |
| A1 | [Database design](architecture/database-design.md) | Proposed v0.3 | 2026-10-15 |
| A2 | [Tenant isolation & security model](architecture/tenant-isolation-and-security.md) | Proposed v0.3.1 | 2026-10-15 |
| ADR | [Architecture decision records](adr/) — 0001–0005 | All Proposed | 2026-10-14 |

## Conventions

- **Evidence labels** (used in research documents):
  - **[V]** Verified on the vendor's own page, documentation, or policy (source cited).
  - **[S]** Supported by a secondary source (review site, analyst report, app store mirror). Treat as likely, not confirmed.
  - **[I]** Inferred from a public description; not directly stated.
  - **[?]** Not found / unverified. Absence of evidence is not evidence of absence.
  - **[R]** Our recommendation.
- **Assumptions** are tagged `ASSUMPTION-n` and must be validated or converted to a decision.
- **Decisions** are tagged `D-n` and tracked in [04-open-decisions.md](04-open-decisions.md).
- **Requirements** are tagged `FR-<area>-n` / `NFR-n` with priority P0 / P1 / P2.
- Documents change via pull request. Do not overwrite earlier reasoning; supersede it and say why.

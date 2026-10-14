# Project Documentation

Multi-tenant service discovery, booking, and real-time queue platform.
Initial vertical: barbershops and independent barbers.

| # | Document | Status | Last updated |
|---|----------|--------|--------------|
| 01 | [Market research & competitor matrix](01-market-research.md) | Draft v0.1 | 2026-10-14 |
| 02 | [Product requirements (initial PRD)](02-product-requirements.md) | Draft v0.1 | 2026-10-14 |
| 03 | [MVP & architecture recommendation](03-mvp-and-architecture.md) | Draft v0.1 | 2026-10-14 |
| 04 | [Open decisions (blocking database design)](04-open-decisions.md) | Draft v0.1 | 2026-10-14 |
| ADR | [Architecture decision records](adr/) | Proposed | 2026-10-14 |

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

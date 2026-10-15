# ADR-0005: Cloud-portable deployment pending the Egypt data-residency decision

- Status: **Proposed** · 2026-10-14 · Blocks: production data processing (not development)
- Related decisions: D-1 (Egypt, resolved), D-17, D-23 (open — legal)

## Context
- Egypt's PDPL (Law 151/2020) executive regulations were issued in November 2025, with a compliance period reported to end on 1 November 2026.
- Transferring personal data outside Egypt reportedly requires a **separate licence/permit** from the Personal Data Protection Center (Art. 16), with fees and a decision period.
- Sources disagree on details and on how operational the regulator is. See [05 §8](../05-design-review.md).
- Push notifications (APNs/FCM), SMS/WhatsApp providers, and error-tracking SaaS may all be cross-border processors.

## Decision (provisional)
1. **Stay cloud-portable** until counsel advises:
   - Containers;
   - standard PostgreSQL + PostGIS (no proprietary DB features);
   - S3-compatible object storage;
   - OpenTelemetry for observability;
   - no managed services without an equivalent in-country.
2. **Two candidate deployment models** are evaluated in validation activity A5/A6:
   - **(a) In-country hosting** (provider TBD): no transfer licence for primary storage. Possibly higher ops cost and fewer managed services.
   - **(b) Regional cloud + transfer licence:** more managed services. Licence cost and lead time; legal risk if the licence is delayed.
3. **Minimize cross-border data regardless of the choice:**
   - push payloads carry no personal data;
   - error tracking scrubs PII (or is self-hosted);
   - SMS/WhatsApp providers receive only the phone and the message text needed.
4. **Development and staging use synthetic data only** until this ADR is accepted.

## Consequences
- ➕ Implementation can start without waiting for counsel. Nothing in the schema depends on the region.
- ➖ We may forgo some managed-service conveniences until the decision is made.
- ⚠ Launch is blocked until D-23 is resolved and the required licences/permits are in place.

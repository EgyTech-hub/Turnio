# ADR-0002: Shared schema multi-tenancy enforced by PostgreSQL RLS

- Status: **Proposed** · 2026-10-14
- Context docs: [03 §5](../03-mvp-and-architecture.md) · [tenant-isolation-and-security](../architecture/tenant-isolation-and-security.md) · [database-design](../architecture/database-design.md)
- 2026-10-14 review: D-5 (employees-only, no cross-tenant queues) and D-5c (the business owns customers) were approved, which removes the main redesign risk for this ADR. It stays *Proposed* until technical sign-off. Additions: composite `(tenant_id, id)` foreign keys; per-context policies (tenant, customer, customer-command, public, admin-grant, worker); RLS is explicitly defense in depth, not the primary authorization mechanism.

## Context
We expect thousands of small tenants. The platform also needs cross-tenant features: discovery, and a customer's own history across shops. Tenant isolation is a security requirement.

## Decision
- Every tenant-owned row carries `tenant_id`.
- RLS is enabled and forced on all tenant-owned tables.
- The API sets `SET LOCAL app.tenant_id` / `app.user_id` per transaction, after it has authorized the caller's membership.
- The application database role does not own tables and has no `BYPASSRLS`.
- Public discovery reads a PII-free projection.
- Admin access goes through an audited, time-boxed context.
- *Updated 2026-10-14 (audit round 2, B6; status still Proposed):*
  - tables are protected according to an explicit **classification registry**, not inferred from `tenant_id`;
  - each surface uses its **own DB role** (staff, customer, auth, worker, admin, public), and every policy is `TO`-scoped;
  - customer commands never get full business context;
  - the admin grant is transaction-local;
  - a separate maintenance role handles partition DDL;
  - no runtime view or function bypasses RLS.
  See [tenant-isolation-and-security v0.2](../architecture/tenant-isolation-and-security.md).

## Consequences
- ➕ One migration path. Cross-tenant platform features stay simple. Isolation is enforced even when application code has a bug.
- ➖ RLS mistakes are subtle (pooling, `SECURITY DEFINER`, views). Mitigations:
  - a CI check against the classification registry (forced RLS, `TO`-scoped policies, no `BYPASSRLS`, no definer views/functions);
  - a cross-tenant test suite;
  - `security_invoker` views only.
- ➖ Noisy neighbors share one DB. Large or regulated tenants can later be moved to a dedicated DB, because ids are UUIDs and there are no cross-tenant foreign keys.

## Alternatives
- **Schema-per-tenant:** rejected; migration fan-out across thousands of schemas.
- **DB-per-tenant:** rejected for now; kept as an option for enterprise or regulated verticals.

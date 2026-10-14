# ADR-0002: Shared schema multi-tenancy enforced by PostgreSQL RLS

- Status: **Proposed** · 2026-10-14
- Context docs: [03 §5](../03-mvp-and-architecture.md)

## Context
We expect thousands of small tenants. The platform also needs cross-tenant features: discovery, and a customer's own history across shops. Tenant isolation is a security requirement.

## Decision
- Every tenant-owned row carries `tenant_id`.
- RLS is enabled and forced on all tenant-owned tables.
- The API sets `SET LOCAL app.tenant_id` / `app.user_id` per transaction, after it has authorized the caller's membership.
- The application database role does not own tables and has no `BYPASSRLS`.
- Public discovery reads a PII-free projection.
- Admin access goes through an audited, time-boxed context.

## Consequences
- ➕ One migration path. Cross-tenant platform features stay simple. Isolation is enforced even when application code has a bug.
- ➖ RLS mistakes are subtle (pooling, `SECURITY DEFINER`, views). Mitigations:
  - a CI check that every tenant table has a policy;
  - a cross-tenant test suite;
  - `security_invoker` views only.
- ➖ Noisy neighbors share one DB. Large or regulated tenants can later be moved to a dedicated DB, because ids are UUIDs and there are no cross-tenant foreign keys.

## Alternatives
- **Schema-per-tenant:** rejected; migration fan-out across thousands of schemas.
- **DB-per-tenant:** rejected for now; kept as an option for enterprise or regulated verticals.

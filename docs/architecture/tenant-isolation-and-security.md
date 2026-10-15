# Tenant Isolation & Data Security Model

Version 0.3 · 2026-10-15 · Status: **Proposed**. Refines [ADR-0002](../adr/0002-tenancy-shared-schema-rls.md) and [03 §5](../03-mvp-and-architecture.md). Table classes come from [database-design §3.0](database-design.md).

> **Changelog v0.2** (audit round 2, B6; see [05 §9](../05-design-review.md)):
> 1. Explicit table classification instead of inferring from `tenant_id`.
> 2. The public reviews view is replaced by a PII-free projection table; **no runtime view bypasses RLS**.
> 3. Separate maintenance role.
> 4. Every policy is scoped with `TO <role>`, plus an access matrix.
> 5. Access controls for identity and notification tables.
> 6. Customer operations use a separate database role, with narrow own-row access and no access to business customer lists. This adds a split of queue-entry personal data into `queue.queue_entry_party`.
>
> The admin grant context is **transaction-scoped**. The `due_*` RLS-exempt views from v0.1 are withdrawn.
>
> **Changelog v0.3** (focused security review, 2026-10-15; see [05 §10](../05-design-review.md)):
> - Least-privilege login path (§5.7).
> - Explicit per-role access for `queue_entry_party` and customer write paths (§5.3a).
> - Claim invitations bound to the presented token (§5.3).
> - Idempotency rows scoped to the caller (§3.1).
> - OTPs no longer routed through notifications.
> - Tests added (§10).
>
> **v0.3.1 (pre-commit gate, 2026-10-15; [05 §11](../05-design-review.md)):**
> - OTP parameters and concurrent verification.
> - Phone-hash key rotation and compromise recovery.
> - Customer-role column grants on `queue_entry`, `staff_commitment`, and `crm.customer`.
> - Read context for customer status and real-time reads.

## 1. What we protect against

| Threat | Example |
|---|---|
| Cross-tenant read/write | A manager of shop A reads shop B's customers by guessing or reusing an id |
| Customer-to-business overreach | A customer endpoint bug returns the shop's customer list or other customers' phones |
| Privilege escalation inside a tenant | A provider (barber) changes prices or exports customer phones |
| Customer data exposure on public surfaces | Reviews, queue displays, or discovery revealing names, phones, or locations |
| Abuse | OTP pumping, fake queue joins, review fraud, scraping, visit-claim hijacking |
| Insider/admin misuse | Support staff browsing tenant data without a reason |
| Context leakage through infrastructure | A pooled DB connection carrying the previous request's tenant; cache keys without a tenant id; real-time channels |

## 2. Defense layers

RLS is one layer. **Application authorization (L3) is the primary control. RLS (L4) is defense in depth.**

| # | Layer | Responsibility | Fails closed? |
|---|---|---|---|
| L1 | **Authentication** | OTP login; short-lived access token (≤ 15 min) + rotating refresh token with reuse detection; admin MFA | ✔ |
| L2 | **Request context resolution** | Derive the *acting context* from the token and endpoint: `staff` (membership chosen by a selector, **verified against the user's active memberships**), `customer`, `customer_command`, `public`, `admin`. A client-supplied tenant id is only a selector. | ✔ |
| L3 | **Application authorization** | Role permissions (Owner/Manager/Front-desk/Provider); **branch scope**; resource ownership; field-level rules (phones masked for Provider); state-machine guards | ✔ (deny by default; permission matrix tested) |
| L4 | **Database roles + RLS** | A separate DB role per surface (§3). Policies scoped `TO` those roles. No context → no rows. | ✔ |
| L5 | **Schema integrity** | Composite `(tenant_id, id)` FKs: no cross-tenant references can be written | ✔ |
| L6 | **Public projections** | Discovery and published reviews are served from PII-free tables | ✔ |
| L7 | **Audit & detection** | Append-only audit; alerts on admin grants, export volume, claim failures | detective |
| L8 | **Tests** | Isolation suite, permission matrix, catalog checks | preventive |

## 3. Database roles

| Role | Login | Used by | Access summary | RLS |
|---|---|---|---|---|
| `turnio_owner` | **NOLOGIN** | — | Owns schemas and tables | — |
| `turnio_migrator` | CI/CD only | Migrations | Member of `turnio_owner` | n/a |
| `turnio_maint` | Separate scheduled maintenance task only | Partition create/detach/drop, VACUUM/ANALYZE | Member of `turnio_owner`; never deployed with API/worker | n/a (no data reads by design; all actions audited) |
| `turnio_app_staff` | API + real-time (staff surfaces) | Business dashboard, staff app | TENANT_OWNED via the tenant policy; limited identity reads | Subject to RLS |
| `turnio_app_customer` | API + real-time (customer surfaces) | Customer app/web | Own-row policies on personal tables; **non-PII** operational reads of a target tenant during commands (§5.3) | Subject to RLS |
| `turnio_auth` | Identity endpoints only (OTP, login, refresh) | Identity module | One OTP-verified phone and its resolved user per transaction (§5.7); no plaintext phone reads; **no tenant tables** | Subject to RLS (auth policies) |
| `turnio_worker` | Worker | Outbox relay, jobs, notifications, projections | TENANT_OWNED per job tenant context; notification and identity reads per §5.6–5.7; writes projections | Subject to RLS |
| `turnio_admin` | Admin backend (separate deployment, restricted network) | Moderation, verification, support | Moderation tables; tenant data only under an active grant (§5.5) | Subject to RLS |
| `turnio_public` | Public discovery endpoints | Anonymous search, profiles, reviews | `SELECT` on `discovery.branch_listing`, `discovery.published_review`, REFERENCE tables only | n/a (no protected tables granted) |

**Hard rules, checked in CI against the catalog (§10):**
- No login role has `BYPASSRLS`, and no runtime role is a member of `turnio_owner`.
- There are **no** `SECURITY DEFINER` functions and **no** views without `security_invoker = true`, unless they appear in a reviewed allowlist. The allowlist is **empty** for the MVP.
- Every table is classified. Each class has its required RLS state and policy roles.

### 3.1 Access matrix (by data class)

| Data class / table group | app_staff | app_customer | auth | worker | admin | public | maint |
|---|---|---|---|---|---|---|---|
| TENANT_OWNED: operational, non-PII (catalog, scheduling, queue, queue_entry, service lines, staff_live_status, commitments, queue_event) | own tenant R/W | target tenant **R** (non-PII) during `customer_command`, plus the **narrow writes listed in §5.3a** | — | job tenant R/W | grant-gated | — | — |
| TENANT_OWNED: personal (`crm.customer`, `queue_entry_party`, `remote_session`, `appointment`, `visit`, `visit_claim*`, `guardian_consent`) | own tenant R/W (field masking in L3) | **own rows only** (§5.2–5.3) | — | job tenant R/W | grant-gated | — | — |
| TENANT_OWNED: reviews | own tenant R; reply W | own reviews R/W | — | R/W (moderation pipeline) | moderation R/W | — | — |
| TENANT_ROOT `tenancy.tenant` | own row R | — | — | R (ids for sweeps) | R/W | — | — |
| PLATFORM_PERSONAL: identity | names of users with a membership in the current tenant (column-limited) | own rows | login needs (§5.7) | recipients of pending notifications (§5.6) | R (audited) | — | — |
| PLATFORM_PERSONAL: notification | — | own inbox R | — | R/W per job | R (audited) | — | — |
| PLATFORM_PRIVATE | INSERT audit/outbox; idempotency **own rows only** R/W (`user_id = app.user_id()`); **enqueue jobs** (INSERT on the job table only) | same as app_staff | INSERT audit | R/W outbox, jobs, checkpoints | R audit, grants W | — | partitions DDL |
| PUBLIC_PROJECTION | — | R | — | W | R | R | — |
| REFERENCE | R | R | — | R | R/W | R | — |

## 4. Setting context safely with connection pooling

- Each runtime role has **its own connection pool**. A request uses exactly one pool, chosen by its surface (staff, customer, auth, admin, public).
- All database work goes through one helper, e.g. `db.withContext(ctx, tx => …)`. It:
  1. opens a transaction;
  2. runs `SELECT set_config('app.ctx', $1, true), set_config('app.tenant_id', $2, true), set_config('app.user_id', $3, true), set_config('app.verified_phone_hash', $4, true), set_config('app.admin_grant_id', $5, true)`. The auth surface additionally sets `app.login_phone_hash` and `app.refresh_token_hash`, and claims set `app.claim_token_hash`; each is used only by the policies that name it. The `true` flag means **transaction-local** (`SET LOCAL` semantics); every setting, **including the admin grant**, vanishes at COMMIT/ROLLBACK;
  3. runs the work;
  4. commits or rolls back.
- **No session-level `SET`.** This makes the model safe with an application-side pool and with PgBouncer in transaction mode: no state survives into another transaction on the same server connection.
- **Raw pool access is banned** by lint/architecture tests, except in the helper, migrations, maintenance, and the LISTEN connection.
- **Fail closed:** helper functions return NULL when a setting is missing, and NULL comparisons are never true. A query run without context returns zero rows and cannot insert (`WITH CHECK` fails).
- **LISTEN/NOTIFY** uses a dedicated, non-pooled session connection that only listens. It never queries tenant data.
- **Prepared statements** are fine. Policies reference settings at execution time through STABLE functions wrapped in `(SELECT …)`.

## 5. RLS policies by role and context

Helper functions (STABLE, `LANGUAGE sql`, owned by `turnio_owner`, not SECURITY DEFINER) read the settings with `current_setting(name, true)`:
- `app.ctx()`
- `app.tenant_id()`
- `app.user_id()`
- `app.verified_phone_hash()`
- `app.admin_grant_id()`
- `app.login_phone_hash()`, `app.refresh_token_hash()` (auth only)
- `app.claim_token_hash()` (claims only)

Every policy names its roles with **`TO`**. A role with no matching policy sees nothing.

### 5.1 Staff (business dashboard, staff app)

```sql
-- illustrative, on every TENANT_OWNED table
ALTER TABLE booking.appointment ENABLE ROW LEVEL SECURITY;
ALTER TABLE booking.appointment FORCE ROW LEVEL SECURITY;

CREATE POLICY staff_tenant ON booking.appointment
  TO turnio_app_staff
  USING      ((SELECT app.ctx()) = 'staff' AND tenant_id = (SELECT app.tenant_id()))
  WITH CHECK ((SELECT app.ctx()) = 'staff' AND tenant_id = (SELECT app.tenant_id()));

CREATE POLICY worker_tenant ON booking.appointment
  TO turnio_worker
  USING      ((SELECT app.ctx()) = 'worker' AND tenant_id = (SELECT app.tenant_id()))
  WITH CHECK ((SELECT app.ctx()) = 'worker' AND tenant_id = (SELECT app.tenant_id()));
```

Branch scope, role permissions, and phone masking are **L3 application checks**. RLS answers only "which tenant?".

### 5.2 Customer reading their own data (`ctx = 'customer'`)

```sql
CREATE POLICY customer_own ON booking.appointment FOR SELECT
  TO turnio_app_customer
  USING ((SELECT app.ctx()) IN ('customer','customer_command')
         AND customer_user_id = (SELECT app.user_id()));

CREATE POLICY customer_own ON visit.visit FOR SELECT
  TO turnio_app_customer
  USING ((SELECT app.ctx()) IN ('customer','customer_command')
         AND (customer_user_id = (SELECT app.user_id())
              OR EXISTS (SELECT 1 FROM visit.visit_claim c
                         WHERE c.visit_id = visit.id AND c.user_id = (SELECT app.user_id()))));
```

The same pattern applies to:
- `queue.queue_entry_party` and `queue.remote_session` (`customer_user_id`);
- `visit.visit_claim` (`user_id`);
- `review.review` (`author_user_id`);
- `crm.customer` (`user_id`).

`customer_user_id` is set only at creation, and visit access otherwise requires a per-visit claim. Linking a Customer record therefore never exposes historical visits (D-25, B2).

### 5.3 Customer commands (`ctx = 'customer_command'`) — no full business context (B6-6)

Customer commands (book, join/add person, cancel own, claim, review) run as **`turnio_app_customer`** with `app.tenant_id = <target tenant>`, which is resolved from a public branch id or from the caller's own row. They **never** use the staff role or the staff context.

| Need | How it is met without exposing the business's customers |
|---|---|
| Availability, ETA, caps, ticket number | The role reads **non-PII operational tables** of the target tenant (policy: `ctx = 'customer_command' AND tenant_id = app.tenant_id()`): `catalog.*`, `scheduling.*` (including commitments, which carry no customer data), `queue.queue`, **`queue.queue_entry`** (no personal columns, see below), `queue_entry_service_line`, `staff_live_status` (carries `expected_free_at` maintained by the queue protocol, so in-progress visits need not be read), public columns of `tenancy.staff_member`, `tenancy.branch`. |
| Personal data of queue entries | Split into **`queue.queue_entry_party`** (`entry_id` PK, tenant_id, customer_id, customer_user_id, display_name, attendee_label). The customer role sees **only rows where `customer_user_id = app.user_id()`**. The staff role sees all of the tenant's rows. |
| Own customer record | `crm.customer` policy for the customer role: `user_id = app.user_id()` **or** (`user_id IS NULL AND phone_hash = app.verified_phone_hash()`), which is a record bearing the caller's own verified phone (for linking, D-25). INSERT `WITH CHECK (user_id = app.user_id())`. No other rows are ever visible. |
| Appointments, sessions | Own rows only (§5.2). Inserts require `WITH CHECK (customer_user_id = app.user_id() AND tenant_id = app.tenant_id())`. |
| Visits | Read own (§5.2). **Customers never insert or update visits.** |
| P-1 conflict (session at another branch) | Detected by the unique index (§6). No read of other sessions is needed. |
| Visit claim | `visit.visit_claim_invitation` policy `TO turnio_app_customer`: `ctx = 'customer_command' AND tenant_id = app.tenant_id() AND destination_phone_hash = app.verified_phone_hash() AND token_hash = app.claim_token_hash()`. Only the row of the **presented** token is visible; other invitations sent to the same phone (e.g., a family member's visits) are not. The tenant id comes from the claim link. INSERT into `visit_claim` `WITH CHECK (user_id = app.user_id())`. |
| Staff phone numbers, other customers' visits/reviews in moderation | Not granted |

### 5.3a Queue personal data and customer write paths (v0.3)

Separating `queue_entry_party` from `queue_entry` keeps personal data out of the rows that customer commands read for ETA. **The split is not sufficient on its own.** It works together with the per-role policies and grants below, application authorization (L3), and tests (§10).

**Tenant-safe relationships**
- `queue_entry_party (tenant_id, entry_id)` → `queue_entry (tenant_id, id)`.
- `queue_entry_party (tenant_id, customer_id)` → `crm.customer (tenant_id, id)`.
- `queue_entry (tenant_id, queue_id, remote_session_id)` → `remote_session (tenant_id, queue_id, id)`.
- A party row can never point at another tenant's entry or customer.

**`queue.queue_entry_party`: operations per role**

| Role | INSERT | SELECT | UPDATE | DELETE |
|---|---|---|---|---|
| `turnio_app_staff` (tenant) | With its entry (walk-in, direct, converted) | All of the tenant's rows; L3 masks nothing here (no phone in this table), but Provider screens show display name + attendee label only | `display_name`, `attendee_label` corrections (audited) | — |
| `turnio_app_customer` | Own remote party rows only: `WITH CHECK (customer_user_id = app.user_id() AND tenant_id = app.tenant_id())` | Own rows only (`customer_user_id = app.user_id()`) | — (to change companions, cancel and re-add) | — |
| `turnio_worker` | — | Job tenant (notifications, projections never copy names) | Anonymization (retention/erasure) | — |
| `turnio_admin` | — | Only under an active grant | — | — |
| `turnio_public` | — | — | — | — |

No runtime role deletes party rows. Retention anonymizes them in place ([database-design §10](database-design.md)).

**Group organizer**
- In the MVP, every person in a remote session is recorded under the **organizer's** account: `customer_user_id` = organizer, with `attendee_label` naming the companion.
- The organizer therefore sees and can cancel all linked entries through the own-row policy.
- Companions have no separate access; companion accounts are not an MVP feature.

**Customer writes during `customer_command`.** These are all narrow and all require the target tenant.

| Table | Write allowed to `turnio_app_customer` |
|---|---|
| `queue.queue` | Column-level `UPDATE (seq, next_ticket_number)` only, policy `tenant_id = app.tenant_id()`. This is the queue protocol's lock-and-sequence step (database-design §6.1). No other column is updatable. |
| `queue.queue_event` | INSERT `WITH CHECK (tenant_id = app.tenant_id())`; payloads carry ids and states only |
| `queue.queue_entry` | INSERT `WITH CHECK (tenant_id = app.tenant_id() AND source IN ('REMOTE_APP','REMOTE_WEB') AND EXISTS (own ACTIVE remote_session))`. UPDATE (cancel, and self-arrival if D-36 is approved) only `USING EXISTS (party row with customer_user_id = app.user_id())`, restricted by column grants to `state, ended_at, end_reason, presence, arrived_at, arrival_confirmed_by, arrival_confirmed_by_id, version`. Legal transitions are enforced by the application. |
| `queue.remote_session` | Own rows (INSERT/UPDATE for cancel/release) |
| `scheduling.staff_commitment` | INSERT/deactivate only `WITH CHECK (kind = 'APPOINTMENT' AND EXISTS (own appointment))` |
| `crm.customer` | Own row (§5.3 table) |
| `platform.outbox_event`, `platform.audit_event`, jobs | INSERT only |

**Customers without a platform account**
- Name-only and phone-only walk-ins have **no** digital access path in the MVP.
- Every operation on their entries goes through staff (`turnio_app_staff` + L3).
- A phone-only walk-in receives SMS/WhatsApp updates; the message content comes from the worker under the tenant context, with no link to live data.
- After service, they can obtain **one** visit through a claim invitation (§5.3, D-30).
- A tokenized, account-less status page is **not** designed. Adding one would be a product decision (**D-40, open**) and would need its own capability-token design.

**Column-level SELECT grants for `turnio_app_customer`** (rows from other customers are visible to it for ETA, so the columns must carry no identifiers):

| Table | Granted columns (all others denied) |
|---|---|
| `queue.queue_entry` | `id, tenant_id, queue_id, branch_id, business_day, ticket_number, source, state, presence, preferred_staff_id, assigned_staff_id, sort_key, arrive_by, called_at, version`. **Not** `arrival_confirmed_by_id` (a user or membership id), `remote_session_id`, `group_id`, `origin_appointment_id`, or `distance_check_passed`. The customer's own values for those columns are returned through own-row queries on `queue_entry_party` / `remote_session`, or in the command's response. |
| `scheduling.staff_commitment` | `id, tenant_id, resource_id, kind, period, active`. **Not** `appointment_id` or `schedule_exception_id`. |
| `crm.customer` | `id, tenant_id, user_id, linked_at, first_seen_at`. **Not** `display_name`, `phone_e164`, `phone_hash`, `no_show_count`, `remote_cooldown_until`, or `blocked_at`. The own-phone linking match (§5.3) is evaluated by the policy, which needs no SELECT on `phone_hash`. A staff-typed name on a shared family phone (possibly another person's) is therefore never shown. Cooldown and block checks are enforced by the application during the command and returned only as a generic refusal reason. |

**Read context for customer status pages and the customer real-time channel**
- These reads run in `customer_command` context.
- `app.tenant_id` is resolved **from the caller's own** `remote_session`, entry, or appointment row (read first in `customer` context), never from the request.
- They therefore see the non-PII, column-limited operational rows above plus their own personal rows, and nothing else.

**Public access** to queue status uses only `discovery.branch_listing` (open/closed, wait bucket, next slot). No table or column with queue personal data is granted to `turnio_public`. A shop TV display (P1) would need a dedicated PII-free projection (ticket numbers and barber display names), which is not yet designed.

Column-level grants also apply: `turnio_app_customer` has **no** `SELECT` privilege on personal columns of staff-only tables (e.g., `branch.phone_e164` is public, but `invitation`, `membership`, and `customer.phone_e164` are not granted).

### 5.4 Public context

- `turnio_public` has grants only on `discovery.branch_listing`, `discovery.published_review`, and REFERENCE tables.
- `published_review` is a **table** maintained by the worker. It holds rating, body, public display name, visit month, and reply, and no user ids.
- **No view bypasses RLS**, which resolves the v0.1 conflict with the view allowlist.
- No tenant table is reachable.

### 5.5 Platform admin and support

- **No standing access** to tenant operational data.
- **Moderation/verification:** policies `TO turnio_admin` with `app.ctx() = 'admin'` allow cross-tenant SELECT/UPDATE on these tables **only**:
  - `review.review`, `review.review_reply`, `review.content_report`;
  - `tenancy.business`, `tenancy.branch`, `tenancy.tenant`.
- **"View as tenant" (support):**
  1. An `admin_access_grant` row is created (reason, ticket reference, READ/WRITE, max 8 h), and an audit event is written.
  2. For **each transaction**, the admin backend sets `app.admin_grant_id` with `set_config(..., true)`. It is transaction-local, never session-level.
  3. Tenant-table policies `TO turnio_admin` allow rows where an unexpired, unrevoked grant matches `(id = app.admin_grant_id(), admin_user_id = app.user_id(), tenant_id = row.tenant_id)`. WRITE access requires `scope = 'WRITE'`.
  4. **Every request** under a grant writes an audit event.
  5. The tenant Owner can see grants on their tenant (P1 screen).
- **No admin action bypasses domain rules.** Admin commands call the same domain services; there are no "fix it in SQL" endpoints.

### 5.6 Background jobs and notifications

- Jobs carry `(tenant_id, object id)`. They run inside `withContext({ctx: 'worker', tenant_id})` as `turnio_worker`. There are **no RLS-exempt views**; due work is enqueued transactionally as jobs ([database-design §8](database-design.md)).
- **`notification.notification`** (PLATFORM_PERSONAL), RLS enabled + forced:
  - `TO turnio_worker`: rows where `tenant_id = app.tenant_id()` (tenant-originated), or `tenant_id IS NULL AND app.ctx() = 'worker_platform'` (platform messages to registered users, e.g., account notices). **OTP codes never go through this table** (§5.7). They are sent directly by the auth service, because an OTP recipient may have no account, and the `num_nonnulls(recipient_user_id, recipient_customer_id) = 1` rule could not represent them.
  - `TO turnio_app_customer`: `SELECT` where `recipient_user_id = app.user_id()` (in-app inbox).
  - Staff and public roles: no access.
- **Sending:**
  - The worker reads the recipient phone from `crm.customer` under the tenant context, or from `identity.user_phone` (§5.7).
  - It sends, and never copies the phone into logs or the notification row.
  - Payloads carry only ids, business/branch names, ticket numbers, times, and the recipient's own display name.
- **Discovery projection** consumers write `discovery.*` and read tenant rows only under the source tenant's context.

### 5.7 Identity tables and the login path (B6-5; least privilege revised v0.3)

All `identity.*` tables are PLATFORM_PERSONAL with RLS enabled + forced.

**v0.3 change:** v0.2 gave `turnio_auth` *broad* read access to `user_phone`, which was more than login needs. The login role is now limited to **one phone per transaction, and only after that phone has passed OTP verification**. No privileged function or view is introduced; this uses ordinary `TO`-scoped policies and column grants.

**Phone representation**
- `phone_hash = HMAC-SHA-256(pepper, E.164 phone)`.
- The pepper lives in the secret store, never in the database. Without it, the hashes can't be reversed or enumerated by dictionary.
- Plaintext `phone_e164` is kept only where a message must be sent (`user_phone`, `crm.customer`).

**Login flow**

| Step | What the auth service does | What the DB allows `turnio_auth` to touch |
|---|---|---|
| 1. Request OTP | Normalize the phone and compute `phone_hash`. Apply rate limits *before* any DB work. Insert `otp_challenge(phone_hash, code_hash, purpose, expires_at)`. Send the code **directly** through the SMS provider adapter. OTPs are **not** routed through `notification.notification`. Return the same response whether or not an account exists. | INSERT on `otp_challenge` (`WITH CHECK phone_hash = app.login_phone_hash()`). **No read of `user_phone` at this step**, so a request reveals nothing about account existence. |
| 2. Verify OTP | Load the challenge by `(id, phone_hash)`, compare the code hash in constant time, and increment `attempts` (max 5, then lock the challenge). On success, mark it `VERIFIED` (single use). | SELECT/UPDATE on `otp_challenge` rows where `phone_hash = app.login_phone_hash()`. |
| 3. Resolve account | In the **same transaction** as step 2, look up the current `user_phone` row for that `phone_hash`; create the user and phone if none exists (signup). | `user_phone` policy (below): only rows whose `phone_hash` equals the hash of a challenge **verified within 5 minutes and not yet consumed**. |
| 4. Issue session | Read `user_account` status and `platform_role_grant` for the resolved user; create `auth_session`; consume the challenge. | Rows of the resolved user only (`user_id = app.user_id()`, set after step 3 in the same transaction). |
| Refresh | Look up the session by `refresh_token_hash`; rotate it. | `auth_session` rows where `refresh_token_hash = app.refresh_token_hash()`, or `user_id = app.user_id()` after resolution. |

**Login-code parameters and concurrency** (proposed values; subject to specialist review, F-13):
- **Code:** 6 digits from a CSPRNG, stored only as `code_hash = HMAC-SHA-256(pepper, challenge_id ‖ code)`. It expires **5 minutes** after issue.
- **One live challenge per phone:** issuing a new challenge marks any previous `ISSUED` challenge for the same `phone_hash` as `EXPIRED` in the same transaction. A partial unique index `(phone_hash) WHERE status = 'ISSUED'` makes concurrent issues safe: the loser gets the same generic response.
- **Attempts are counted atomically, even under parallel requests:**
  `UPDATE otp_challenge SET attempts = attempts + 1 WHERE id = $id AND phone_hash = app.login_phone_hash() AND status = 'ISSUED' AND attempts < 5 AND expires_at > now() RETURNING …`.
  The code comparison happens only after this row-locking increment, so parallel guesses can't exceed 5.
- **Verification is single-use:** `UPDATE … SET status = 'VERIFIED', verified_at = now() WHERE id = $id AND status = 'ISSUED'`. Of two concurrent correct submissions, only one transitions.
- **Consumption:** session issuance sets `consumed_at`, which ends the 5-minute account-lookup window (step 3) for that challenge.

**Phone-hash key (pepper) rotation and compromise recovery** (procedure outline; needs specialist review before production):
- **Versioning:** every hash column has a sibling `*_key_version`. The secret store holds the current key and, during rotation, the previous key.
- **Planned rotation:**
  1. Write new rows with the new key.
  2. A worker job **rehashes** `user_phone` and `crm.customer` from their plaintext `phone_e164` in batches, under each row's tenant context.
  3. Short-lived hashes are left to expire rather than rehashed: `otp_challenge` (5 min) and `visit_claim_invitation` (≤ 14 days, re-issued on request).
  4. During the overlap, lookups try both versions.
  5. When no rows reference the old version, destroy the old key.
- **Suspected key compromise:** an attacker with the key and a hash dump could brute-force the Egyptian mobile number space. Response:
  1. rotate immediately;
  2. expire all `ISSUED` OTP challenges and unclaimed visit invitations;
  3. rehash;
  4. treat it as a security incident (assess the PDPL breach-notification duty with counsel, D-17).
- Holding the key alone, without database access, reveals nothing.

**Policies and grants** (illustrative):

```sql
-- user_phone: login may see one verified phone per transaction
CREATE POLICY auth_verified_phone ON identity.user_phone FOR SELECT
  TO turnio_auth
  USING ((SELECT app.ctx()) = 'auth'
         AND phone_hash = (SELECT app.login_phone_hash())
         AND EXISTS (SELECT 1 FROM identity.otp_challenge c
                     WHERE c.phone_hash = user_phone.phone_hash
                       AND c.status = 'VERIFIED' AND c.consumed_at IS NULL
                       AND c.verified_at > now() - interval '5 minutes'));
-- inserts (signup) are limited to the verified phone of the transaction
CREATE POLICY auth_signup_phone ON identity.user_phone FOR INSERT
  TO turnio_auth WITH CHECK ((SELECT app.ctx()) = 'auth' AND phone_hash = (SELECT app.login_phone_hash()));
-- column grants: no plaintext phone read for the auth role
GRANT SELECT (id, user_id, phone_hash, is_current, verified_at) ON identity.user_phone TO turnio_auth;
GRANT INSERT (id, user_id, phone_e164, phone_hash, is_current, verified_at) ON identity.user_phone TO turnio_auth;
```

(Inserts must not use `RETURNING phone_e164`, which needs SELECT on that column.)

**What `turnio_auth` can and cannot reach**

| Can | Cannot |
|---|---|
| The challenge rows of the phone hash set for the transaction | Any `user_phone` row without a recently verified, unconsumed challenge for it; any plaintext `phone_e164` |
| The account, session, and platform-role rows of the user resolved in that transaction | Other users' accounts, sessions, or roles; `user_email` except the resolved user's |
| INSERT audit events | Any tenant table, notification, CRM, queue, or visit data (no grants at all) |

**Can it enumerate unrelated accounts?**
- **Through the DB role, no.** A bug or SQL injection in the auth service can see at most the account of a phone that has just passed OTP verification.
- **Through the API, only by acquiring OTPs.** That means controlling the phones, and is rate-limited.
- **Residual risk:** a fully compromised auth service can still issue sessions. That is inherent to any login component, and is mitigated by isolation, secrets handling, and monitoring.

**Rate limiting and monitoring** (the store is Redis or a dedicated non-identity table; chosen in Phase 0):
- **Limits:**
  - per `phone_hash`: 3 OTP requests / 15 min and 10 / day;
  - per IP: 10 / h;
  - per device or installation id: 10 / h;
  - global circuit breaker on OTP sends per minute;
  - 5 verification attempts per challenge.
- **Alerts:**
  - OTP request spikes by number prefix, country, or ASN (SMS-pumping fraud);
  - a verify/request ratio below the baseline;
  - repeated lockouts.
  These are audit events in the Security category.
- **Which numbers are accepted:** restricting sign-up to Egyptian mobile numbers (`+20 1…`) for the MVP would sharply cut SMS-pumping risk. That is a product decision: **D-39, open.**

**Other identity access** (unchanged from v0.2 unless noted):

| Table | auth | app_customer | app_staff | worker | admin |
|---|---|---|---|---|---|
| `user_account` | resolved user only; INSERT on signup | own row R/W | `SELECT (id, display_name, public_display_name)` where the user has a membership in `app.tenant_id()` | R for recipients of pending notifications | R (audited) |
| `user_phone` | verified phone only (above); no plaintext column | own rows R | — | R (incl. plaintext) where an unsent notification targets that user | R (audited per request) |
| `user_email` | resolved user only | own rows R/W | — | as `user_phone` | R (audited) |
| `otp_challenge` (`phone_hash` only, no plaintext) | rows of the transaction's phone hash | — | — | delete (retention) | — |
| `auth_session` | by token hash / resolved user | own rows R (device list), revoke | — | delete (retention) | revoke (audited) |
| `push_device` | — | own rows R/W | — | R for pending-notification recipients | — |
| `platform_role_grant` *(PLATFORM_PRIVATE)* | resolved user only | — | — | — | R/W (ADMIN only, audited) |

### 5.8 Real-time gateway

- Authorization happens at **subscribe time**, through the pool of the surface:
  - **staff channel** `branch:<id>:board` uses the staff pool and requires an active membership with that branch in scope;
  - **customer channel** `session:<id>` / `entry:<id>` uses the customer pool and requires the caller to own the session or entry (`queue_entry_party.customer_user_id`).
- **Re-checked** on token refresh (≤ 15 min). Membership removal revokes subscriptions within one refresh cycle, or immediately via an `authz.membership.changed` event.
- Events are re-read from the database **under the subscriber's context** before they are pushed. NOTIFY payloads carry only ids.
- **Customer payloads** contain only their own entries, the count of people ahead, and the ETA (FR-RT-3).

## 6. Unique constraints below RLS — leakage analysis

Unique indexes and FKs are checked regardless of RLS, so a constraint violation can reveal that a row exists elsewhere.

| Constraint | Could reveal | Assessment |
|---|---|---|
| P-1 one ACTIVE remote session per user (cross-tenant) | That *this user* has an active session somewhere | Only the user themselves can trigger it. The response links to their own session. ✔ acceptable |
| `identity.user_phone` current phone unique | Whether a phone is registered | Only reachable after OTP verification (§5.7). The OTP request step never reads `user_phone` and responds identically ✔ |
| `crm.customer (tenant_id, phone_e164)` | That the shop has a record for a phone | Only staff insert phones. Customer-role inserts carry no phone. ✔ |
| `visit_claim` PK / invitation single use | That a visit was already claimed | The claim API returns a generic `404 INVITATION_INVALID` for every failure ✔ |
| `business.slug` unique | That a slug is taken | Public anyway ✔ |
| `staff_commitment` exclusion | Only within the same provider (tenant-scoped) | ✔ |
| Composite FKs | Rejection of a cross-tenant reference | Mapped to a generic `404 NOT_FOUND` ✔ |

**Rule:** database errors are never returned verbatim. Constraint names map to domain error codes.

## 7. What RLS does **not** protect

1. **Wrong context set by the application.** If L2 resolves the wrong tenant, RLS enforces the wrong tenant. Mitigations:
   - membership verification in one audited code path;
   - tests;
   - optionally, a later hardening step: set the context through a reviewed function that re-checks membership in the DB.
2. **Authorization inside a tenant** (provider vs manager, branch scope, field masking): L3.
3. **Business-rule bypass**: domain services and state machines.
4. **Data outside PostgreSQL**:
   - Redis keys must be prefixed with the tenant id;
   - object storage keys use `tenants/<tenant_id>/…` with short-lived signed URLs;
   - logs and traces contain no phone numbers or names (hashed user ids only);
   - analytics exports are reviewed.
5. **Owner, migrator, and maintenance roles; backups; logical replication.** These are handled by operational access control: CI/CD and the maintenance task hold the only credentials, and access is audited.
6. **Third-party processors** (SMS/WhatsApp, push, error tracking): data minimization and a transfer licence where needed (D-23, [ADR-0005](../adr/0005-hosting-and-data-residency.md)).
7. **Shared or recycled phones** in visit claims (D-30): an identity-assurance limit, not an isolation bug. The blast radius is one visit.

## 8. Other controls

| Area | Control |
|---|---|
| Rate limiting | Per phone and per IP for OTP (e.g., 3/15 min/phone, 10/h/IP); per user for queue join/cancel loops and claim attempts (e.g., 10/h); per IP for discovery; per tenant for exports. A shared store (Redis) once there is more than one instance. |
| Input validation | Zod schemas at the API edge; DB CHECKs as the last line; size limits on text and uploads |
| Uploads | Signed upload URLs; content-type and size checks; image re-encode with EXIF strip (removes GPS); moderation status before public display |
| Secrets | Environment-specific secret store; no secrets in the repo (`.gitignore` covers `.env*`, keys); separate credentials per DB role; rotation runbook |
| Sessions | Refresh token rotation; revoke all sessions on phone change or suspicion; device list |
| Visit claims | 128-bit random single-use tokens (CSPRNG). Stored as SHA-256 hashes; plaintext is never stored or logged. Expiry = the visit's review window (14 days, D-10). Atomic redemption by conditional update. The token must match **and** the caller's current OTP-verified phone hash must equal the destination hash. Generic `404` on any failure. Rate limits: 10 claim attempts per user per hour and 20 per IP per hour; repeated failures raise an alert. The link carries the token in the URL **fragment or app deep-link path**, and the token is sent in the request body. Access logs and analytics strip it, and `Referrer-Policy: no-referrer` applies. Every claim and revocation is audited. Shared/recycled-phone risk stays open as **D-30**. |
| Privacy (PDPL) | Consent records; data-subject request workflow (export/erasure) with audit; breach runbook covering the **reported** 72 h regulator notification; DPO and licence obligations per counsel (D-17). **No compliance is claimed** until counsel confirms. |
| Push | Payloads contain no personal data (D-23) |

## 9. RLS evaluation — verdict

| Aspect | Assessment |
|---|---|
| Benefit | Turns most application authorization bugs that would otherwise be **cross-tenant leaks** or **customer-to-business overreach** into empty results. Protects new endpoints and jobs by default. |
| Cost | Policy on every protected table; context helper; per-surface pools; the `queue_entry_party` split; CI checks; slight planning overhead (to confirm in the load test). |
| Pitfalls | Session-level SET with pooling; definer views/functions; forgetting FORCE or `TO`; owners bypassing; error leakage; non-DB stores. **All have explicit mitigations above.** |
| Alternatives | App-only filtering (one missed `WHERE` = a leak); schema-per-tenant (operational cost, see ADR-0002). |
| **Recommendation** | **Adopt RLS as defense in depth (L4), never as the primary authorization mechanism.** ADR-0002 remains *Proposed* until technical sign-off. |

## 10. Security tests

- **Catalog checks (CI):**
  - Every table appears in the classification registry; unclassified tables fail.
  - Every TENANT_OWNED, TENANT_ROOT, PLATFORM_PERSONAL, and PLATFORM_PRIVATE table has RLS **enabled and forced**.
  - Every policy has an explicit `TO` list that matches the access matrix (§3.1). No policy targets `PUBLIC`.
  - No login role has `BYPASSRLS`. No runtime role is a member of `turnio_owner`.
  - No `SECURITY DEFINER` function and no non-`security_invoker` view exists outside the (empty) allowlist.
  - PUBLIC_PROJECTION tables match their column allowlists.
- **Isolation suite** (staff and worker roles): for each TENANT_OWNED table, seed tenants A and B and run as context A:
  - SELECT returns 0 rows of B;
  - INSERT with B's tenant_id fails;
  - UPDATE/DELETE affect 0 rows of B;
  - inserting a child that references B's parent fails (composite FK).
- **Customer-overreach suite** (`turnio_app_customer`, `customer_command` targeting tenant A):
  - `crm.customer` returns only the caller's own row (or the unlinked own-phone row);
  - `queue_entry_party` returns only own rows;
  - appointments, visits, and reviews of other customers → 0 rows;
  - ETA and availability computation still succeed;
  - any attempt to read `crm.customer.phone_e164` of others fails.
- **Claim suite:**
  - wrong token, wrong phone, expired, revoked, or already claimed → identical `404`;
  - a successful claim exposes exactly one visit;
  - linking a customer record exposes no pre-link visits.
- **No-context test:** every repository query run without context returns 0 rows or fails.
- **Pool leakage test:** interleave 1,000 transactions for alternating tenants and users on a pool of 2 connections per role and assert there is never a cross-read. Run it with PgBouncer in transaction mode as well.
- **Admin:**
  - without a grant → 0 tenant rows;
  - expired or revoked grant → 0 rows;
  - a READ grant cannot write;
  - a grant id set in one transaction is absent in the next;
  - every granted request → an audit row.
- **Maintenance:** the API/worker credentials cannot run partition DDL. Maintenance actions produce audit rows.
- **Real-time:** subscribing to another tenant's branch channel or another user's session is rejected. A removed membership loses its subscription within the SLA.
- **API matrix:** every endpoint × role × (own tenant / other tenant / no membership) → expected status. Cross-tenant ids → `404`.
- **Login least privilege (v0.3)**, as `turnio_auth`:
  - with no `login_phone_hash` → `user_phone` returns 0 rows;
  - with a hash but no verified challenge → 0 rows;
  - with a verified challenge → only that phone's rows;
  - after consumption or 5 minutes → 0 rows;
  - `SELECT phone_e164` → permission denied;
  - another user's `user_account`/`auth_session` → 0 rows;
  - any tenant, CRM, queue, or notification table → permission denied.
  API level: identical responses and timing bands for registered and unregistered phones; rate limits and lockout enforced.
- **Queue personal data (v0.3):**
  - the customer role sees only own `queue_entry_party` rows;
  - it can update only own entries, and only the granted columns;
  - it can update only `seq` and `next_ticket_number` on `queue.queue`;
  - it cannot insert a party row with another user's id;
  - the organizer sees all session companions, and nobody else does;
  - the public role gets permission denied on all `queue.*`;
  - the customer role gets permission denied when selecting `queue_entry.arrival_confirmed_by_id`, `staff_commitment.appointment_id`, `crm.customer.display_name`, or `crm.customer.no_show_count`.
- **Policy/column interplay (v0.3.1):** confirm on the target PostgreSQL version that the `crm.customer` own-phone policy (which references `phone_hash`) works for a role without SELECT on `phone_hash`, and that the `queue_entry` ownership policy (which references `queue_entry_party`) works with the restricted column grants. If not, the fallback (a separate linking step run by the worker) must be designed before Phase 1.
- **OTP concurrency (v0.3.1):**
  - 20 parallel wrong guesses on one challenge → at most 5 attempts recorded, then locked;
  - two parallel correct submissions → exactly one `VERIFIED`;
  - two parallel OTP requests for one phone → at most one `ISSUED`.
- **Key rotation (v0.3.1):** during the overlap, logins and claims succeed for rows on both key versions; after cutover, old-version lookups fail closed.
- **Claims (v0.3):** with a valid token for visit A, a user cannot see the invitation for visit B sent to the same phone.
- **Idempotency (v0.3):** user B presenting user A's key gets no replay of A's response (RLS on `user_id`); the same key with a different tenant or branch → `422`.

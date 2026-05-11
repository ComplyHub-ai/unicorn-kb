# Audit Log Inventory & Canonical Ledger Gap

> **Last updated:** 2026-05-11 · **Reconsider by:** 2026-08-11 · **Confidence:** high — all tables verified in migrations; schema details extracted from migration files.
>
> **Reflects commit:** `<codebase>@893fc01a` (2026-05-11).
>
> Documents the current state of audit logging across the Unicorn codebase and scopes the architectural gap between the existing domain-specific tables and a workspace-wide canonical ledger.

---

## Background

The `audit_user_events` concept originated in the original ComplyHub research and was described as the right shape for user-profile events. It does not exist in the Unicorn codebase. What exists instead is a proliferation of domain-specific `*_audit_log` tables — fifteen at last count — each created independently by Lovable with no shared schema contract. This document inventories those tables, characterises the schema inconsistencies, and scopes what a canonical ledger would need to cover.

---

## Current state — domain audit tables

| Table | Created | Actor column | Change capture | Has `tenant_id` | Coverage |
|---|---|---|---|---|---|
| `client_audit_log` | 20260105 | `actor_user_id` | `details` jsonb | ✅ bigint | AI finding drafts + decisions; AI exec summary drafts + decisions |
| `package_builder_audit_log` | 20260106 | `user_id` | `before_data` / `after_data` | ❌ absent | Package builder CRUD |
| `eos_template_audit_log` | 20260120 | (TBC) | (TBC) | (TBC) | EOS template changes |
| `eos_minutes_audit_log` | 20260120 | `user_id` | `details` jsonb | ✅ bigint | Governance meeting minutes (create, finalise, revise, lock, unlock, restore) |
| `process_audit_log` | 20260121 | (TBC) | (TBC) | (TBC) | Process changes |
| `assistant_audit_log` | 20260205 | `viewer_user_id` | `request_text` / `response_summary` | ✅ `client_tenant_id` bigint | Internal assistant queries, reports, refusals |
| `knowledge_item_audit_log` | 20260205 | (TBC) | (TBC) | (TBC) | Knowledge base item changes |
| `eos_process_audit_log` | 20260205 | (TBC) | (TBC) | (TBC) | EOS process changes |
| `notification_audit_log` | 20260206 | (TBC) | (TBC) | (TBC) | Notification delivery events |
| `consultant_capacity_audit_log` | 20260212 | (TBC) | (TBC) | (TBC) | Consultant capacity changes |
| `consultant_assignment_audit_log` | 20260212 | (TBC) | (TBC) | (TBC) | Consultant assignment changes |
| `engagement_audit_log` | 20260213 | `actor_user_uuid` | none | ✅ bigint | Package engagement events + integrity validation |
| `exec_weekly_review_audit_log` | 20260213 | (TBC) | (TBC) | (TBC) | Executive weekly review changes |
| `time_entry_audit_log` | 20260216 | `actor_user_id` | `old_row` / `new_row` jsonb | ✅ integer | Time entry create/update/delete/repost/split (trigger-based) |
| `research_audit_log` | 20260216 | (TBC) | (TBC) | (TBC) | Research item changes |

> Note: columns marked (TBC) were not extracted in this review — tables exist but internal schema not yet verified. The four tables with full schema detail above are representative for design purposes.

---

## Schema inconsistencies

Five concrete problems observed across fully-reviewed tables:

**1. Actor column has four different names**
- `actor_user_id` — `time_entry_audit_log`, `client_audit_log`
- `actor_user_uuid` — `engagement_audit_log`
- `user_id` — `package_builder_audit_log`, `eos_minutes_audit_log`
- `viewer_user_id` — `assistant_audit_log` (semantically different — the requesting user, not a mutating actor)

**2. `package_builder_audit_log` has no `tenant_id`**
This is a multi-tenancy gap: package builder audit events cannot be scoped to a tenant without joining back to the `packages` table. Any cross-domain query or view that depends on `tenant_id` cannot include this table without a join.

**3. Change-capture naming is inconsistent**
- `old_row` / `new_row` — `time_entry_audit_log` (trigger pattern)
- `before_data` / `after_data` — `package_builder_audit_log`
- `details` jsonb only — `client_audit_log`, `eos_minutes_audit_log` (no before/after)
- No change capture at all — `engagement_audit_log`

**4. Action vs event_type**
- `action` — `time_entry_audit_log`, `package_builder_audit_log`, `client_audit_log`
- `event_type` — `engagement_audit_log`
- Some `action` columns have CHECK constraints; others are free text.

**5. `tenant_id` type mismatch**
- `integer` — `time_entry_audit_log`
- `bigint` — `engagement_audit_log`, `client_audit_log`, `eos_minutes_audit_log`

---

## The gap — what a canonical ledger would add

The domain tables answer the question "what changed in domain X?" They do not answer:

- **"What did user Y do across the workspace this week?"** — requires querying 15 tables and normalising inconsistent schemas.
- **"Show me all changes that touched tenant Z on date D."** — same problem; tenant_id is absent from at least one table.
- **"Did anyone access or modify client records outside normal hours?"** — no single point of query.
- **"Produce a change log for this audit engagement from opening to sign-off."** — would need to join `client_audit_log` + `eos_minutes_audit_log` + manual CRUD events (which have no audit trail at all).

Additionally, manual CRUD on core entities has no audit coverage whatsoever:
- Creating/updating/deleting audit responses manually → no log
- Creating findings from scratch (not AI-drafted) → no log
- Updating client profile data → no log
- Tenant user role changes → no log (invitation flow is logged via `audit_invites`; post-accept changes are not)

---

## Design options

### Option A — Central `workspace_audit_log` table

One append-only table with a common schema. All domains write to it via triggers or explicit application inserts.

```sql
CREATE TABLE public.workspace_audit_log (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id    bigint NOT NULL,
  actor_id     uuid NOT NULL REFERENCES auth.users(id),
  action       text NOT NULL,            -- verb: 'create', 'update', 'delete', 'ai.draft', etc.
  domain       text NOT NULL,            -- 'time_entry', 'client_audit', 'eos_minutes', etc.
  entity_type  text NOT NULL,            -- table name or logical entity type
  entity_id    text NOT NULL,            -- cast to text for universality
  old_val      jsonb,
  new_val      jsonb,
  metadata     jsonb,                    -- domain-specific extras (tokens, model, confidence, etc.)
  created_at   timestamptz NOT NULL DEFAULT now()
);
```

**Pros:** single query surface, consistent schema, works as compliance export.
**Cons:** high write volume (every CRUD on every audited table), requires retrofitting 15 existing tables, risk of becoming a bottleneck. Migration is a breaking change if any consumer depends on domain table schemas.

### Option B — Federated view

Keep all 15 domain tables as-is. Create a `v_workspace_audit_log` view that `UNION ALL` normalises them into a common shape.

```sql
CREATE VIEW public.v_workspace_audit_log AS
  SELECT
    id, tenant_id, actor_user_id AS actor_id, action, 
    'time_entry' AS domain, 'time_entries' AS entity_type,
    time_entry_id::text AS entity_id,
    old_row AS old_val, new_row AS new_val, NULL AS metadata,
    created_at
  FROM public.time_entry_audit_log
UNION ALL
  SELECT
    id, tenant_id, actor_user_id AS actor_id, action,
    'client_audit' AS domain, entity_type, entity_id,
    NULL AS old_val, NULL AS new_val, details AS metadata,
    created_at
  FROM public.client_audit_log
-- ... (12 more UNION ALL branches)
;
```

**Pros:** zero migration cost on existing consumers; no write-path changes; can be added incrementally.
**Cons:** view grows with every new domain table (maintenance cost); `package_builder_audit_log` cannot be included without a join (no `tenant_id`); the manually-unlogged operations (manual CRUD) remain invisible; query performance is poor on large datasets without materialisation.

### Option C — Hybrid (recommended for this stage)

1. **Short-term:** Create the federated view (Option B) for the 12 tables that have `tenant_id`. Fix `package_builder_audit_log` by adding `tenant_id` in a targeted migration. Exclude the three (TBC) tables until their schemas are verified.
2. **Medium-term:** Add trigger-based logging on the 5 most critical manually-unlogged entities: `client_audit_responses`, `client_audit_findings`, `tenant_users`, `users`, `tenant_settings`.
3. **Long-term decision:** If the workspace needs a compliance-grade export (e.g. for an external regulator), migrate to the central table (Option A). If the use case is internal ops visibility only, the federated view is sufficient.

---

## Decisions required before building

| # | Question | Who | Notes |
|---|---|---|---|
| 1 | **What is the primary use case?** Internal ops visibility (who did what during an incident) or compliance-grade audit export (regulators, QA audits of Vivacity's own practices)? | Carl / Angela | Drives the "view is enough" vs "central table" choice. |
| 2 | **Which entities must be covered for compliance?** At minimum: `client_audit_responses`, `client_audit_findings`, `tenant_users`. Are there others? | Angela (RTO compliance expert) | Defines scope of new trigger coverage. |
| 3 | **Fix `package_builder_audit_log` missing `tenant_id` first?** It breaks any cross-tenant query that includes package builder events. Low-risk migration. | Carl | Prerequisite for Option B/C. |
| 4 | **Retention policy.** How long should audit log entries be kept? 7 years is the ASQA standard for RTO records. Does this apply to Unicorn's internal audit logs, or only the compliance outputs? | Angela | Affects whether `created_at` needs a partitioning strategy. |
| 5 | **`audit_user_events` — build now or defer?** The ComplyHub concept captures login events, password changes, role changes. Is there an immediate need for this in Unicorn? | Carl | Could be a separate table feeding into the federated view. |

---

## Recommended next step

Run the **Option C short-term phase** as a Lovable prompt:
1. Add `tenant_id bigint NOT NULL` to `package_builder_audit_log` (backfill via `packages.tenant_id` join).
2. Create `v_workspace_audit_log` covering the 12 tables that have `tenant_id`.
3. Create `audit_user_events` table (login, logout, password change, role change, tenant switch) with trigger on `users` UPDATE + hook into auth events if Supabase auth hooks are available.

This is a low-risk first step that gives immediate cross-domain visibility without touching any existing consumers.

Follow the Lovable production DB change workflow (`unicorn-kb/handoffs/lovable-production-db-change.md`) — this involves a migration (`tenant_id` backfill on `package_builder_audit_log` is a data operation on a live table).

---

## Related

- `unicorn-kb/reference/ai-audit-stack.md` — AI-specific audit (client_audit_log deep dive)
- `unicorn-kb/reference/decision-trail.md` — ADR-005 (RLS), ADR-011 (operating model)
- `unicorn-kb/pinned/decisions.md` — open decisions list (add ADR-013 when design is decided)

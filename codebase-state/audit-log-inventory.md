# Audit Log Inventory & Canonical Ledger Gap

> **Last updated:** 2026-05-13 · **Reconsider by:** 2026-08-13 · **Confidence:** high — all tables verified via live DB query against `yxkgdalkbrriasiyyrwk` on 13 May 2026.
>
> **Reflects commit:** `<codebase>@a171ca97` (2026-05-13). Live DB queried directly; prior versions were based on migration file inspection and undercounted by ~25 tables.

---

## Background

Unicorn has approximately **42 tables** with "audit", "log", or "events" in their name. They fall into three distinct categories — only one of which is relevant to the canonical ledger gap.

**Category 1 — RTO compliance audit module (not event logs)**
These tables store the actual RTO audit work delivered to clients: templates, sections, questions, responses, findings, actions. They are domain data, not system event history. Examples: `audit`, `audit_section`, `audit_question`, `audit_response`, `audit_finding`, `audit_action`, `audit_templates`, `reusable_audit_templates`, `audit_intelligence_packs`. Not in scope for this document.

**Category 2 — Domain-specific event log tables**
Operational logs created by Lovable across different modules as features were built. No shared schema contract.

**Category 3 — Security and behaviour event tables**
Tables explicitly tracking who did what at the security and access layer: impersonation sessions, restricted-action attempts, invitation outcomes, access denials.

The gap: none of the tables across Categories 2 and 3 are queryable together. There is no single surface for "what happened to Tenant X this week" or "who changed this record."

---

## Complete event log inventory — live DB state (13 May 2026)

### Active event log tables

| Table | Rows (approx) | `tenant_id` | Type | Actor column | Notes |
|-------|----------|-------------|------|--------------|-------|
| `audit_eos_events` | 2,791 | ✅ | bigint NOT NULL | `user_id` | Most active event log; EOS + invitation flow writes here |
| `audit_dashboard_events` | 2,739 | ✅ | bigint | `actor_user_id` | Staff dashboard interactions |
| `time_entry_audit_log` | 1,286 | ✅ | **integer** ⚠️ | `actor_user_id` | Trigger-based; old/new row capture; `action` CHECK constraint |
| `audit_events` | 1,202 | ❌ | — | `user_id` | **Instrumentation only** — errors, message reads, SharePoint settings noise; not a compliance log |
| `client_audit_log` | 1,125 | ✅ | bigint | `actor_user_id` | AI finding + exec summary draft decisions |
| `audit_client_impersonation` | active | ✅ | **integer** ⚠️ | `actor_user_id` | Staff-impersonates-client sessions; has `started_at`/`ended_at` |
| `process_audit_log` | active | ✅ | bigint **NULLABLE** ⚠️ | `actor_user_id` | Process changes; `before_data`/`after_data` jsonb; `tenant_id` nullable |
| `tga_import_audit` | active | ✅ | bigint | `triggered_by` | TGA import runs per tenant |
| `audit_invites` | active | ✅ | bigint | `actor_user_id` (text) | Invitation send/accept outcomes; `actor_user_id` is `text` not `uuid` ⚠️ |
| `sharepoint_access_log` | active | ✅ | bigint | `user_id` | SharePoint file access events |
| `document_activity_log` | active | ✅ | bigint | `actor_user_id` | Document view/download/upload with `actor_role` |
| `portal_document_audit` | active | ✅ | bigint | `actor_user_id` | Portal document actions with `actor_role` |
| `meeting_sync_audit` | active | ✅ | bigint | `user_id` | Outlook calendar sync results |
| `engagement_audit_log` | active | ✅ | bigint | `actor_user_uuid` | Package engagement; uses `event_type` not `action` ⚠️ |
| `ai_events` | active | ✅ | bigint | `actor_user_id` | AI feature performance metrics (latency, confidence, model) |
| `eos_minutes_audit_log` | active | ✅ | bigint | `user_id` | Governance meeting minutes lifecycle |
| `eos_template_audit_log` | active | ✅ | bigint | `user_id` | EOS template changes |
| `consultant_assignment_audit_log` | active | ✅ | bigint | `created_by` (nullable) | Consultant assignment decisions with capacity snapshot |
| `assistant_audit_log` | active | ✅ | bigint **as `client_tenant_id`** ⚠️ | `viewer_user_id` | AI assistant queries, refusals; column name non-standard |
| `audit_restricted_actions` | active | ✅ | bigint | `user_id` | Attempted actions the user lacked permission for |
| `consultant_capacity_audit_log` | active | ✅ | bigint **NULLABLE** ⚠️ | `created_by` (nullable) | Capacity calculation snapshots |
| `audit_user_events` | **0 rows** | ✅ | bigint NULLABLE | `actor_user_uuid` | New table (May 2026); write path not yet wired |
| `package_builder_audit_log` | active | ❌ **MISSING** | — | `user_id` | Package builder CRUD; `before_data`/`after_data` jsonb |

### Inactive / excluded tables

| Table | Rows | Reason for exclusion |
|-------|------|----------------------|
| `audit_log` | **0** | Unicorn 1.0 legacy (field-level diffs: `field_name`, `old_value`, `new_value`, `editor_name`). Nothing writes to it in Unicorn 2.0. |
| `audit_events` | 1,202 | Application instrumentation noise (error boundaries, profile prompts, message reads). No `tenant_id`. Not a compliance-grade source. |
| `sch_audit_log` | low | Scheduling module; uses `org_id uuid` not `tenant_id` — incompatible join key |
| `knowledge_item_audit_log` | low | No `tenant_id` |
| `eos_process_audit_log` | low | No `tenant_id` |
| `exec_weekly_review_audit_log` | low | No `tenant_id` |
| `notification_audit_log` | low | No `tenant_id` |
| `research_audit_log` | low | No `tenant_id` |
| `document_link_audit` | low | No `tenant_id` |
| `document_ai_audit` | low | No `tenant_id` |
| `meeting_capture_audit` | low | No `tenant_id` |
| `stage_state_audit_log` | low | No `tenant_id` |
| `package_instance_state_log` | low | No `tenant_id` |
| `strategic_decision_log` | low | No `tenant_id` |

---

## Schema inconsistencies (updated — live DB verified)

**Actor column has six different names**
`actor_user_id`, `actor_user_uuid`, `user_id`, `viewer_user_id`, `created_by`, `triggered_by`

**`tenant_id` type inconsistencies**
- `integer`: `time_entry_audit_log`, `audit_client_impersonation` — need `CAST(tenant_id AS bigint)` in the view
- `bigint NULLABLE`: `process_audit_log`, `consultant_capacity_audit_log` — rows with NULL tenant_id exist
- `client_tenant_id` (non-standard column name): `assistant_audit_log`

**Action vs event_type**
`engagement_audit_log` uses `event_type` rather than `action`

**`audit_invites.actor_user_id` is `text` not `uuid`**
Edge function writes the actor UUID as a text string. Cannot directly JOIN to `auth.users(id)` without `::uuid` cast.

**`package_builder_audit_log` has no `tenant_id`**
Cannot be included in the federated view until Phase 1 fix is applied.

---

## The gap — what a canonical ledger would add

Even with all 19 includable tables above, the following remain unlogged:

- Manual edits to `client_audit_responses` and `client_audit_findings` — no log anywhere
- `tenant_users` role and access scope changes after invitation acceptance — not logged
- `users` profile changes — `audit_user_events` table exists but write path not wired
- `tenant_settings` configuration changes — not logged

---

## Design decisions — confirmed 13 May 2026

| # | Question | Decision |
|---|---|---|
| 1 | **Primary use case?** | **Compliance-grade export** — Option A (central table) is the long-term destination; Option C (federated view) is the first step |
| 2 | **Trigger entities for medium-term?** | `tenant_users`, `users`, `client_audit_responses` + `client_audit_findings`, `tenant_settings` — all four confirmed |
| 3 | **Fix `package_builder_audit_log` now?** | **Yes** — Phase 1 of current session |
| 4 | **Retention policy** | Open — Angela to confirm. 7-year ASQA standard likely applies |
| 5 | **`audit_user_events` — build now?** | Already exists with `tenant_id`. INSERT policy + trigger writes in Phase 3 |

---

## Implementation plan — Option C (hybrid)

### Phase 1 — Fix `package_builder_audit_log` (in progress, 13 May session)

Add `tenant_id bigint` via migration with backfill from `packages.tenant_id`. Make NOT NULL after backfill. Prerequisite for including this table in the view.

### Phase 2 — Create `v_workspace_audit_log` (in progress, 13 May session)

`SECURITY INVOKER` view `UNION ALL`-ing the following 19 tables:

**Straight includes** (bigint tenant_id, standard actor column):
`audit_eos_events`, `audit_dashboard_events`, `client_audit_log`, `tga_import_audit`, `sharepoint_access_log`, `document_activity_log`, `portal_document_audit`, `meeting_sync_audit`, `ai_events`, `eos_minutes_audit_log`, `eos_template_audit_log`, `consultant_assignment_audit_log`, `audit_restricted_actions`, `audit_user_events`, `engagement_audit_log`

**Requires special handling:**
- `time_entry_audit_log` — `CAST(tenant_id AS bigint)`
- `audit_client_impersonation` — `CAST(tenant_id AS bigint)`
- `assistant_audit_log` — `client_tenant_id` aliased as `tenant_id`
- `process_audit_log` — `tenant_id` nullable; include with NULL (platform-level events)
- `consultant_capacity_audit_log` — `tenant_id` nullable; same treatment
- `audit_invites` — `actor_user_id::uuid` cast needed
- `package_builder_audit_log` — include only after Phase 1 is applied

**Excluded from view** (no tenant_id or incompatible schema):
See "Inactive / excluded tables" section above.

Normalised output shape:
```
id uuid, tenant_id bigint, actor_id uuid, action text,
domain text, entity_type text, entity_id text,
old_val jsonb, new_val jsonb, metadata jsonb, created_at timestamptz
```

### Phase 3 — Trigger-based logging (separate session)

Add `AFTER INSERT OR UPDATE OR DELETE` triggers writing to `audit_user_events` on:
- `tenant_users` — role, access_scope, relationship_role changes
- `users` — profile field changes, global_role changes
- `client_audit_responses` — manual CRUD (currently silent)
- `client_audit_findings` — manual CRUD (currently silent)
- `tenant_settings` — workspace configuration changes

Add INSERT policy to `audit_user_events` for the trigger function (SECURITY DEFINER).

### Long-term — Central `workspace_audit_log` table (Option A)

When row volumes or compliance export requirements warrant it, migrate to a single append-only table. The federated view (Phase 2) provides the same query interface and makes the migration transparent to consumers.

---

## Current phase status

| Phase | Status |
|-------|--------|
| Phase 1 — `package_builder_audit_log.tenant_id` | **In progress — 13 May session** |
| Phase 2 — `v_workspace_audit_log` | **In progress — 13 May session** |
| Phase 3 — Trigger logging (4 entities) | Queued — separate session |
| Long-term — Option A central table | Deferred |

---

## Related

- `unicorn-kb/reference/ai-audit-stack.md` — AI-specific audit (`client_audit_log` deep dive)
- `unicorn-kb/reference/decision-trail.md` — ADR-005 (RLS), ADR-011 (operating model)
- `unicorn-kb/handoffs/lovable-production-db-change.md` — workflow for Phases 1–3

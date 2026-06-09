# RBAC v5 Implementation Plan

> **Created:** 09 June 2026 | **Project:** Unicorn 2.0
> **Source documents:** `rbac-v5.md`, `impl-academy-rbac-and-permission-editor.md`
> **Total Lovable prompts:** 24 | **Direct SQL steps:** 1
> **Status:** In planning — Phase 0 ready to run

---

## Design decisions confirmed

| # | Decision | Answer |
|---|---|---|
| 1 | Role name case convention | Title case — match existing (`'Super Admin'`, `'Team Leader'`). New values: `'Integrator'`, `'BGT'`, `'CHC'`, `'CET'` |
| 2 | Multiple roles per user | Not supported — single `unicorn_role` column. Dave → Team Leader (superset of BGT permissions). Multi-role is a future architectural change. |
| 3 | Sam Holtham role | CHC |
| 4 | Beverly Pastor-Ambo role | BGT |
| 5 | CET members | Nobody assigned for now — implement role definition, leave unassigned |
| 6 | `is_vivacity_internal` | Already `true` for all 14 staff. Column is a boolean double-check in RLS helpers. Not a blocker. |

---

## Staff role assignments (apply in Phase 4)

| Name | Email | Current | New Role |
|---|---|---|---|
| Angela Connell-Richards | angela@vivacity.com.au | Super Admin | **Team Leader** |
| Dave Richards | dave@vivacity.com.au | Super Admin | **Team Leader** |
| Nova Canto | nova@vivacity.com.au | Super Admin | **Integrator** |
| Sharwari Rajurkar | Sharwari@vivacity.com.au | Super Admin | **CHC** |
| Kelly Xu | kelly@vivacity.com.au | Super Admin | **CHC** |
| Tanya Janklin | tanya@vivacity.com.au | Super Admin | **CHC** |
| AJ Delostrico | AJ@vivacity.com.au | Super Admin | **CHC** |
| Ezel Olores | ezel@vivacity.com.au | Super Admin | **CHC** |
| Samantha Holtham | sam@vivacity.com.au | Super Admin | **CHC** |
| Beverly Pastor-Ambo | beverly@vivacity.com.au | Super Admin | **BGT** |
| Carl Simpao | carl@vivacity.com.au | Super Admin | **Super Admin** (keep) |
| Khian Sismundo | brian@vivacity.com.au | Super Admin | **Super Admin** (keep) |
| RJ Badua | Rhald@vivacity.com.au | Super Admin | **Super Admin** (keep) |
| CET | — | — | Nobody assigned yet |

---

## Phase tracking

| Phase | Name | Prompts | Status | Depends on |
|---|---|---|---|---|
| 0 | Security patches | 1 | ⬜ Todo | Nothing |
| 1 | DB foundation | 5 | ⬜ Todo | Design decisions |
| 2 | Edge function updates | 3 | ⬜ Todo | Phase 1 deployed |
| 3 | `useRBAC.tsx` update | 1 | ⬜ Todo | Phase 1 deployed |
| 4 | Staff reassignments | SQL only | ⬜ Todo | Phases 2 + 3 deployed |
| 5 | Academy gates + bug fixes | 7 | ⬜ Todo | Phase 3 deployed |
| 6 | Permission EF | 1 | ⬜ Todo | Phase 1 deployed |
| 7 | Permission editor UI | 1 | ⬜ Todo | Phase 6 deployed |
| 8 | Wire `usePermission` | 5 | ⬜ Todo | Phase 7 + seed verified |

---

## Phase 0 — Security patches

**Run immediately. No dependencies.**

Two live gaps where Client Admins (`unicorn_role = 'Admin'`, `user_type = 'Client Parent'`) can call internal-only functions because `user_type` is not checked.

### Prompt 0.1 — Fix edge function auth gaps

> **Files:** `supabase/functions/generate-release-documents/index.ts`, `supabase/functions/export-compliance-pack/index.ts`
>
> Both functions check `unicorn_role in ['Super Admin', 'Admin']` but do not check `user_type`. A Client Admin can currently call these. Add a `user_type` guard to both: the call is only permitted if `user_type in ('Vivacity', 'Vivacity Team')` — model the fix on the pattern in `send-password-reset` which correctly checks both fields. No other changes.

---

## Phase 1 — DB foundation (full production DB protocol)

**Triggers `unicorn-kb/handoffs/lovable-production-db-change.md`. Run all 5 sub-prompts in sequence.**

### Sub-prompt 1.1 — Audit (plan mode ON, read-only)

> Conduct a read-only audit of:
> - The `unicorn_role` type — current values, every table column and function that references it
> - The RLS helper functions `is_vivacity_team_safe`, `is_super_admin_safe`, `is_any_team_member` — exact current DDL
> - `public.users` columns `is_vivacity_internal`, `unicorn_role`, `user_type`
> - All FK constraints on the `users` table
>
> Report findings. No code. Finish with a proposed implementation plan and any remaining design decisions.

**Design decisions gate — review audit output before 1.2.**

### Sub-prompt 1.2 — Implementation plan (plan mode ON)

> Using the confirmed decisions, produce the complete migration plan for (1) enum additions, (2) new permission tables, (3) RLS helper function updates. Include per-migration lock impact, rollback plan, and verification queries for each step.

### Sub-prompt 1.3 — Migration A: Enum additions (plan mode ON)

> Add the following values to `public.unicorn_role` using `ALTER TYPE ... ADD VALUE IF NOT EXISTS`:
> - `'Integrator'`, `'BGT'`, `'CHC'`, `'CET'`
>
> Do not rename or remove existing values (`'Super Admin'`, `'Team Leader'`, `'Team Member'`).
>
> Verification: `SELECT enum_range(NULL::public.unicorn_role)` — confirm all values present including new four.

### Sub-prompt 1.4 — Migration B: Permission tables (plan mode ON)

> Create three tables exactly as specified in `impl-academy-rbac-and-permission-editor.md` PR 1, with this correction:
> - `role_permissions.updated_by` FK must reference `public.users(user_uuid)` — the spec has a typo (`public.users(id)`)
>
> Apply all RLS policies from the spec. Seed all 40 rows of `permission_features`. Seed `role_permissions` from the RBAC v5 permission matrix (40 features × 6 roles = 240 rows) — SA always `full`, all others per matrix §4.
>
> Use `'Super Admin'`, `'Team Leader'`, `'Integrator'`, `'BGT'`, `'CHC'`, `'CET'` as the role string values (title case, matching the enum).
>
> Verification: `SELECT COUNT(*) FROM permission_features` (expect 40), `SELECT COUNT(*) FROM role_permissions` (expect 240).

### Sub-prompt 1.5 — Migration C: RLS helper functions (plan mode ON)

> Create or replace the following DB functions. Every function must use `SET search_path = ''` with fully qualified object references (per `pinned/conventions.md → Function hardening`). `REVOKE ALL ON FUNCTION ... FROM PUBLIC`, grant to `authenticated` only.
>
> - Update `is_vivacity_team_safe` — check `unicorn_role IN ('Super Admin', 'Team Leader', 'Integrator', 'BGT', 'CHC', 'CET') AND is_vivacity_internal = true`
> - Update `is_super_admin_safe` — check `unicorn_role = 'Super Admin' AND is_vivacity_internal = true`
> - Add `is_team_leader_or_above` — `unicorn_role IN ('Super Admin', 'Team Leader') AND is_vivacity_internal = true`
> - Add `is_integrator_or_above` — `unicorn_role IN ('Super Admin', 'Team Leader', 'Integrator') AND is_vivacity_internal = true`
> - Update `is_any_team_member` — same list as `is_vivacity_team_safe`
>
> Verify each function returns correct results for a known SA user and a known CHC user.

---

## Phase 2 — Edge function updates

**Must deploy before Phase 4 staff reassignments. No DB protocol needed.**

### Prompt 2.1 — Update shared helpers

> **Files:** `supabase/functions/_shared/auth-helpers.ts`, `supabase/functions/_shared/ask-viv-access.ts`
>
> In `auth-helpers.ts`: update `VIVACITY_ROLES` to include all six: `['Super Admin', 'Team Leader', 'Team Member', 'Integrator', 'BGT', 'CHC', 'CET']`. Keep `'Team Member'` during the transition period. Update `SUPER_ADMIN_ROLE` constant. Export a shared `VIVACITY_STAFF_ROLES` constant.
>
> In `ask-viv-access.ts`: update `VIVACITY_INTERNAL_ROLES` to the same list.
>
> No other files changed.

### Prompt 2.2 — Narrow administration functions to SA only

> Update the following edge functions. For the "Vivacity staff" path only — do not change client-side gates:
> - `update-user-role` — already SA only, update role string constant
> - `bulk-user-action` — already SA only, update role string, remove `global_role` fallback
> - `bulk-send-invitations` — same
> - `generate-recovery-link` — same
> - `delete-user`, `toggle-user-status` — Vivacity path only, tighten to SA
> - `bulk-account-actions`, `activate-ghost-user`, `cohort-access-sender-worker` — currently allow all Vivacity team via `is_vivacity_team_safe`; narrow to SA + Team Leader only (per RBAC v5 §3 — `admin.*` features are SA-only; `admin.team_users.manage` is TL + SA)

### Prompt 2.3 — Open TL access where RBAC v5 grants it

> - `invite-user` Vivacity path: change `isVivacityStaff` check (all team) to TL + SA only — `unicorn_role IN ('Super Admin', 'Team Leader')`
> - `send-password-reset`: expand Vivacity check from SA-only to TL + SA
> - `update-user-profile` admin path: expand to TL + SA for Vivacity staff profiles
> - `resend-invite` / `cancel-invite`: update Super Admin check to include `'Team Leader'` (RBAC v5 §4 — "Add team members to Unicorn: TL + SA")

---

## Phase 3 — `useRBAC.tsx` update

**Prompt 3.1 — Add new roles to RBAC hook**

> **File:** `src/hooks/useRBAC.tsx`
>
> Add entries to `ROLE_PERMISSIONS` for the 4 new role codes, mapping permissions per RBAC v5 §4:
>
> - `'Integrator'`: `advanced_features:access`, `eos:access`, `ask_viv:access`, `vto:edit`, `eos_meetings:schedule`, `eos_meetings:edit`, `qc:schedule`, `rocks:create`, `rocks:edit_own`, `rocks:edit_others`, `risks:create`, `risks:escalate`, `agenda_templates:manage`
> - `'BGT'`: `advanced_features:access`, `eos:access`, `ask_viv:access`, `rocks:create`, `rocks:edit_own`, `risks:create`
> - `'CHC'`: same as BGT + `risks:escalate`
> - `'CET'`: `eos:access`, `ask_viv:access`, `rocks:create`, `rocks:edit_own`, `risks:create`
>
> Update `is_vivacity_team` check: `['Super Admin', 'Team Leader', 'Team Member', 'Integrator', 'BGT', 'CHC', 'CET']`.
> Keep existing `'Team Member'` entry unchanged — valid during transition.

---

## Phase 4 — Staff role reassignments (direct SQL, not Lovable)

**Run after Phases 1–3 are deployed.** Use the staff assignment table at the top of this document.

```sql
UPDATE public.users SET unicorn_role = 'Team Leader', updated_at = now()
  WHERE email IN ('angela@vivacity.com.au', 'dave@vivacity.com.au');

UPDATE public.users SET unicorn_role = 'Integrator', updated_at = now()
  WHERE email = 'nova@vivacity.com.au';

UPDATE public.users SET unicorn_role = 'CHC', updated_at = now()
  WHERE email IN (
    'Sharwari@vivacity.com.au',
    'kelly@vivacity.com.au',
    'tanya@vivacity.com.au',
    'AJ@vivacity.com.au',
    'ezel@vivacity.com.au',
    'sam@vivacity.com.au'
  );

UPDATE public.users SET unicorn_role = 'BGT', updated_at = now()
  WHERE email = 'beverly@vivacity.com.au';

-- Carl, Khian, RJ stay as 'Super Admin' — no change needed
```

Dry-run first: `SELECT email, unicorn_role FROM users WHERE email IN (...)` to confirm rows.

---

## Phase 5 — Part A: Academy gates + bug fixes (7 prompts)

**Run after Phase 3 deployed. Prompts are independent of each other.**

### Prompt 5.1 — A1: Certificate user name bug

> **Route:** `/superadmin/academy/certificates`
>
> The User column shows email only. Fix to display full name on line 1 + email on line 2 (muted), matching the Enrolments page layout. Join the certificates query to `public.users` via the user FK to surface `first_name + last_name`. Test against Certificate No. VA-2026-000009 (the only issued certificate).

### Prompt 5.2 — A2: Breadcrumb titles

> Fix the final breadcrumb segment on all five Academy pages — currently shows `"Page"` for all. Correct values per route:
> - `/superadmin/academy/certificates` → `Certificates`
> - `/superadmin/academy/enrolments` → `Enrolments`
> - `/superadmin/academy/tenant-access` → `Tenant Access`
> - `/superadmin/academy/builder` → `Academy Builder`
> - `/superadmin/academy/package-course` → `Package → Course Mapping`

### Prompt 5.3 — A3: Tenant Access gating

> **Access: TL + SA only.**
>
> - Hide the Tenant Access nav item entirely for Integrator, BGT, CHC, CET — must not appear in Academy sub-navigation
> - Gate the enable/disable tenant toggle and the Edit action row button: TL + SA only
> - Gate the Enrolments link per row: TL + SA only
> - No data model changes — UI gates only via `useRBAC` (check `isSuperAdmin` or `unicorn_role === 'Team Leader'`)

### Prompt 5.4 — A4: Enrolments RBAC

> - All roles can view the Enrolments page and list
> - `+ New Enrolment` button: TL + BGT + CHC — hidden for Integrator and CET
> - `Export CSV` button: TL + SA only
> - Revoke / expire actions on rows: TL + SA only

### Prompt 5.5 — A5: Certificates RBAC

> - Page visible to all team
> - `Issue Certificate Manually` button: TL + SA only — hidden for all other roles
> - Three-dot Actions menu per certificate row: TL + SA only

### Prompt 5.6 — A6: Academy Builder RBAC

> - All team can view and browse the course library
> - `+ New Course` button: TL + BGT only — hidden for Integrator, CHC, CET
> - `Backfill Video Durations` button: TL + SA only
> - Within a course: edit controls (modules, lessons, content): TL + BGT
> - Publish / Unpublish action: TL + SA only
> - Delete / Archive course: TL + SA only

### Prompt 5.7 — A7: Package → Course Mapping RBAC

> - Hide page from BGT, CHC, CET navigation entirely
> - Page accessible to TL, Integrator, SA only
> - Integrator sees the matrix read-only: cells non-interactive, all buttons hidden
> - `+ New rule`, `Copy mappings`, `Select row/column` controls: TL + SA only
> - Matrix checkboxes: TL + SA only can click

---

## Phase 6 — Part B PR 2: `update-role-permission` edge function

**Depends on Phase 1 tables existing.**

### Prompt 6.1 — Implement `update-role-permission` edge function

> **Path:** `supabase/functions/update-role-permission/index.ts`
>
> Implement exactly as specified in `impl-academy-rbac-and-permission-editor.md` PR 2. Key requirements:
> - Caller must be `unicorn_role = 'Super Admin'` AND `is_vivacity_internal = true`
> - Hard guard: SA role can never be set below `full` — reject with 400 `CANNOT_RESTRICT_SUPER_ADMIN`
> - Writes to both `role_permissions` (upsert) and `permission_change_log` (insert)
> - Valid roles: `'Super Admin'`, `'Team Leader'`, `'Integrator'`, `'BGT'`, `'CHC'`, `'CET'` (title case, matching DB)
> - Valid permission levels: `full`, `limited`, `owner_only`, `none`
>
> Required unit test cases in the same PR:
> - Attempt to set SA to `'none'` → must reject 400 `CANNOT_RESTRICT_SUPER_ADMIN`
> - Non-SA caller → must reject 403 `FORBIDDEN`
> - Invalid `feature_key` → must reject 404 `FEATURE_NOT_FOUND`

---

## Phase 7 — Part B PR 3: Permission Editor UI

**Depends on Phase 6.**

### Prompt 7.1 — Permission editor UI

> **Route:** `/administration/role-permissions`
> **Access:** Super Admin only — not in nav for any other role
>
> Implement exactly as specified in `impl-academy-rbac-and-permission-editor.md` PR 3:
> - Permission matrix table with 6 role columns (SA, TL, INT, BGT, CHC, CET)
> - SA column is read-only with lock icon — always Full
> - Cells are dropdown selects: Full (purple), Limited (cyan), Owner only (amber), None (grey)
> - Staged saves — changes highlighted until `Save All Changes` is clicked
> - `Save All Changes` calls `update-role-permission` once per changed cell with progress indicator
> - Change Log drawer (right slide-out) showing `permission_change_log` records
> - Module filter tabs: All | Administration | Clients | Packages | EOS | Audits | Academy | Resource Hub

---

## Phase 8 — Part B PR 4: Wire `usePermission` hook

**Run after Phase 7 deployed and seed data verified. One prompt per module.**

### Prompt 8.1 — `usePermission` hook

> **File:** `src/hooks/usePermission.ts` (new file)
>
> Implement exactly as specified in `impl-academy-rbac-and-permission-editor.md` PR 4. Cache entire `role_permissions` table for 5 minutes per session. Invalidate immediately after a successful permission editor save.

### Prompt 8.2 — Wire Academy module

> Replace hardcoded `isSuperAdmin`/`unicorn_role` checks in all Academy pages with `usePermission('academy.*')` calls. Do not remove Phase 5 gates — replace them. Academy is first because it has the freshest code.

### Prompt 8.3 — Wire Resource Hub

> Replace hardcoded role checks in Resource Hub pages with `usePermission('resource_hub.*')` calls.

### Prompt 8.4 — Wire EOS

> Replace hardcoded role checks across EOS pages with `usePermission('eos.*')` calls.

### Prompt 8.5 — Wire Clients + Packages

> Replace hardcoded role checks in client management and package pages with `usePermission('clients.*')` and `usePermission('packages.*')` calls.

---

## Key constraints (do not violate)

- Never combine enum migrations with table creation in one prompt — separate migrations per sub-prompt 1.3 / 1.4
- Never deploy Phase 4 SQL before Phases 2 and 3 are in production
- The SA column in the permission editor is always Full — enforced in both UI and edge function
- All RBAC gates enforced at both UI layer (hide/show) and RLS/server layer independently
- Do not use `any` in TypeScript in new code
- Do not expose `role_permissions` to browser write — all writes go through the `update-role-permission` edge function

---

## Open questions

- [ ] When `'Team Member'` is fully retired (all staff reassigned), decide whether to archive the old enum value
- [ ] Multi-role support (e.g. BGT + TL for Dave) — separate architectural discussion, out of scope here
- [ ] CET members — assign when confirmed by Angela
- [ ] Dave: assigned Team Leader. If BGT work requires BGT-specific restrictions in future, revisit.

# Tasks Feature Overhaul — Implementation Plan

> **Created:** 16 June 2026 · **Status:** Planning
>
> Covers the full overhaul of the client portal Tasks feature — unifying stage tasks and action items into a single client-facing model.

---

## Background

Two task systems currently coexist and are disconnected:

- **Stage tasks** (`client_task_instances`) — predefined checklist items attached to a package stage. Hidden from client until `stage_instances.released_client_tasks = true` (all-or-nothing boolean per stage). No assignee.
- **Action items** (`client_action_items`) — free-form tasks created by consultants. Never shown in the client portal (admin/consultant side only). Has `owner_user_id` for Vivacity staff but no client-side assignee.

The client portal `/client/tasks` page only shows stage tasks. Action items are invisible to clients entirely.

---

## Agreed Design Decisions

| Decision | Resolution |
|---|---|
| Unification model | Option B — stage tasks publish as action items. Action item is the client-facing record. |
| Status ownership | Action item owns status after publish. `client_task_instances.status` freezes at publish time. |
| Link between models | `published_action_item_id uuid NULL` on `client_task_instances` → `client_action_items(id)` |
| Client-side assignee | Add `assigned_to_tenant_user_id uuid NULL` to `client_action_items` (separate from `owner_user_id` which stays Vivacity staff) |
| Unassigned visibility | All portal users for the tenant can see all action items |
| Visibility gate | Add `is_visible_to_client boolean NOT NULL DEFAULT true` to `client_action_items` |
| Portal user permissions | Can: update status, add comments, upload attachments, reassign within their team. Cannot: edit title/description/due date/priority, create, delete |
| Existing dashboard views | `v_client_package_dashboard` and `v_client_package_whats_next` must LEFT JOIN to `client_action_items` via `published_action_item_id` to read live status for published tasks |

---

## Phase Overview

| Phase | Description | Migration | DB Protocol | Status |
|---|---|---|---|---|
| 1 | Stage task release UI | No | No | Not started |
| 2 | Schema changes | Yes | Yes | Not started |
| 3 | Unified client portal view | No | No | Not started |
| 4 | Publish flow (RPC + admin UI) | Yes | Yes | Not started |
| 5 | Legacy deprecation | Yes | Yes | Deferred |

---

## Phase 1 — Stage Task Release UI

**Goal:** Unblock the existing task backlog immediately. No migration risk.

**What to build:**
- "Release tasks to client" button on the admin stage management view
- Shows current release state per stage (released / unreleased)
- On release: `UPDATE stage_instances SET released_client_tasks = true, released_client_tasks_date = now()`
- Include a "Recall" option (set back to false) for safety

**Note:** This is a bridge while Phases 2–4 are built. Phase 4 will replace this toggle with the publish flow.

**Lovable prompt type:** UI only — no migration, no DB protocol needed.

---

## Phase 2 — Schema Changes

**Goal:** Add the columns needed for the unified model and fix the RLS gap.

### `client_task_instances` — add 1 column

```sql
published_action_item_id uuid NULL REFERENCES public.client_action_items(id) ON DELETE SET NULL
```

- `NULL` = task is unreleased (internal only)
- Set = task has been published; action item at that ID is the live client-facing record

### `client_action_items` — add 2 columns

```sql
assigned_to_tenant_user_id uuid NULL REFERENCES auth.users(id) ON DELETE SET NULL
is_visible_to_client       boolean NOT NULL DEFAULT true
```

- Index: `(tenant_id, assigned_to_tenant_user_id, status)` on `client_action_items`

### RLS rewrite on `client_action_items`

Current RLS is too permissive — portal users can INSERT/UPDATE/DELETE. Must be fixed.

| Role | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| Vivacity staff (Super Admin / Team Leader) | All items | Yes | All fields | Yes |
| Portal users | `is_visible_to_client = true` items for their tenant | No | `status`, `completed_at`, `completed_by`, `assigned_to_tenant_user_id` only | No |

### RPC update

`rpc_create_action_item` — add `p_assigned_to_tenant_user_id uuid DEFAULT NULL` parameter.

**Lovable prompt type:** Migration + RLS — full DB protocol applies.

---

## Phase 3 — Unified Client Portal View

**Goal:** Client portal shows both legacy tasks and action items in one unified list. Depends on Phase 2.

**`useClientAllTasks` changes:**
- Second query fetching `client_action_items` where `is_visible_to_client = true` for the tenant
- Normalise both into a shared `UnifiedTask` type with a `source` discriminator (`'stage_task' | 'action_item'`)
- Merge and sort together (overdue first, then due soon, then rest)

**Client portal UI (`ClientTasksPage.tsx`):**
- "My Tasks" tab — filtered by `assigned_to_tenant_user_id = current user`
- "All Tasks" tab — all visible items for the tenant
- Inline status update control on action items (portal users only)
- Source badge: stage name (e.g. "Stage 1") for `stage_rule` items, nothing for `manual`
- Assignee column showing tenant user name
- Legacy `client_task_instances` remain read-only in portal (no status update) until Phase 4 replaces them

**Lovable prompt type:** UI only — no migration needed.

---

## Phase 4 — Publish Flow

**Goal:** Replace the Phase 1 boolean toggle with per-task publish. Depends on Phase 2.

### RPC `rpc_publish_stage_tasks`

Parameters: `p_stage_instance_id integer`

Behaviour:
- Reads all `client_task_instances` for the stage
- Creates a `client_action_items` record for each (`source = 'stage_rule'`, `source_task_instance_id` set)
- Sets `published_action_item_id` on each `client_task_instance`
- Idempotent — skips tasks where `published_action_item_id` is already set
- Logs a timeline event per item

### Admin UI

- Replace Phase 1 boolean toggle with a "Publish tasks" panel per stage
- Shows unpublished stage tasks as a checklist
- "Publish selected" and "Publish all" actions
- Published tasks show their current action item status inline (read from `client_action_items` via `published_action_item_id`)

### Dashboard views fix

`v_client_package_dashboard` and `v_client_package_whats_next`:
- LEFT JOIN to `client_action_items` via `published_action_item_id`
- For published tasks: read status from `client_action_items.status`
- For unpublished tasks: fall back to `client_task_instances.status` as before

**Lovable prompt type:** Migration (1 RPC) + UI — full DB protocol applies.

---

## Phase 5 — Legacy Deprecation (Deferred)

**Trigger:** Phase 4 validated in production and all new stage tasks are being published as action items.

**What:**
- Backfill existing released `client_task_instances` into `client_action_items`
- Remove legacy branch from `useClientAllTasks`
- Retire `released_client_tasks` column (or leave inert)

**Lovable prompt type:** Data backfill — full DB protocol applies including dry-run verification.

---

## Key Files

| File | Relevance |
|---|---|
| `src/pages/ClientTasksPage.tsx` | Client portal tasks UI — rewritten in Phase 3 |
| `src/hooks/useClientAllTasks.ts` | Data hook — extended in Phase 3 |
| `src/components/client/ClientActionItemsTab.tsx` | Admin action items UI — updated in Phase 2 (assignee field) |
| `supabase/migrations/20260107054614_*.sql` | `client_action_items` schema origin |
| `supabase/migrations/20260502061224_*.sql` | Dashboard views (`v_client_package_dashboard`, `v_client_package_whats_next`) — updated in Phase 4 |

---

## Cross-references

- DB change protocol: `unicorn-kb/handoffs/lovable-production-db-change.md`
- Module status: `unicorn-kb/codebase-state/module-status.md` (Tasks section)
- Feature matrix: `unicorn-kb/codebase-state/feature-matrix-2026-05-20.md`

# Messaging Pipeline

> **Last updated:** 2026-05-12 · **Reconsider by:** 2026-08-12 · **Confidence:** high — verified directly against live DB and migration history.
>
> **Reflects codebase:** `unicorn-cms-f09c59e5@2ac73d59` · **Migration verified:** `20260511071820`

---

## Two tables, one live

There are two message tables. Only one is active.

| Table | Status | Rows | Notes |
|-------|--------|------|-------|
| `public.messages` | **DEPRECATED** | 0 | Superseded by `tenant_messages` at the messaging cutover sprint. Hard-drop deferred. RLS, indexes, and triggers intentionally left in place until drop. |
| `public.tenant_messages` | **LIVE** | 86+ | Active table — all message writes go here. |

When debugging or fixing anything related to messages or message notifications, **work only with `tenant_messages` and its triggers**. Do not touch the `messages` table — it will be hard-dropped in a follow-up sprint.

---

## Trigger chain (live)

New row in `tenant_messages` fires two triggers:

```
tenant_messages INSERT
  ├── trg_tm_on_message_insert        → fn_tm_on_message_insert()
  │     ├── Updates tenant_conversations.last_message_at / last_message_preview
  │     └── Inserts user_notifications for each eligible participant
  └── trg_audit_tenant_message_send   → fn_audit_tenant_message_send()
        └── Writes audit_events row (message_sent)
```

The deprecated `messages` table also has `trg_notify_conversation_participants` → `fn_notify_conversation_participants()` and `trg_update_conversation_on_message` → `fn_update_conversation_on_message()`. These **never fire** (zero rows) and will be dropped with the table.

---

## `fn_tm_on_message_insert` — notification logic

```sql
FOR _participant IN
  SELECT cp.user_id
    FROM public.conversation_participants cp
    JOIN public.tenant_users tu
      ON tu.user_id = cp.user_id
     AND tu.tenant_id = _tenant_id        -- scoped to this conversation's tenant
   WHERE cp.conversation_id = NEW.conversation_id
     AND cp.user_id <> NEW.sender_user_uuid
     AND COALESCE(tu.access_scope, '') <> 'academy_only'  -- academy filter
LOOP
  INSERT INTO public.user_notifications (...)
  ON CONFLICT (dedupe_key) DO NOTHING;
END LOOP;
```

Key behaviours:

| Scenario | Result |
|----------|--------|
| `access_scope = 'academy_only'` | Excluded — no notification |
| No `tenant_users` row for this tenant | Excluded by INNER JOIN — no notification, no error |
| User in `tenant_users` for multiple tenants | Only the row matching the conversation's `tenant_id` is evaluated |
| `access_scope = 'full'` (primary/secondary contact) | Receives notification |
| Duplicate message event | `ON CONFLICT (dedupe_key) DO NOTHING` — safe |
| Notification failure | Wrapped in `EXCEPTION WHEN OTHERS → RAISE WARNING` — message save is never blocked |

---

## academy_only notification bug — resolved

**Bug:** Academy-only users (`access_scope = 'academy_only'`, `relationship_role = 'academy_user'`) were receiving `message`-type notifications via `fn_tm_on_message_insert`, which originally had no `access_scope` filter.

**Fix:** Migration `20260511071820` (applied 11 May 2026, 07:18 UTC) — unnamed MCP migration, present in `supabase_migrations.schema_migrations`.

**Backfill:** The same migration deleted 27 contaminated `user_notifications` rows for academy users. As of 12 May 2026, the academy user has zero message notifications.

**What the fix did NOT change:**
- RLS policies on `user_notifications` — untouched
- Any frontend notification code — untouched
- `fn_notify_conversation_participants` on the deprecated `messages` table — also patched in the same migration for completeness, but irrelevant since that trigger never fires

---

## Staff members in conversation_participants

Vivacity staff (e.g. `AJ@vivacity.com.au`, `carl@vivacity.com.au`) appear in `conversation_participants` for client conversations but have **no `tenant_users` row** for the client's tenant. The INNER JOIN in `fn_tm_on_message_insert` silently excludes them from message notifications. This is consistent with staff accessing messages via a separate internal inbox path — not a bug.

---

## Columns reference — `tenant_messages`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `conversation_id` | uuid | FK → `tenant_conversations.id` |
| `tenant_id` | bigint | NOT NULL |
| `sender_user_uuid` | uuid | NOT NULL — used by trigger as `NEW.sender_user_uuid` |
| `sender_type` | text | NOT NULL |
| `body` | text | NOT NULL |
| `meta` | jsonb | nullable |
| `created_at` | timestamptz | NOT NULL |

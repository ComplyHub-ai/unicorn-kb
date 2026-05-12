# System Design — Conventions & Patterns

> **Last updated:** 2026-04-28 · **Reconsider by:** 2026-10-27 · **Confidence:** medium-high — canonical patterns derived from invite-user + useAuth; tenant ID corrected to 6372 in April 2026 audit. Edge function auth pattern note added. Template/instance pattern (added 2026-04-23) is descriptive of 1.0 with one confirmed 2.0 use; further extension is open. **2026-04-27:** scope clarified — these conventions are aspirational targets for hand-written code, not enforced via pre-merge review. See `reference/decision-trail.md → ADR-011`.
>
> The "how we do things here" doc. Rules to follow when adding code, with rationale so you can handle edge cases.

---

## Scope of these conventions

These patterns are the target. **They are not enforced via pre-merge review** — see [decision-trail.md → ADR-011](../reference/decision-trail.md#adr-011) for why. When hand-writing code via Claude Code, follow them. Lovable-generated code may not match — flag drift you spot in `reference/brainstorm-log.md`, but don't expect it to be caught at merge time.

The technical patterns below (RLS three-step ritual, edge function auth, coercion triggers) are still **correct** even when not enforced. Self-review against them is the operating discipline.

---

## Multi-tenant model

### Identity shape
Every tenant-scoped row has:
- `tenant_id: int` (FK → `tenants.id`)
- `created_at`, `updated_at` (timestamps, `updated_at` maintained by trigger)

User identity:
- `auth.users` — Supabase-managed, UUID primary key
- `users` — app profile, keyed by `user_uuid` matching `auth.users.id`. Holds `unicorn_role`, `tenant_id`, name, avatar.
- `tenant_members` — platform RBAC table. Roles: `Admin` / `General User`. Has `status` (active/inactive/pending), `invited_at`, `joined_at`. Used for auth checks (`useAuth`), billing signals, seat limits, and AI feature gates. Used in RLS policies for client-user SELECT checks (see ritual below).
- `tenant_users` — client-side contact/membership table. Roles: `parent` (can manage) / `child` (read-only). Has `primary_contact` and `secondary_contact` booleans (auto-set by trigger when `role = 'parent'`). Used for client management UI, email delivery, document generation, invite-user provisioning, and M365 provisioning.

> **Two-table note:** both tables store user–tenant relationships but serve different purposes. The RLS SELECT policy template below references `tenant_members` — that governs platform access. `tenant_users` governs client-org contact roles and is written by the invite-user edge function. Do not conflate them.

### Role matrix

| `tenant_id` | Role values | Capability |
|---|---|---|
| `6372` (Vivacity) | `Super Admin` | All access. Can switch tenant. Manages invites globally. |
| `6372` | `Team Leader` | Staff-level access. Cross-tenant read for assigned work. |
| `6372` | `Team Member` | Staff. Scoped to assignments. |
| any other | `Admin` | Client org admin. Manages their org's users + settings. |
| any other | `User` | Client user. Operational access only. |

Enforced in `invite-user` edge function ([supabase/functions/invite-user/index.ts:22-36](../supabase/functions/invite-user/index.ts)).

---

## RLS (Row Level Security)

### The three-step ritual

**For every new table with mixed staff + client access:**

```sql
-- 1. Tenant-scoped SELECT (client RTO access)
CREATE POLICY tenant_read ON <table>
  FOR SELECT
  USING (tenant_id IN (
    SELECT tenant_id FROM tenant_members WHERE user_uuid = auth.uid()
  ));

-- 2. Vivacity staff ALL (consultants)
CREATE POLICY staff_all ON <table>
  FOR ALL
  USING (is_vivacity())
  WITH CHECK (is_vivacity());

-- 3. Explicitly enable RLS — the step everyone forgets
ALTER TABLE <table> ENABLE ROW LEVEL SECURITY;

-- 4. Verify in Supabase Studio that RLS shows as enabled.
```

**Why all three, not two:**
- Skipping step 1 locks clients out of their own data.
- Skipping step 2 locks Vivacity staff out of every tenant's data. Staff are not `tenant_members` of client RTOs — they're members of tenant 6372. The tenant-scoped SELECT alone blocks them entirely.
- Skipping step 3 makes the policies completely inactive. Silent failure — no errors, Studio shows the policies.

**History:** Both failure modes occurred in production on the sibling Vivacity project in April 2026. See [decision-trail.md → ADR-005](../reference/decision-trail.md#adr-005). Treat this as a solved problem — don't re-litigate it.

### Tables that don't need both policies

- **Vivacity-only tables** (e.g., internal staff tools) — only need `is_vivacity()` ALL.
- **Cross-tenant system tables** (e.g., `tenants` itself) — special-case policies; flag in `reference/brainstorm-log.md` and tag Carl.

### Public helper functions
Always available in RLS policies:
- `is_vivacity()` — current user is in tenant 6372
- `is_superadmin()` — current user's `unicorn_role = 'Super Admin'`
- `current_tenant()` — active tenant id

> **Edge function auth diverges from RLS policies.** Edge functions do not call these Postgres functions. They check the `is_vivacity_internal` boolean column on the `users` table directly (e.g. `.select('is_vivacity_internal')`), or call the `is_vivacity_staff` RPC. Client-portal routes use `has_tenant_access_safe(tenant_id, auth.uid())`. See `_shared/addin-auth.ts` for the shared pattern.

---

## Edge functions

### Canonical pattern

```ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";
import { corsHeaders } from "../_shared/cors.ts";

serve(async (req) => {
  if (req.method === "OPTIONS") return new Response("ok", { headers: corsHeaders });

  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!,
    { auth: { persistSession: false } }
  );

  // 1. Extract + validate caller token
  const callerToken = req.headers.get("Authorization")?.replace(/^Bearer\s+/i, "");
  if (!callerToken) return jsonResponse(401, { ok: false, code: "NO_AUTH", ... });

  // 2. Resolve caller identity
  const { data: callerUser, error } = await supabase.auth.getUser(callerToken);
  if (error || !callerUser?.user) return jsonResponse(401, { ok: false, code: "AUTH_FAILED", ... });

  // 3. Check caller's role/tenant from `users` table (NOT just JWT claims)
  const { data: callerProfile } = await supabase.from('users')
    .select('unicorn_role, tenant_id')
    .eq('user_uuid', callerUser.user.id)
    .maybeSingle();

  // 4. Authorise — role + tenant + action-specific
  if (callerProfile?.unicorn_role !== 'Super Admin') {
    return jsonResponse(403, { ok: false, code: "FORBIDDEN", ... });
  }

  // 5. Validate input payload with explicit shape check

  // 6. Do the work, return structured JSON
});
```

### Response envelope

Always return this shape:
```json
{ "ok": true,  "data": {...} }
{ "ok": false, "code": "MACHINE_READABLE_CODE", "detail": "human-readable explanation" }
```

The frontend keys off `code`. See [docs/INVITE_USER_DIAGNOSTICS.md](../docs/INVITE_USER_DIAGNOSTICS.md) for the canonical error code map.

### Why service-role

Edge functions use the service-role key because they perform actions the caller can't do under RLS (create auth users, bypass tenant checks for cross-tenant operations). This is also why **manual authorization is mandatory** — RLS is off, so the function itself is the security boundary.

---

## Frontend patterns

### Query keys

Convention: `[domain, subentity, ...args]` — e.g. `['audits', auditId, 'findings']`, `['eos', 'rocks', tenantId, quarter]`. Keys must be stable across rerenders.

### Mutations + invalidation

```ts
const queryClient = useQueryClient();
const mutation = useMutation({
  mutationFn: async (input) => { /* supabase call */ },
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['rocks', tenantId] });
  },
});
```

Do not use `refetchQueries` unless you need immediate data-in-hand. `invalidateQueries` is cheaper and sufficient for most UI refreshes.

### Real-time subscriptions

Pattern (see [src/hooks/useMeetingRealtime.ts](../src/hooks/useMeetingRealtime.ts)):

```ts
useEffect(() => {
  const channel = supabase
    .channel(`meeting-${meetingId}`)
    .on('postgres_changes',
        { event: '*', schema: 'public', table: 'meeting_segments', filter: `meeting_id=eq.${meetingId}` },
        (payload) => { /* update local or invalidate query */ }
    )
    .subscribe();
  return () => { supabase.removeChannel(channel); };
}, [meetingId]);
```

Rule: channel names are unique per entity instance. Always clean up in the effect return.

### Forms

1. Define zod schema at top of component.
2. `useForm<SchemaType>({ resolver: zodResolver(schema), defaultValues: ... })`
3. Submit via mutation, not direct supabase call — gives you `isPending`, error handling, and invalidation for free.
4. For NOT NULL DB columns with frontend writes, **still** coerce nulls with a DB trigger. Don't rely on zod or form defaults alone.

### Wrapper pages

Many pages have a `*Wrapper` variant. Convention: `Wrapper` mounts layout chrome + auth gates; the inner `Page` is pure. When in doubt, `App.tsx` imports the Wrapper. Follow the existing pattern when adding pages.

### Role-aware rendering

Use the `profile.unicorn_role` and `profile.tenant_id` from `useAuth()`:

```ts
import { VIVACITY_TENANT_ID } from '@/hooks/useVivacityTeamUsers'; // = 6372
const { profile } = useAuth();
const isVivacityStaff = profile?.tenant_id === VIVACITY_TENANT_ID;
const isSuperAdmin = profile?.unicorn_role === 'Super Admin';
```

Do not duplicate this logic per-component — put role checks in a hook or util if they appear more than twice. (Forward note: the Clean Architecture refactor — see `reference/clean-architecture-refactor.md` — proposes consolidating these in `src/domain/access/can.ts`. New role checks should follow that target shape where practical.)

---

## Database conventions

### Standard columns
```sql
id           bigserial PRIMARY KEY
tenant_id    int NOT NULL REFERENCES tenants(id)
created_at   timestamptz NOT NULL DEFAULT now()
updated_at   timestamptz NOT NULL DEFAULT now()
```

Apply the `updated_at` trigger (standard across the project — see existing migrations for the trigger function).

### Coercion triggers for NOT NULL + frontend writes

Lovable-generated forms may send explicit `NULL` on submit, which overrides column defaults. Protect NOT NULL columns:

```sql
CREATE OR REPLACE FUNCTION coerce_<table>_nulls()
RETURNS trigger AS $$
BEGIN
  NEW.<col1> := COALESCE(NEW.<col1>, <default1>);
  NEW.<col2> := COALESCE(NEW.<col2>, <default2>);
  RETURN NEW;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_coerce_<table>
BEFORE INSERT OR UPDATE ON <table>
FOR EACH ROW EXECUTE FUNCTION coerce_<table>_nulls();
```

Reference: ADR-008 in [decision-trail.md](../reference/decision-trail.md#adr-008).

### Template → instance pattern (inherited from Unicorn 1.0)

Many domain entities split into:
- a **template** table — canonical definition, low row count, often global or tenant-scoped (e.g. `documents`, `emails`, the various `*_tasks` tables in 1.0)
- an **instance** table — per-stage materialisation, FK to a `stage_instances`-equivalent, high row count (e.g. `document_instances`, `email_instances`, `*_task_instances`)

This is load-bearing in 1.0 — see [migration-1to2.md → Unicorn 1.0 data model reference](../reference/migration-1to2.md#unicorn-10-data-model-reference) for the row-count fan-out (~150× for staff tasks, ~143× for documents).

In 2.0, `package_stage_instances` follows this pattern (template = `package_stages`, instance = `package_stage_instances`). When designing a new entity that gets materialised per stage / per engagement / per recurrence, prefer this split over a single denormalised table — it keeps reporting and lifecycle hooks composable, and matches the mental model the team already has from 1.0.

Whether to apply this pattern to tasks/emails/documents in 2.0 is an open question — flag in `reference/brainstorm-log.md` when scoping.

### Migrations
- Files in [supabase/migrations/](../supabase/migrations/) — timestamped, one migration per logical change.
- Naming: `<timestamp>_<uuid>.sql` (Lovable-generated default).
- There is no pre-merge migration review gate today (see `reference/decision-trail.md → ADR-011`). Lovable-generated migrations land direct on `main`; hand-written migrations should be self-reviewed against the New table checklist below.
- If the migration adds a table with mixed staff+client access, the three-step RLS ritual MUST be in the same migration.

### Function hardening

Every new or replaced `public` schema function **must** include all three of these, in this order, immediately after `LANGUAGE` / volatility / `SECURITY DEFINER`:

```sql
SECURITY DEFINER
SET search_path = ''   -- empty string, not 'public'
AS $function$
```

**Why empty string, not `'public'`:**
- `SET search_path = 'public'` prevents the caller injecting their session search_path (good), but still allows a malicious object in `public` to shadow an unqualified call inside the function body.
- `SET search_path = ''` closes that second vector entirely — every reference must be schema-qualified or PostgreSQL refuses to resolve it.

**Consequence:** with `search_path = ''`, every object reference in the function body must carry an explicit schema prefix:

```sql
-- Wrong (breaks with empty search_path)
PERFORM normalise_abn(NEW.abn);
SELECT id FROM tenant_users WHERE ...

-- Correct
PERFORM public.normalise_abn(NEW.abn);
SELECT id FROM public.tenant_users WHERE ...

-- These are always fine — pg_catalog is always searched regardless
SELECT now(), COALESCE(...), BTRIM(...)
```

`auth.uid()`, `auth.users`, `extensions.*` — already schema-qualified everywhere; no change needed.

**Existing 28 functions patched 12 May 2026** currently use `SET search_path = 'public'` (P1-e). Upgrading them to `= ''` requires a body audit first — see `pinned/decisions.md → P1-e-ii`. Do not swap them without that audit.

**Lovable prompt guardrail — include this line in every prompt that creates or replaces a function:**

> *"All new or replaced functions must use `SET search_path = ''` (empty string). All object references in the function body must be fully schema-qualified (e.g. `public.my_function()`, not `my_function()`). Do not use `SET search_path = 'public'`."*

---

## New function checklist (literal, use this)

Before merging a migration that creates or replaces a function:

- [ ] `SECURITY DEFINER` — included if the function accesses `auth.*` or bypasses RLS?
- [ ] `SET search_path = ''` — empty string, not `'public'`?
- [ ] Every non-`pg_catalog` reference fully qualified with schema prefix?
- [ ] `REVOKE ALL ON FUNCTION ... FROM PUBLIC` + `GRANT EXECUTE ... TO authenticated` (or `service_role`) — added?
- [ ] `anon` EXECUTE **not** granted (Supabase may auto-extend — check `information_schema.routine_privileges`)?

---

## New table checklist (literal, use this)

Before merging a migration that creates a table:

- [ ] Does the table really not exist? (Search schema + views first.)
- [ ] `tenant_id int NOT NULL REFERENCES tenants(id)` — included?
- [ ] `created_at`, `updated_at` — included with defaults?
- [ ] `updated_at` trigger wired up?
- [ ] RLS policies written:
  - [ ] Tenant-read SELECT (`tenant_members` check)
  - [ ] Staff ALL (`is_vivacity()`)
- [ ] `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` — actually run?
- [ ] Verified in Supabase Studio that RLS is enabled?
- [ ] If any NOT NULL columns can receive writes from Lovable forms → coercion trigger added?

---

## What NOT to do

- Don't call LLM APIs from the frontend.
- Don't write RLS policies without enabling RLS.
- Don't skip caller validation in an edge function just because "RLS is on" — the service-role key bypasses RLS.
- Don't cache the Supabase client per-component — use the singleton from `src/integrations/supabase/client.ts`.
- Don't put tenant-ID checks on the frontend as the only line of defense. RLS first; frontend filters for UX.
- Don't mix `react-query` with ad-hoc `useEffect` fetches for the same data. Pick one per surface.
- Don't use `SET search_path = 'public'` in new functions — use `SET search_path = ''` and fully-qualify all object references. `= 'public'` is the legacy state of 28 existing functions awaiting P1-e-ii; don't add to that list.
- Don't grant `EXECUTE` to `anon` or `PUBLIC` on any function — Supabase may auto-extend grants; always check `information_schema.routine_privileges` after applying a function migration.

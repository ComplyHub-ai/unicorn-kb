# Flow Patterns — Conceptual Walkthroughs

> **Last updated:** 2026-04-22 · **Reconsider by:** 2026-10-22 · **Confidence:** medium — flows are partially inferred; happy paths verified, edge cases and failure modes should be traced against real code before being trusted.
>
> How the core product flows work end-to-end. Use these when tracing bugs, onboarding, or explaining the system to someone who hasn't seen the code.

Each flow: **trigger → journey → side effects → failure modes**.

---

## Auth flows

### Login (password)

```
User submits email+password on /login
  → supabase.auth.signInWithPassword()
  → onAuthStateChange fires in AuthProvider
  → setUser / setSession
  → setTimeout(0) → fetchUserProfile() loads from `users` table
  → ProtectedRoute allows navigation
  → Redirect to default landing (currently /manage-packages)
```

Files: [src/pages/Login.tsx](../src/pages/Login.tsx), [src/hooks/useAuth.tsx](../src/hooks/useAuth.tsx), [src/components/ProtectedRoute.tsx](../src/components/ProtectedRoute.tsx).

**Deadlock note:** `fetchUserProfile` is wrapped in `setTimeout(fn, 0)` inside `onAuthStateChange` to avoid a Supabase deadlock where synchronous DB calls inside the auth callback block the session update. Don't "refactor" this away.

**Failure modes:**
- `users` row missing for the auth user → profile stays `null`, role checks fail silently. Fix: ensure invite flow upserts into `users`.
- Stale session from a deleted user → `getSession()` returns a session but `getUser()` 401s. Handle both.

### Invite user

```
Super Admin submits invite form
  → invoke `invite-user` edge function (Bearer token)
    → verify caller is Super Admin
    → hash token + insert into user_invitations
    → create auth user (service role)
    → insert tenant_members row
    → invoke `send-invitation-email` edge function
      → Mailgun send with invite token link
  → Invitee clicks link → /accept-invitation?token=...
    → hash incoming token, lookup in user_invitations
    → on match, set password + profile via Supabase auth
    → mark invitation as accepted
    → redirect to login
```

Files: [supabase/functions/invite-user/index.ts](../supabase/functions/invite-user/index.ts), [supabase/functions/send-invitation-email/](../supabase/functions/send-invitation-email/), [src/pages/AcceptInvitation.tsx](../src/pages/AcceptInvitation.tsx), [docs/INVITE_USER_DIAGNOSTICS.md](../docs/INVITE_USER_DIAGNOSTICS.md).

**Failure modes:** The diagnostics doc is the canonical reference. Key classes: `NO_AUTH` (missing token), `AUTH_FAILED` (bad JWT), `FORBIDDEN` (non-Super Admin caller), `ROLE_NOT_ALLOWED` (e.g. client roles on Vivacity tenant or vice versa), `EMAIL_EXISTS`, `EMAIL_SEND_FAILED` (Mailgun).

### Password reset

```
User enters email on /reset-password (step 1)
  → supabase.auth.resetPasswordForEmail()
  → Supabase sends recovery email (template: password-reset.html)
User clicks link → lands on /reset-password with recovery token
  → supabase handles token exchange in background
  → user enters new password
  → supabase.auth.updateUser({ password })
  → redirect to login
```

Super Admin can bypass and force-change a user's password via `admin-change-password` edge function.

---

## EOS Level 10 Meeting flow

Canonical spec: [docs/EOS_LEVEL10_SPECIFICATION.md](../docs/EOS_LEVEL10_SPECIFICATION.md). Paraphrase below.

### Weekly meeting lifecycle

```
Scheduled time arrives (recurrence expanded by `generate-meeting-recurrence`)
  → Facilitator opens /eos/meetings/:meetingId/live
  → LiveMeetingView subscribes to Supabase channel `meeting-:id`
  → Agenda loaded from active Rocks, Scorecard entries, Issues
  → Facilitator clicks through segments (Segue, Scorecard, Rock Review,
     Customer/Employee Headlines, To-Do Review, IDS, Conclude)
  → Each segment update publishes realtime event → all participants see it
  → Issues resolved via IDS → new To-Dos generated
  → Meeting concluded → summary saved, ratings captured,
     To-Dos assigned with 7-day due dates
  → /eos/meetings/:id/summary shows post-meeting recap
```

Hook: [src/hooks/useMeetingRealtime.ts](../src/hooks/useMeetingRealtime.ts). View: [src/components/eos/LiveMeetingView.tsx](../src/components/eos/LiveMeetingView.tsx).

**Failure modes:**
- Channel drops mid-meeting → reconnect logic in the hook; if stale state persists, force `invalidateQueries`.
- Two facilitators in the same meeting → last write wins. Product decision; don't add locking unless asked.
- Client viewer joins → RLS should filter to only tagged items. Verify `client_id` on rocks/issues is correctly scoped.

### Quarterly Conversation (QC)

```
QC scheduled in /eos/qc
  → Session starts at /eos/qc/:id
  → Structured 5-question format (EOS QC protocol)
  → Notes saved + signed
  → Result feeds into next quarter's Rocks planning
```

Hook: [src/hooks/useQuarterlyConversations.ts](../src/hooks/useQuarterlyConversations.ts).

### V/TO editing

```
/eos/vto → VtoEditor
  → Sections: Core Values, Core Focus, 10-Year Target, Marketing Strategy,
              3-Year Picture, 1-Year Plan, Quarterly Rocks
  → Autosave per section (debounced mutation)
  → Real-time sync if multiple admins edit simultaneously
```

### Rocks

Quarterly priorities. Can be tagged `company`, `team`, or `client_id` (scoped to a tenant's relationship). Client-tagged Rocks appear in `/client/eos` for that client.

---

## Audit flow

```
Admin creates/selects template (/audits/create-template or via library)
  → AuditTemplateBuilder configures sections + questions
  → Template saved with tenant_id scoping
Admin starts an audit for a client
  → /audits/:id — workspace view (6-tab interface per old docs)
  → Findings captured per question (/audits/:id/findings)
  → Corrective actions tracked (/audits/:id/actions)
  → Report generated (/audits/:id/report)
```

Hooks: [src/hooks/useAudits.ts](../src/hooks/useAudits.ts), [src/hooks/useAuditTemplates.ts](../src/hooks/useAuditTemplates.ts), [src/hooks/useReusableAuditTemplates.ts](../src/hooks/useReusableAuditTemplates.ts).

**Distinct from old docs:** The current Audits module does **not** include AI-generated reports, document analysis, or the six named AI agents. That's the other Vivacity project. If the spec references `generate-audit-report` edge function, it's not in this codebase.

---

## Data fetching pattern (canonical)

```ts
// Inside a component
const { profile } = useAuth();
const tenantId = profile?.tenant_id;

const { data, isLoading, error } = useQuery({
  queryKey: ['rocks', tenantId, quarter],
  queryFn: async () => {
    const { data, error } = await supabase
      .from('rocks')
      .select('*, owner:users(user_uuid, first_name, last_name)')
      .eq('tenant_id', tenantId)
      .eq('quarter', quarter);
    if (error) throw error;
    return data;
  },
  enabled: !!tenantId,
});
```

Notes:
- `enabled` gate prevents running the query before auth resolves.
- Tenant filter is applied at the frontend AND enforced at RLS — belt and braces.
- Joined selects use Supabase's relational syntax (`owner:users(...)`) — prefer this over multiple round-trips.

---

## Mutation pattern (canonical)

```ts
const queryClient = useQueryClient();
const { toast } = useToast();

const createRock = useMutation({
  mutationFn: async (input: RockInput) => {
    const { data, error } = await supabase
      .from('rocks')
      .insert({ ...input, tenant_id: tenantId })
      .select()
      .single();
    if (error) throw error;
    return data;
  },
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['rocks', tenantId] });
    toast({ title: 'Rock created' });
  },
  onError: (err) => {
    toast({ title: 'Could not create rock', description: err.message, variant: 'destructive' });
  },
});
```

---

## Edge function invocation pattern

```ts
const { data, error } = await supabase.functions.invoke('invite-user', {
  body: { email, first_name, last_name, invite_as, tenant_id, unicorn_role },
});

if (error) { /* network/infra failure */ }
if (!data?.ok) {
  // structured business error — key off data.code
  switch (data.code) {
    case 'EMAIL_EXISTS': ...
    case 'FORBIDDEN': ...
    default: ...
  }
}
```

Never parse edge function responses by string matching on `detail`. Always switch on `code`.

---

## Role-aware rendering

```ts
const { profile } = useAuth();
const isVivacityStaff = profile?.tenant_id === 6372;
const isSuperAdmin = profile?.unicorn_role === 'Super Admin';
const isClientAdmin = !isVivacityStaff && profile?.unicorn_role === 'Admin';

// Use in JSX
{isSuperAdmin && <DeleteTenantButton />}
{isClientAdmin && <InviteUserButton />}
```

Do NOT use the presence of a session as permission. Always check role + tenant.

---

## File upload flow (documents)

```
User selects file in <input type="file">
  → supabase.storage.from('bucket').upload(path, file)
  → on success, insert metadata row into documents table with storage_path
  → RLS on both bucket and table gates visibility
```

**Path convention:** `tenant-{tenantId}/category/filename-{timestamp}.ext`. Storage policies extract tenant from the path. Do not use flat paths — they bypass tenant isolation.

---

## When flows collide

- **Invite + active session** — if a logged-in user clicks an invite link for a different account, the flow silently accepts into the wrong account. Mitigation: sign out before accepting, or detect session mismatch in `AcceptInvitation`.
- **Realtime + React strict mode** — double-mount in dev can double-subscribe channels. Cleanup effect is mandatory.
- **Mutation during navigation** — if the user navigates mid-mutation, the toast may show on the wrong page. Accept this for now; add AbortController if it becomes a UX issue.

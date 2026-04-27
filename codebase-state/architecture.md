# Architecture

> **Last updated:** 2026-04-27 · **Reconsider by:** 2026-07-27 · **Confidence:** medium-high — tenant ID, EOS table names, pg_cron, and LLM provider verified in codebase (April 2026 audit).
>
> **Reflects commit:** `<codebase>@cf8d1314` (2026-04-25). Edge function count unchanged at 117 over the past week (a `create-client-audit` function landed on `65c426aa` and was reverted same-day on `084a5e17`). Three new RPC migrations added — see Helper functions section.
>
> System design reference for Unicorn 2.0. How everything connects, where logic lives, and the constraints to respect.
> Reconciled against the `unicorn-cms-f09c59e5` working tree — April 2026. Note: a separate Vivacity Supabase project previously shared the "Unicorn 2.0" name but had different edge functions and module scope. That sibling project is now largely superseded — this doc describes the current codebase.

---

## System overview

```
Lovable (React + Vite + TS + shadcn + Tailwind — admin/consultant + client UI)
    │
    ├──► Supabase project: yxkgdalkbrriasiyyrwk
    │        ├── PostgreSQL (multi-tenant, RLS)
    │        │     ├── tenants, tenant_members, user_invitations, tenant_settings
    │        │     ├── users (profile)
    │        │     ├── packages, package_stages, package_stage_instances
    │        │     ├── EOS schema (eos_vto, eos_rocks, eos_issues, eos_todos,
    │        │     │              eos_scorecard_metrics, eos_scorecard_entries,
    │        │     │              eos_meetings, eos_meeting_segments, eos_meeting_participants,
    │        │     │              eos_meeting_ratings, eos_meeting_series,
    │        │     │              eos_qc, eos_agenda_templates, eos_workspaces,
    │        │     │              eos_flight_plans, eos_health_snapshots, eos_alerts,
    │        │     │              accountability_charts, accountability_chart_versions,
    │        │     │              accountability_functions, accountability_seats,
    │        │     │              accountability_seat_roles, accountability_seat_assignments)
    │        │     └── Audit schema (audit_templates, audit_sections, audit_questions,
    │        │                       client_audits, audit_findings, audit_actions,
    │        │                       audit_reports)
    │        ├── Auth (email/password, invitation token flow, password reset)
    │        ├── Realtime (EOS live meeting sync via channels)
    │        ├── Storage (tenant documents, audit evidence, avatars)
    │        └── Edge Functions (see list below)
    │
    ├──► Mailgun (transactional email via `send-invitation-email` + Supabase auth)
    │
    ├──► Microsoft 365 / Outlook (calendar sync, email capture, addin, SharePoint)
    ├──► ClickUp (task and time sync)
    └──► [Planned] Stripe
```

**Supabase project ID:** `yxkgdalkbrriasiyyrwk` (from `supabase/config.toml`).

---

## Edge functions (117 live)

All in [supabase/functions/](../supabase/functions/). Pattern: service-role Supabase client, manual caller validation, JSON response with `{ ok, code, detail }` error shape.

**Core user/auth functions:**

| Function | Purpose |
|---|---|
| `invite-user` | Creates user in `auth.users` + `tenant_members`; dispatches Mailgun invite email. Validates caller is Super Admin. |
| `resend-invite` | Resend an existing pending invite. |
| `cancel-invite` | Cancel a pending invitation. |
| `send-invitation-email` | Template-driven Mailgun send (invite template). |
| ~~`admin-change-password`~~ | ~~Super Admin password change for another user (service-role).~~ **Not found in `supabase/functions/` — function may have been removed or renamed. Verify with RJ.** |
| `delete-user` | Hard delete from `auth.users` + cleanup. |
| `bulk-user-action` | Batch user operations. |
| `toggle-user-status` | Enable/disable user. |
| `update-user-profile` | Update user profile fields (name, avatar, etc.). |
| `update-user-role` | Change `unicorn_role` for a user. |
| `send-password-reset` / `send-self-password-reset` | Password reset emails. |
| `send-staff-onboarding-email` | Staff onboarding flow. |
| `provision-m365-user` | Microsoft 365 user provisioning. |

**AI / intelligence functions:**

| Function | Purpose |
|---|---|
| `ai-generate-suggestions` | LLM-backed suggestion generator (Rocks drafts, agenda hints). |
| `ai-orchestrator` | Central AI orchestration hub. |
| `ai-suggest-rock` | AI-powered Rock suggestions. |
| `academy-ai-generate` | Academy content generation. |
| `analyze-document` | Document AI analysis. |
| `assistant-answer` / `copilot-chat` | AI assistant / copilot chat. |
| `compliance-assistant` / `client-ai-companion` | Compliance and client-facing AI. |
| `research-answer` / `research-scrape` | Research intelligence. |
| `research-audit-intelligence` | Audit intelligence packs. |
| `research-evidence-gap-check` / `research-template-gap-analysis` | Gap analysis tools. |
| `research-enrich-tenant` / `research-public-snapshot` | Tenant enrichment. |
| `research-tas-context` / `research-jobs` variant | TAS-specific research. |
| `calculate-predictive-risk` / `run-tenant-risk-forecast` | Risk forecasting. |
| `run-strategic-signal-analysis` / `strategic-orchestration` | Strategic intelligence. |
| `query-knowledge-graph` / `vector-search` | Knowledge graph and vector search. |
| `vector-index-rebuild` / `vector-index-update` / `vector-index-remove` | Vector index management. |
| `chunk-document` / `scan-document` | Document processing pipeline. |
| `extract-document-fields` | AI field extraction from documents. |
| `generate-document` / `generate-excel-document` | Document generation. |
| `help-center-chat` | Help center AI chat. |

**Meetings / EOS functions:**

| Function | Purpose |
|---|---|
| `generate-meeting-recurrence` | Expand a recurrence rule into instances. |
| `generate-meeting-summary` | Post-meeting AI summary. |
| `generate-minutes-draft` / `generate-minutes-from-transcript` | Meeting minutes generation. |
| `extract-copilot-minutes` | Extract minutes from Copilot transcripts. |
| `publish-meeting-minutes` | Publish approved minutes. |
| `create-meeting-time-drafts` / `create-tasks-from-minutes` | Meeting artifact creation. |
| `sync-meeting-artifacts` | Sync meeting artifacts across systems. |
| `scorecard-refresh` | EOS scorecard data refresh. |

**Outlook / Microsoft 365 integration:**

| Function | Purpose |
|---|---|
| `outlook-auth` | Outlook OAuth flow. |
| `sync-outlook-calendar` | Calendar sync. |
| `outlook-time-draft-worker` | Time draft worker. |
| `capture-outlook-email` / `addin-email-capture` | Outlook addin email capture. |
| `addin-auth-exchange` / `addin-diagnostics-usage` | Outlook addin auth and diagnostics. |
| `addin-email-create-task` / `addin-email-link-attachments` | Addin task/attachment management. |
| `addin-meeting-capture` / `addin-meeting-create-time-draft` | Addin meeting capture. |
| `send-email-graph` / `send-composed-email` | Email sending via Microsoft Graph. |
| `send-stage-email` | Stage-triggered emails. |

**SharePoint functions:**

| Function | Purpose |
|---|---|
| `browse-sharepoint-folder` | Browse SharePoint folders. |
| `import-sharepoint-template` | Import document templates from SharePoint. |
| `link-sharepoint-document` | Link documents to SharePoint. |
| `provision-tenant-sharepoint-folder` | Provision tenant folder structure. |
| `resolve-tenant-folder` | Resolve tenant SharePoint path. |
| `validate-sharepoint-root-folder` | Validate root folder config. |
| `deliver-governance-document` | Deliver governance docs. |

**TGA / RTO integration:**

| Function | Purpose |
|---|---|
| `search-organisations` / `get-organisation-details` | Lookup against training.gov.au. |
| `tga-integration` / `tga-sync` / `tga-fetch-scope` | TGA data sync. |
| `tga-product-lookup` / `tga-rto-import` / `tga-rto-preview` / `tga-rto-sync` | RTO product management. |

**Compliance / audit functions:**

| Function | Purpose |
|---|---|
| `export-compliance-pack` / `export-client-timeline-pdf` | Compliance pack export. |
| `generate-pack` / `generate-release-documents` | Document pack generation. |
| `generate-staff-checklist` | Staff checklist generation. |
| `bulk-generate-phase-documents` | Bulk document generation for phases. |
| `regulator-watch-check` | Regulator monitoring. |

**Package / lifecycle functions:**

| Function | Purpose |
|---|---|
| `add-missing-packages` | Ensure all tenants have the default package set. |
| `calculate-phase-completeness` | Phase completion calculation. |
| `run-stage-health-monitor` | Stage health monitoring. |
| `tenant-lifecycle` | Tenant lifecycle management. |

**ClickUp integration:**

| Function | Purpose |
|---|---|
| `sync-clickup-tasks` / `sync-clickup-time` | ClickUp sync. |
| `fetch-clickup-comments` / `import-clickup-csv` | ClickUp data import. |
| `clickup-ai-search` | AI-powered ClickUp search. |

**Other:**

| Function | Purpose |
|---|---|
| `notify-chat` | Outbound chat/webhook notifications. |
| `generate-notifications` / `process-notification-outbox` | Notification pipeline. |
| `import-unicorn1-client` / `lookup-unicorn1-client` / `search-unicorn1-users` | Unicorn 1.0 migration. |
| `run-retention-forecast` / `run-workload-forecast` / `run-workflow-optimisation` | Operational forecasting. |
| `risk-command-engine` / `scan-risk-radar` | Risk management. |
| `extract-note-title` / `extract-suggest-title` | AI-assisted title extraction for notes and suggestions. |
| `verify-compliance-folder` | Verify compliance folder structure (SharePoint-adjacent). |
| `dashboard-test-seed` | Test data seeding. |
| `_shared/*` | Shared utilities (CORS, auth helpers, etc.). |

---

## Multi-tenancy

Every tenant-scoped row carries `tenant_id: int`. Tenant `6372` is Vivacity (staff consultancy); every other tenant is a client RTO.

**Role split:**
- Vivacity tenant → roles `Super Admin`, `Team Leader`, `Team Member`
- Client tenants → roles `Admin`, `User`

**Helper functions (Postgres):**
- `is_vivacity()` — current user belongs to tenant 6372 (used in RLS policies)
- `is_superadmin()` — current user's `unicorn_role = 'Super Admin'`
- `current_tenant()` — user's active tenant
- `set_active_tenant(tenant_id)` — Vivacity Super Admins can switch context
- `fn_package_stream(p_package_id)` (added 2026-04-23, migration `20260423093423_…`) — derives a regulatory-stream tag (`'rto' | 'cricos' | 'gto' | 'generic'`) from a package's name/slug. Used by the updated `start_client_package` RPC to enforce a duplicate-stream guard.

**Staff-only Academy admin RPCs (added 2026-04-21, all gated by `is_vivacity()`):**
- `fn_academy_enrollment_stats()` → `jsonb` of six dashboard tiles (total/active/completed/expired/revoked/auto_lifetime).
- `fn_academy_enrollment_lesson_detail(p_enrollment_id bigint)` → per-enrolment lesson progress rows.
- `fn_academy_rule_dashboard_stats()` → package-rule dashboard counts (active rules / total mappings / auto-enrolments / unmapped packages).
- Migrations: `20260421082533_da37ce62-…sql`, `20260421085406_b2a157f8-…sql`. Surfaced in [src/pages/superadmin/AcademyEnrolmentsPage.tsx](../src/pages/superadmin/AcademyEnrolmentsPage.tsx) via [src/hooks/academy/useAcademyEnrollments.ts](../src/hooks/academy/useAcademyEnrollments.ts).

**Edge function auth note:** Edge functions do NOT call `is_vivacity()` directly. They check the `is_vivacity_internal` boolean column on the `users` table (e.g. `.select('is_vivacity_internal')`), or use the `is_vivacity_staff` RPC (`supabase.rpc('is_vivacity_staff', {p_user: userId})`). The `has_tenant_access_safe` RPC handles tenant membership checks in shared addin auth (`_shared/addin-auth.ts`). The canonical pattern in `02-system-design.md` covers the simplified form; real functions vary.

**RLS conventions (see [02-system-design.md](02-system-design.md#rls) for full rationale and checklist):**
1. Every new table with mixed staff + client access needs **two policies**:
   - Tenant-read SELECT via tenant membership check (client RTO users)
   - Staff ALL via `is_vivacity()` (Vivacity consultants)
2. Then `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` — **explicitly**. Writing policies without enabling is a silent failure.

---

## Auth

- Supabase Auth with email/password primary.
- Invitation token flow: `user_invitations` table holds SHA256-hashed tokens; [src/pages/AcceptInvitation.tsx](../src/pages/AcceptInvitation.tsx) validates, creates auth user, upserts `tenant_members`.
- Password reset via Supabase's native recovery flow + [src/pages/ResetPassword.tsx](../src/pages/ResetPassword.tsx).
- Magic link — email template exists (`supabase/email-templates/magic-link.html`) but no dedicated frontend handler. Either unused or relies on Supabase default.
- Session state lives in [src/hooks/useAuth.tsx](../src/hooks/useAuth.tsx) — `AuthProvider` wraps the router, exposes `{ user, session, profile, loading, signOut, refreshProfile }`.
- Protected routes via [src/components/ProtectedRoute.tsx](../src/components/ProtectedRoute.tsx).

Diagnostics reference: [docs/INVITE_USER_DIAGNOSTICS.md](../docs/INVITE_USER_DIAGNOSTICS.md) — error codes + SMTP verification.

---

## Frontend (src/)

- **Pages** (`src/pages/`, ~247 files) — 180 top-level + 67 in subdirectories (`academy/`, `addin/`, `admin/`, `client/`, `internal/`, `superadmin/`, `teams/`). Many pages have `*Wrapper` variants that handle layout/auth boilerplate around the core page. When in doubt, the `Wrapper` is what `App.tsx` mounts.
- **Components** (`src/components/`):
  - `ui/` — shadcn-ui primitives
  - `layout/` — nav, sidebar, header
  - `admin/` — user/tenant admin tables & dialogs
  - `eos/` — the EOS module (largest subtree): rocks, issues, todos, scorecard, V/TO editor, meeting controls, live meeting view, QC scheduler, accountability chart. Subdirs `client/` (client-facing) and `qc/` (quarterly conversations).
  - `audit/` — audit inspection dialogs and workspace pieces
  - `dashboard/` — stats, charts
  - `profile/`, `tenant/`
- **Contexts** (`src/contexts/ViewModeContext.tsx`) — view-mode switcher (e.g., admin vs. client view).
- **Hooks** (`src/hooks/`) — ~230 top-level items + academy/ subdirectory (~9 hooks), ≈238 total. Domain clusters include EOS, Audits, Academy (`src/hooks/academy/`), plus `useAuth`, `useMobile`, `useToast`, `useAISuggestions`, `useNotifications`, `useDashboardData`, and more.
- **Integrations** (`src/integrations/supabase/client.ts`) — the Supabase JS client singleton.
- **Types** (`src/types/audit.ts`, `eos.ts`, `qc.ts`) — domain types.

---

## Data layer conventions

- **Reads** — `@tanstack/react-query` via `useQuery`. `QueryClient` is instantiated once at the top of `App.tsx` and never re-created.
- **Mutations** — `useMutation`; on success, invalidate relevant query keys.
- **Real-time** — Supabase channels, typically inside a domain hook (see [useMeetingRealtime](../src/hooks/useMeetingRealtime.ts) for the live-meeting example).
- **Forms** — `react-hook-form` + `zod` schemas. Validation at the form layer, RLS at the DB layer.

Full patterns in [02-system-design.md](02-system-design.md).

---

## Storage

Storage buckets are tenant-scoped with path-based policies (convention: `tenant-{id}/...` prefix). Current buckets (inferred from UI surfaces — confirm with RJ before assuming exact names):
- Tenant documents
- Audit evidence / references / reports
- User avatars

Historical Vivacity docs list five audit-specific storage buckets by name — those belong to the sibling Supabase project, not this one.

---

## Automation layer

| Type | Current state |
|---|---|
| Email automation | Mailgun + Microsoft Graph (`send-email-graph`, `send-composed-email`, `send-stage-email`). Auth emails via Supabase. |
| Outlook calendar | ✅ Live via `sync-outlook-calendar`, `outlook-auth`, addin functions. |
| SharePoint | ✅ Live via multiple SharePoint edge functions (browse, import, link, provision). |
| ClickUp | ✅ Live via `sync-clickup-tasks`, `sync-clickup-time`, `import-clickup-csv`. |
| Scheduled jobs (pg_cron) | ✅ Deployed. `pg_cron` extension enabled in migration `20260209232822`. One scheduled job confirmed: `seed-compliance-tasks-nightly` runs `run_seed_compliance_tasks_job()` at `0 2 * * *` (2am daily). |
| Stripe | Not wired. No subscription tables, no webhook handlers confirmed. |

---

## Constraints (hard rules)

1. **AI logic → server-side only** (Edge Functions or n8n). Never frontend. API keys must not leak.
2. **New tables → three-step RLS ritual** (tenant-read SELECT + staff ALL + `ENABLE ROW LEVEL SECURITY`). Omitting any step is a silent failure.
3. **Lovable → UI layer only.** No schema decisions, no business logic. If Lovable scaffolds a table, remove it before migration.
4. **Service-role edge functions validate the caller manually.** Always `supabase.auth.getUser(callerToken)` then check `unicorn_role` + `tenant_id`.
5. **NOT NULL columns with frontend writes → add a coercion trigger.** Column defaults do not protect against explicit `NULL` values sent by Lovable-generated forms.
6. **Tenant 319 is Vivacity** — hardcoded constant in `invite-user` and checked throughout. If this changes, grep `319` across the repo.
7. **Multi-tenant defaults.** Every domain query should filter by `tenant_id`. Cross-tenant access is Vivacity-staff-only and goes through `is_vivacity()`.

---

## Open architectural questions

Track these in [05-product-decisions.md → Open Decisions](05-product-decisions.md#open-decisions). Highlights:

- **Stripe webhooks** — Edge Function or n8n? Subscriptions still not wired.
- **LLM providers (resolved):** Primary gateway: `https://ai.gateway.lovable.dev/v1/chat/completions` with `LOVABLE_API_KEY`, serving `google/gemini-2.5-flash` and `google/gemini-3-flash-preview` models via Lovable's AI proxy. Secondary: Direct OpenAI `gpt-4o-mini` in `assistant-answer` with `OPENAI_API_KEY`. Prompt ownership and orchestration routing are still undocumented — flag to RJ.
- **Lovable environments** — dev / staging / prod separation is not documented.
- **AI orchestrator scope** — `ai-orchestrator` is a central hub; how is routing and model selection configured?

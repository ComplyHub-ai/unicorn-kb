# Module Status

> **Last updated:** 2026-04-22 · **Reconsider by:** 2026-05-22 · **Confidence:** medium — module presence confirmed by files/routes; "shipped" vs "partial" calls need RJ confirmation for several modules.
>
> Live tracker for each module. Grounded in the actual `unicorn-cms-f09c59e5` codebase (April 2026). Many features previously flagged as "historical reference only" are now present in this codebase.

---

## Status key

| Symbol | Meaning |
|---|---|
| 🔲 | Not started |
| 🟡 | In progress / partial |
| 🔵 | In review |
| ✅ | Shipped |
| 🚫 | Blocked |

---

## 1. Auth & User Management

**Status:** ✅ Shipped — active stabilisation

**What exists:**
- Email/password login ([src/pages/Login.tsx](../src/pages/Login.tsx))
- Invitation flow ([src/pages/AcceptInvitation.tsx](../src/pages/AcceptInvitation.tsx), [supabase/functions/invite-user/](../supabase/functions/invite-user/))
- Password reset + Super Admin force-change (`send-password-reset`, `send-self-password-reset` edge functions confirmed. ~~`admin-change-password`~~ **not found in `supabase/functions/` — verify with RJ whether removed or renamed**)
- User management UI (`/manage-users`, `/manage-invites`)
- Role matrix enforcement (Vivacity vs. client roles)
- Audit of Super Admin actions (inferred from recent commits)

**Recent activity (last 2 weeks):** Nearly all commits. Super Admin password change, auth user check for reset, password reset validation.

**Remaining:**
- Magic link flow — template exists but no dedicated route. Decide: ship or delete template.
- SSO / OAuth providers — not started, not confirmed in scope.

---

## 2. Tenants & Multi-tenancy

**Status:** ✅ Shipped

**What exists:**
- `tenants`, `tenant_members`, `user_invitations`, `tenant_settings` schema ([sql-setup/01-tenant-schema.sql](../sql-setup/01-tenant-schema.sql))
- Helper functions `is_vivacity()`, `is_superadmin()`, `current_tenant()`, `set_active_tenant()` ([sql-setup/02-tenant-functions.sql](../sql-setup/02-tenant-functions.sql))
- RLS policies ([sql-setup/03-tenant-policies.sql](../sql-setup/03-tenant-policies.sql))
- Admin tenant management (`/manage-tenants`, `/tenant/:id/*`)
- Tenant switcher for Vivacity staff

**Remaining:**
- Parent-child org relationships — mentioned in old docs as live; confirm equivalent in this project.

---

## 3. EOS Level 10 Meeting Module

**Status:** ✅ Shipped — the flagship feature

**What exists:**
- V/TO editor (`/eos/vto`)
- Rocks — quarterly priorities, client/team/company-tagged (`/eos/rocks`)
- Issues — IDS workflow (`/eos/issues`)
- To-Dos (`/eos/todos`)
- Scorecard — weekly metrics (`/eos/scorecard`)
- Meeting scheduler + calendar (`/eos/meetings`, `/eos/calendar`)
- Live meeting view — real-time multi-participant sync ([src/components/eos/LiveMeetingView.tsx](../src/components/eos/LiveMeetingView.tsx))
- Meeting summary (`/eos/meetings/:id/summary`)
- Quarterly Conversations (`/eos/qc`, `/eos/qc/:id`)
- Client view (`/client/eos`) — filtered to client-tagged items
- Accountability Chart (inferred from spec, components present in `src/components/eos/`)
- Hooks: `useEos`, `useEosAgendaTemplates`, `useEosDrafts`, `useEosHeadlines`, `useEosMeetingRecurrences`, `useEosMeetingSegments`, `useEosScorecardEntries`, `useEosScorecardMetrics`, `useMeetingIssues`, `useMeetingRealtime`, `useMeetingTodos`, `useQuarterlyConversations`

**Spec:** [docs/EOS_LEVEL10_SPECIFICATION.md](../docs/EOS_LEVEL10_SPECIFICATION.md)

**Remaining:**
- Tablet / facilitator focus mode (if planned)
- Meeting recording / transcription (if planned — AI suggestion hooks are present but not wired)
- Meeting rating analytics over time

---

## 4. Audits & Assessments

**Status:** 🟡 Shipped, simpler than old-doc version

**What exists:**
- Audit templates ([sql-setup/05-audit-schema.sql](../sql-setup/05-audit-schema.sql), `/audits/create-template`)
- Question bank seed (395 questions, 91 sections, 5 templates — [sql-setup/08-audit-question-bank-seed.sql](../sql-setup/08-audit-question-bank-seed.sql))
- Audit workspace (`/audits/:id`)
- Findings (`/audits/:id/findings`)
- Corrective actions (`/audits/:id/actions`)
- Report view (`/audits/:id/report`)
- RLS hardening ([sql-setup/06-audit-rls-policies.sql](../sql-setup/06-audit-rls-policies.sql))
- RPC functions ([sql-setup/07-audit-rpc-functions.sql](../sql-setup/07-audit-rpc-functions.sql))

**Now present in this codebase:**
- `analyze-document` edge function — ✅ exists
- `research-audit-intelligence` edge function — ✅ exists
- `research-evidence-gap-check` / `research-template-gap-analysis` — ✅ exist
- `scan-document` / `chunk-document` pipeline — ✅ exists
- New page: `AuditWorkspaceNew.tsx` — may indicate the three-phase lifecycle is in progress

**Still not confirmed in codebase:**
- `generate-audit-report` — not found; confirm with RJ
- Three-phase workflow (Preparation / Conducting / Closing) with auto-completion — verify
- `audit_appointments` scheduling table — verify in DB

**Remaining:**
- AI report generation — verify scope
- Three-phase lifecycle — verify scope
- Client portal audit views — partial; confirm scope

---

## 5. Packages / Pipeline Stages

**Status:** ✅ Shipped

**What exists:**
- `packages`, `package_stages`, `package_stage_instances` schema
- Admin view (`/admin/manage-packages`, `/admin/package/:id`, `/admin/package/:id/tenant/:tenantId`)
- Tenant view (`/manage-packages`, `/package/:id`)
- Stages management (`/manage-stages`)
- `add-missing-packages` edge function — ensures default packages per tenant

**Remaining:**
- Pipeline automation / auto-stage-moves — old docs reference DB-trigger-based stage completion. Not yet evident here. Clarify scope.

---

## 6. Client Portal

**Status:** 🟡 Partial

**What exists:**
- `/client/eos` — client-facing EOS view
- Tenant documents (`/tenant/:tenantId/documents`, `/tenant/:tenantId/document/:documentId`)
- Tenant-scoped notes (`/tenant/:tenantId/notes`)
- Client task views (`/tenant/:tenantId/tasks`)

**Remaining:**
- Dedicated client-portal surface (vs. reusing admin routes with filters) — architectural clarification needed.
- Audit portal views for clients (reports, action plan, prep checklist) — old docs list as live; not visibly scoped in current code.

---

## 7. Documents

**Status:** 🟡 Partial

**What exists:**
- Document management UI (`/manage-documents`, `/documents`)
- Document detail view (`/document/:id`)
- Tenant documents
- Categories (`/manage-categories`)
- Fields (`/manage-fields`) — custom metadata

**Remaining:**
- Version history / tracking (if planned)
- Document templates vs. instances distinction

---

## 8. Tasks

**Status:** ✅ Shipped

**What exists:**
- Global tasks (`/tasks`)
- Tenant-scoped tasks (`/tenant/:tenantId/tasks`)
- Task management wrapper

**Remaining:**
- Automation tying tasks to pipeline stages (old doc feature — "stage task auto-completion") — not evident here.

---

## 9. Campaigns / Email

**Status:** 🟡 Partial

**What exists:**
- Admin email template management (`/admin/manage-emails`)
- Mailgun templates in [templates/mailgun/](../templates/mailgun/) and [supabase/email-templates/](../supabase/email-templates/)
- Transactional email via `send-invitation-email` edge function
- Auth emails (invite, confirm-signup, password-reset, magic-link, reauth, email-change)

**Also present in this codebase:**
- `send-email-graph` and `send-composed-email` via Microsoft Graph (not Mailgun)
- `send-stage-email` — stage-triggered emails
- `generate-notifications` + `process-notification-outbox` — notification pipeline

**Not yet confirmed:**
- `send-automated-email` multi-handler dispatcher
- Broadcast / campaign builder UI
- Segmentation

**Remaining:**
- Confirm whether campaign engine is in scope for 2.0 or deferred.

---

## 10. RTO Tips

**Status:** ✅ Shipped

**What exists:**
- `/rto-tips` — [src/pages/RtoTips.tsx](../src/pages/RtoTips.tsx) / Wrapper
- Hook: [src/hooks/useRtoTips.ts](../src/hooks/useRtoTips.ts)

**Purpose:** Domain knowledge / quick-reference content for RTO clients.

---

## 11. Settings

**Status:** ✅ Shipped

**What exists:**
- General settings (`/settings`)
- Team settings (`/team-settings`)
- Notification settings (`/settings/notifications`)
- Integration settings (`/settings/integrations`)

---

## 12. Subscription / Membership (Stripe)

**Status:** 🔲 Not started

**No evidence in code:** no Stripe imports, no webhook handlers, no subscription schema. The old doc's "Superhero" and "Academy" membership tiers are aspirational, not present.

**Blockers:** Product decision on Stripe webhook handling (Edge Function vs. n8n), self-service portal scope.

---

## 13. Booking / Scheduling

**Status:** 🟡 EOS calendar only

**What exists:**
- EOS calendar (`/eos/calendar`) — meeting scheduling within the EOS module
- Meeting scheduler component
- QC scheduler component
- `generate-meeting-recurrence` edge function

**Now present:**
- Outlook calendar sync (`sync-outlook-calendar`) — ✅ exists
- `CalendarTimeCapture.tsx` / `TimeInbox.tsx` / `WorkCalendar.tsx` pages — ✅ exist (time tracking angle)
- `OutlookCallback.tsx` — Outlook OAuth callback

**Not confirmed:**
- General-purpose booking (OnceHub replacement for non-EOS contexts)
- Google calendar integration

---

## 14. AI / Automation

**Status:** 🟡 Partial

**What exists:**
- `ai-generate-suggestions` edge function ([supabase/functions/ai-generate-suggestions/](../supabase/functions/ai-generate-suggestions/))
- `useAISuggestions` hook

**Now present in this codebase:**
- `analyze-document` — ✅ exists
- `research-audit-intelligence` — ✅ exists
- `ai-orchestrator` — central AI routing hub — ✅ exists
- `compliance-assistant`, `client-ai-companion`, `copilot-chat`, `assistant-answer`, `help-center-chat` — ✅ all exist
- `research-answer`, `research-scrape`, `research-enrich-tenant`, `research-evidence-gap-check` — ✅ exist
- `calculate-predictive-risk`, `run-tenant-risk-forecast`, `run-strategic-signal-analysis` — ✅ exist
- `vector-search`, `vector-index-rebuild`, `query-knowledge-graph` — ✅ vector/knowledge layer exists

**Not confirmed:**
- Six named AI agents (Alex, Casey, Morgan, Jordan, Riley, Sam) — check if these map to existing functions
- `generate-audit-report` — not found

**Unresolved:**
- LLM provider(s) and model configuration across the AI functions — undocumented. Grep function sources.

---

## 15. Integrations

| Integration | Status |
|---|---|
| Mailgun | ✅ Shipped (invites, auth emails) |
| Microsoft Graph email | ✅ Shipped (`send-email-graph`, `send-composed-email`, `send-stage-email`) |
| training.gov.au / TGA | ✅ Shipped (`search-organisations`, `get-organisation-details`, `tga-sync`, `tga-rto-import`, etc.) — see [docs/training-gov-au-integration.md](../docs/training-gov-au-integration.md) |
| Outlook calendar | ✅ Shipped (`sync-outlook-calendar`, `outlook-auth`, addin functions) |
| SharePoint | ✅ Shipped (browse, import, link, provision, deliver functions) |
| ClickUp | ✅ Shipped (`sync-clickup-tasks`, `sync-clickup-time`, `import-clickup-csv`) |
| Microsoft 365 / M365 | ✅ Shipped (`provision-m365-user`, Outlook addin suite) |
| Stripe | 🔲 Not started |
| n8n / Awesomate | 🔲 Not confirmed in this codebase |

---

## 16. Academy

**Status:** ✅ Shipped

**What exists:**
- Admin surface: `/academy`, `/academy/courses`, `/academy/certificates`, `/academy/events`, `/academy/community`, `/academy/team`, `/academy/settings`
- Role-specific client views: Trainer, Compliance Manager, Governance Person, Student Support Officer, Administration Assistant
- Course detail + lesson viewer (`/academy/course/:slug/*`), assessment player, assessment results
- Superadmin builder: `/superadmin/academy/builder`, `builder/:courseId`, enrollments, tenant access, package-course rules
- Components (`src/components/academy/`): `AcademyAccessGate`, `CourseCard`, `SeatLimitBanner`, `EnrolmentProgressDrawer`, builder tools, layout shell
- Hooks (`src/hooks/academy/`): `useAcademyCourses`, `useAcademyEnrollments`, `useAcademyCertificates`, `useAcademyPackageRules`, `useTenantAcademyAccess`, `useVideoLibrary`, builder hooks
- Edge function: `academy-ai-generate`
- Total: ~30 page/wrapper files, 22 components, 9+ hooks

**Remaining:**
- No `docs/` spec — verify course content schema and role-gate logic with RJ
- Seat access gating (`AcademyAccessGate`, `SeatLimitBanner`) — confirm enforcement is RLS-backed, not frontend-only
- Stripe for seat-based billing — not wired

---

## 17. Resource Hub

**Status:** ✅ Shipped

**What exists:**
- Staff-facing hub (`/resource-hub`) with 11 category routes: audit-evidence, checklists, CI tools, favourites, guides-howto, most-used, recently-added, registers-forms, templates, training-webinars, updates
- Client-facing view (`/client/resource-hub`, `/client/resource-hub/:categoryId`)
- Pages: `ResourceHubDashboard`, `ResourceCategoryPage`, `ResourceRecentlyAdded`, `ResourceMostUsed`, `ResourceFavourites`, `ResourceUpdatesLog`, `ClientResourceHubPage`
- Components: `ResourceCard`, `ResourceGrid`, `ResourceSearch`
- Schema managed in timestamped migrations (no separate sql-setup folder — earlier KB reference to `sql-setup/09–12` is stale)

**Remaining:**
- No dedicated hook file found — confirm data fetching pattern with RJ
- Content upload/management workflow for new resources not confirmed

---

## Summary matrix

| Module | Current status | Delta from earlier survey |
|---|---|---|
| Auth & Users | ✅ | + bulk actions, resend/cancel invite, M365 provisioning |
| Multi-tenancy | ✅ | Same conceptual model |
| EOS Level 10 | ✅ | Flagship feature — fully shipped |
| Audits | 🟡 | + `analyze-document`, `research-audit-intelligence` now present; `generate-audit-report` still missing |
| Packages / Pipeline | ✅ | `run-stage-health-monitor`, `calculate-phase-completeness` added |
| Client Portal | 🟡 | Many new client pages (`ClientDetail`, `ClientImpact`, `ClientInbox`, `ClientCommunications`) |
| Documents | 🟡 | + AI analysis, versions, categories, scan pipeline |
| Tasks | ✅ | Same scope |
| Campaigns/Email | 🟡 | + Microsoft Graph email; campaign builder still open |
| Subscriptions/Membership | 🔲 | `MembershipDashboard.tsx` manages tenant memberships/roles — NOT Stripe. Stripe is 🔲 Not started. |
| Booking | 🟡 | + Outlook calendar sync now live; time capture pages added |
| AI automation | ✅ | 40+ AI functions; orchestrator, vector layer, research, compliance AI |
| SharePoint | ✅ | Full SharePoint suite of edge functions |
| Outlook / M365 | ✅ | Calendar sync, addin suite, M365 provisioning |
| ClickUp | ✅ | Task + time sync |
| Academy | ✅ | Fully shipped — ~30 pages, 22 components, 9 hooks, role-specific client views, superadmin builder |
| Resource Hub | ✅ | Shipped — 12 staff + client routes, categorised library |

See [07-migration-map.md](07-migration-map.md) for what came from Unicorn 1.0 and what's net-new.

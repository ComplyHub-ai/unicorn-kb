# Start Here — Unicorn 2.0

> **Last updated:** 2026-04-27 · **Reconsider by:** 2026-10-27 · **Confidence:** medium-high — tenant ID verified in codebase; team-roles section rewritten 2026-04-27 in seat-centric restructure; ground rules updated 2026-04-27 to reflect operating-model ADR-011.
>
> Shared orientation for the Unicorn 2.0 engineering team. Product overview, mental model, ground rules, and key landmarks. Read this before diving into any other KB file.

---

## What is Unicorn 2.0?

**One-line:** A multi-tenant React + Supabase platform that runs Vivacity's compliance-consulting business end-to-end — clients, pipeline, weekly EOS meetings, quarterly conversations, RTO compliance audits.

**Scope of this KB.** Vivacity has two products: Unicorn (this codebase) and ComplyHub (separate; not covered here). The GitHub org name `ComplyHub-ai` reflects the company, not a product-specific scope. If you're looking for ComplyHub docs, this isn't the place.

**Who uses it:**
- **Vivacity staff** (tenant ID `6372`) — consultants working across many client RTOs. Roles: Super Admin, Team Leader, Team Member.
- **Client RTOs** (every other tenant) — Australian Registered Training Organisations consuming compliance services. Roles: Admin, User.

**Who it replaces:**
- Unicorn 1.0 / legacy Vivacity CMS (internal CRM)
- Keap (contacts, pipeline, campaign automation)
- OnceHub (booking — partially superseded, see [migration-1to2.md](../reference/migration-1to2.md))
- Ad-hoc spreadsheets for EOS meeting tracking

**What is novel about 2.0:** EOS Level 10 Meeting methodology is a **first-class product surface**, not a bolt-on. V/TO, Rocks, Scorecard, Issues (IDS), To-Dos, Accountability Chart, live real-time meetings, Quarterly Conversations (QC) — all live. See [docs/EOS_LEVEL10_SPECIFICATION.md](../docs/EOS_LEVEL10_SPECIFICATION.md) for the authoritative product spec.

---

## Stack in one glance

| Layer | Tech |
|---|---|
| Frontend | React 18 + Vite + TypeScript + shadcn-ui + Tailwind (built in Lovable) |
| Data fetching | `@tanstack/react-query` (global `QueryClientProvider` in [src/App.tsx](../src/App.tsx)) |
| Forms | `react-hook-form` + `zod` schema validation |
| Animations / DnD | `framer-motion`, `@dnd-kit/*` |
| Backend | Supabase — Postgres + RLS + Auth + Storage + Edge Functions + Realtime |
| Email | Mailgun via edge functions (templates in [templates/mailgun/](../templates/mailgun/) and [supabase/email-templates/](../supabase/email-templates/)) |
| Automation | n8n now, migrating to Awesomate (Australia-hosted). Stripe not yet wired. |

---

## Mental model

Three concepts unlock 80% of the codebase:

### 1. Tenancy
Every business-domain row has a `tenant_id`. Tenant `6372` is Vivacity — staff. All other tenants are client RTOs. A user belongs to one tenant via `tenant_members`; Vivacity Super Admins can switch active tenant via `set_active_tenant()`.

### 2. Two distinct UI surfaces
- **Admin / consultant surface** — Vivacity staff views (`/manage-*`, `/tenant/:id/*`, `/admin/*`, `/eos/*`, `/audits/*`)
- **Client surface** — client RTO views (`/client/eos`, tenant-scoped documents, their own tasks)

They share components but render different data based on `unicorn_role` + `tenant_id`. See [flow-patterns.md](../reference/flow-patterns.md#role-aware-rendering).

### 3. RLS is the security boundary
All tenant isolation happens at the database layer via Postgres RLS. The frontend never enforces access control — it reads what RLS lets it read. This is non-negotiable. See [conventions.md](conventions.md#rls) and [decision-trail.md](../reference/decision-trail.md#adr-005) for the failure modes that make this rule hard.

Helper functions (Postgres):
- `is_vivacity()` → true if current user belongs to tenant 6372
- `is_superadmin()` → true if current user's `unicorn_role = 'Super Admin'`
- `current_tenant()` → the user's active tenant for this session

> **Note:** Edge functions do NOT call `is_vivacity()` directly — they check the `is_vivacity_internal` boolean column on the `users` table, or the `is_vivacity_staff` RPC. The `is_vivacity()` function is used in RLS policies only.

---

## Where to look first

| If you're looking for… | Go to… |
|---|---|
| Entry point / routes | [src/App.tsx](../src/App.tsx) |
| Auth state | [src/hooks/useAuth.tsx](../src/hooks/useAuth.tsx) |
| Supabase client | [src/integrations/supabase/client.ts](../src/integrations/supabase/client.ts) |
| Edge functions | [supabase/functions/](../supabase/functions/) |
| Schema seeds | [sql-setup/](../sql-setup/) |
| Migrations | [supabase/migrations/](../supabase/migrations/) |
| EOS spec | [docs/EOS_LEVEL10_SPECIFICATION.md](../docs/EOS_LEVEL10_SPECIFICATION.md) |
| Invite error codes | [docs/INVITE_USER_DIAGNOSTICS.md](../docs/INVITE_USER_DIAGNOSTICS.md) |

Full navigational map: [codebase-map.md](../codebase-state/codebase-map.md).

---

## Ground rules (non-negotiable)

1. **AI logic → server only.** Never in the frontend. Edge Functions or n8n.
2. **New tables require the three-step RLS ritual.** Tenant-read SELECT policy + staff ALL policy + `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`. Skipping any one is a silent failure. See [conventions.md](conventions.md#new-table-checklist).
3. **Lovable owns schema in practice.** Lovable users (Angela, Dave, Khian — see `team-roles.md`) push schema migrations alongside frontend code direct to `main`. There is no pre-merge review or sign-off gate. The three-step RLS ritual (item 2 above) is the *correct* technical pattern, but enforcement is post-merge — see `reference/decision-trail.md → ADR-011` for the operating-model rationale and risk acceptance. Hand-written code via Claude Code lands via PR on a feature branch (also without mandatory review).
4. **Edge functions validate the caller manually.** Always `supabase.auth.getUser(callerToken)` then check `unicorn_role` / `tenant_id`. The service-role key bypasses RLS, so the function is the only enforcement point.
5. **Column defaults are not enough.** For NOT NULL columns the frontend may explicitly set to `NULL`, add a coercion trigger. See [decision-trail.md](../reference/decision-trail.md#adr-008).

---

## Who's who

The team is organised around four **seats** (responsibilities), not titles. The same person may sit in multiple seats. See `pinned/team-roles.md` for full seat definitions, tool access, and handoff routing.

| Person | Seats |
|---|---|
| Angela | Product owner + Dev |
| Carl | Project lead + KB owner + Dev |
| Dave | Dev |
| RJ | Dev (works mostly on ComplyHub) |
| Khian (Brian) | Dev — junior, growing scope |

**Seat summary:**
- **Product owner** (Angela) — owns Unicorn the product; final say on what ships.
- **Project lead + KB owner** (Carl) — engineering direction; KB stewardship; sole audit-repo access; Lovable remix trigger.
- **Dev** — feature implementation, almost entirely through Lovable; no peer review or sign-off gates today.
- **Non-technical KB reader** (no current named occupants) — read-only seat for new hires before access, external reviewers, deep-reading internals.

---

## What's live (April 2026) — `unicorn-cms-f09c59e5`

**EOS Level 10 Meeting module** — The flagship. Implements EOS methodology end-to-end: V/TO, Rocks, Scorecard, Issues (IDS), To-Dos, live multi-participant Level 10 Meetings, Quarterly Conversations, and a filtered client-facing view. See [module-status.md#3-eos-level-10-meeting-module](../codebase-state/module-status.md#3-eos-level-10-meeting-module) for build status.

**Audits module** — Runs Vivacity's compliance audit engagements (CHC, Mock, CRICOS, Due Diligence) against a 395-question bank; staff log findings, assign corrective actions, and generate reports with AI-assisted document analysis. See [module-status.md#4-audits--assessments](../codebase-state/module-status.md#4-audits--assessments) for build status.

**Multi-tenant user management** — Invite, role-assign, and manage users within any tenant. Covers the full invite-to-onboard flow, Super Admin password resets, and bulk actions across Vivacity and client roles. See [module-status.md#1-auth--user-management](../codebase-state/module-status.md#1-auth--user-management) for build status.

**Tenant / client management** — Vivacity staff workspace for each client RTO: details, team members, documents, notes, tasks, engagement timeline, and impact metrics in one place. See [module-status.md#2-tenants--multi-tenancy](../codebase-state/module-status.md#2-tenants--multi-tenancy) for build status.

**Package / pipeline stages** — Models Vivacity's service offerings as Packages with ordered stages; tracks each client's stage-instance progress with a health monitor and phase-completeness calculator. See [module-status.md#5-packages--pipeline-stages](../codebase-state/module-status.md#5-packages--pipeline-stages) for build status.

**Outlook calendar sync + addin suite** — Syncs EOS meeting events to Microsoft Outlook; the addin functions let staff capture emails, log meeting notes, and draft time entries without leaving Outlook. See [module-status.md#15-integrations](../codebase-state/module-status.md#15-integrations) for build status.

**SharePoint integration** — Browse a client's SharePoint, import or link documents into Unicorn, and provision structured SharePoint folders for new clients. See [module-status.md#15-integrations](../codebase-state/module-status.md#15-integrations) for build status.

**Microsoft 365 / Graph email** — Sends pipeline stage and consultant-composed emails via Microsoft Graph (bypassing Mailgun), and provisions M365 accounts for new client users. See [module-status.md#15-integrations](../codebase-state/module-status.md#15-integrations) for build status.

**ClickUp integration** — Bidirectional task and time sync between Unicorn and ClickUp, plus bulk CSV import for migrating ClickUp-heavy clients. See [module-status.md#15-integrations](../codebase-state/module-status.md#15-integrations) for build status.

**TGA / training.gov.au integration** — Queries the national training register to search, import, and keep RTO organisational records current — used at client onboarding and during compliance engagements. See [module-status.md#15-integrations](../codebase-state/module-status.md#15-integrations) for build status.

**AI layer** — 40+ server-side functions routed through a central `ai-orchestrator`: compliance assistant chat, document analysis, vector knowledge search, research intelligence, evidence gap checks, and predictive risk scoring. AI logic is server-only (edge functions). See [module-status.md#14-ai--automation](../codebase-state/module-status.md#14-ai--automation) for build status.

**Academy module** — A learning platform for client RTO staff. Delivers role-specific course views (Trainer, Compliance Manager, Governance Person, Student Support Officer, Administration Assistant), certificates, assessments, events, and community. Vivacity Super Admins build and manage content via a dedicated course builder. Seat access gating is implemented; Stripe billing is not yet wired. See [module-status.md#16-academy](../codebase-state/module-status.md#16-academy) for build status.

**Resource Hub** — A categorised library of compliance documents, templates, checklists, and guides for client RTOs. Browsable by category (audit evidence, CI tools, registers, templates, training resources, etc.) with search and favourites. Both staff and client portal surfaces are live. See [module-status.md#17-resource-hub](../codebase-state/module-status.md#17-resource-hub) for build status.

**RTO tips / Tasks / Settings / Notifications** — Four lightweight modules: RTO Tips serves quick-reference compliance content to client RTOs; Tasks provides global and tenant-scoped task lists; Settings covers user profile, team config, and integration credentials; Notifications runs an in-app alert outbox pipeline. See [module-status.md#10-rto-tips](../codebase-state/module-status.md#10-rto-tips), [#8-tasks](../codebase-state/module-status.md#8-tasks), [#11-settings](../codebase-state/module-status.md#11-settings).

## What's NOT live (don't assume it is)

- Stripe / subscriptions — not wired. `MembershipDashboard` exists but no Stripe. See [module-status.md](../codebase-state/module-status.md).
- `generate-audit-report` — not found in codebase; confirm with RJ
- The six named AI agents (Alex, Casey, Morgan, Jordan, Riley, Sam) — not confirmed as named entities (many AI functions exist but agent naming is unclear)

---

## Onboarding checklist (for new team members)

- [ ] Run the app locally. Login as a Vivacity Super Admin; hop into a client tenant via the switcher.
- [ ] Read [architecture.md](../codebase-state/architecture.md) end-to-end.
- [ ] Skim every file in [src/pages/](../src/pages/) — names map directly to product surfaces.
- [ ] Read one edge function front-to-back: [supabase/functions/invite-user/index.ts](../supabase/functions/invite-user/index.ts). It's the canonical service-role pattern.
- [ ] Read [docs/EOS_LEVEL10_SPECIFICATION.md](../docs/EOS_LEVEL10_SPECIFICATION.md). It's the core product thesis.
- [ ] Open [module-status.md](../codebase-state/module-status.md) and identify which modules are 🟡 partial — these are the highest-value areas to contribute to.
- [ ] Ask Carl which open decisions in [decisions.md](decisions.md#open-decisions) are actively blocking work.

---

## Standing questions to always ask

When a decision feels implicit or invisible, it probably *was* implicit. Surface it to Carl and capture it in [decisions.md](decisions.md). This codebase has a history of implicit decisions biting later (RLS incidents, NULL coercion, naming divergence). When in doubt, write it down.

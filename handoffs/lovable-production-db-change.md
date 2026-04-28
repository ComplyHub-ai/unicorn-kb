# Handoff: Lovable Production Database Change

> **Last updated:** 2026-04-28 · **Reconsider by:** 2026-10-28 · **Confidence:** high.
>
> Use this handoff when implementing a database change via Lovable that touches production schema, constraints, triggers, RLS policies, or data. Established from the staff UUID relink fix (28 April 2026) incorporating Dave's depth requirements.

---

## When to use this handoff

- Any Lovable prompt that generates a Supabase migration
- Changes to FK constraints, RLS policies, triggers, or enums
- Data backfill or repair operations
- Edge functions that write to production tables

Do **not** use for UI-only Lovable changes with no migration.

---

## The workflow

### Before you start

1. Identify the problem clearly — root cause, not just symptom
2. Gather the specific values needed (UUIDs, emails, constraint names, etc.) from Supabase SQL editor before opening Lovable
3. Check whether a similar fix has been done before — search `unicorn-audit/` and `unicorn-kb/codebase-state/`

---

### Prompt 1 — Audit (plan mode ON)

Ask Lovable to conduct a read-only deep dive and produce a written findings report. **No code yet.**

The audit must cover:
- Every FK constraint, trigger, RLS policy, and index touched by the proposed change
- Backward compatibility with existing application code and edge functions
- Lock impact per table (identify high-traffic tables; recommend `NOT VALID` + `VALIDATE CONSTRAINT` strategy)
- Side-effect triggers on cascaded tables
- Rollback plan for every step
- Any design decisions required before coding (surface these explicitly)

End the prompt with: *"Finish with a proposed implementation plan in numbered steps, a risk assessment, and a list of anything that needs a design decision before proceeding."*

Review the findings report with the relevant stakeholder before proceeding.

---

### Design decisions gate

Answer every open question the audit surfaces before moving to Prompt 2. Common decision types:
- Audit-log immutability (cascade historical rows vs. preserve old UUIDs)
- Backfill scope (known accounts only vs. full sweep)
- Dead code cleanup (now vs. deferred)
- Deployment window (off-peak hours for VALIDATE steps on large tables — use 22:00–04:00 AEST)

---

### Prompt 2 — Implementation plan (plan mode ON)

Reply to Lovable with your confirmed decisions and ask for the complete implementation plan before any code is written. The plan must include:

- Migration ordering and deploy schedule
- Per-migration lock impact and safe rollout strategy
- Rollback plan for every migration
- Verification steps (pre and post deploy)
- Off-peak scheduling for any VALIDATE CONSTRAINT steps

Review the plan. Only approve once you are satisfied.

---

### Prompts 3–N — Phased implementation (plan mode ON for each)

Break implementation into independent migrations. Never combine structural changes with data fixes in one prompt. Typical phases:

| Phase | What | Plan mode |
|---|---|---|
| Supporting objects | New tables, helper functions, audit tables | ON — review before implement |
| Structural change | FK constraint changes, trigger rewrites | ON — highest risk, most important to review |
| Data fix | Backfill, UUID corrections, sweep | ON — verify row counts before approving |
| Validation | VALIDATE CONSTRAINT loop (off-peak) | ON — confirm timing before approving |
| Verification | Smoke tests, dry runs, summary report | OFF — read-only |

**Always turn plan mode ON** before pasting a migration prompt. Review Lovable's proposed changes before approving implementation.

---

### Before any live data operation

1. Run `?dry_run=true` (or equivalent) first
2. Paste the dry-run output for review — confirm only the expected rows are affected
3. Run live only after dry-run is verified

---

### Final prompt — Verification and sign-off

Ask Lovable to produce a written summary covering:
- Every DB object changed and how
- What changed in system behaviour
- What stayed the same
- Benefits delivered
- Residual risk assessment with mitigations
- Confirmation that the fix is self-contained (no ongoing manual intervention required)

This is the sign-off document for stakeholder review.

---

## The Dave standard

When a stakeholder requests a thorough review, include this requirement in every prompt:

> *"Please take your time and conduct a thorough deep dive covering every nuance of the codebase and database structures this plan touches. Identify all gaps, improvements, shortcomings, issues, bugs, and conflicts. Ensure the implementation is backward-compatible, audit-complete, and production-ready. Confirm it does not negatively impact any existing functionality, RLS policies, or FK constraints. Finish with a summary of changes, benefits, and a risk assessment."*

---

## Off-peak deployment

VALIDATE CONSTRAINT steps on large tables must run in the off-peak window: **22:00–04:00 AEST**.

High-traffic tables that warrant off-peak VALIDATE:
`audit_log`, `notification_outbox`, `eos_todos`, `meetings`, `email_messages`, `ai_interaction_logs`, `consult_entries`, `ops_time_logs`

Use pg_cron for scheduling. Always set the job to self-unschedule after one run.

---

## After deploy

1. Run verification queries the morning after off-peak steps
2. Confirm affected users can log in / feature works end-to-end
3. Document the session in `unicorn-audit/` following the audit template
4. Update `unicorn-audit/INDEX.md`
5. Update any stale KB docs in `unicorn-kb/codebase-state/` if the migration changed documented architecture

---

## Reference

- First use: `unicorn-audit/audit/2026-04-28-staff-uuid-relink-fix.md`
- Audit template: `unicorn-audit/README.md`
- KB hygiene: `unicorn-kb/pinned/kb-hygiene.md`

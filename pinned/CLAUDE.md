# Unicorn Workspace — Root CLAUDE.md

## GitHub repos

| Alias | Org | Repo | Notes |
|-------|-----|------|-------|
| `unicorn-kb` | `ComplyHub-ai` | `unicorn-kb` | Team KB — this repo |
| `<codebase>` | `vivacityrto` | `unicorn-cms-f09c59e5` | Codebase — Lovable territory |
| `unicorn-audit` | _(local only)_ | _(not on GitHub)_ | Carl's audit trail |

Use these org/repo values for all GitHub MCP calls. No need to ask the user for them.

Post-remix: update the `<codebase>` row above (org stays `vivacityrto`; repo name changes to the new Lovable-generated name).


## Workspace layout

```
~/repository/unicorn-workspace/
├── CLAUDE.md                              ← this file
├── unicorn-audit/                         ← audit trail (Carl-authored for reconciliations/remixes; dev-authored for Lovable prod DB changes)
├── unicorn-kb/                            ← team KB
└── unicorn-cms-f09c59e5/                  ← codebase (Lovable territory)
```

## The `<codebase>` alias

`<codebase>/` currently resolves to: **`unicorn-cms-f09c59e5/`**

This file is the canonical source. All KB docs reference `<codebase>/` symbolically;
this file pins it to a concrete directory name.

Post-remix: update the `<codebase>` alias above to the new directory name.
No other files in `unicorn-kb/` need editing for the rename.

## Session start ritual (mandatory first action)

Before any other work, run in order:

1. `cd unicorn-kb && git pull --ff-only && cd ..`
2. `cd unicorn-audit && git pull --ff-only && cd ..`
3. `cd unicorn-cms-f09c59e5 && git fetch && cd ..`
4. Report:
   - Latest commit in `unicorn-kb`
   - Latest commit in `unicorn-audit`
   - Local HEAD vs `origin/HEAD` in `<codebase>` — **flag if they differ**; user decides whether to pull

If any pull fails (conflict, divergence, dirty working tree): **STOP and report. Do not attempt to resolve conflicts autonomously.**

## Write permissions (strict, non-negotiable)

| Repo | Permitted | Forbidden |
|------|-----------|-----------|
| `unicorn-kb/` | commit + push to feature branches | push to `main`, force-push, delete branches/tags |
| `unicorn-audit/` | commit + push to feature branches | push to `main`, force-push, delete branches/tags |
| `unicorn-cms-f09c59e5/` | `git fetch`; `git pull` only on explicit user request | **everything else — no commit, no push, no `git add`, no file edits** |

Feature branch naming:
- `unicorn-kb/`: `fix/<slug>`, `kb/<slug>`, `adr-<NNN>`, `restructure/<slug>`
- `unicorn-audit/`: `audit/YYYY-MM-DD-<slug>`, `remix/YYYY-MM-DD-<slug>`

**`<codebase>/` is Lovable's territory.** If asked to edit codebase files, refuse and offer either
(a) a Lovable prompt the user can run, or (b) a hotfix flag requiring explicit override.

## Before writing a Lovable prompt (mandatory for data-fetch changes)

Any prompt that touches a Supabase fetch, React Query hook, useState, or pagination MUST follow these steps. Skip none of them.

**1. Blast-radius check — run this first, before drafting the prompt**
Ask: what else in the file or codebase reads from or depends on the data this change touches? Identify every consumer (stat cards, filters, load bars, row counts, realtime handlers). If consumers exist, the prompt must account for all of them.

**2. Scope the prompt — include both of these**
- An explicit "do not touch" list naming every UI element that must produce the same result after the change.
- A dependency note for any consumer the blast-radius check found, so Lovable knows the relationship exists.

**3. Post-pull verification — check in the browser before marking done**
- Summary/stat cards show correct DB totals (not capped at page size)
- Table row counts match the stat cards
- Filters, realtime updates, and mutations still work correctly

Full checklist and regression trap reference: `unicorn-kb/reference/dev-guardrails.md`

**Why this is here:** ManageTenants pagination fix (April 2026) silently broke the four stat cards because they derived counts from `tenants.length`. Adding `.range(0, 99)` capped the array at 100 rows; cards showed 100 regardless of real DB total. Caught by a stakeholder, not the team.

Across ALL repos:
- Never push to `main`
- Never force-push
- Never delete branches or tags without explicit user confirmation
- Never amend commits that have been pushed
- Never `git reset --hard` except on a fresh feature branch you just created

## Session end ritual

For each repo touched (`unicorn-kb`, `unicorn-audit`):

1. Confirm branch is clean except for intended changes
2. Commit with conventional-commits message (`fix:`, `audit:`, `kb:`, `remix:`, etc.)
3. `git push -u origin <branch>`
4. Open PR via `gh pr create` (fallback: GitHub MCP). PR title = commit summary; PR description includes:
   - What changed
   - SHAs of any cross-repo references
   - Reviewer hint (RJ for RLS/conventions, Carl otherwise)
5. **Do NOT auto-merge** — stop after PR creation
6. For `unicorn-audit/` specifically: also `git push --tags` after PR creation
7. If `<codebase>/` has a dirty working tree at session end, flag it — do NOT stash, commit, or revert
8. Summarise what shipped: per repo — branch name, commit SHA, PR URL

If both `gh` CLI and GitHub MCP are unavailable: stop at push and report the branch URL.

## Single-repo sessions

Open Claude Code at the sub-repo directly (not workspace root). The sub-repo's own
`CLAUDE.md` governs. This file's session rituals apply only to cross-repo work.

## General rules

- Never edit across repos in a single commit
- Codebase-state docs live in `unicorn-kb/codebase-state/`, not in the codebase repo
- When the codebase directory name differs from what this file says, update this file first
- Flag past-shelf-life citations in `unicorn-kb/` (see `unicorn-kb/pinned/kb-hygiene.md`)

## Entry docs

| Path | Purpose |
|------|---------|
| `unicorn-kb/README.md` | KB orientation |
| `unicorn-kb/pinned/kb-hygiene.md` | KB policy |
| `unicorn-kb/handoffs/README.md` | Scenario procedures |
| `unicorn-kb/handoffs/post-lovable-remix.md` | What to do when `<codebase>` directory name changes |
| `unicorn-audit/README.md` | Audit repo — authorship split by session type; see Lovable production DB change sessions section in workspace root CLAUDE.md |
| `unicorn-audit/CLAUDE.md` | Audit-session rituals (takes precedence for unicorn-audit/ sessions) |

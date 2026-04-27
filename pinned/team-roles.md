# Team Roles & Tool Access

> **Last updated:** 2026-04-27 · **Reconsider by:** 2026-07-27 · **Confidence:** high — reflects the seat-centric model agreed in the 2026-04-27 restructure session.
>
> Vivacity has two products: Unicorn and ComplyHub. This KB covers Unicorn. Roles below describe **seats** (responsibilities), not people — the same person may sit in multiple seats. See "People → seat mapping" for who sits where today.

---

## Seats

The team is organised as four seats — three contributor, one consumer. Seats define responsibilities; people are mapped onto them below. The mental model is the EOS Accountability Chart from `pinned/glossary.md` — define seats, not people.

### 1. Product owner

**Purpose**: Owns Unicorn the product. Decides what gets built and why; sets client and business priorities.

**Owns**: roadmap and quarterly priorities; client-facing priorities; business direction and product positioning.

**Contributes to**: frontend implementation via Lovable (vibe-codes; doesn't touch hand-written backend code). Reads KB; flags drift but doesn't fix it.

**Authority**: final say on product *and* engineering decisions when they collide. Project lead surfaces tradeoffs; product owner decides.

### 2. Project lead + KB owner

**Purpose**: Leads the Unicorn engineering effort day-to-day and stewards the knowledge base. Translates product direction into shipped code; keeps the KB honest as the codebase and team evolve.

**Owns**: KB structure and hygiene; `unicorn-audit/` repo (sole access, sole author); Lovable remix trigger; post-remix ritual; KB pinned-file changes; `conventions.md` and `flow-patterns.md`; ADR merging (anyone proposes; project lead merges); Open Decision resolution; curation of `glossary.md` and `brainstorm-log.md`; reconciliation cadence.

**Contributes to**: feature implementation as a Dev (the project lead also wears the Dev seat).

**Authority**: KB pinned-file merges (final say); Lovable remix timing (final say, but product owner can override); audit events (sole authority). Not schema or RLS sign-off — those don't exist as gates today.

### 3. Dev

**Purpose**: Builds and maintains Unicorn. In practice today, this means Lovable work — schema and frontend land together via Lovable, no peer review. Hand-written code via Claude Code is possible but not the norm.

**Owns**: feature implementation (almost entirely through Lovable); adherence to `conventions.md` patterns *when* hand-writing code via Claude Code; following the canonical edge-function pattern from `conventions.md → Edge functions` for any hand-written edge function.

**Contributes to**: `conventions.md`, `flow-patterns.md`, `glossary.md`, ADR proposals, brainstorm-log — anyone proposes; project lead merges. `codebase-state/*` drift flags — Devs notice, project lead regenerates.

**Authority**: none. No schema sign-off, no RLS sign-off, no code review gate, no merge approval.

### 4. Non-technical KB reader

**Purpose**: Reads the KB to understand Unicorn — what it is, who works on it, how it's built — without authoring, committing, or being expected to act on what they read.

**Typical occupants**: new hires before commit access; external reviewers / consultants Angela brings in; rare deep-reading internal people.

**Owns**: nothing. By design.

**Contributes to**: optional fresh-eyes feedback via `handoffs/non-technical-proposal.md`. KB owner decides what (if anything) lands.

**Authority**: none.

---

## People → seat mapping

| Person | Seats |
|---|---|
| Angela | Product owner + Dev |
| Carl | Project lead + KB owner + Dev |
| Dave | Dev |
| RJ | Dev (access to Unicorn but primarily on ComplyHub — Vivacity's other product) |
| Khian (Brian) | Dev — junior, growing scope |

---

## Tool access matrix

Per occupant. Marked at the person level because tool access varies by individual, not by seat.

| Tool | Angela | Carl | Dave | RJ | Khian | Non-technical KB reader |
|---|---|---|---|---|---|---|
| Claude.ai (Unicorn project) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Claude Code | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Lovable | ✅ | ✅ | ✅ | ✅ (access, doesn't use) | ✅ | ❌ |
| GitHub MCP read | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| GitHub MCP write | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `<codebase>/` commit (via Lovable) | ✅ | ✅ | ✅ | ❌ (doesn't use) | ✅ | ❌ |
| `unicorn-kb/` commit (via Claude Code) | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `unicorn-audit/` access | ❌ | ✅ (sole) | ❌ | ❌ | ❌ | ❌ |
| Supabase console | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |

**GitHub MCP write path.** The GitHub MCP connector in claude.ai chat is configured with a **read-only PAT by design** — chat users cannot open PRs through it, even for `unicorn-kb/`. Writes happen exclusively from Claude Code sessions, which use local `gh`/git credentials. This is intentional — it keeps the chat connector's blast radius small. Verified 2026-04-27.

**`<codebase>/` access.** Lovable users push direct to `main` from the Lovable UI. Claude Code is forbidden from writing to `<codebase>/` per `~/repository/unicorn-workspace/CLAUDE.md → Write permissions` regardless of who's running it.

---

## Which handoff applies to whom

| Handoff | Angela | Carl | Dave | RJ | Khian | Non-technical KB reader |
|---|---|---|---|---|---|---|
| [claude-project-to-claude-code.md](../handoffs/claude-project-to-claude-code.md) | ✅ user | ✅ user | ✅ user | ✅ user | ✅ user | ❌ |
| [lovable-to-codebase.md](../handoffs/lovable-to-codebase.md) | generates events | ✅ reconciler | generates events | ❌ | generates events | ❌ |
| [post-lovable-remix.md](../handoffs/post-lovable-remix.md) | ❌ | ✅ ritual owner | ❌ | ❌ | ❌ | ❌ |
| [non-technical-proposal.md](../handoffs/non-technical-proposal.md) | ❌ | recipient | ❌ | ❌ | ❌ | ✅ author |

**`lovable-to-codebase.md` semantics.** Lovable users don't "author" a handoff doc — they generate the events (direct-to-`main` Lovable commits) that the project lead reconciles. The handoff is purely KB-owner-side: Carl watches `main`, reconciles `codebase-state/*` and any affected `pinned/*` files when drift accumulates.

---

## Authority

- **KB pinned-file changes**: project lead (Carl) reviews and merges. No external reviewer required.
- **Open Decision resolution**: anyone proposes; project lead merges the ADR.
- **Lovable remix trigger**: project lead's call, communicated before remix so the team can pause Lovable work.
- **Audit events**: project lead only. Not a team-visible process.
- **Schema and RLS sign-off**: **does not exist as a gate today.** Lovable pushes schema changes direct to `main`. This is a known risk; an ADR documenting the operating-model rationale and the trace path for failures lands in `reference/decision-trail.md` as part of PR2.

---

## Maintenance

When team composition or tool access changes, update both the People → seat mapping and the tool access matrix in this file, plus any affected entry in `pinned/orientation.md → Who's who` (cross-reference). Per `pinned/kb-hygiene.md → Update triggers`.

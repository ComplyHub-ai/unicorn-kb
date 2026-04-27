# Team Roles & Tool Access

> **Last updated:** 2026-04-24 · **Reconsider by:** 2026-07-24 · **Confidence:** high.
>
> Who's on the team, what they do, and which tools they touch. Drives which
> [handoffs](../handoffs/README.md) apply to whom.

---

## People

| Person | Role | Notes |
|---|---|---|
| Angela | Product owner + dev + KB owner | The only person with audit-repo access; owns most KB files; runs reconciliations |
| RJ | Tech lead / architect | Signs off RLS changes, co-owns conventions + flow-patterns |
| Carl | Senior dev | Full codebase, chat-based for KB |
| Dave | Architect / dev | Full codebase, chat-based for KB |
| Khian (Brian) | Junior dev | Growing scope |

---

## Tool access matrix

| Tool | Angela | RJ | Khian | Carl | Dave | Non-technical stakeholders |
|---|---|---|---|---|---|---|
| Claude.ai (Unicorn 2.0 project) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Claude Code | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| GitHub MCP (read) | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| GitHub MCP (PR write) | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `<codebase>/` commit access | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `unicorn-kb/` commit access | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `unicorn-audit/` access | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Lovable | ✅ | ✅ | ? | ? | ? | ❌ |
| Supabase console | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |

Lovable access per-person isn't firmly set — update this row as the team
stabilises.

---

## Which handoff applies to whom

| Handoff | Angela | RJ | Khian | Carl | Dave | Non-technical |
|---|---|---|---|---|---|---|
| [claude-project-to-claude-code.md](../handoffs/claude-project-to-claude-code.md) | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| [lovable-to-codebase.md](../handoffs/lovable-to-codebase.md) | ✅ | ✅ | ? | ? | ? | ❌ |
| [post-lovable-remix.md](../handoffs/post-lovable-remix.md) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| [non-technical-proposal.md](../handoffs/non-technical-proposal.md) | Process as recipient | ❌ | ❌ | ❌ | ❌ | ✅ (author) |

---

## Authority

- **RLS migrations:** RJ signs off before merge. No exceptions. Rule from
  `reference/cadence.md → Shipping discipline`.
- **KB pinned-file changes:** Carl reviews and merges. RJ consulted on
  `conventions.md` and `flow-patterns.md` changes.
- **Open Decision resolution:** Anyone can propose resolution; RJ or Carl
  merges the ADR.
- **Lovable remix trigger:** Carl's call, communicated before remix so
  the team can pause commits.
- **Audit events:** Carl only. Not a team-visible process.

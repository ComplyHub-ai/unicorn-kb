# Unicorn 2.0 — Knowledge Base

Source of truth for the Unicorn 2.0 Claude Project's knowledge layer. This
repo is one of three that work together:

| Repo | Purpose | Lovable sees |
|---|---|---|
| `<codebase>/` | The actual codebase. Lovable builds here. | Yes |
| `unicorn-kb/` | This repo. Team opinion, decisions, patterns, handoffs, and as-shipped state docs. | No |
| `unicorn-audit/` | Angela's audit trail — narrative record of reconciliations. | No |

## Folder structure

```
unicorn-kb/
├── pinned/          ← uploaded to the Claude Project (always in context)
├── reference/       ← fetched via GitHub MCP on demand (team opinion)
├── codebase-state/  ← fetched via GitHub MCP on demand (as-shipped state)
└── handoffs/        ← scenario-specific procedures
```

**Pinned files** are the stable opinion layer. They change in months, not
weeks. Uploading them to the Claude Project means every chat has them in
context without a fetch.

**Reference files** are longer-form opinion — full ADRs, flow patterns,
cadence, migration history. Fetched via GitHub MCP on demand because the
cost-per-token of keeping them pinned isn't worth it.

**Codebase-state files** describe the shipped `<codebase>/` codebase — what's
built, where things live, as-shipped architecture. They live here (not in
`<codebase>/`) so Lovable doesn't have to reason about files it didn't create.
Every file in this folder carries a **Reflects commit** SHA pointing at the
`<codebase>/` HEAD it was generated from. Regeneration ritual lives in
[handoffs/post-lovable-remix.md](handoffs/post-lovable-remix.md).

**Handoff files** are role- and scenario-specific procedures — e.g. "I just
finished a Claude Code session, what goes back to the KB?" Start at
[handoffs/README.md](handoffs/README.md) for the lookup table.

## Source precedence

The Claude Project's custom instructions encode, in order:

1. Pinned KB (always in context)
2. `unicorn-kb/reference/` + `unicorn-kb/handoffs/` via GitHub MCP
3. `unicorn-kb/codebase-state/` via GitHub MCP
4. `<codebase>/` source code via GitHub MCP
5. Inference (flagged with "Inferring from …")

When any KB layer and the actual codebase disagree, the codebase wins.
Full rules in [reference/source-precedence.md](reference/source-precedence.md).

## How to update

- Conventions, decisions, patterns, ADRs → PR against `unicorn-kb/` on a branch.
- Codebase state (module status, codebase map, architecture) → PR against
  `unicorn-kb/codebase-state/`; full regeneration after a Lovable remix,
  surgical edits per feature ship.
- Audit narrative → `unicorn-audit/` (Angela only).
- Non-git stakeholders → paste into the designated Claude Project inbox
  thread; see [handoffs/non-technical-proposal.md](handoffs/non-technical-proposal.md).

See [pinned/kb-hygiene.md](pinned/kb-hygiene.md) for the full policy on
shelf life, review cadence, and what never goes in.

## What's NOT here

- Secrets, API keys, Supabase service-role keys, Stripe keys (never).
- Raw migration SQL — points to `supabase/migrations/` in `<codebase>/`.
- Lovable preview URLs / project IDs beyond what's in `<codebase>/supabase/config.toml`.
- Personal identifying info about clients beyond role names.
- The product spec (EOS) — lives in `<codebase>/` or wherever the spec canonically lives.

# Product Decisions (Index)

> **Last updated:** 2026-04-30 · **Reconsider by:** 2026-07-30 · **Confidence:** high for recorded entries.
>
> One-line-per-decision index. For full rationale, alternatives, and
> supersession history, see [../reference/decision-trail.md](../reference/decision-trail.md).
>
> **Maintenance:** when a decision is made, add a row here AND a full ADR in
> `decision-trail.md`. When an Open Decision is resolved, move it to the
> decided log.

---

## Decided

> Format: `[ADR-NNN] [YYYY-MM-DD] [one-line summary] — full: reference/decision-trail.md#adr-NNN`

The legacy ADR-001 through ADR-010 records live in reference/decision-trail.md but were not migrated forward as one-liners — they'll be re-engaged here as they become relevant in real work. New decisions from this point forward are recorded as normal (one-line index entry here + full ADR in decision-trail.md).

- [ADR-011] [2026-04-27] Operating model — Lovable is the workflow; no peer review or sign-off gates — full: [reference/decision-trail.md#adr-011](../reference/decision-trail.md#adr-011)
- [ADR-012] [2026-04-28] Audit-entry authorship — devs author Lovable prod DB change sessions; Carl sole author for everything else — full: [reference/decision-trail.md#adr-012](../reference/decision-trail.md#adr-012)

---

## Open decisions

> Format: `[YYYY-MM-DD opened] [topic] [owner] — next step`
>
> **Rule:** open > 6 months with no movement → either delete or escalate.
> See `kb-hygiene.md → Pruning discipline`.

- [2026-04-30] **AI executive summary exemplars** [Angela / Sam] — `draft-executive-summary` system prompt contains `{{EXEMPLARS_PENDING}}`. Sam to supply 2–3 redacted historical executive summaries (one CRICOS/Combined, one CHC/Mock, one Due Diligence) so Angela can replace the placeholder. Without them, output voice is untuned. — next step: Sam delivers redacted examples; Angela pastes into the function and deploys.
- [2026-04-30] **Corpus population — SRTO 2025** [Angela] — `embed-srto-corpus` exists but no confirmation it has been run against the actual Standards PDFs. Until run, all AI audit tools operate with `corpus_empty = true` (lower quality, but still functional). — next step: Angela confirms run date or schedules the ingestion.
- [2026-04-30] **Corpus population — National Code 2018 / ESOS Act** [Angela] — National Code 2018, CRICOS Practice Guides, and ESOS Act 2000 PDFs need to be uploaded to `srto-source-documents` and `embed-srto-corpus` run against them before CRICOS audits get framework-specific context. — next step: Angela uploads PDFs and runs embed function.
- [2026-04-30] **Gemini 2.5 Pro vs Claude for audit voice** [Carl / Angela] — Lovable AI Gateway does not route Anthropic models. Audit AI stack uses Gemini 2.5 Pro. If voice fidelity against Vivacity's existing findings style is insufficient, an Anthropic key + direct API call is the swap path (gateway URL + model ID only). — next step: Angela/Sam run test drafts against real findings; Carl decides.

---

## Retired / superseded

<!--
  ADRs marked superseded retain their entry in decision-trail.md. Reflect
  here only if relevant to current product shape.
-->

# CLAUDE.md

Context and conventions for working in this repo. Read this first.

## What this repo is

A structured design log for the AST10x0 BSP work in OpenPRoT — board, peripherals, and platform integration. ADR-style: decisions and the evidence behind them are first-class, durable artifacts. Code lives in the OpenPRoT fork, not here.

This repo exists because planning docs and analyses accumulate during LLM-assisted development, and curation is what turns that pile into something future agents can actually use. Adding entries here is the act of curation.

## Repo structure

- `decisions/` — Architecture Decision Records (ADRs). One file per decision.
- `evidence/` — Evidence cards: claims linked to sources (code, specs, benchmarks, incidents) with confidence levels.
- `experiments/` — Validation plans for uncertain assumptions.
- `outcomes/` — Post-implementation reviews tied back to decisions.
- `templates/` — Canonical templates. Use these when creating new records.
- `prompts/` — Reusable prompts for this workflow.
- `archive/` — Raw source material (planning docs, exploratory analyses) being curated. Not authoritative; the distilled records are.
- `index.md` — Dashboard tracking open and accepted decisions, evidence backlog, active experiments.

## Naming conventions

- Decisions: `DEC-YYYYMMDD-short-title.md`
- Evidence: `EVD-YYYYMMDD-short-claim.md`
- Experiments: `EXP-YYYYMMDD-short-hypothesis.md`
- Outcomes: `OUT-YYYYMMDD-decision-id.md`

Date is when the record was created. Short title is kebab-case, 2-5 words, descriptive.

## Writing decisions

**Match the house style.** Read at least one existing record before drafting a new one. Match its depth and tone. Don't invent a new format.

**Retrospective vs fresh.** Many records here document patterns already in the code. Mark these as `Status: accepted (retrospective — documents existing pattern)`. For retrospective records:

- Section 3 (Options Considered): list alternatives in the design space honestly, not a fake deliberation. Don't fabricate weighing that didn't happen.
- Section 5 (Scorecard): mark `Not applicable — retrospective`.
- Section 7 (Risks): focus on risks of the *current* pattern, not risks of *adopting* it.
- Section 8 (Validation): point at existing tests and peripherals that already exercise the pattern.

**Verify callers, not just definitions.** When claiming a component "has landed" / "is wired" / "is the source of truth" in a retrospective record, verify by following the call chain from a known production entry point — not just by confirming the type/function/module exists. A definition with zero callers is dead code, not landed work; if you find that, note it explicitly in the record (typically as an evidence row plus a §6 sub-point flagging the gap). The companion-repo equivalent of "did this actually wire through?" is a `grep` for callers of the public entry points.

**Code references.** Reference files by path and concept, not line numbers — line numbers rot. Good: "see `peripherals/smc/controller.rs` — the `Smc<B, Mode>` type-state struct." Bad: "see line 45 of controller.rs."

**One decision per record.** If you find yourself writing about three different things, that's three records. Split.

## Writing evidence cards

Evidence cards are short. They state a claim, point at the source, and rate confidence (high / medium / low per the README criteria). They are pointers with attached claims, not summaries of source documents.

If you find yourself copying significant content from the source into the card, stop. The source is the source of truth; the card just makes the claim findable.

## Workflow for new entries

1. Read the relevant template under `templates/`.
2. Read at least one existing record of the same type for tone-matching.
3. Draft the new record under the correct directory with the correct naming convention.
4. Add the entry to `index.md` in the appropriate table.
5. If a decision references new evidence, create the evidence cards in the same change — don't leave dangling references.

## What NOT to put here

- Raw planning docs from a working session — those go in `archive/` if they're worth keeping at all.
- Status updates, checkpoints, work-in-progress notes — these are ephemeral and don't belong in any structured directory.
- Code excerpts longer than ~10 lines — link to the source instead.
- Full reproductions of analyses that exist elsewhere — extract the durable claims into evidence cards and link.
- Tutorials, how-to guides, onboarding docs — those belong in the OpenPRoT repo's docs, not here.

## Companion repo

The code this design log describes lives at `github.com/rusty1968/openprot`, primary branch of interest `smc-peripheral`. When a decision references code, use repo-relative paths (e.g. `peripherals/smc/controller.rs`); avoid cross-repo URLs, which are fragile.

## Curation discipline

The point of this repo is to convert sprawling planning material into a small number of durable, retrievable records. Every time something is added, it should be the *distilled* version. If you're tempted to dump a long doc in here, that's the signal to extract the 2-3 durable claims out of it instead and put those in.

Fewer, sharper records beat many comprehensive ones. Optimize for what future-you (or a future agent) will actually want to read mid-task — not for completeness.

## When asked to curate from `archive/`

A common task: take a doc from `archive/` and produce one or more decision/evidence records. Default approach:

1. Read the source doc fully before proposing anything.
2. Identify the durable claims — the things that would still be true and useful in six months. Skip status updates, abandoned ideas, and details already encoded in the code.
3. Decide whether each claim is a *decision* (an architectural choice) or *evidence* (a factual claim about the code or system).
4. Propose a list of records before writing them. Let the human confirm scope before drafting.
5. Draft. Cross-link evidence to decisions. Update `index.md`.
6. Before recommending deletion of the source doc, grep the new records for references to labels, sections, or numbering that only made sense inside the source (e.g., "Phase N", "Step N", "see §X of the original"). Replace each with the concrete content it stood for so the records read coherently after the source is gone.
7. Recommend whether the source doc should be kept in `archive/`, deleted, or replaced by a shorter pointer.

Most archive docs yield 2-4 durable records, not one record per doc. A 700-line analysis is rarely one decision.

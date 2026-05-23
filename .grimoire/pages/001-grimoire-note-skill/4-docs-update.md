# Step 4 — Document `grimoire-note` in READMEs and `CLAUDE.md`

## Source of truth

Read [`.grimoire/pages/001-grimoire-note-skill/SPEC.md`](./SPEC.md), specifically:
- **Goals → "Document the new skill in `README.md`, `README.pt-BR.md`, and `CLAUDE.md`."**
- **Scope → Modified files → `README.md` and `README.pt-BR.md`**, and **`CLAUDE.md`**.
- **Acceptance criteria → "`README.md`, `README.pt-BR.md`, and `CLAUDE.md` all list `grimoire-note` alongside the other six skills…"** — **NOTE: the SPEC says "the total reads 'seven skills' everywhere"; this is a stale-from-before-`grimoire-know` oversight in the SPEC.** Per planner decision at planning time (resolved with the user via `AskUserQuestion`), the docs already say "seven skills" today (counting `grimoire-know`), so adding `grimoire-note` makes **eight**. Update wording to **"eight skills"** everywhere. Do not honor the literal "seven" — honor the intent (list all skills, count them correctly).

This step edits **only** the three documentation files. No SKILL.md, no conventions, no manifests, no `CHANGELOG.md`.

## Files to edit

### 1. `README.md`

Read first, then locate and update:

- **Skill count line** (currently around line 9): *"A Claude Code plugin bundling **seven** skills…"* → change to **"eight"** skills.
- **Installation success line** (around line 33): the inline list of available slash commands. Insert `/grimoire-note` in an order that reads naturally — recommendation: after `/grimoire-update`, since `grimoire-note` is orthogonal-to-pipeline like `grimoire-update` and `grimoire-know`. Update the comma list accordingly.
- **Skills table** (around lines 55–62): add a new row for `/grimoire-note <note>` with a one-line description matching the SKILL.md frontmatter (Step 1). Place it adjacent to `/grimoire-know` or `/grimoire-update`, matching the SPEC's framing of `grimoire-note` as an orthogonal skill.
- **Per-skill section** (the `### /grimoire-know` / `### /grimoire-update` style sections, around lines 103–): add a new `### /grimoire-note` section in the same prose density and structure used for `grimoire-know` and `grimoire-update`. Cover:
  - What it does (free-text in, surgical edit to `PROJECT.md`'s `Key Conventions` and `Notes`).
  - Scope guardrails (never bootstraps, never adds sections, never touches `HISTORIC.md` or pages).
  - Mention the dual-writer contract with `grimoire-init` (cross-reference `GRIMOIRE-CONVENTIONS.md § Project context`).
  - IDE-aware review pause-point and the single atomic commit (`docs(grimoire): refine project context`).
- **"Pipeline skills" footer** (around line 165): the sentence ending *"`grimoire-update` is plugin self-maintenance and `grimoire-know` is read-only Q&A, so neither participates in those rules."* Extend it to include `grimoire-note` in the "orthogonal-to-pipeline" group (e.g., *"`grimoire-update` is plugin self-maintenance, `grimoire-know` is read-only Q&A, and `grimoire-note` is a surgical `PROJECT.md` writer, so none of them participate in those rules."*). Confirm the exact wording fits the surrounding paragraph.

### 2. `README.pt-BR.md`

Mirror every change made to `README.md`, translated into Portuguese-BR. Match the existing tone (the file already uses pt-BR for the same content). Specifically:

- Skill count: **"sete"** → **"oito"** skills (or the language's natural way to phrase it; check the current sentence).
- Installation success line (around line 33): insert `/grimoire-note` in the same position relative to `/grimoire-update`.
- Skills table (around lines 55–62): new row, pt-BR description.
- Per-skill section (around lines 103–): new `### /grimoire-note` section, pt-BR prose.
- "Pipeline skills" footer (around line 165): extend the pt-BR equivalent sentence to include `grimoire-note` in the orthogonal-to-pipeline group.

### 3. `CLAUDE.md`

- **Skill count line** (around line 9): *"bundling **seven** skills"* → **"eight"** skills.
- **Frontmatter list of SKILL.md paths** (around line 11): add `[skills/grimoire-note/SKILL.md](skills/grimoire-note/SKILL.md)` in the inline comma-separated list. Place it adjacent to the `grimoire-know` / `grimoire-update` entries.
- **"The skills and how they connect"** section: add a new `- **`grimoire-note`**` bullet alongside the existing `grimoire-init`, `grimoire-know`, `grimoire-update` bullets. The description must:
  - Frame it as orthogonal to Spec → Plan → Execute (same family as `grimoire-know` and `grimoire-update`).
  - State scope: writes only `## Key Conventions / Constraints` and `## Notes` in `.grimoire/PROJECT.md`, incrementally.
  - Note the hard-stop when `PROJECT.md` is missing.
  - Note the single atomic commit message: `docs(grimoire): refine project context`.
  - Cross-reference the dual-writer contract amendment to `GRIMOIRE-CONVENTIONS.md § Project context` (Step 2).
- **Any other sentence enumerating the skills** — sweep the file for `seven`, `six skills`, and the original five-skill enumeration. Update so every enumeration is consistent with eight skills total.

## Pre-work reading

1. [`README.md`](../../../../README.md), [`README.pt-BR.md`](../../../../README.pt-BR.md), [`CLAUDE.md`](../../../../CLAUDE.md) — the three files to edit. Read each in full so you can match the existing density, prose style, and skill-section template.
2. [`skills/grimoire-note/SKILL.md`](../../../../skills/grimoire-note/SKILL.md) — completed in Step 1; use its `description` frontmatter as the source for the one-line table description.
3. [`GRIMOIRE-CONVENTIONS.md`](../../../../GRIMOIRE-CONVENTIONS.md) `§ Project context` — amended in Step 2; cite when explaining the dual-writer contract.

## Style guardrails

- **Match existing voice.** README sections have a recognizable prose density — match it.
- **Bilingual parity.** Every change to `README.md` has an equivalent change in `README.pt-BR.md`. Do not skip the pt-BR file.
- **Skill ordering.** Place `grimoire-note` adjacent to its orthogonal-pipeline siblings (`grimoire-know`, `grimoire-update`) in every list, table, and section enumeration.
- **Numeric consistency.** Every "skill count" mention must be updated to **eight**. Sweep for `seven`, `7`, `seis`, `sete`, etc. — do not leave a stale count.
- **No new conventions content.** The dual-writer contract lives in `GRIMOIRE-CONVENTIONS.md § Project context` — cross-reference, do not duplicate the contract prose into these files.

## Acceptance check before commit

- All three files list `grimoire-note` alongside the other seven skills (count: eight total).
- The skill count number is consistent in all three files (no "seven" left behind).
- The new `### /grimoire-note` per-skill section appears in both README files with bilingual parity.
- `CLAUDE.md` "skills and how they connect" has a new bullet for `grimoire-note` with full scope, hard-stop, and dual-writer contract reference.
- No other file is modified.

## Commit

Per `§ Commits`, one atomic commit covering only the three documentation files:

```
docs(grimoire): document grimoire-note in READMEs and CLAUDE.md
```

## Out of scope (do NOT do here)

- Do **not** edit `GRIMOIRE-CONVENTIONS.md`, `SKILL.md` files, manifests, or `CHANGELOG.md` — Steps 1–3 and Step 5 own those.
- Do **not** touch `HISTORIC.md`.
- `§ TDD` does not apply.

# Step 2 — Amend `§ Project context` in `GRIMOIRE-CONVENTIONS.md`

## Source of truth

Read [`.grimoire/pages/001-grimoire-note-skill/SPEC.md`](./SPEC.md), specifically the **Goals → Amend `§ Project context`** bullet and **Scope → Modified files → `GRIMOIRE-CONVENTIONS.md`** entry. This step amends **only** the existing `§ Project context` section in place. Do **not** add a new section, do **not** touch any other `§` section, do **not** edit any other file.

## Pre-work reading

1. [`GRIMOIRE-CONVENTIONS.md`](../../../../GRIMOIRE-CONVENTIONS.md) — the file you will edit. Read `§ Project context` in its current form so you understand exactly what is being amended.
2. [`.grimoire/pages/001-grimoire-note-skill/1-grimoire-note-skill.md`](./1-grimoire-note-skill.md) — Step 1, already completed; the new skill exists and its `[Required Reading]` references this section. Your amendment must remain consistent with the SKILL.md it claims to govern.
3. SPEC sections: **Goals**, **Scope → Modified files**, **Constraints → Single source of truth**, **Acceptance criteria** (the `GRIMOIRE-CONVENTIONS.md § Project context describes both writers and their scopes in-place` bullet).

## What the current `§ Project context` says (for reference, do NOT copy verbatim — read the file)

The existing section establishes:
- Every Grimoire skill loads `.grimoire/PROJECT.md` if present.
- If absent, proceed without it and remind the user to run `grimoire-init`.
- **`grimoire-init` is the only skill that writes `PROJECT.md`. Other skills must not modify it as a side effect.** ← this invariant evolves.
- Update mode for `grimoire-init`.

## What to change

Amend `§ Project context` **in place** so that the single-writer invariant becomes a **dual-writer contract**. The amendment must capture, in the prose of the section itself (not as a footnote):

1. **`grimoire-init` is the only skill that may create `PROJECT.md`** (bootstrap is exclusive).
2. **`grimoire-init` is the only skill authorized to write outside `## Key Conventions / Constraints` and `## Notes`** — Purpose, Audience, Tech Stack, Repository Layout, Current Status are init-only.
3. **`grimoire-init` is the only skill that may add new sections** to `PROJECT.md`.
4. **`grimoire-note` is authorized to write only `## Key Conventions / Constraints` and `## Notes`, incrementally** — never creates the file, never adds sections, never edits other sections.
5. **Other skills (`grimoire-spec`, `grimoire-plan`, `grimoire-execute`, `grimoire-quick`, `grimoire-know`, `grimoire-update`) never write `PROJECT.md` at all.**

Keep the existing bullets that still apply (load-on-startup behavior, the "missing → suggest `grimoire-init`, non-blocking" rule, update-mode behavior of `grimoire-init`). The amendment is additive — not a rewrite.

## Style guardrails

- **Single source of truth, preserved.** The contract lives here and **only** here. The new SKILL.md (Step 1) references `§ Project context` and does not duplicate the contract.
- Keep the existing voice and density of `GRIMOIRE-CONVENTIONS.md` — short bullets, terse declarative sentences.
- Do **not** rename the section. It must remain `## § Project context`.
- Do **not** reorder sections. Do **not** edit any other `§` section.
- Preserve all surrounding section dividers (`---`) and the table-of-contents flow as-is.

## Acceptance check before commit

- The amended section names both writers (`grimoire-init`, `grimoire-note`) and their scopes.
- The "only `grimoire-init` writes `PROJECT.md`" invariant is replaced with the dual-writer contract; no stale wording remains.
- No new top-level section was added; only `§ Project context` is modified.
- The Step 1 SKILL.md (`skills/grimoire-note/SKILL.md`) and the amended `§ Project context` are mutually consistent (no contradiction in scope or wording).

## Commit

Per `§ Commits`, one atomic commit covering only `GRIMOIRE-CONVENTIONS.md`:

```
docs(grimoire): amend § Project context for dual-writer contract
```

## Out of scope (do NOT do here)

- Do **not** edit any other `§` section of `GRIMOIRE-CONVENTIONS.md`.
- Do **not** edit `skills/grimoire-init/SKILL.md` — that is Step 3.
- Do **not** edit READMEs, `CLAUDE.md`, manifests, or `CHANGELOG.md` — Steps 4 and 5.
- Do **not** touch `HISTORIC.md`.
- `§ TDD` does not apply — no tests.

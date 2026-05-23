# Step 3 — Update `grimoire-init` Phase 2 with preserve-by-default for Key Conventions and Notes

## Source of truth

Read [`.grimoire/pages/001-grimoire-note-skill/SPEC.md`](./SPEC.md), specifically:
- **Goals → "Update `grimoire-init`'s update mode…"** bullet.
- **Scope → Modified files → `skills/grimoire-init/SKILL.md`** entry.
- **Acceptance criteria → "`skills/grimoire-init/SKILL.md` Phase 2 explicitly describes the preserve-by-default behavior…"** bullet.

This step edits **only** `skills/grimoire-init/SKILL.md` Phase 2.

## Pre-work reading

1. [`skills/grimoire-init/SKILL.md`](../../../../skills/grimoire-init/SKILL.md) — the file you will edit. Read it in full, but the change is scoped to **Phase 2: Clarifying Questions**.
2. [`GRIMOIRE-CONVENTIONS.md`](../../../../GRIMOIRE-CONVENTIONS.md) `§ Project context` (just amended in Step 2) — the dual-writer contract that motivates this Phase 2 change. The Step 2 amendment establishes that `grimoire-note` writes the two sections; the SPEC's intent here is that `grimoire-init`'s update-mode interview no longer assumes Key Conventions / Notes are fully owned by init.
3. [`.grimoire/PROJECT.md`](../../../../.grimoire/PROJECT.md) — example of a `PROJECT.md` with populated `## Key Conventions / Constraints` and `## Notes` sections, for context on what kinds of entries the preserve-by-default behavior protects.

## What to change

Edit **Phase 2: Clarifying Questions** in `skills/grimoire-init/SKILL.md` to add a clearly-scoped paragraph that activates **only in UPDATE mode**, after the existing "In UPDATE mode, ask only about deltas" sentence. The new paragraph must establish:

1. **Trigger condition.** When `## Key Conventions / Constraints` and/or `## Notes` in the existing `PROJECT.md` already contain entries (one or more bullets).
2. **Behavior.** List those existing entries back to the user (briefly — slug or first ~10 words is enough) and ask: *"Anything stale here?"*
3. **Default action.** Preserve existing entries **in place**. The user must give an explicit delta to remove or rewrite a specific entry. Silence = keep everything.
4. **Additions still allowed.** The user can still add new entries via this Phase 2 interview, exactly as before.
5. **Other sections unchanged.** The full-rewrite/refresh behavior of init for `Purpose`, `Audience`, `Tech Stack`, `Repository Layout`, `Current Status` is **unchanged** in update mode. Only `## Key Conventions / Constraints` and `## Notes` get the preserve-by-default protection.

A short cross-reference to `§ Project context` (the dual-writer contract from Step 2) is appropriate, e.g. *"per `§ Project context`'s dual-writer contract, these two sections are also owned by `grimoire-note`, so init's default is preserve-in-place"*.

## Style guardrails

- Keep the existing structure of Phase 2 intact — do **not** rewrite Phase 1, Phase 3, Phase 4, Phase 5, or the `[Required Reading]` block.
- Do **not** change the YAML frontmatter (`name`, `description`, `argument-hint`).
- Do **not** add a new phase number; this is an in-place additive paragraph inside Phase 2.
- INITIALIZATION mode behavior is unchanged — the new paragraph applies **only** in UPDATE mode.
- Match the surrounding voice (short imperative sentences, no narrative buildup).

## Acceptance check before commit

- Phase 2 explicitly describes the preserve-by-default behavior for `## Key Conventions / Constraints` and `## Notes` in UPDATE mode only.
- The interview surfaces existing entries and asks about staleness — does not silently drop or rewrite them.
- Additions remain permitted.
- Other sections' update-mode behavior is unchanged.
- The `[Required Reading]` block and frontmatter were not touched.
- No other file was modified.

## Commit

Per `§ Commits`, one atomic commit covering only `skills/grimoire-init/SKILL.md`:

```
docs(grimoire): preserve-by-default for Key Conventions and Notes in init update mode
```

## Out of scope (do NOT do here)

- Do **not** edit `GRIMOIRE-CONVENTIONS.md` — Step 2 already did it.
- Do **not** edit any other `SKILL.md` file.
- Do **not** edit READMEs, `CLAUDE.md`, manifests, or `CHANGELOG.md` — Steps 4 and 5.
- Do **not** touch `HISTORIC.md`.
- `§ TDD` does not apply.

# Step 1 — Create `skills/grimoire-note/SKILL.md`

## Source of truth

Read [`.grimoire/pages/001-grimoire-note-skill/SPEC.md`](./SPEC.md) — it is the authoritative description of `grimoire-note`. This step implements the **new SKILL.md only**. Do **not** edit `GRIMOIRE-CONVENTIONS.md`, `grimoire-init`, READMEs, `CLAUDE.md`, `plugin.json`, `marketplace.json`, or `CHANGELOG.md` here — those are separate steps.

## Pre-work reading (for the sub-agent)

1. `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md` — the conventions this new skill must declare it loads (`§ Project context`, `§ IDE-aware review`, `§ Commits` apply; everything else does not). Source-of-truth discipline: do **not** duplicate the dual-writer contract into this file — it lives in `§ Project context` (Step 2).
2. [`skills/grimoire-know/SKILL.md`](../../../skills/grimoire-know/SKILL.md) — closest precedent for the "orthogonal-to-pipeline" skill pattern. Reuse the structure of its `[Required Reading]` block (explicit list of which conventions apply / do not).
3. [`skills/grimoire-update/SKILL.md`](../../../skills/grimoire-update/SKILL.md) — another orthogonal skill; reference for "no page, no HISTORIC, no commits-in-consumer" framing and IDE-aware draft handling.
4. [`.grimoire/PROJECT.md`](../../../../.grimoire/PROJECT.md) — the file `grimoire-note` will write to. Note the exact section headings (`## Key Conventions / Constraints`, `## Notes`).
5. SPEC sections: **Goals**, **Scope → Created files**, **Acceptance criteria**, **Constraints**.

## What to write

Create the new file at:

```
skills/grimoire-note/SKILL.md
```

### Frontmatter (YAML, load-bearing — preserve exact keys)

```yaml
---
name: grimoire-note
description: Surgically refine .grimoire/PROJECT.md's Key Conventions and Notes from a free-text input — semantic dedup, retroactive consolidation, IDE-aware diff review.
argument-hint: Free-text note(s) to fold into PROJECT.md's Key Conventions or Notes
---
```

The `description` must be a single line. The `argument-hint` must describe the free-text input.

### Prompt body — required sections (in order)

1. **`[Required Reading]`**
   - Load `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md`. Explicitly state which sections apply (`§ Project context`, `§ IDE-aware review`, `§ Commits`) and which do **not** (`§ TDD`, `§ Sub-agent spawning`, `§ Historic`, `§ Pause-point pattern` for pipeline pages, `§ .grimoire/ layout` for pages). Follow the wording precedent in `grimoire-know`'s `[Required Reading]`.
   - Load `.grimoire/PROJECT.md` — but defer the hard-stop check to Phase 1 below so the message is presented clearly.

2. **`[Objective]`** — one short paragraph plus the `$ARGUMENTS` substitution placeholder for the user's free-text note(s). Frame: incremental, surgical edits to `## Key Conventions / Constraints` and `## Notes` only — never the whole file, never new sections, never a skeleton rewrite.

3. **`[Hard constraints]`** — write-scope guardrails:
   - Only `.grimoire/PROJECT.md` may be edited; no other file is touched.
   - Only `## Key Conventions / Constraints` and `## Notes` may be modified; never add or remove a section header, never edit Purpose / Audience / Tech Stack / Repository Layout / Current Status.
   - Never create `.grimoire/PROJECT.md` if missing — hard-stop instead.
   - Never create a page folder, never touch `HISTORIC.md`, never spawn execute sub-agents.

4. **`[Phase 1: Precondition Check (HARD STOP)]`** — per `§ Project context` (no bootstrap):
   - Verify `.grimoire/PROJECT.md` exists. If missing, print the canonical English message verbatim (the message may also be rendered in the user's interaction language when detected; English is the default and canonical reference):

     > ❌ .grimoire/PROJECT.md does not exist.
     > `grimoire-note` edits the `Key Conventions` and `Notes` sections of `PROJECT.md` — without the file there is nowhere to write.
     > Run `/grimoire-init` first to create the base context, then re-invoke `/grimoire-note`.

   - STOP. Do **not** offer to run `grimoire-init` automatically. Do **not** create a skeleton.

5. **`[Phase 2: Input Parsing & Semantic Split]`**
   - Parse the user's free-text input.
   - Semantically split it into N distinct rules where applicable (one input may carry several). Single-rule inputs stay as one entry.
   - Note: the user-facing surface for the split happens at the Phase 5 pause-point ("detected N distinct rules: …") — Phase 2 only produces the structured list.

6. **`[Phase 3: Read & Synthesize Against Existing Sections]`**
   - Read `## Key Conventions / Constraints` and `## Notes` in full from `.grimoire/PROJECT.md`.
   - For each new rule from Phase 2, run an LLM-driven semantic pass against the existing entries in both sections to detect: duplication, refinement (the new rule sharpens an existing one), generalization (the new rule subsumes or is subsumed by an existing one), and contradiction.
   - For each detected relationship, propose the best synthesis: merge, rewrite, generalize, or accept as new. **Minimum-word phrasing is an explicit objective** — state this in the skill body so the agent optimizes for it.

7. **`[Phase 4: Retroactive Consolidation Pass]`**
   - Independently of the new input, scan all existing rules in both sections and propose rewrites where synthesis can be improved (consolidate near-duplicates already in the file, tighten verbose phrasings, merge adjacent rules that say the same thing).
   - This pass runs **every invocation**, not just when the new input triggers it. The cost is acceptable because the synthesis discipline keeps the file naturally small (see SPEC § Constraints).

8. **`[Phase 5: Section Assignment]`**
   - Heuristic:
     - **Key Conventions / Constraints** = normative / hard / "must" rules (`always`, `never`, `must`, `enforce`, "single source of truth", commit-message patterns, language rules).
     - **Notes** = observational / contextual / state ("the runtime currently announces X", "this repo dogfoods itself", "some operator messages are pt-BR by design").
   - When the rule is genuinely ambiguous (hedges like "tends to", "usually", "often"), ask the user via the available question tooling (e.g. `AskUserQuestion`) which section it belongs to before proposing the diff.
   - **Misfit handling.** When the rule fits neither section's intent (e.g., it would belong to Tech Stack or Repository Layout in spirit), force-fit into the closest section (default: `## Notes`) and flag the misfit explicitly in the Phase 6 diff narration so the user can override. **Never create a new section** — that power stays with `grimoire-init`.

9. **`[Phase 6: IDE-Aware Review]`** — per `§ IDE-aware review`:
   - **IDE detected:** write the proposed `PROJECT.md` content directly to `.grimoire/PROJECT.md` (its final destination — same pattern as `grimoire-init`). Tell the user in one short line that the draft is open and what answer you're waiting for, e.g. `📄 PROJECT.md atualizado em .grimoire/PROJECT.md — revise no editor e diga "ok" para seguir, ou liste mudanças.` (Operator-facing message may be pt-BR per project precedent.)
   - **Terminal-only fallback:** present the diff inline in chat.
   - When applicable, surface the Phase 2 split as a short prose narration at this checkpoint: `Detected N distinct rules: …`. Also surface any misfit force-fits from Phase 5.
   - PAUSE per `§ Pause-point pattern`. Wait for explicit user approval, requested edits, or abandonment.
   - On requested edits: re-apply Phases 3–5 with the user's corrections and re-pause.
   - On abandonment: restore `.grimoire/PROJECT.md` to its pre-edit state (read from `git show HEAD:.grimoire/PROJECT.md` and write it back) — never leave an unapproved file on disk.

10. **`[Phase 7: Commit]`** — per `§ Commits`:
    - One atomic commit per invocation, message exactly: `docs(grimoire): refine project context`.
    - The commit must touch **only** `.grimoire/PROJECT.md`. If any other file is dirty in the working tree, do not include it in this commit (warn the user instead).

### Style / discipline guardrails (state these inside the skill body, not just here)

- Operator-facing messages default to English; pt-BR is permitted (precedent: existing skills' hard-stop messages). Commit messages and SPEC-style content stay English.
- **Single source of truth.** The dual-writer contract is described in `GRIMOIRE-CONVENTIONS.md § Project context` — do not duplicate it here.
- **No `[Required Reading]` block change** in any other skill — `PROJECT.md` is already auto-loaded via `§ Project context`.

## Acceptance check before commit

Re-read the SPEC's `## Acceptance criteria` block for the SKILL.md bullets (frontmatter, hard-stop, parsing, dedup, retroactive consolidation, section assignment, misfit, IDE-aware review, commit message, never-touch-other-files). Verify each bullet is reflected in the file you wrote.

## Commit

Per `§ Commits`, one atomic commit covering only the new file:

```
docs(grimoire): add grimoire-note skill
```

The commit body is optional; if added, keep it to one short paragraph. Do **not** bundle any other file into this commit.

## Out of scope (do NOT do here)

- Do **not** edit `GRIMOIRE-CONVENTIONS.md` — that is Step 2.
- Do **not** edit `skills/grimoire-init/SKILL.md` — that is Step 3.
- Do **not** edit READMEs or `CLAUDE.md` — that is Step 4.
- Do **not** bump versions or edit `CHANGELOG.md` — that is Step 5.
- Do **not** touch `HISTORIC.md` — only the orchestrator updates page status (after Step 5).
- `§ TDD` does not apply (this is prompt text, not code). Do not write tests.

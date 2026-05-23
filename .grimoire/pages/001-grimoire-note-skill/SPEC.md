# 001 — grimoire-note Skill

## Context

`.grimoire/PROJECT.md` is the only persistent surface across sessions for project rules, conventions, and patterns, but it can only be written by `grimoire-init` — a heavyweight interview. This creates friction: when the user discovers a rule mid-work ("we always use snake_case in Postgres columns"), there is no fast path to capture it. Decisions and patterns stay in the operator's head or get buried inside SPEC files that rotate to `.grimoire/bag/historic/` after 5 entries.

This page introduces **`grimoire-note`**, a lightweight, independently-invocable skill that accepts free text and surgically updates `PROJECT.md` — no skeleton rewrite, no full interview. It evolves the prior invariant ("only `grimoire-init` writes `PROJECT.md`") into a dual-writer contract: `grimoire-init` owns the file as a whole; `grimoire-note` owns only the `## Key Conventions / Constraints` and `## Notes` sections, and only incrementally.

The skill is orthogonal to the Spec → Plan → Execute pipeline, like [grimoire-know](../../../skills/grimoire-know/SKILL.md) and [grimoire-update](../../../skills/grimoire-update/SKILL.md): it does not create a page folder, does not touch `HISTORIC.md`, and is not part of the page model.

## Goals

- Add a new skill at `skills/grimoire-note/SKILL.md` that accepts free-text input and updates `PROJECT.md` incrementally.
- Restrict `grimoire-note`'s write scope to the `## Key Conventions / Constraints` and `## Notes` sections of `PROJECT.md` only.
- Drive consolidation: every invocation reviews **all** existing rules in both sections and proposes rewrites where synthesis can be improved — not only those related to the new note.
- Detect duplication, refinement, and contradiction semantically (LLM-driven) and propose the best synthesis (merge, rewrite, generalize) before writing. Minimum-word phrasing is an explicit objective.
- Use the IDE-aware review pause-point: write the proposed `PROJECT.md` to disk so the IDE renders the diff, and commit only after explicit user approval.
- Split semantically when the input contains multiple rules: surface N distinct entries at the pause-point ("detected 3 distinct rules: …").
- Apply a heuristic for section assignment (Key Conventions = normative / hard / "must"; Notes = observational / contextual / state) and ask the user only when the rule is genuinely ambiguous (e.g., contains hedges like "tends to", "usually").
- When the input does not fit naturally in either section, force-fit into the closest section (default: Notes) and flag the misfit explicitly in the diff. No new sections are ever created — that power stays with `grimoire-init`.
- Amend `§ Project context` in `GRIMOIRE-CONVENTIONS.md` to describe the dual-writer contract in a single place.
- Update `grimoire-init`'s update mode so that when `## Key Conventions / Constraints` and `## Notes` already have content, the interview lists those entries and asks "anything stale here?" — preserves additions by default and only edits/removes on explicit user delta.
- Document the new skill in `README.md`, `README.pt-BR.md`, and `CLAUDE.md`.
- Bump plugin version (minor) and add a matching `CHANGELOG.md` entry.

## Non-goals

- **No new sections in `PROJECT.md`.** Section creation remains a `grimoire-init` responsibility.
- **No bootstrap.** If `.grimoire/PROJECT.md` does not exist, the skill hard-stops and instructs the user to run `grimoire-init` first. It never creates a skeleton.
- **No participation in the page model.** No page folder, no `HISTORIC.md` entry, no sub-agent execution loop, no commit-and-status dance.
- **No `[Required Reading]` changes in other skills.** `PROJECT.md` is already auto-loaded via `§ Project context`; no propagation needed.
- **No changes to other skills' hard-stop messages.** The language-adaptive policy applies only to `grimoire-note`'s own hard-stop (see Open questions for whether to generalize later).
- **No application of `§ TDD` or `§ Sub-agent spawning`.** This page produces prompt text, not code; the skill itself runs inline without spawning execution sub-agents.

## Scope

### Created files

- `skills/grimoire-note/SKILL.md` — full SKILL.md with YAML frontmatter (`name: grimoire-note`, a one-line `description`, an `argument-hint` for the free-text note input) and a prompt body covering:
  - `[Required Reading]` block loading `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md` (only `§ Project context`, `§ IDE-aware review`, `§ Commits` apply; the rest do not) and `.grimoire/PROJECT.md`.
  - Hard-stop precondition check for `.grimoire/PROJECT.md` existence.
  - Input parsing and semantic split into N distinct rules.
  - Full-section read of `## Key Conventions / Constraints` and `## Notes`, with LLM-driven semantic dedup, refinement, and contradiction detection against the new input.
  - Retroactive consolidation pass over all existing rules in both sections (independent of new input).
  - Heuristic section assignment plus ambiguity ask.
  - Misfit handling (force-fit + flag) when input matches neither section's intent.
  - Pause-point per `§ IDE-aware review` showing the full diff of `PROJECT.md`.
  - Atomic commit per `§ Commits` after approval.

### Modified files

- `GRIMOIRE-CONVENTIONS.md` — amend `§ Project context` to describe the dual-writer contract:
  - `grimoire-init` is the only skill that may create `PROJECT.md`, the only skill authorized to write outside the `## Key Conventions / Constraints` and `## Notes` sections, and the only skill that may add new sections.
  - `grimoire-note` is authorized to write only those two sections (incrementally), never creates the file, and never adds sections.
- `skills/grimoire-init/SKILL.md` — extend Phase 2 (Clarifying Questions, **update mode only**) so that when `## Key Conventions / Constraints` and/or `## Notes` already contain entries, the interview lists those entries to the user and asks which (if any) have gone stale; additions remain permitted, but the default action on existing entries is preserve-in-place. The full-rewrite of other sections in update mode is unchanged.
- `README.md` and `README.pt-BR.md` — add `grimoire-note` to the skills list with a one-line description matching the SKILL.md frontmatter.
- `CLAUDE.md` — add `grimoire-note` to the maintainer-facing "skills and how they connect" section alongside `grimoire-init`, `grimoire-update`, `grimoire-know`. Update any sentence that enumerates the seven skills.
- `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` — bump version in lockstep (minor: 0.6.0 → 0.7.0 assumed; `grimoire-plan` confirms).
- `CHANGELOG.md` — add a new version section covering: new `grimoire-note` skill, `§ Project context` amendment, `grimoire-init` update-mode change.

### Untouched

- All other SKILL.md files (`grimoire-spec`, `grimoire-plan`, `grimoire-execute`, `grimoire-quick`, `grimoire-update`, `grimoire-know`) — their `[Required Reading]` and bodies do not change.
- All other sections of `GRIMOIRE-CONVENTIONS.md` (`§ TDD`, `§ Sub-agent spawning`, `§ Commits`, `§ .grimoire/ layout`, `§ Historic`, `§ IDE-aware review`, `§ Pause-point pattern`) — only `§ Project context` is amended.
- The `[Required Reading]` blocks of all existing skills.

## Acceptance criteria

- `skills/grimoire-note/SKILL.md` exists with valid YAML frontmatter (`name: grimoire-note`, `description`, `argument-hint`) and a prompt body that:
  - References `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md` and explicitly states which sections apply (`§ Project context`, `§ IDE-aware review`, `§ Commits`) and which do not (`§ TDD`, `§ Sub-agent spawning`, `§ Historic`, `§ Pause-point pattern` for pipeline pages, `§ .grimoire/ layout` for pages).
  - Hard-stops if `.grimoire/PROJECT.md` does not exist. The canonical English message is:
    > ❌ .grimoire/PROJECT.md does not exist.
    > `grimoire-note` edits the `Key Conventions` and `Notes` sections of `PROJECT.md` — without the file there is nowhere to write.
    > Run `/grimoire-init` first to create the base context, then re-invoke `/grimoire-note`.

    The message may be rendered in the user's interaction language when detected (e.g., pt-BR); English is the default and the canonical reference.
  - Parses the user's free-text input, splits it semantically into N distinct rules when applicable, and surfaces the split at the pause-point (e.g., "detected 3 distinct rules: …").
  - Reads `## Key Conventions / Constraints` and `## Notes` in full and performs an LLM-driven semantic pass against the new input to detect duplication, refinement, generalization, and contradiction. Proposes the best synthesis (merge, rewrite, generalize) — minimum-word phrasing is an explicit objective stated in the skill body.
  - Performs a retroactive consolidation pass over all existing rules in both sections (independent of the new input) and proposes rewrites where synthesis can be improved.
  - Assigns each accepted rule to a section using the heuristic (Key Conventions = normative / hard / "must"; Notes = observational / contextual / state). When the rule is genuinely ambiguous, asks the user to choose before proposing the diff.
  - When a rule does not fit either section naturally, force-fits into the closest section (default: Notes) and surfaces the misfit explicitly in the diff narration so the user can override.
  - Presents the resulting `PROJECT.md` diff for review per `§ IDE-aware review`. Writes the proposed file content to `.grimoire/PROJECT.md` (its final destination) so the IDE renders it; on abandon, restores the file to its pre-edit state.
  - Commits per `§ Commits` with exactly: `docs(grimoire): refine project context`. One atomic commit per invocation, covering only the `PROJECT.md` edit.
  - Never creates a page folder, never touches `HISTORIC.md`, never spawns execute sub-agents, never modifies any file other than `PROJECT.md`.
- `GRIMOIRE-CONVENTIONS.md` `§ Project context` describes both writers and their scopes in-place (amendment, not new section). The single-source-of-truth discipline is preserved — no duplication of the contract into SKILL.md bodies.
- `skills/grimoire-init/SKILL.md` Phase 2 explicitly describes the preserve-by-default behavior for `## Key Conventions / Constraints` and `## Notes` in update mode, and prompts the user only about stale entries.
- `README.md`, `README.pt-BR.md`, and `CLAUDE.md` all list `grimoire-note` alongside the other six skills (so the total reads "seven skills" everywhere).
- `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` versions match and have been bumped.
- `CHANGELOG.md` has a section for the new version covering: new `grimoire-note` skill, `§ Project context` amendment, `grimoire-init` update-mode change.
- Manual smoke test passes on this repo: invoking `/grimoire-note "we always use snake_case in Postgres columns"` correctly proposes adding that rule to `Key Conventions`, shows a diff via the IDE-aware review path, and after approval produces a single `docs(grimoire): refine project context` commit touching only `.grimoire/PROJECT.md`.

## Constraints

- **Single source of truth.** Shared, load-bearing rules go to `GRIMOIRE-CONVENTIONS.md`. The dual-writer contract (which other skills also need to know about) lives in `§ Project context`. Skill-specific behavior (section heuristic, retroactive consolidation policy, semantic dedup approach, misfit handling) lives in the SKILL.md body.
- **Page vocabulary.** Any reference to the Spec → Plan → Execute model uses the term "page". `grimoire-note` itself does not create pages — it is orthogonal to the pipeline.
- **Languages.** SPEC content is English. Commit messages are Conventional Commits in English. Operator-facing messages inside the skill (hard-stop, pause-point prompts, diff narration) default to English and may switch to the user's interaction language when detected — consistent with the precedent that existing skills use pt-BR strings while SPEC/PLAN/commit content stays English.
- **IDE-aware review is required.** The skill must implement both the IDE-detected path (write to `.grimoire/PROJECT.md` for diff preview) and the terminal-only fallback (inline diff in chat).
- **No new `PROJECT.md` sections.** The skill writes only to `## Key Conventions / Constraints` and `## Notes`. Adding new sections remains `grimoire-init`'s exclusive responsibility.
- **No bypass on missing file.** When `PROJECT.md` is absent, hard-stop with the canonical message. Never silently create a skeleton.
- **Semantic detection budget.** Retroactive consolidation reviews **all** existing rules in both sections per invocation. The cost is acceptable because the synthesis discipline keeps the file naturally small.
- **`§ TDD` does not apply.** This page produces prompt text (SKILL.md, conventions edit, docs, manifests, changelog). No code, no test suite.
- **`grimoire-note` itself does not register a `HISTORIC.md` entry.** Only the page that introduces `grimoire-note` (this one) is registered in `HISTORIC.md`, per `§ Historic` as bootstrapped by this `grimoire-spec` run.
- **Atomic edits per file class.** The `grimoire-plan` step files chosen for this page must keep skill-body, conventions, init-edit, docs, and version-bump in separate atomic commits (per `§ Commits`, "one logical change per commit").

## Open questions

- **Language-adaptive hard-stop rollout.** Should `§ Pause-point pattern` later be amended so all hard-stop messages (currently hard-coded pt-BR in `grimoire-plan` and `grimoire-execute`) become language-adaptive like the new `grimoire-note` hard-stop? Out of scope for this page — track separately if desired.
- **Heuristic formalization in conventions.** The Key Conventions vs Notes heuristic lives in the new skill body for now. If another skill ever needs to reference it, lift to `§ Project context` or a new sub-section. Not lifted preemptively to keep the conventions file lean.
- **Step-file breakdown.** A reasonable split for `grimoire-plan` is: (1) new `skills/grimoire-note/SKILL.md` + `§ Project context` amendment; (2) `skills/grimoire-init/SKILL.md` Phase 2 update; (3) docs (`README.md`, `README.pt-BR.md`, `CLAUDE.md`) + version bump (`plugin.json`, `marketplace.json`) + `CHANGELOG.md` entry. Final breakdown is `grimoire-plan`'s call.
- **Version bump magnitude.** Default assumption is minor (`0.6.0` → `0.7.0`), matching the precedent of `grimoire-know`'s `0.5.0` → `0.6.0`. `grimoire-plan` should confirm at planning time.

## References

- [GRIMOIRE-CONVENTIONS.md](../../../GRIMOIRE-CONVENTIONS.md) — `§ Project context` (amendment target), `§ IDE-aware review`, `§ Commits`, `§ Historic`.
- [skills/grimoire-init/SKILL.md](../../../skills/grimoire-init/SKILL.md) — current writer of `PROJECT.md`; Phase 2 (update mode) is the integration point for the dual-writer contract.
- [skills/grimoire-know/SKILL.md](../../../skills/grimoire-know/SKILL.md) — closest precedent for an "orthogonal-to-pipeline" skill; reference for the "which conventions apply / which do not" pattern in `[Required Reading]`.
- [skills/grimoire-update/SKILL.md](../../../skills/grimoire-update/SKILL.md) — another orthogonal skill; reference for "no page, no HISTORIC, no commits-in-consumer" framing and IDE-aware draft handling.
- [.grimoire/PROJECT.md](../../../.grimoire/PROJECT.md) — dogfooded example of `PROJECT.md` structure, including the `## Key Conventions / Constraints` and `## Notes` sections this skill targets.
- [.claude-plugin/plugin.json](../../../.claude-plugin/plugin.json) and [.claude-plugin/marketplace.json](../../../.claude-plugin/marketplace.json) — version files; both must be bumped in lockstep.
- [CHANGELOG.md](../../../CHANGELOG.md) — pattern for release notes.
- [CLAUDE.md](../../../CLAUDE.md) — maintainer-facing skill enumeration; must include `grimoire-note`.

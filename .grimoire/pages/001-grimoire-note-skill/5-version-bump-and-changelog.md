# Step 5 — Bump version to 0.7.0 and add `CHANGELOG.md` entry

## Source of truth

Read [`.grimoire/pages/001-grimoire-note-skill/SPEC.md`](./SPEC.md), specifically:
- **Goals → "Bump plugin version (minor) and add a matching `CHANGELOG.md` entry."**
- **Scope → Modified files → `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`**, and **`CHANGELOG.md`**.
- **Acceptance criteria → "`.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` versions match and have been bumped."** and **"`CHANGELOG.md` has a section for the new version covering: new `grimoire-note` skill, `§ Project context` amendment, `grimoire-init` update-mode change."**
- **Open questions → "Version bump magnitude" → confirmed at planning time as `0.6.0 → 0.7.0` (minor), matching the precedent of `grimoire-know`'s `0.5.0 → 0.6.0` bump.**

This step performs the release-shipping commit. After this commit, the page is fully implemented and ready for the planner-orchestrator's HISTORIC status update.

## Files to edit

### 1. `.claude-plugin/plugin.json`

Bump `"version"` from `"0.6.0"` to `"0.7.0"`. No other field changes.

### 2. `.claude-plugin/marketplace.json`

Bump `"version"` (inside `plugins[0]`) from `"0.6.0"` to `"0.7.0"`. No other field changes. CI enforces that this matches `plugin.json`.

### 3. `CHANGELOG.md`

Prepend a new top section for `[0.7.0]` immediately above the existing `[0.6.0]` section. Use today's date — **2026-05-23** — in the section header.

Required content (use Keep-a-Changelog headings `### Added` and `### Changed` as the file already does):

- **`### Added`**
  - New `grimoire-note` skill — describe in one prose paragraph matching the density of the existing `[0.6.0]` "grimoire-know" entry. Cover: free-text-in / surgical-edit-out, scope (only `## Key Conventions / Constraints` and `## Notes` in `.grimoire/PROJECT.md`), LLM-driven semantic dedup + refinement + contradiction + retroactive consolidation, heuristic section assignment with ambiguity ask, misfit force-fit + flag, IDE-aware diff review, single atomic commit `docs(grimoire): refine project context`, orthogonal-to-pipeline framing (same family as `grimoire-know` and `grimoire-update`).
- **`### Changed`**
  - `GRIMOIRE-CONVENTIONS.md § Project context` amended to a **dual-writer contract**: `grimoire-init` owns the file as a whole (bootstrap, all sections, section creation); `grimoire-note` owns only `## Key Conventions / Constraints` and `## Notes`, incrementally.
  - `grimoire-init` Phase 2 (UPDATE mode only): preserve-by-default for `## Key Conventions / Constraints` and `## Notes` — existing entries are listed back to the user with an "anything stale?" prompt, additions remain permitted, silence keeps everything.

After the section content, append the new reference-link line at the bottom of the file (where `[0.6.0]: …` etc. are listed):

```
[0.7.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.7.0
```

Preserve all existing reference-link lines unchanged.

## Pre-work reading

1. [`.claude-plugin/plugin.json`](../../../../.claude-plugin/plugin.json) and [`.claude-plugin/marketplace.json`](../../../../.claude-plugin/marketplace.json) — confirm current version is `0.6.0` in both.
2. [`CHANGELOG.md`](../../../../CHANGELOG.md) — read the `[0.6.0]` and `[0.5.0]` sections so you match their density, section-heading style, and reference-link pattern at the bottom of the file.
3. [`skills/grimoire-note/SKILL.md`](../../../../skills/grimoire-note/SKILL.md) (Step 1 output), the amended `GRIMOIRE-CONVENTIONS.md § Project context` (Step 2 output), and updated `skills/grimoire-init/SKILL.md` Phase 2 (Step 3 output) — these are the three changes the `[0.7.0]` entry documents. Read them so the changelog accurately describes the shipped behavior.

## Style guardrails

- **Lockstep version bump.** Both `plugin.json` and `marketplace.json` must change from `0.6.0` → `0.7.0` in the same commit. CI fails the release if they diverge.
- **Date is 2026-05-23** in the `[0.7.0]` header.
- **Match Keep-a-Changelog tone.** Use `### Added` and `### Changed`. No `### Removed`, `### Deprecated`, `### Fixed`, `### Security` here — they do not apply.
- **Reference link at the bottom.** Add the `[0.7.0]` link; do not edit any other existing link.
- **No release-workflow tweaks.** This step is content-only. Do not touch `.github/workflows/release.yml`.

## Acceptance check before commit

- `plugin.json` version is `0.7.0`.
- `marketplace.json` (inside `plugins[0]`) version is `0.7.0`.
- `CHANGELOG.md` has a new `[0.7.0] — 2026-05-23` section above `[0.6.0]`, covering: new `grimoire-note` skill (Added) + dual-writer contract amendment + `grimoire-init` update-mode change (Changed).
- The `[0.7.0]: …` reference-link line is appended at the bottom; the existing reference-link lines are unchanged.
- No other file is modified.

## Commit

Per `§ Commits`, one atomic commit covering all three files (plugin.json + marketplace.json + CHANGELOG.md). Matching the precedent of the `0.6.0` release commit (`chore(release): bump version to 0.6.0 for grimoire-know`):

```
chore(release): bump version to 0.7.0 for grimoire-note
```

## Out of scope (do NOT do here)

- Do **not** edit any `SKILL.md` file, `GRIMOIRE-CONVENTIONS.md`, READMEs, or `CLAUDE.md` — Steps 1–4 own those.
- Do **not** touch `HISTORIC.md` — the orchestrator updates page status from `[planned]` to `[finished]` only after a successful `/grimoire-execute 1` run.
- Do **not** edit `.github/workflows/release.yml`.
- Do **not** push, tag, or trigger the release manually. Pushing `main` with the bumped version is the release trigger; that is a separate, user-driven action.
- `§ TDD` does not apply.

## End-of-run state

After this step's commit, the working tree should be clean and the page is implemented end-to-end. The orchestrator (`grimoire-execute`) will then update `.grimoire/HISTORIC.md` to mark `001-grimoire-note-skill` as `[finished]`.

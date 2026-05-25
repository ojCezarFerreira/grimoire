# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.8.0] — 2026-05-25

### Added
- `GRIMOIRE-CONVENTIONS.md` — new `§ Parallel execution` section: opt-in YAML frontmatter (`depends-on`, `touches`) on step files unlocks a dependency-graph execution model for `grimoire-execute`. The orchestrator builds a DAG, runs Kahn-style topological waves, spawns one sub-agent per step inside a deterministic git worktree under `.grimoire/bag/worktrees/page-NNN-step-K/`, cherry-picks successful siblings back to the main workspace in step-number order, and tears down every worktree unconditionally (`git worktree remove --force` + defensive filesystem cleanup) before the next wave and at end of run — `.grimoire/bag/worktrees/` and `git worktree list` are guaranteed clean after every run, success or failure.

### Changed
- `GRIMOIRE-CONVENTIONS.md § Sub-agent spawning` amended to admit parallel sub-agents inside a single wave when the page's step files declare `depends-on` frontmatter; sequencing across waves remains strict; legacy strict-serial is the default when no step file carries frontmatter.
- `GRIMOIRE-CONVENTIONS.md § .grimoire/ layout` notes that step files MAY carry YAML frontmatter per `§ Parallel execution`.
- `GRIMOIRE-CONVENTIONS.md § Pause-point pattern`:
  - `grimoire-plan: final clarity check` extended to require DAG presentation (wave list + per-step deps + advisory `touches`-overlap warnings) when frontmatter will be emitted, so the user can veto a suspect parallelization before step files are written.
  - `grimoire-execute: precondition check (HARD STOP)` gains two new hard-stop cases: cyclic `depends-on` graph and `depends-on` reference to a non-existent step.
- `skills/grimoire-plan/SKILL.md` Phase 3 extended: when steps are genuinely independent, the planner MAY emit the new frontmatter; builds and validates the DAG (cycle + dangling-reference detection, advisory `touches`-overlap), computes waves, surfaces the wave plan at the final-clarity-check pause-point, and falls back to legacy serial if the user vetoes parallelization. Phase 4 writes the YAML frontmatter block on each step file when the approved DAG calls for it.
- `skills/grimoire-execute/SKILL.md` Phase 1 detects frontmatter and selects between the legacy strict-serial path and the parallel-wave path. Phase 2 is split into 2A (legacy) and 2B (parallel-wave: worktree creation, concurrent sub-agent spawn, wave-settle wait, cherry-pick in step-number order, unconditional teardown as a try/finally invariant). Phase 4 splits finalization: fully-successful runs mark `[finished]`; partial-failure runs keep HISTORIC at `[planned]` and report finished / failed / skipped steps to the user.
- `README.md`, `README.pt-BR.md`, `CLAUDE.md` describe the new wave model in their `grimoire-plan` and `grimoire-execute` entries, pointing to `§ Parallel execution` for details.

### Compatibility
- **Pure opt-in.** Pages whose step files declare no `depends-on` frontmatter execute byte-identically to pre-0.8.0 — no worktrees created, no behavior change. Existing pages (e.g., `001-grimoire-note-skill`) are unaffected; no auto-migration.

## [0.7.0] — 2026-05-23

### Added
- New `grimoire-note` skill: free-text-in, surgical-edit-out maintenance of `.grimoire/PROJECT.md`. Scope is strictly limited to the `## Key Conventions / Constraints` and `## Notes` sections; the rest of the file is owned by `grimoire-init`. Uses LLM-driven semantic deduplication, refinement of stale phrasing, contradiction handling, and retroactive consolidation across pre-existing entries; assigns each incoming line heuristically to Conventions vs. Notes and pauses to ask when ambiguous; force-fits content that does not match either section into the closer one and flags the misfit back to the user. Honors `§ IDE-aware review` for diff inspection and lands a single atomic commit `docs(grimoire): refine project context`. Writes nothing outside the two governed sections, creates no page folder, and does not touch `HISTORIC.md` — fully orthogonal to the Spec → Plan → Execute pipeline (same family as `grimoire-know` and `grimoire-update`).

### Changed
- `GRIMOIRE-CONVENTIONS.md § Project context` amended to a **dual-writer contract**: `grimoire-init` owns `.grimoire/PROJECT.md` as a whole (bootstrap, every section, section creation), while `grimoire-note` owns only `## Key Conventions / Constraints` and `## Notes` and edits them incrementally.
- `grimoire-init` Phase 2 (UPDATE mode only): preserve-by-default for `## Key Conventions / Constraints` and `## Notes`. The skill now lists existing entries back to the user with an "anything stale?" prompt; additions remain permitted, and silence keeps every existing entry untouched.

## [0.6.0] — 2026-05-22

### Added
- New `grimoire-know` skill: read-only Q&A about the repository or the application it builds. Spawns a sub-agent with `WebSearch`/`WebFetch` access for facts outside the repo, returns the answer with a "References" list of consulted URLs, and explicitly flags anything it is not confident about. Writes nothing, makes no commits, and does not touch `HISTORIC.md` or `.grimoire/pages/` — it is fully ephemeral and orthogonal to the Spec → Plan → Execute pipeline (like `grimoire-update`).

## [0.5.0] — 2026-05-22

### Added
- New pause-point in `GRIMOIRE-CONVENTIONS.md § Pause-point pattern`: **final clarity check** for both `grimoire-plan` and `grimoire-quick`. Before a plan is closed and presented to the user, the agent must self-review for hidden assumptions and surface them via the available question tooling (e.g. `AskUserQuestion`) instead of silently guessing. Skip allowed only when the plan is fully unambiguous.

### Changed
- `grimoire-plan` Phase 3: ends with the new final clarity check before Phase 4 writes step files.
- `grimoire-quick` Phase 2: runs the new final clarity check before composing/presenting the plan draft, replacing the prior "ask any clarifying questions if necessary" wording with an explicit self-review + pause requirement.

## [0.4.2] — 2026-05-22

### Changed
- Ephemeral draft files in `§ IDE-aware review` now include a Unix-epoch suffix in their filenames (`quick-plan-<epoch>.md`, `grimoire-update-changelog-<epoch>.md`) so each run writes to a unique path. This defeats the VSCode markdown preview cache, which is keyed by file path — previously, a second invocation could show the stale preview from the prior run. Final-destination drafts (`PROJECT.md`, `SPEC.md`) are unaffected and keep their canonical names.
- `grimoire-quick` Phases 2/5 and `grimoire-update` Phases 4/5/8: write and cleanup now use the memorized per-run path instead of a fixed filename.

## [0.4.1] — 2026-05-21

### Added
- New `§ IDE-aware review` convention in `GRIMOIRE-CONVENTIONS.md`: when Claude Code is running inside an IDE extension (VSCode, JetBrains, Cursor, etc.), skills now write the draft/plan to disk so the editor renders it with full markdown preview, instead of pasting raw markdown into chat. Falls back to inline-chat output when no IDE is detected.

### Changed
- `grimoire-init` draft review (Phase 3/4): writes the proposed `PROJECT.md` straight to its final path in IDE mode for review; commit phase unchanged.
- `grimoire-spec` draft review (Phase 4/5): writes the proposed `SPEC.md` straight to its final page-folder path in IDE mode for review; commit phase unchanged.
- `grimoire-quick` plan authorization (Phase 2): writes the plan to `.grimoire/bag/drafts/quick-plan.md` in IDE mode and adds a Phase 5 that deletes the draft after execution (or on abandonment).
- `grimoire-update` changelog display (Phase 4): writes the extracted changelog slice to `.grimoire/bag/drafts/grimoire-update-changelog.md` in IDE mode; Phases 5/8 delete the draft on rejection or completion. Notes/Objective updated to reflect that the skill may now write an ephemeral draft.

## [0.4.0] — 2026-05-21

### Added
- `grimoire-update` skill: guided plugin self-update. Reads the installed `plugin.json` version, fetches the latest from GitHub, prints the changelog diff between the two, and then walks the user through `/plugin marketplace update grimoire` + `/reload-plugins`. Exits early when already on the latest version. Writes nothing to the consumer project and makes no commits.

### Changed
- README and CLAUDE.md updated to document the new sixth skill.

## [0.3.0] — 2026-05-21

### Added
- `grimoire-spec` skill: page specification with HISTORIC.md bootstrap/rotation.
- `grimoire-plan` and `grimoire-execute` accept page numbers with precondition checks.

### Changed
- `grimoire-init` and `grimoire-quick` aligned with the Spec → Plan → Execute pipeline.
- README and CLAUDE.md updated for the new pipeline.

[0.8.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.8.0
[0.7.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.7.0
[0.6.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.6.0
[0.5.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.5.0
[0.4.2]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.4.2
[0.4.1]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.4.1
[0.4.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.4.0
[0.3.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.3.0

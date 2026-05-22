# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

[0.4.2]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.4.2
[0.4.1]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.4.1
[0.4.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.4.0
[0.3.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.3.0

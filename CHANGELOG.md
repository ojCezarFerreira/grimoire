# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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

[0.4.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.4.0
[0.3.0]: https://github.com/ojCezarFerreira/grimoire/releases/tag/v0.3.0

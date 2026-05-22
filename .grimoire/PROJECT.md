# Grimoire

## Purpose

Grimoire exists because other Claude Code context orchestrators kept producing mediocre output — too heavy, too chatty, too eager to take decisions away from the user. Projects drifted from what the author actually wanted.

This plugin is the opposite: a disciplined **Spec → Plan → Execute** pipeline (with a **Quick** fast-path, an **Init** step for shared project context, and an **Update** step for plugin self-maintenance) that asks until the user is clear on what they want, then ships exactly that. It will help think through a fuzzy idea, but it will not pretend to know the project better than the user, and it will not silently choose for them.

The unit of work is the **page**: a folder under `.grimoire/pages/NNN-[page-name]/` containing a `SPEC.md` and one or more sequential step files, plus a single entry in `.grimoire/HISTORIC.md` whose status moves `[spec]` → `[planned]` → `[finished]`.

## Audience

Developers using Claude Code who can read code, evaluate a plan, and own the decisions a workflow surfaces. Not for non-technical users — Grimoire assumes the operator can push back on a plan and judge whether a SPEC is sufficient.

## Tech Stack

- **No source code, no build system, no test suite.** Every artifact in this repo is markdown or JSON. Edits land in the prompt text of `SKILL.md` files and `GRIMOIRE-CONVENTIONS.md`.
- **Plugin manifest:** `.claude-plugin/plugin.json` (Anthropic's required path). `marketplace.json` lives next to it; both versions must stay in sync.
- **Skills:** one `skills/<name>/SKILL.md` per skill, each with YAML frontmatter (`name`, `description`, `argument-hint`) and a prompt body. Auto-discovered by Claude Code from `skills/` — no `skills` field in the manifest.
- **Shared workflow rules:** a single `GRIMOIRE-CONVENTIONS.md` at the repo root, referenced from every SKILL.md via `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md`.
- **Release automation:** `.github/workflows/release.yml` — pushes to `main` that change `.claude-plugin/plugin.json` cut a tag, validate that `plugin.json` and `marketplace.json` versions match, extract the matching CHANGELOG section, and publish a GitHub Release.
- **Docs are bilingual:** `README.md` (English) and `README.pt-BR.md` (Portuguese — Brazil). User-facing strings in skills are sometimes Portuguese; the spirit is: messages to the operator can be PT-BR, code/commit/spec content is English.

## Repository Layout

```
.claude-plugin/
  plugin.json          ← required manifest path; version source of truth
  marketplace.json     ← version must match plugin.json (CI enforces)
skills/
  grimoire-init/SKILL.md
  grimoire-spec/SKILL.md
  grimoire-plan/SKILL.md
  grimoire-execute/SKILL.md
  grimoire-quick/SKILL.md
  grimoire-update/SKILL.md
GRIMOIRE-CONVENTIONS.md  ← single source of truth for workflow rules
CLAUDE.md                ← maintainer-only notes (NOT loaded for end users)
README.md / README.pt-BR.md
CHANGELOG.md
LICENSE                  ← MIT
assets/                  ← logo/images for README
.github/workflows/release.yml
```

## Key Conventions / Constraints

- **`GRIMOIRE-CONVENTIONS.md` is the single source of truth** for TDD, sub-agent spawning, commits, `.grimoire/` layout, project context, historic, IDE-aware review, and pause-point pattern. Never duplicate any of those rules inside a `SKILL.md` body — when a rule changes, it changes there once.
- **Conventional Commits, in English.** `feat:`, `fix:`, `test:`, `refactor:`, `chore:`, `style:`, `docs:`. One logical change per commit, atomic.
- **`docs(grimoire): ...` is the namespace for plugin-content changes** to skill bodies, conventions, and the README/CHANGELOG.
- **SKILL.md frontmatter is load-bearing.** Don't rename `name`, `description`, or `argument-hint`; don't drop the `---` delimiters. The skills system consumes the frontmatter verbatim.
- **`$ARGUMENTS` and `$1` in prompt bodies are substitutions** the Claude Code runtime fills at invocation. Preserve them verbatim.
- **`${CLAUDE_PLUGIN_ROOT}` is the runtime-substituted absolute path** of the installed plugin. Use it whenever a SKILL.md references a sibling file (e.g., `GRIMOIRE-CONVENTIONS.md`).
- **Releases are version-driven.** Bumping `.claude-plugin/plugin.json` on `main` is the release trigger. `marketplace.json` must match. CHANGELOG must already have a section for the new version (the release workflow fails otherwise).
- **A new CHANGELOG entry is part of the release commit, not a follow-up.** The release workflow extracts release notes from it.
- **`CLAUDE.md` at the plugin root is NOT loaded for end users.** Per Anthropic's plugin spec, it is ignored at install time. End-user context comes from the skills themselves and from `GRIMOIRE-CONVENTIONS.md`.
- **Vocabulary discipline.** Every change Grimoire tracks is a **page**, not a "feature" / "ticket" / "task" / "plan". Use "page" consistently in skill bodies, docs, and commit messages where the unit of work is referenced.

## Current Status

Early/active — brand-new plugin, working day-to-day for the author but not yet battle-tested across many projects or teams. Conventions and skill bodies are still evolving (recent CHANGELOG entries iterate on the IDE-aware review flow, ephemeral-draft path hygiene, and the new final-clarity-check pause point). **No stability promise across minor versions** — breaking changes between minors are expected until things settle. Issues and PRs are welcome.

## Notes

- The runtime currently announces a **VSCode native extension** environment for the maintainer, so the IDE-aware review path (`§ IDE-aware review`) is the active flow when developing the plugin itself — drafts are written to disk for preview, not pasted inline.
- Some operator-facing messages in skill bodies (e.g., the hard-stop messages in `§ Pause-point pattern`) are in Portuguese-BR by design; SPEC/PLAN/commit content stays in English.
- The `.grimoire/` directory in this repo is the plugin **dogfooding** itself — pages authored here track changes to the plugin source.

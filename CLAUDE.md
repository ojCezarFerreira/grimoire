# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Note: this CLAUDE.md is for developers maintaining the Grimoire plugin. It is **not** loaded as context for end users — per Anthropic's plugin spec, a CLAUDE.md at the plugin root is ignored at install time. End-user context comes from the skills and from [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md).

## What this repo is

A Claude Code **plugin** ([.claude-plugin/plugin.json](.claude-plugin/plugin.json) at the root) bundling four skills that define the **Grimoire workflow**: a disciplined Plan → Execute pipeline with a Quick fast-path, plus an Init step that establishes per-project context.

Each skill is a single `skills/<name>/SKILL.md` file with YAML frontmatter (`name`, `description`, `argument-hint`) and a prompt body: [skills/grimoire-init/SKILL.md](skills/grimoire-init/SKILL.md), [skills/grimoire-plan/SKILL.md](skills/grimoire-plan/SKILL.md), [skills/grimoire-execute/SKILL.md](skills/grimoire-execute/SKILL.md), [skills/grimoire-quick/SKILL.md](skills/grimoire-quick/SKILL.md).

The shared, load-bearing workflow rules — TDD, sub-agent spawning, commits, `.grimoire/` layout, project context, pause points — live in a **single source of truth**: [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md). Each SKILL.md references it via `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md` so the rules aren't duplicated across skill bodies.

There is no source code, no build system, no test suite — edits land in the prompt text of the markdown files above.

## The skills and how they connect

- **`grimoire-init`** — Analyzes the *consuming* project and interviews the user, then writes `.grimoire/PROJECT.md` (purpose, audience, tech stack, layout, conventions, status). Every other Grimoire skill reads that file at the top of its `[Required Reading]`. Supports both initialization and update mode; pauses for clarifying questions and draft review before writing.
- **`grimoire-plan`** — Writes plan files into `.grimoire/` in the *consuming* project. Single plans get `.grimoire/NNN-[feature].md`; large features get split into `.grimoire/NNN-[feature]/NNN.X-[subplan].md`. The plan-or-split decision is the model's call based on projected context size during execution.
- **`grimoire-execute`** — Takes a path (file or directory). For a directory, executes sub-plans in strict numeric order and **spawns one sub-agent per sub-plan** to preserve context lucidity. On success, moves the plan file or whole directory into `.grimoire/finished/`.
- **`grimoire-quick`** — Fast-path for small fixes. Has a **scope gatekeeper**: if the task is too large, it must STOP and tell the user to switch to `grimoire-plan`. Requires explicit user authorization of the inline plan before any code is written.

## Shared conventions

All shared, load-bearing rules live in [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md):

- § TDD (Red/Green/Refactor + the filter-only rule)
- § Sub-agent spawning
- § Commits (atomic + Conventional Commits in English)
- § `.grimoire/` layout (NNN naming, sub-plan folders, `.grimoire/finished/`, `.grimoire/PROJECT.md`)
- § Project context (every skill loads `.grimoire/PROJECT.md` if present; missing → suggest `grimoire-init`; only `grimoire-init` writes it)
- § Pause-point pattern (grimoire-init: clarifying questions + draft review; grimoire-plan: unclear requirements; grimoire-quick: scope gatekeeper + plan authorization)

**Do not duplicate these rules in SKILL.md bodies.** When a rule changes, update `GRIMOIRE-CONVENTIONS.md` only.

## Editing notes

- SKILL.md frontmatter is consumed by the Claude Code skills system — don't rename `name`, `description`, or `argument-hint`, and don't drop the leading `---` delimiters.
- `$ARGUMENTS` and `$1` in the prompt bodies are skill-invocation substitutions — preserve them verbatim.
- `${CLAUDE_PLUGIN_ROOT}` is substituted by Claude Code at runtime to the absolute path of the installed plugin. Use it whenever a SKILL.md references a sibling file like `GRIMOIRE-CONVENTIONS.md`.
- When you add or remove a section in `GRIMOIRE-CONVENTIONS.md`, update the `[Required Reading]` block at the top of each SKILL.md that depends on it.
- `plugin.json` lives at `.claude-plugin/plugin.json` (Anthropic's required path). Skills are auto-discovered from `skills/` — do not add a `skills` field to the manifest unless adding custom paths.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Note: this CLAUDE.md is for developers maintaining the Grimoire plugin. It is **not** loaded as context for end users — per Anthropic's plugin spec, a CLAUDE.md at the plugin root is ignored at install time. End-user context comes from the skills and from [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md).

## What this repo is

A Claude Code **plugin** ([.claude-plugin/plugin.json](.claude-plugin/plugin.json) at the root) bundling four skills that define the **Grimoire workflow**: a disciplined Plan → Execute pipeline with a Quick fast-path, plus an Init step that establishes per-project context.

Each skill is a single `skills/<name>/SKILL.md` file with YAML frontmatter (`name`, `description`, `argument-hint`) and a prompt body: [skills/grimoire-init/SKILL.md](skills/grimoire-init/SKILL.md), [skills/grimoire-plan/SKILL.md](skills/grimoire-plan/SKILL.md), [skills/grimoire-execute/SKILL.md](skills/grimoire-execute/SKILL.md), [skills/grimoire-quick/SKILL.md](skills/grimoire-quick/SKILL.md).

The shared, load-bearing workflow rules — TDD, sub-agent spawning, commits, `.grimoire/` layout, project context, historic, pause points — live in a **single source of truth**: [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md). Each SKILL.md references it via `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md` so the rules aren't duplicated across skill bodies.

There is no source code, no build system, no test suite — edits land in the prompt text of the markdown files above.

## Vocabulary: "page"

Every feature, change, or alteration tracked by Grimoire is called a **page**. A page is the unit of work: it has a folder (`.grimoire/pages/NNN-[page-name]/`), one or more sequential step files inside (`1-[step].md`, `2-[step].md`, …), and a single entry in `.grimoire/HISTORIC.md` with status `[planned]` or `[finished]`. When this repo (or any skill body) refers to "the feature", "the alteration", "the plan", or "the work", it means **the page**. Use this term consistently in any new docs or skill edits.

## The skills and how they connect

- **`grimoire-init`** — Analyzes the *consuming* project and interviews the user, then writes `.grimoire/PROJECT.md` (purpose, audience, tech stack, layout, conventions, status). Every other Grimoire skill reads that file at the top of its `[Required Reading]`. Supports both initialization and update mode; pauses for clarifying questions and draft review before writing. Does **not** write `HISTORIC.md`.
- **`grimoire-plan`** — Writes the page into `.grimoire/pages/NNN-[page-name]/` in the *consuming* project, always as a folder with one or more sequential step files (`1-[step].md`, `2-[step].md`, …). Then bootstraps/appends/rotates `.grimoire/HISTORIC.md` to register the page with status `[planned]`. The number-of-steps decision is the model's call based on projected context lucidity per step.
- **`grimoire-execute`** — Takes a path to a page folder (or one of its step files; resolves up to the folder). Executes step files in strict numeric order and **spawns one sub-agent per step file** to preserve context lucidity. On success, updates the page's `HISTORIC.md` entry from `[planned]` to `[finished]` in place — **no files are moved**.
- **`grimoire-quick`** — Fast-path for small fixes. Has a **scope gatekeeper**: if the task is too large, it must STOP and tell the user to switch to `grimoire-plan`. Requires explicit user authorization of the inline plan before any code is written. Stays fully ephemeral: no page folder, no `HISTORIC.md` entry.

## Shared conventions

All shared, load-bearing rules live in [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md):

- § TDD (Red/Green/Refactor + the filter-only rule)
- § Sub-agent spawning
- § Commits (atomic + Conventional Commits in English; `chore: register page <name> in historic` after plan, `chore: mark page <name> finished` after execute)
- § `.grimoire/` layout (every page lives in `.grimoire/pages/NNN-[page-name]/` with sequential step files; no `finished/` directory; `.grimoire/PROJECT.md` for project context)
- § Project context (every skill loads `.grimoire/PROJECT.md` if present; missing → suggest `grimoire-init`; only `grimoire-init` writes it)
- § Historic (`.grimoire/HISTORIC.md` keeps the last 5 pages with `[planned]`/`[finished]` status; `grimoire-plan` bootstraps, appends, and rotates to `.grimoire/bag/historic/HISTORIC-N.md`; `grimoire-execute` only updates status in place)
- § Pause-point pattern (grimoire-init: clarifying questions + draft review; grimoire-plan: unclear requirements; grimoire-quick: scope gatekeeper + plan authorization)

**Do not duplicate these rules in SKILL.md bodies.** When a rule changes, update `GRIMOIRE-CONVENTIONS.md` only.

## Editing notes

- SKILL.md frontmatter is consumed by the Claude Code skills system — don't rename `name`, `description`, or `argument-hint`, and don't drop the leading `---` delimiters.
- `$ARGUMENTS` and `$1` in the prompt bodies are skill-invocation substitutions — preserve them verbatim.
- `${CLAUDE_PLUGIN_ROOT}` is substituted by Claude Code at runtime to the absolute path of the installed plugin. Use it whenever a SKILL.md references a sibling file like `GRIMOIRE-CONVENTIONS.md`.
- When you add or remove a section in `GRIMOIRE-CONVENTIONS.md`, update the `[Required Reading]` block at the top of each SKILL.md that depends on it.
- `plugin.json` lives at `.claude-plugin/plugin.json` (Anthropic's required path). Skills are auto-discovered from `skills/` — do not add a `skills` field to the manifest unless adding custom paths.

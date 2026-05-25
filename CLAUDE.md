# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Note: this CLAUDE.md is for developers maintaining the Grimoire plugin. It is **not** loaded as context for end users — per Anthropic's plugin spec, a CLAUDE.md at the plugin root is ignored at install time. End-user context comes from the skills and from [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md).

## What this repo is

A Claude Code **plugin** ([.claude-plugin/plugin.json](.claude-plugin/plugin.json) at the root) bundling eight skills that define the **Grimoire workflow**: a disciplined Spec → Plan → Execute pipeline with a Quick fast-path, plus an Init step that establishes per-project context and an Update step that maintains the plugin itself.

Each skill is a single `skills/<name>/SKILL.md` file with YAML frontmatter (`name`, `description`, `argument-hint`) and a prompt body: [skills/grimoire-init/SKILL.md](skills/grimoire-init/SKILL.md), [skills/grimoire-spec/SKILL.md](skills/grimoire-spec/SKILL.md), [skills/grimoire-plan/SKILL.md](skills/grimoire-plan/SKILL.md), [skills/grimoire-execute/SKILL.md](skills/grimoire-execute/SKILL.md), [skills/grimoire-quick/SKILL.md](skills/grimoire-quick/SKILL.md), [skills/grimoire-know/SKILL.md](skills/grimoire-know/SKILL.md), [skills/grimoire-update/SKILL.md](skills/grimoire-update/SKILL.md), [skills/grimoire-note/SKILL.md](skills/grimoire-note/SKILL.md).

The shared, load-bearing workflow rules — TDD, sub-agent spawning, commits, `.grimoire/` layout, project context, historic, pause points — live in a **single source of truth**: [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md). Each SKILL.md references it via `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md` so the rules aren't duplicated across skill bodies.

There is no source code, no build system, no test suite — edits land in the prompt text of the markdown files above.

## Vocabulary: "page"

Every feature, change, or alteration tracked by Grimoire is called a **page**. A page is the unit of work: it has a folder (`.grimoire/pages/NNN-[page-name]/`) containing a `SPEC.md` and one or more sequential step files (`1-[step].md`, `2-[step].md`, …), plus a single entry in `.grimoire/HISTORIC.md` whose status progresses `[spec]` → `[planned]` → `[finished]`. When this repo (or any skill body) refers to "the feature", "the alteration", "the plan", or "the work", it means **the page**. Use this term consistently in any new docs or skill edits.

## The skills and how they connect

- **`grimoire-init`** — Analyzes the *consuming* project and interviews the user, then writes `.grimoire/PROJECT.md` (purpose, audience, tech stack, layout, conventions, status). Every other Grimoire skill reads that file at the top of its `[Required Reading]`. Supports both initialization and update mode; pauses for clarifying questions and draft review before writing. Does **not** write `HISTORIC.md`.
- **`grimoire-spec`** — Entry point of the long-form pipeline. Takes the user's request in free text, analyzes the codebase, interviews until consensus, then writes `.grimoire/pages/NNN-[page-name]/SPEC.md`. Owns `HISTORIC.md`: bootstraps it if missing, prepends the new page with status `[spec]`, and rotates to `.grimoire/bag/historic/HISTORIC-N.md` once the file reaches 5 entries. The only skill that creates a page folder.
- **`grimoire-plan`** — Takes a page number (`42` or `042`). Hard-stops if the page folder is missing, has no `SPEC.md`, or the HISTORIC entry is not in `[spec]` status. Reads `SPEC.md` and writes sequential step files (`1-[step].md`, …) into the existing folder. Updates the HISTORIC entry from `[spec]` to `[planned]` in place. Never bootstraps, appends, or rotates `HISTORIC.md`. When the step breakdown contains genuinely independent steps, the planner MAY emit `depends-on`/`touches` frontmatter on step files; the resulting wave plan is surfaced to the user at the final-clarity-check pause-point before any step file is written — see `§ Parallel execution`.
- **`grimoire-execute`** — Takes a page number. Hard-stops if the page folder is missing, has no step files, or the HISTORIC entry is not in `[planned]` status. Executes step files in strict numeric order and **spawns one sub-agent per step file** to preserve context lucidity. On success, updates the page's `HISTORIC.md` entry from `[planned]` to `[finished]` in place — **no files are moved**. When step files carry `depends-on` frontmatter, execution switches from strict serial to dependency-graph waves with per-step git-worktree isolation and unconditional worktree teardown; absence of frontmatter falls back to the legacy strict-serial path — see `§ Parallel execution`.
- **`grimoire-quick`** — Fast-path for small fixes. Has a **scope gatekeeper**: if the task is too large, it must STOP and tell the user to switch to `grimoire-spec` (the entry point of the long-form pipeline). Requires explicit user authorization of the inline plan before any code is written. Stays fully ephemeral: no page folder, no `HISTORIC.md` entry.
- **`grimoire-know`** — Read-only Q&A about the repository or the application it builds. Takes a question in free text; reads `.grimoire/PROJECT.md` and `HISTORIC.md` for context if present; spawns a single sub-agent with `Read`/`Grep`/`Glob`/read-only `Bash` plus `WebSearch`/`WebFetch` to inspect only the files it needs and consult the web when the answer depends on facts outside the repo. Returns the direct answer, an explicit "uncertain about" list when applicable, and a `References` section citing each consulted URL when the web was actually used. **Never** writes, **never** commits, **never** touches `.grimoire/` state files — orthogonal to the Spec → Plan → Execute pipeline (like `grimoire-update`).
- **`grimoire-update`** — Plugin self-maintenance. Reads the installed version from `${CLAUDE_PLUGIN_ROOT}/.claude-plugin/plugin.json`, `WebFetch`es the latest `plugin.json` and `CHANGELOG.md` from the GitHub `main` branch, and — only if versions differ — shows the changelog diff and guides the user through `/plugin marketplace update grimoire` followed by `/reload-plugins`. Does **not** write to `.grimoire/`, does **not** create a page, does **not** make commits in the consumer project. Pure orchestration of two slash commands the user runs themselves.
- **`grimoire-note`** — Surgical, incremental writer for `.grimoire/PROJECT.md`. Takes a free-text note, semantically splits it into N rules, reads `## Key Conventions / Constraints` and `## Notes` in full, and proposes the best synthesis (merge, rewrite, generalize, or accept as new) — minimum-word phrasing is an explicit objective. Every invocation also runs a retroactive consolidation pass over both sections. Orthogonal to Spec → Plan → Execute (same family as `grimoire-know` and `grimoire-update`): writes **only** `## Key Conventions / Constraints` and `## Notes` in `.grimoire/PROJECT.md`, incrementally; never adds sections; never edits other sections; never touches `HISTORIC.md` or any page folder. **Hard-stops** if `.grimoire/PROJECT.md` is missing and points the user at `/grimoire-init`. Ends with a single atomic commit `docs(grimoire): refine project context` covering only `.grimoire/PROJECT.md`. The dual-writer contract between `grimoire-init` and `grimoire-note` lives in [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md) `§ Project context` (amended in Step 2 of this page).

## Shared conventions

All shared, load-bearing rules live in [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md):

- § TDD (Red/Green/Refactor + the filter-only rule) — does not apply to `grimoire-spec`, which produces a specification, not code.
- § Sub-agent spawning
- § Parallel execution (frontmatter schema `depends-on`/`touches`; DAG construction + cycle detection; Kahn-style waves; worktree contract with mandatory unconditional teardown; partial-failure handling; back-compat fallback to legacy serial)
- § Commits (atomic + Conventional Commits in English; `docs(grimoire): spec page <name>` + `chore: register page <name> in historic` after spec; `chore: mark page <name> planned` after plan; `chore: mark page <name> finished` after execute)
- § `.grimoire/` layout (every page lives in `.grimoire/pages/NNN-[page-name]/` with `SPEC.md` and sequential step files; no `finished/` directory; `.grimoire/PROJECT.md` for project context)
- § Project context (every skill loads `.grimoire/PROJECT.md` if present; missing → suggest `grimoire-init`; only `grimoire-init` writes it)
- § Historic (`.grimoire/HISTORIC.md` keeps the last 5 pages with `[spec]`/`[planned]`/`[finished]` status; `grimoire-spec` bootstraps, appends, and rotates to `.grimoire/bag/historic/HISTORIC-N.md`; `grimoire-plan` and `grimoire-execute` only update status in place)
- § Pause-point pattern (grimoire-init: clarifying questions + draft review; grimoire-spec: clarifying questions + draft review; grimoire-plan: precondition check (hard-stop) + unclear requirements; grimoire-execute: precondition check (hard-stop); grimoire-quick: scope gatekeeper + plan authorization)

**Do not duplicate these rules in SKILL.md bodies.** When a rule changes, update `GRIMOIRE-CONVENTIONS.md` only.

## Editing notes

- SKILL.md frontmatter is consumed by the Claude Code skills system — don't rename `name`, `description`, or `argument-hint`, and don't drop the leading `---` delimiters.
- `$ARGUMENTS` and `$1` in the prompt bodies are skill-invocation substitutions — preserve them verbatim.
- `${CLAUDE_PLUGIN_ROOT}` is substituted by Claude Code at runtime to the absolute path of the installed plugin. Use it whenever a SKILL.md references a sibling file like `GRIMOIRE-CONVENTIONS.md`.
- When you add or remove a section in `GRIMOIRE-CONVENTIONS.md`, update the `[Required Reading]` block at the top of each SKILL.md that depends on it.
- `plugin.json` lives at `.claude-plugin/plugin.json` (Anthropic's required path). Skills are auto-discovered from `skills/` — do not add a `skills` field to the manifest unless adding custom paths.

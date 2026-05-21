# Grimoire

A Claude Code plugin bundling three skills for a disciplined **Plan → Execute** pipeline with a **Quick** fast-path.

- `/grimoire-plan <feature>` — analyze the codebase, ask clarifying questions, then write a plan file into `.grimoire/`. Splits into sub-plans automatically when the scope is large enough to threaten context lucidity.
- `/grimoire-execute <path>` — execute a plan (single file or sub-plan directory) via sub-agents, one per sub-plan in strict numeric order. On success, moves the plan(s) into `.grimoire/finished/`.
- `/grimoire-quick <fix>` — fast-path for small fixes. Has a scope gatekeeper (stops and redirects you to `/grimoire-plan` if the task is too big) and requires explicit authorization of the inline plan before any code is written.

All three skills share a single source of truth for the workflow rules — strict TDD, atomic Conventional Commits, sub-agent orchestration, and the `.grimoire/` directory layout — in [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md).

## Install

Add this repo as a marketplace, then install the plugin:

```
/plugin marketplace add <github-owner>/grimoire
/plugin install grimoire
```

Replace `<github-owner>/grimoire` with the repo path (for example `cezarleferr/grimoire`). Other install options:

```
# Install to project scope (shared via .claude/settings.json)
claude plugin install grimoire --scope project

# Local trial without a marketplace
claude --plugin-dir /path/to/this/repo
```

After install, the three skills are available as `/grimoire-plan`, `/grimoire-execute`, `/grimoire-quick` in any session.

## Workflow

1. `/grimoire-plan "add user-auth endpoint"` → produces `.grimoire/001-add-user-auth-endpoint.md` (or a `.grimoire/001-add-user-auth-endpoint/` folder of sub-plans).
2. `/grimoire-execute .grimoire/001-add-user-auth-endpoint.md` → runs the plan via a sub-agent under strict TDD, commits atomically, then moves the plan to `.grimoire/finished/`.
3. Use `/grimoire-quick "fix typo in login error message"` for trivial fixes — Grimoire will stop you and route you back to `/grimoire-plan` if the request is too large.

## Editing the plugin

See [CLAUDE.md](CLAUDE.md) for maintainer notes.

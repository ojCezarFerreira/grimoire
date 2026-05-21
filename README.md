# Grimoire

A Claude Code plugin bundling four skills for a disciplined **Plan → Execute** pipeline with a **Quick** fast-path, plus an **Init** step that gives every session shared project context.

Every change tracked by Grimoire is called a **page**: a folder under `.grimoire/pages/` with one or more sequential step files inside, plus a single entry in `.grimoire/HISTORIC.md` that records its status.

- `/grimoire-init` — analyze the project and interview you to produce `.grimoire/PROJECT.md` (purpose, audience, tech stack, conventions). Every other Grimoire skill loads that file on startup so the orchestrator and its sub-agents share baseline context. Re-running enters update mode and asks only about deltas.
- `/grimoire-plan <page>` — analyze the codebase, ask clarifying questions, then write the page as `.grimoire/pages/NNN-[page-name]/` with one or more sequential step files (`1-[step].md`, `2-[step].md`, …). Bootstraps/appends/rotates `.grimoire/HISTORIC.md`, registering the page with status `[planned]`.
- `/grimoire-execute <path>` — execute the page's step files via sub-agents, one per step file in strict numeric order. On success, updates the page's `HISTORIC.md` entry to `[finished]` in place — no files are moved.
- `/grimoire-quick <fix>` — fast-path for small fixes. Has a scope gatekeeper (stops and redirects you to `/grimoire-plan` if the task is too big) and requires explicit authorization of the inline plan before any code is written. Stays ephemeral: no page folder, no `HISTORIC.md` entry.

All four skills share a single source of truth for the workflow rules — strict TDD, atomic Conventional Commits, sub-agent orchestration, the `.grimoire/pages/` layout, project context loading, and the `HISTORIC.md` recency log and status-of-record — in [GRIMOIRE-CONVENTIONS.md](GRIMOIRE-CONVENTIONS.md).

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

After install, the four skills are available as `/grimoire-init`, `/grimoire-plan`, `/grimoire-execute`, `/grimoire-quick` in any session.

## Workflow

1. `/grimoire-init` (once per project) → produces `.grimoire/PROJECT.md` from a short interview; every subsequent skill loads it automatically. Re-run it later in update mode whenever the project's purpose, stack, or constraints have shifted.
2. `/grimoire-plan "add user-auth endpoint"` → produces `.grimoire/pages/001-add-user-auth-endpoint/` with one or more sequential step files (e.g., `1-schema.md`, `2-endpoints.md`, `3-tests.md`) and registers the page in `.grimoire/HISTORIC.md` as `1. **001-add-user-auth-endpoint** [planned] — …` (rotating to `.grimoire/bag/historic/` if the log was already full).
3. `/grimoire-execute .grimoire/pages/001-add-user-auth-endpoint/` → runs each step file via a sub-agent under strict TDD, commits atomically, then updates the page's `HISTORIC.md` entry in place to `[finished]`. The page folder stays in `.grimoire/pages/`.
4. Use `/grimoire-quick "fix typo in login error message"` for trivial fixes — Grimoire will stop you and route you back to `/grimoire-plan` if the request is too large.

## Editing the plugin

See [CLAUDE.md](CLAUDE.md) for maintainer notes.

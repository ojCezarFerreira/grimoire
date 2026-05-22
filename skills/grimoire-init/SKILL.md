---
name: grimoire-init
description: Initialize project context by analyzing the codebase and writing .grimoire/PROJECT.md.
argument-hint: (optional) extra context or focus areas
---

**[Required Reading]**
1. Read `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md`. Its rules (§ .grimoire/ layout, § Project context, § Historic, § Commits, § IDE-aware review, § Pause-point pattern) are load-bearing for this skill.
2. If `.grimoire/HISTORIC.md` exists, read it for recent-page context (older context lives in `.grimoire/bag/historic/`, newest suffix first). Missing → proceed without it; do not create it (bootstrap is owned by `grimoire-spec`, not this skill).

**[Objective]**
Build a shared understanding of the current project — what it is, who it's for, how it's built — and persist it as `.grimoire/PROJECT.md`. From then on every other Grimoire skill loads that file on startup as project context.

Optional extra context or focus areas from the user:

$ARGUMENTS

**[Phase 1: Mode Detection & Codebase Analysis]**
- Check whether `.grimoire/PROJECT.md` already exists in the project root.
  - **If it exists:** read it first. This run is an **UPDATE** — preserve everything still accurate and only revise what has changed or is missing.
  - **If it does not exist:** this run is an **INITIALIZATION** — start from scratch.
- Scan the project for signals. Read only what is needed to form an opinion:
  - README files at the root.
  - Package manifests (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `Gemfile`, `composer.json`, etc.).
  - Build / language config (`tsconfig.json`, `vite.config.*`, `next.config.*`, `pnpm-workspace.yaml`, etc.).
  - CI/CD definitions under `.github/workflows/`, `.gitlab-ci.yml`, etc.
  - Top-level directory layout (one level deep).
  - `LICENSE`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` if present.
- Collect candidate facts: project name, primary languages / frameworks, notable dependencies, build & test commands, repository layout, audience hints. **Do not invent facts** — anything you cannot infer goes into Phase 2 as a question.

**[Phase 2: Clarifying Questions]**
Per `§ Pause-point pattern` (grimoire-init: clarifying questions), ask the user concise, targeted questions about ONLY what cannot be inferred from the codebase. Typical topics:
- The project's purpose — what problem it solves.
- The intended audience or users.
- The current stage (prototype / active development / production / maintenance).
- Non-obvious conventions, constraints, or invariants a new contributor should know.
- Anything to deliberately exclude from `PROJECT.md`.

In **UPDATE** mode, ask only about deltas — what has changed since the existing `PROJECT.md` was written, and whether any section is now stale.

Wait for the user's answers before drafting `PROJECT.md`.

**[Phase 3: Draft Review]**
- Compose the proposed `PROJECT.md` using this template:

  ```
  # <Project Name>

  ## Purpose
  ## Audience
  ## Tech Stack
  ## Repository Layout
  ## Key Conventions / Constraints
  ## Current Status
  ## Notes
  ```

- Present the proposed content for review per `§ IDE-aware review`:
  - **IDE detected:** create `.grimoire/` in the project root if it does not exist, then write the draft to `.grimoire/PROJECT.md` so the IDE renders it. Tell the user in one short line that the draft is open in the editor.
  - **Terminal-only fallback:** output the proposed content inline in the chat.
- Pause per `§ Pause-point pattern` (grimoire-init: draft review): wait for the user's confirmation or edits. Do not advance to Phase 4 until approved. If the user abandons the draft, delete the file (IDE mode only).

**[Phase 4: Finalize PROJECT.md]**
- **IDE mode:** the approved file is already at `.grimoire/PROJECT.md`. No additional write is needed.
- **Inline fallback:** create `.grimoire/` if it does not exist, then write the approved content to `.grimoire/PROJECT.md`.
- Inform the user that subsequent Grimoire skills (`grimoire-spec`, `grimoire-plan`, `grimoire-execute`, `grimoire-quick`) will load this file automatically as part of their `[Required Reading]`.

**[Phase 5: Version Control]**
- Commit per `§ Commits`:
  - Initialization: `docs(grimoire): initialize project context`
  - Update: `docs(grimoire): update project context`

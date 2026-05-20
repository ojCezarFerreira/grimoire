# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of three Claude Code skills defining the **Sage workflow**: a disciplined Plan → Execute pipeline with a Quick fast-path. Each skill is a single [skills/](skills/)`<name>/SKILL.md` file with YAML frontmatter (`name`, `description`, `argument-hint`) and a prompt body. There is no source code, no build system, no test suite — edits land in the prompt text of [skills/sage-plan/SKILL.md](skills/sage-plan/SKILL.md), [skills/sage-execute/SKILL.md](skills/sage-execute/SKILL.md), or [skills/sage-quick/SKILL.md](skills/sage-quick/SKILL.md). [README.md](README.md) is intentionally empty.

## The three skills and how they connect

- **`sage-plan`** — Writes plan files into `.planning/` in the *consuming* project. Single plans get `.planning/NNN-[feature].md`; large features get split into `.planning/NNN-[feature]/NNN.X-[subplan].md`. Numeric prefix is incremental and chronological. The plan-or-split decision is the model's call based on projected context size during execution.
- **`sage-execute`** — Takes a path (file or directory). For a directory, executes sub-plans in strict numeric order and **spawns one sub-agent per sub-plan** to preserve context lucidity. On success, moves the plan file or whole directory into `.planning/finished/`.
- **`sage-quick`** — Fast-path for small fixes. Has a **scope gatekeeper**: if the task is too large, it must STOP and tell the user to switch to `sage-plan`. Requires explicit user authorization of the inline plan before any code is written.

## Shared conventions (load-bearing — keep aligned across all three skills)

- **Strict TDD**: Red → Green → Refactor on every step.
- **Never run the full test suite.** Always filter to only the tests created/modified in the current step (`--filter`, exact file paths, etc.).
- **Sub-agent spawning** for execution work — the main context only orchestrates.
- **Atomic Conventional Commits in English** (`feat:`, `fix:`, `test:`, `refactor:`, `chore:`).
- **Explicit pause points**: `sage-plan` pauses on unclear requirements; `sage-quick` pauses for plan authorization before coding.
- **`.planning/finished/`** is the terminal location for completed plans.

## Editing notes

- SKILL.md frontmatter is consumed by the Claude Code skills system — don't rename `name`, `description`, or `argument-hint`, and don't drop the leading `---` delimiters.
- `$ARGUMENTS` and `$1` in the prompt bodies are skill-invocation substitutions — preserve them verbatim.
- When changing a rule that appears in multiple SKILL.md files (TDD steps, commit style, sub-agent spawning, `.planning/` layout), update all three together so the workflow stays coherent.

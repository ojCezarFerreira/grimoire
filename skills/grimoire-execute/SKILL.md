---
name: grimoire-execute
description: Execute a page with Grimoire — runs each step file via sub-agents and marks the page [finished] in HISTORIC.md.
argument-hint: @path-to-page-folder
---

**[Required Reading]**
1. Read `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md`. Its rules (§ TDD, § Sub-agent spawning, § Commits, § .grimoire/ layout, § Project context, § Historic) are load-bearing for this skill. **Note:** this skill only **updates the page's `HISTORIC.md` entry status** from `[planned]` to `[finished]` — it never appends new entries and never rotates. Bootstrap, append, and rotation are owned by `grimoire-plan`.
2. If `.grimoire/PROJECT.md` exists in the project root, read it for project context. If it does not exist, proceed without it and suggest the user run `grimoire-init` once this task is complete.
3. If `.grimoire/HISTORIC.md` exists, read it for recent-page context. If older context seems relevant, browse `.grimoire/bag/historic/` (newest suffix first). Missing → proceed without it.

**[Objective]**
Execute the page located at: $1

**[Phase 1: Detection & Page Review]**
- Resolve the input path to a **page folder** under `.grimoire/pages/NNN-[page-name]/`. If the user passed one of the page's step files, resolve up to the containing folder.
- Read every step file inside the page folder to understand the full scope.
- Prepare to execute the step files in strict sequential order based on their numeric prefix (`1-…`, `2-…`, `3-…`).
- If any step or requirement seems contradictory, ask me for clarification before starting.

**[Phase 2: Execution]**
- Spawn sub-agents per `§ Sub-agent spawning`: one sub-agent per step file, strictly sequential. Never start step `N+1` until the sub-agent for step `N` has fully completed its assignment.
- Each sub-agent must execute its step file step-by-step per `§ TDD`.

**[Phase 3: Version Control]**
- Make commits per `§ Commits` as the execution progresses.

**[Phase 4: Finalization]**
- The page folder and its step files **stay in place** — nothing is moved.
- Update the page's entry in `.grimoire/HISTORIC.md` per `§ Historic`:
  - Find the entry whose `<page-name>` matches the page just executed.
  - Update its status from `[planned]` to `[finished]` **in place**. Do not change its position, do not append a new entry, do not rotate.
  - If the matching entry is not found (e.g., it was rotated to `.grimoire/bag/historic/` while the page was being executed), skip silently. This is non-blocking.
- Commit the HISTORIC status update per `§ Commits` as a final commit: `chore: mark page <name> finished`.

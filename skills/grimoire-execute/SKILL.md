---
name: grimoire-execute
description: Run a plan file with Grimoire.
argument-hint: @caminho-para-o-plano
---

**[Required Reading]**
1. Read `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md`. Its rules (§ TDD, § Sub-agent spawning, § Commits, § .grimoire/ layout, § Project context) are load-bearing for this skill.
2. If `.grimoire/PROJECT.md` exists in the project root, read it for project context. If it does not exist, proceed without it and suggest the user run `grimoire-init` once this task is complete.

**[Objective]**
Execute the development plan(s) located at: $1

**[Phase 1: Detection & Plan Review]**
- Determine if the provided path is a single file (isolated plan) or a directory (partitioned sub-plans).
- **If it is a single file:** Read and understand the outlined steps.
- **If it is a directory:** Read all sub-plans within it to understand the full scope. Prepare to execute them in strict sequential order based on their numeric prefixes (e.g., `001.1`, `001.2`).
- If any step or requirement seems contradictory, ask me for clarification before starting.

**[Phase 2: Execution]**
- Spawn sub-agents per `§ Sub-agent spawning` (single file → one sub-agent; directory → one sub-agent per sub-plan, strictly sequential).
- Each sub-agent must execute its plan step-by-step per `§ TDD`.

**[Phase 3: Version Control]**
- Make commits per `§ Commits`.

**[Phase 4: Cleanup & Finalization]**
- Once all execution is verified and successful, move the plan to `.grimoire/finished/` per `§ .grimoire/ layout`:
  - **Single file:** move the `.md` file to `.grimoire/finished/`.
  - **Directory:** move the ENTIRE parent folder (containing all sub-plans) into `.grimoire/finished/`, preserving its internal structure.
- Create a final commit for this organizational change: `chore: move [plan-name] to finished`.

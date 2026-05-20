---
name: sage-execute
description: Run a plan file with Sage.
argument-hint: @caminho-para-o-plano
---

**[Objective]**
Execute the development plan(s) located at: $1

**[Phase 1: Detection & Plan Review]**
- Determine if the provided path is a single file (isolated plan) or a directory (partitioned sub-plans).
- **If it is a single file:** Read and understand the outlined steps.
- **If it is a directory:** Read all sub-plans within it to understand the full scope. Prepare to execute them in strict sequential order based on their numeric prefixes (e.g., `001.1`, `001.2`).
- If any step or requirement seems contradictory, ask me for clarification before starting.

**[Phase 2: Execution via Sub-Agents & Strict TDD]**
**If executing a directory of sub-plans:**
- You must spawn a sub-agent for EACH sub-plan sequentially to maintain context lucidity.
- Instruct the sub-agent with the specific sub-plan file.
- Do not start or spawn the next sub-plan until the current sub-agent has fully completed its assignment.

**For every plan or sub-plan, execute step-by-step using strict TDD:**
**CRITICAL:** NEVER execute the entire test suite. Use test filters (e.g., `--filter` in PHPUnit/Pest) or specify exact file paths to run ONLY the tests created or modified in the current step.
1. **Red:** Write tests for the current step. Run ONLY these tests and verify failure.
2. **Green:** Implement minimum code. Run the specific tests again to verify they pass.
3. **Refactor:** Clean up the code while ensuring tests remain green.

**[Phase 3: Version Control & Organization]**
- Make granular, atomic commits as you progress.
- Commit changes immediately after each logical step is completed (Green/Refactor phase).
- Use Conventional Commits (e.g., `feat:`, `test:`, `refactor:`) in English.

**[Phase 4: Cleanup & Finalization]**
- Once all execution is verified and successful:
  - **If it was a single file:** Move the `.md` file to the `.planning/finished/` subdirectory.
  - **If it was a directory:** Move the ENTIRE parent folder (containing all sub-plans) into `.planning/finished/`, preserving its internal structure (e.g., move `.planning/001-[feature]/` to `.planning/finished/001-[feature]/`).
- Create a final commit for this organizational change: `chore: move [plan-name] to finished`.

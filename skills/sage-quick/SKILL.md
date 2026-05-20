---
name: sage-quick
description: Quick task or bug fix with Sage.
argument-hint: Task/fix description
---

**[Objective]**
I need you to quickly resolve the following minor adjustment or bug fix:

$ARGUMENTS

**[Phase 1: Analysis & Scope Gatekeeper]**
- Analyze the relevant codebase architecture for this specific request.
- Evaluate the complexity and scope of the requested change.
- **CRITICAL - SCOPE CHECK:** If you determine that this request is too large, complex, or risks context degradation, you MUST STOP immediately. Do not generate a plan. Inform me with the following message: *"This task is too large for a quick execution. Please start a new session and use the standard Planning prompt."*

**[Phase 2: Quick Planning & User Authorization]**
- If the scope is confirmed as small and safe for quick execution, ask any clarifying questions if necessary.
- Output a clear, step-by-step execution plan directly in our chat for my review.
- PAUSE: Wait for my explicit authorization or corrections before proceeding. Do NOT start coding or spawning agents yet.

**[Phase 3: Execution via Sub-Agent & Strict TDD]**
- Once I authorize the plan, you must spawn a single sub-agent to handle the execution. This ensures the main context remains lucid.
- Instruct the sub-agent to execute the approved plan step-by-step using a strict Test-Driven Development workflow.
- **CRITICAL:** NEVER execute the entire test suite. Use test filters (e.g., `--filter` in PHPUnit/Pest) or specify exact file paths to run ONLY the tests created or modified for this quick fix.
  1. **Red:** Write/modify tests for the current step. Run ONLY these tests and verify failure.
  2. **Green:** Implement minimum code. Run the specific tests again to verify they pass.
  3. **Refactor:** Clean up the code while ensuring tests remain green.

**[Phase 4: Version Control]**
- Instruct the sub-agent to make granular, atomic commits as it progresses through the fix.
- Commit changes immediately after each logical step is completed (Green/Refactor phase).
- Use Conventional Commits (e.g., `fix:`, `refactor:`, `style:`) in English.

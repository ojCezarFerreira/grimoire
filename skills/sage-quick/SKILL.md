---
name: sage-quick
description: Quick task or bug fix with Sage.
argument-hint: Task/fix description
---

**[Required Reading]**
Before proceeding, read `${CLAUDE_PLUGIN_ROOT}/SAGE-CONVENTIONS.md`. Its rules (§ TDD, § Sub-agent spawning, § Commits, § Pause-point pattern) are load-bearing for this skill.

**[Objective]**
I need you to quickly resolve the following minor adjustment or bug fix:

$ARGUMENTS

**[Phase 1: Analysis & Scope Gatekeeper]**
- Analyze the relevant codebase architecture for this specific request.
- Evaluate the complexity and scope of the requested change.
- **CRITICAL — SCOPE CHECK** (per `§ Pause-point pattern`, sage-quick: scope gatekeeper): if this request is too large, complex, or risks context degradation, you MUST STOP immediately. Do not generate a plan. Inform me with the following message: *"This task is too large for a quick execution. Please start a new session and use the standard Planning prompt."*

**[Phase 2: Quick Planning & User Authorization]**
- If the scope is confirmed as small and safe for quick execution, ask any clarifying questions if necessary.
- Output a clear, step-by-step execution plan directly in our chat for my review.
- Pause per `§ Pause-point pattern` (sage-quick: plan authorization): wait for my explicit authorization or corrections before proceeding. Do NOT start coding or spawning agents yet.

**[Phase 3: Execution]**
- Once I authorize the plan, spawn a sub-agent per `§ Sub-agent spawning` (sage-quick: one sub-agent for the whole fix).
- Instruct the sub-agent to execute the approved plan step-by-step per `§ TDD`.

**[Phase 4: Version Control]**
- Instruct the sub-agent to make commits per `§ Commits`.

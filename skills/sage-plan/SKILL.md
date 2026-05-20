---
name: sage-plan
description: Make a plan file with Sage.
argument-hint: Feature description
---

**[Objective]**
I need you to plan the implementation of the following feature:

$ARGUMENTS

**[Phase 1: Analysis & Clarification]**
- Analyze the current codebase architecture.
- Research and identify the best practices for this specific implementation.
- PAUSE: If any requirements are unclear, or if you need to make critical architectural decisions, ask me clarifying questions before proceeding.

**[Phase 2: Planning, Scope & Context Management]**
- CRITICAL: You are entirely responsible for evaluating the complexity and scope of the requested feature. 
- You must actively assess the potential token usage and context window size required for the execution phase. Your primary goal is to maintain maximum context lucidity and prevent degradation during execution.

**If it is a Standard/Small Scope Feature (fits comfortably within context without losing lucidity):**
- Generate a single, comprehensive, step-by-step execution plan.
- Save this plan inside the `.planning/` directory.
- The file MUST have an incremental numeric prefix to maintain chronological order (e.g., `.planning/001-[feature-name].md`).

**If it is a Large Scope Feature (risk of high token usage, context degradation, or loss of lucidity):**
- You must break the execution plan down into smaller, highly cohesive, and sequential sub-plans.
- Create a subfolder inside `.planning/` using the incremental numeric prefix and the feature name (e.g., `.planning/001-[feature-name]/`).
- Save each sub-plan inside this new subfolder.
- Name the sub-plans sequentially using the folder's number followed by a sub-sequence number (e.g., `001.1-[subplan-name].md`, `001.2-[subplan-name].md`, etc.).

**[Phase 3: Documentation Commit]**
- Once the plan (or folder of sub-plans) is saved, create a commit for these planning files.

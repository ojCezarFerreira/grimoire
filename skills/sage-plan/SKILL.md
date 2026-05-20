---
name: sage-plan
description: Make a plan file with Sage.
argument-hint: Feature description
---

**[Required Reading]**
Before proceeding, read `${CLAUDE_PLUGIN_ROOT}/SAGE-CONVENTIONS.md`. Its rules (§ TDD, § Sub-agent spawning, § Commits, § .planning/ layout, § Pause-point pattern) are load-bearing for this skill.

**[Objective]**
I need you to plan the implementation of the following feature:

$ARGUMENTS

**[Phase 1: Analysis & Clarification]**
- Analyze the current codebase architecture.
- Research and identify the best practices for this specific implementation.
- Pause per `§ Pause-point pattern` (sage-plan: unclear requirements): if any requirement is unclear or a critical architectural decision is needed, ask me clarifying questions before proceeding.

**[Phase 2: Planning, Scope & Context Management]**
- CRITICAL: You are entirely responsible for evaluating the complexity and scope of the requested feature.
- You must actively assess the potential token usage and context window size required for the execution phase. Your primary goal is to maintain maximum context lucidity and prevent degradation during execution.

**If it is a Standard/Small Scope Feature (fits comfortably within context without losing lucidity):**
- Generate a single, comprehensive, step-by-step execution plan.
- Save it per `§ .planning/ layout` (single-plan naming: `.planning/NNN-[feature-name].md`).

**If it is a Large Scope Feature (risk of high token usage, context degradation, or loss of lucidity):**
- Break the execution plan down into smaller, highly cohesive, and sequential sub-plans.
- Save them per `§ .planning/ layout` (split-plan naming: `.planning/NNN-[feature-name]/NNN.X-[subplan-name].md`).

**[Phase 3: Documentation Commit]**
- Once the plan (or folder of sub-plans) is saved, create a commit for these planning files per `§ Commits`.

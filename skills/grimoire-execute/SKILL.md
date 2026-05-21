---
name: grimoire-execute
description: Execute a page with Grimoire — runs each step file via sub-agents and marks the page [finished] in HISTORIC.md.
argument-hint: Page number (e.g. 42)
---

**[Required Reading]**
1. Read `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md`. Its rules (§ TDD, § Sub-agent spawning, § Commits, § .grimoire/ layout, § Project context, § Historic, § Pause-point pattern) are load-bearing for this skill. **Note:** this skill only **updates the page's `HISTORIC.md` entry status** from `[planned]` to `[finished]` — it never appends new entries and never rotates. Bootstrap, append, and rotation are owned by `grimoire-spec`. This skill requires the page to already be planned (`[planned]` status in `HISTORIC.md`, step files present); if not, hard-stop per `§ Pause-point pattern` (grimoire-execute: precondition check).
2. If `.grimoire/PROJECT.md` exists in the project root, read it for project context. If it does not exist, proceed without it and suggest the user run `grimoire-init` once this task is complete.
3. If `.grimoire/HISTORIC.md` exists, read it for recent-page context. If older context seems relevant, browse `.grimoire/bag/historic/` (newest suffix first). Missing → the page-status check below requires it; treat as a hard-stop unless the page is verifiable through step files alone (see Phase 1).

**[Objective]**
Execute the page identified by:

$ARGUMENTS

**[Phase 1: Page Resolution & Precondition Check (HARD STOP)]**
Per `§ Pause-point pattern` (grimoire-execute: precondition check). Run these checks **before** spawning any sub-agent; STOP on the first failure with the literal message shown.

1. **Parse the page number.** Accept `$ARGUMENTS` as a bare integer with or without zero-padding (`42` or `042`); normalize to `NNN`. Reject anything not parseable as an integer.
2. **Locate the page folder.** Glob `.grimoire/pages/NNN-*/`. Exactly one match expected.
   - No match → `❌ Page NNN não existe.` and STOP.
3. **Confirm step files.** Verify at least `1-*.md` exists inside the folder.
   - No step files → `❌ Page NNN ainda não tem plano. Rode /grimoire-plan NNN primeiro.` and STOP.
4. **Check HISTORIC status.** Find the entry whose `<page-name>` matches the folder.
   - Status `[spec]` → `❌ Page NNN ainda está em spec. Rode /grimoire-plan NNN primeiro.` and STOP.
   - Status `[finished]` → `❌ Page NNN já foi finalizada.` and STOP.
   - Entry missing entirely (rotated mid-execution) → proceed; this is the non-blocking "skip silently if rotated" rule from `§ Historic`. Note this in the run summary.
   - Status `[planned]` → proceed.
5. **Read all step files** in the page folder to understand the full scope of execution. If any step or requirement seems contradictory, ask the user for clarification before starting.

**[Phase 2: Execution]**
- Spawn sub-agents per `§ Sub-agent spawning`: one sub-agent per step file, strictly sequential. Never start step `N+1` until the sub-agent for step `N` has fully completed its assignment.
- Each sub-agent must execute its step file step-by-step per `§ TDD`.

**[Phase 3: Version Control]**
- Make commits per `§ Commits` as the execution progresses.

**[Phase 4: Finalization]**
- The page folder, its `SPEC.md`, and its step files **stay in place** — nothing is moved.
- Update the page's entry in `.grimoire/HISTORIC.md` per `§ Historic`:
  - Find the entry whose `<page-name>` matches the page just executed.
  - Update its status from `[planned]` to `[finished]` **in place**. Do not change its position, do not append a new entry, do not rotate.
  - If the matching entry is not found (rotated mid-execution per the Phase 1 note), skip silently. This is non-blocking.
- Commit the HISTORIC status update per `§ Commits` as a final commit: `chore: mark page <name> finished`.

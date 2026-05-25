---
name: grimoire-execute
description: Execute a page with Grimoire — runs each step file via sub-agents and marks the page [finished] in HISTORIC.md.
argument-hint: Page number (e.g. 42)
---

**[Required Reading]**
1. Read `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md`. Its rules (§ TDD, § Sub-agent spawning, § Parallel execution, § Commits, § .grimoire/ layout, § Project context, § Historic, § Pause-point pattern) are load-bearing for this skill. **Note:** this skill only **updates the page's `HISTORIC.md` entry status** from `[planned]` to `[finished]` — it never appends new entries and never rotates. Bootstrap, append, and rotation are owned by `grimoire-spec`. This skill requires the page to already be planned (`[planned]` status in `HISTORIC.md`, step files present); if not, hard-stop per `§ Pause-point pattern` (grimoire-execute: precondition check).
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
6. **Detect execution mode.** Parse each step file's YAML frontmatter:
   - If **no step file** carries `depends-on` frontmatter (or the field is missing/empty on every step), select the **legacy strict-serial path** — proceed to Phase 2A unchanged from prior behavior.
   - If **at least one step file** carries `depends-on` frontmatter, select the **parallel-wave path** per `§ Parallel execution`:
     - Build the DAG (nodes = step files, edges = `depends-on` references).
     - Run cycle detection. If a cycle is found → `❌ Page NNN tem dependências cíclicas entre steps. Rode /grimoire-plan NNN para corrigir.` and STOP.
     - Verify every `depends-on` reference points to an existing step file. Dangling reference → `❌ Page NNN referencia um step inexistente em depends-on. Rode /grimoire-plan NNN para corrigir.` and STOP.
     - Compute waves (Kahn-style topological levels per `§ Parallel execution`).
     - Proceed to Phase 2B's wave loop.

**[Phase 2A: Legacy strict-serial execution]**
Selected at Phase 1 step 6 when no step file carries `depends-on` frontmatter. Behavior is byte-identical to the pre-0.8.0 implementation.

- Spawn sub-agents per `§ Sub-agent spawning`: one sub-agent per step file, strictly sequential. Never start step `N+1` until the sub-agent for step `N` has fully completed its assignment.
- Each sub-agent must execute its step file step-by-step per `§ TDD`.
- Each sub-agent makes its own atomic Conventional Commits directly in the main workspace per `§ Commits`.

**[Phase 2B: Parallel-wave execution]**
Selected at Phase 1 step 6 when at least one step file carries `depends-on` frontmatter. The wave loop runs for each wave `K` from `0` upward, in strictly increasing order. **All worktree, sub-agent isolation, cherry-pick, and failure-handling rules live in `§ Parallel execution`** — this phase orchestrates them; it does not restate them.

For each wave `K`:

1. **Skip filtering.** Determine which steps in wave `K` are eligible to run. Per `§ Parallel execution`, any step whose `depends-on` **transitively** includes a step that **failed** in an earlier wave of this run is **skipped** — mark it as `skipped-due-to-upstream-failure` for the run summary. Steps whose dependencies all succeeded are eligible.

2. **Worktree creation.** For each eligible step `S` in wave `K`, per the worktree contract in `§ Parallel execution`:
   - Create `.grimoire/bag/worktrees/page-NNN-step-S/` (where `NNN` is the zero-padded page number and `S` is the step number).
   - Run `git worktree add` from the main workspace's current `HEAD`. The runtime MAY use detached worktrees or short-lived branches — pick whichever the host environment supports cleanly.

3. **Sub-agent spawn (concurrent).** Spawn one sub-agent per eligible step in this wave, **in parallel**, per the amended `§ Sub-agent spawning`. Each sub-agent receives:
   - The path to its worktree as its working directory.
   - The path to its step file (still under `.grimoire/pages/NNN-[page-name]/`).
   - A strict instruction to confine all edits and commits to its worktree directory — it must not touch the main workspace, sibling worktrees, or any `.grimoire/` state file outside its worktree.
   - The `§ TDD` and `§ Commits` requirements as in the legacy path.

4. **Wait for wave settlement.** Wait for **every** spawned sub-agent in this wave to finish (success or failure). **No mid-wave kill** — even when one step fails, in-flight siblings run to completion per `§ Parallel execution`.

5. **Cherry-pick back to main, in step-number order.** Iterate the wave's eligible steps in ascending step-number order:
   - If the step succeeded → cherry-pick every commit it made in its worktree into the main workspace.
   - If the step failed → skip it (no commits land); record it as `failed` for downstream filtering and the run summary.
   - Cherry-pick conflicts (caused by overlapping `touches` between siblings, etc.) are reported and the step is treated as `failed` for downstream wave filtering.

6. **Unconditional teardown (try/finally invariant).** **Before** moving to wave `K+1`, for **every** worktree the orchestrator created in wave `K` — successful, failed, skipped, or never started — execute teardown:
   - Run `git worktree remove --force <worktree-path>`.
   - Run a defensive filesystem `rm -rf` on the worktree directory in case `git worktree remove --force` left residue behind.
   - Verify with `git worktree list` that no entry remains for any of this wave's worktrees.
   - This teardown is **non-negotiable**. It runs whether the wave succeeded, partially failed, fully failed, or the run was aborted by an exception or hard-abort signal. Implement it as a **try/finally-shaped invariant** in the orchestrator: the cleanup block runs even when the try block panics. This is the dominant safety property of the parallel-wave path — the maintainer has been bitten by stale worktree clutter before, and any code path that can leave residue is a bug.

7. **End-of-run teardown.** At the **end of the run** — success, partial failure, or hard abort — `.grimoire/bag/worktrees/` MUST be empty AND `git worktree list` MUST show only the main workspace. If the orchestrator detects any residue at end of run, it MUST attempt a final defensive cleanup (`git worktree remove --force` + `rm -rf` on every entry) and explicitly report any worktrees it could not remove so the user can clear them manually.

**[Phase 3: Version Control]**
- Make commits per `§ Commits` as the execution progresses.
- In the **legacy strict-serial path (Phase 2A)**, sub-agents commit directly in the main workspace as each step lands.
- In the **parallel-wave path (Phase 2B)**, per-step commits **originate inside the step's worktree** and arrive on the main branch via the wave's cherry-pick step in **ascending step-number order**. The main branch history therefore reads as if the wave had run serially, even though execution was concurrent. Atomic `feat:` / `fix:` / `test:` / `refactor:` commits per step per `§ Commits` are preserved through the cherry-pick.

**[Phase 4: Finalization]**
- The page folder, its `SPEC.md`, and its step files **stay in place** — nothing is moved.
- **If every step in the run succeeded** (legacy strict-serial: every sub-agent finished green; parallel-wave: every wave's every step finished green and cherry-picked cleanly to main):
  - Update the page's entry in `.grimoire/HISTORIC.md` per `§ Historic`:
    - Find the entry whose `<page-name>` matches the page just executed.
    - Update its status from `[planned]` to `[finished]` **in place**. Do not change its position, do not append a new entry, do not rotate.
    - If the matching entry is not found (rotated mid-execution per the Phase 1 note), skip silently. This is non-blocking.
  - Commit the HISTORIC status update per `§ Commits` as a final commit: `chore: mark page <name> finished`.
- **If any step failed or was skipped due to upstream failure** (parallel-wave only — legacy strict-serial has no partial-failure state because step `N+1` never starts when step `N` fails):
  - **Do NOT** update HISTORIC. The page's status stays `[planned]`.
  - Report a run summary to the user listing **finished**, **failed**, and **skipped** steps, and tell them that re-running `/grimoire-execute NNN` will retry the page **from scratch** — they SHOULD inspect and, if necessary, revert commits from the prior partial run before retrying.
  - **Worktree teardown still runs** per Phase 2B step 7 — partial failure is **not** an excuse to leave residue under `.grimoire/bag/worktrees/`.

# Step 3 — Update `skills/grimoire-execute/SKILL.md`

## Scope

Teach `grimoire-execute` to detect step-file frontmatter, compute waves via topological sort, run wave members concurrently — one sub-agent per step, each inside its own git worktree — cherry-pick successful siblings back in step-number order, and tear down every worktree unconditionally before the next wave and at end of run. **The shared rules live in `§ Parallel execution` (added in Step 1) — this SKILL.md only references them, it must not restate them.** No code; markdown only.

## Files touched

- `skills/grimoire-execute/SKILL.md` (only this file)

## Precondition

Step 1 must be committed first — this step relies on `§ Parallel execution`, `§ Sub-agent spawning` (amended), and `§ Pause-point pattern` (extended with the two new hard-stop cases) being live in `GRIMOIRE-CONVENTIONS.md`.

## Edits

### 1. `[Required Reading]` — add `§ Parallel execution`

In the first bullet of `[Required Reading]` (currently lists `§ TDD, § Sub-agent spawning, § Commits, § .grimoire/ layout, § Project context, § Historic, § Pause-point pattern`), insert `§ Parallel execution` so the list reads:

> § TDD, § Sub-agent spawning, **§ Parallel execution**, § Commits, § .grimoire/ layout, § Project context, § Historic, § Pause-point pattern

Order: place `§ Parallel execution` right after `§ Sub-agent spawning` (match conventions order).

### 2. Phase 1 — frontmatter detection + cycle / dangling-reference checks

The current Phase 1 has five steps (Parse / Locate / Confirm step files / Check HISTORIC status / Read all step files). Keep them; **after step 5 (Read all step files)**, add a sixth step:

> 6. **Detect execution mode.** Parse each step file's YAML frontmatter:
>    - If **no step file** carries `depends-on` frontmatter (or the field is missing/empty on every step), select the **legacy strict-serial path** — proceed to Phase 2 unchanged from prior behavior.
>    - If **at least one step file** carries `depends-on` frontmatter, select the **parallel-wave path** per `§ Parallel execution`:
>      - Build the DAG (nodes = step files, edges = `depends-on` references).
>      - Run cycle detection. If a cycle is found → `❌ Page NNN tem dependências cíclicas entre steps. Rode /grimoire-plan NNN para corrigir.` and STOP.
>      - Verify every `depends-on` reference points to an existing step file. Dangling reference → `❌ Page NNN referencia um step inexistente em depends-on. Rode /grimoire-plan NNN para corrigir.` and STOP.
>      - Compute waves (Kahn-style topological levels per `§ Parallel execution`).
>      - Proceed to Phase 2's wave loop.

(The hard-stop messages above match the ones added in Step 1's amendment to `§ Pause-point pattern`. The exact text should be identical.)

### 3. Phase 2 — rewrite to cover both paths

Replace the current Phase 2 with two clearly-labeled paths:

> **Phase 2A — Legacy strict-serial path** (selected at Phase 1 step 6 when no step file carries `depends-on` frontmatter):
> - Spawn sub-agents per `§ Sub-agent spawning`: one sub-agent per step file, strictly sequential. Never start step `N+1` until the sub-agent for step `N` has fully completed its assignment.
> - Each sub-agent must execute its step file step-by-step per `§ TDD`.
> - Each sub-agent makes its own atomic Conventional Commits in the main workspace per `§ Commits`.
>
> **Phase 2B — Parallel-wave path** (selected at Phase 1 step 6 when at least one step file carries `depends-on` frontmatter). For each wave `K` from `0` upward:
>
> 1. **Skip filtering.** Determine which steps in wave `K` are eligible to run: any step whose `depends-on` transitively includes a step that **failed** in an earlier wave of this run is **skipped** (mark it as `skipped-due-to-upstream-failure` for the run summary). Steps whose dependencies all succeeded are eligible.
>
> 2. **Worktree creation.** For each eligible step `S` in wave `K`, per `§ Parallel execution`:
>    - Create `.grimoire/bag/worktrees/page-NNN-step-S/` (where `NNN` is the page number and `S` is the step number).
>    - Run `git worktree add` from the main workspace's current `HEAD`. The runtime MAY use detached worktrees or short-lived branches — pick whichever the host environment supports cleanly.
>
> 3. **Sub-agent spawn (concurrent).** Spawn one sub-agent per eligible step in this wave, **in parallel**, per the amended `§ Sub-agent spawning`. Each sub-agent receives:
>    - The path to its worktree as its working directory.
>    - The path to its step file (still under `.grimoire/pages/NNN-[page-name]/`).
>    - A strict instruction to confine all edits and commits to its worktree directory — must not touch the main workspace, sibling worktrees, or `.grimoire/` state outside the worktree.
>    - The `§ TDD` and `§ Commits` requirements as in the legacy path.
>
> 4. **Wait for wave settlement.** Wait for every spawned sub-agent in this wave to finish (success or failure). **No mid-wave kill** — even if one step fails, in-flight siblings run to completion per `§ Parallel execution`.
>
> 5. **Cherry-pick back to main, in step-number order.** Iterate eligible steps in ascending step-number order:
>    - If the step succeeded → cherry-pick every commit it made in its worktree into the main workspace.
>    - If the step failed → skip it (no commits land); record it as `failed` for the run summary.
>    - Cherry-pick conflicts (caused by overlapping `touches` between siblings, etc.) are reported and the step is treated as failed for downstream wave filtering.
>
> 6. **Unconditional teardown (try/finally invariant).** **Before** moving to wave `K+1`, for **every** worktree created in wave `K` (success, failure, skipped, or never started):
>    - Run `git worktree remove --force <worktree-path>`.
>    - Run a defensive filesystem `rm -rf` on the worktree directory in case `git worktree remove --force` left residue.
>    - Verify with `git worktree list` that no entry remains for this wave's worktrees.
>    - This teardown is **non-negotiable** and runs even on hard-abort/exception paths. Treat it as a try/finally invariant in the skill body.
>
> 7. **End-of-run teardown.** At the end of the run — success, partial failure, or hard abort — `.grimoire/bag/worktrees/` MUST be empty AND `git worktree list` MUST show only the main workspace. If the orchestrator detects any residue at end of run, it MUST attempt a final defensive cleanup and report any worktrees it could not remove.

### 4. Phase 3 — clarify commits coverage

The current Phase 3 is one sentence ("Make commits per `§ Commits` as the execution progresses."). Extend it minimally to note that in the parallel-wave path the per-step commits **originate in the worktree** and arrive on the main branch via cherry-pick in step-number order — so main-branch history reads as if the wave had run serially, even though execution was concurrent. The actual `feat:` / `fix:` / `test:` / `refactor:` commits remain atomic per step per `§ Commits`.

### 5. Phase 4 — finalization split for partial-failure case

The current Phase 4 unconditionally updates HISTORIC from `[planned]` to `[finished]`. Split it:

> **Phase 4 — Finalization:**
> - The page folder, its `SPEC.md`, and its step files **stay in place** — nothing is moved (unchanged from prior behavior).
> - **If every step in the run succeeded** (legacy serial: every sub-agent finished green; parallel-wave: every wave's every step finished green and cherry-picked cleanly):
>   - Update the page's `HISTORIC.md` entry from `[planned]` to `[finished]` per `§ Historic`.
>   - Commit the HISTORIC update per `§ Commits` as `chore: mark page <name> finished`.
> - **If any step failed or was skipped due to upstream failure** (parallel-wave only — legacy serial has no partial-failure state because step `N+1` never starts if step `N` fails):
>   - **Do NOT** update HISTORIC. The status stays `[planned]`.
>   - Report a run summary to the user listing finished steps, failed steps, and skipped steps, and tell them that re-running `/grimoire-execute NNN` will retry from scratch — they SHOULD inspect and, if necessary, revert commits from the partial run before retrying.
>   - **Worktree teardown still runs** per Phase 2B step 7 — partial failure is **not** an excuse to leave residue under `.grimoire/bag/worktrees/`.

## Style + constraints

- **Do not restate the contents of `§ Parallel execution`** in the SKILL.md body — reference it. The worktree contract, frontmatter schema, failure semantics, and back-compat rule all live in conventions.
- Preserve all existing `[Required Reading]` and Objective text unchanged except for the inserted `§ Parallel execution` reference.
- Preserve the SKILL.md frontmatter (`name`, `description`, `argument-hint`) and the `---` delimiters exactly.
- Preserve `$ARGUMENTS` substitutions verbatim.
- The SKILL.md is in English; the new hard-stop messages stay in pt-BR per the existing precedent.
- The teardown invariant must be stated **explicitly and verbosely** in the skill body. This is the dominant safety property of the feature.

## Acceptance

- `[Required Reading]` lists `§ Parallel execution` between `§ Sub-agent spawning` and `§ Commits`.
- Phase 1 has a new step 6 that parses step-file frontmatter, selects legacy-serial vs. parallel-wave path, runs cycle detection + dangling-reference checks for the parallel path, and uses the same hard-stop messages as `§ Pause-point pattern`.
- Phase 2 is split into 2A (legacy strict-serial) and 2B (parallel-wave). 2B describes worktree creation, concurrent sub-agent spawn, wave-settle wait, cherry-pick in step-number order, and the unconditional per-wave + end-of-run teardown invariant — all referencing `§ Parallel execution` for the rules.
- Phase 3 notes the cherry-pick-in-step-number-order property in the parallel path.
- Phase 4 splits finalization: fully-successful run → HISTORIC `[finished]` + commit; partial-failure → HISTORIC stays `[planned]`, run summary reported to the user.
- The SKILL.md never restates the contents of `§ Parallel execution`; it only references the section.

## Commit

After the edit, commit per `§ Commits`:

```
docs(grimoire): teach grimoire-execute to run dependency-graph waves
```

The commit covers only `skills/grimoire-execute/SKILL.md`.

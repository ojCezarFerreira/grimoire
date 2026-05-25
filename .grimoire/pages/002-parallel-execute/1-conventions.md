# Step 1 — Amend `GRIMOIRE-CONVENTIONS.md`

## Scope

This step makes `GRIMOIRE-CONVENTIONS.md` the single source of truth for the parallel-wave execution model. It is the DAG root for this page — steps 2–5 reference the conventions added here. **No code; markdown only.** `§ TDD` does not apply.

## Files touched

- `GRIMOIRE-CONVENTIONS.md` (only this file)

## Edits

### 1. Amend `§ Sub-agent spawning`

The current section mandates strict serial execution for `grimoire-execute`. Amend it to **admit a parallel-wave path under an explicit precondition** (frontmatter present), without weakening the legacy default.

Add a bullet (or rewrite the `grimoire-execute` bullets) so the section reads roughly:

- **`grimoire-execute` on a page with one step file:** execute directly inside one sub-agent.
- **`grimoire-execute` on a page with multiple step files and NO `depends-on` frontmatter:** strict serial — spawn one sub-agent per step file in numeric order; never start step `N+1` until step `N` has fully completed.
- **`grimoire-execute` on a page where any step file declares `depends-on` frontmatter:** parallel-wave mode per `§ Parallel execution`. Steps inside a single wave MAY be spawned concurrently — one sub-agent per step, each in its own git worktree per the worktree contract. Sequencing **across** waves remains strict (wave `K+1` never starts until wave `K` has settled and worktrees have been torn down). Sequencing **inside** a wave is unconstrained.
- **`grimoire-quick`:** unchanged — spawn a single sub-agent for the entire fix once the user has authorized the inline plan.

The "never start the next step until the current sub-agent has fully completed" rule remains the **default** when no frontmatter is present.

### 2. Add a brand-new `§ Parallel execution` section

Place it **after `§ Sub-agent spawning`** and **before `§ Commits`**. This is the single source of truth — SKILL.md bodies must reference it instead of restating the rules.

The section must exhaustively document:

**Frontmatter schema** — YAML at the top of a step file:
- `depends-on: [<step-numbers>]` — list of step numbers this step depends on. Missing or empty means the step is a DAG root.
- `touches: [<repo-relative paths>]` — list of paths the step will modify. Optional but recommended; used for advisory overlap warnings at plan time.
- Body of the step file is markdown as before; the frontmatter is parsed and stripped by the runtime, the markdown body is what the sub-agent executes.

**DAG construction** — nodes are step files; edges are `depends-on` references (edge from `K` to `N` when `N` lists `K` in its `depends-on`).
- **Cycle detection is mandatory** at plan time (refuse to write step files) and at execute time (refuse to start the run). Defense in depth.
- A `depends-on` reference to a non-existent step number is a hard error at both plan time and execute time.

**Wave construction (Kahn-style topological levels)**:
- Wave `0` = all steps with empty `depends-on`.
- Wave `K` = all steps whose entire `depends-on` set lies in waves `< K`.
- All steps in a wave run **concurrently**. Waves run **sequentially** (`0`, then `1`, then `2`, …).

**Worktree contract (load-bearing — non-negotiable)**:
- **Location:** `.grimoire/bag/worktrees/page-NNN-step-K/` (deterministic, page- and step-scoped).
- **Creation:** at the start of each wave, `git worktree add` from the main workspace's current `HEAD` (one worktree per step in the wave). Branch name is derived per worktree; the runtime MAY use detached worktrees or short-lived branches — the only requirement is that the worktree is independently committable.
- **Per-step sub-agent isolation:** each step's sub-agent works **only inside its worktree directory**; it must not touch the main workspace, sibling worktrees, or `.grimoire/` state files outside its worktree.
- **Cherry-pick back to main:** when the wave settles (all step sub-agents have either succeeded or failed), the orchestrator cherry-picks each successful step's commits back to the main workspace **in step-number order** (lowest step number first). The main branch history reads as if the wave had run serially.
- **Unconditional teardown:** before starting the next wave, and at end of run regardless of outcome, the orchestrator runs `git worktree remove --force <path>` for every worktree it created in that wave, plus a defensive filesystem `rm -rf` on any residual directory under `.grimoire/bag/worktrees/`. **At the end of any `grimoire-execute` run (success, partial failure, or hard abort), `.grimoire/bag/worktrees/` MUST be empty, and `git worktree list` MUST show only the main workspace.**
- The skill body must enumerate teardown as a try/finally-shaped invariant. The maintainer has been bitten by stale worktree clutter before; this is the dominant safety property of the feature.

**Failure semantics**:
- A failed step's worktree is **discarded**: no commits from that worktree are cherry-picked back.
- Successful **siblings** in the same wave **still cherry-pick back** in step-number order, skipping the failed step's slot.
- **Downstream waves are filtered**: any step whose `depends-on` set transitively includes the failed step is **skipped**. Independent downstream waves still run.
- **No mid-wave kill.** When one step fails, the orchestrator waits for the other in-flight siblings to finish (success or failure) before declaring the wave result.
- **HISTORIC stays `[planned]`** on partial failure. The page is not marked `[finished]` unless every step succeeded. The user re-runs `/grimoire-execute NNN` to retry; the re-run starts from scratch (no resume state file in v1).
- **No concurrency cap** inside `grimoire-execute` itself. Wave members all spawn at once. The runtime's existing sub-agent limits are the de-facto cap.

**Back-compat fallback**:
- If **no step file** in the page carries `depends-on` frontmatter, `grimoire-execute` falls back to the **legacy strict-serial path** — byte-identical to the pre-0.8.0 behavior. No worktrees, no waves. Detected in `grimoire-execute` Phase 1.
- Existing pages (e.g., `001-grimoire-note-skill`) are unaffected — no auto-migration, no silent re-planning.

**Commits (cross-reference `§ Commits`, do not restate)**:
- Per-step atomic commits remain `feat:`, `fix:`, `test:`, etc., made by each sub-agent inside its worktree.
- The page-level finalization commit `chore: mark page <name> finished` is unchanged.

### 3. Extend `§ .grimoire/ layout`

In the **Step files inside a page folder** sub-section, add a sentence at the end:

> Step files MAY carry YAML frontmatter per `§ Parallel execution` to declare inter-step dependencies and the files they touch. The canonical body of the step file remains markdown; the frontmatter is parsed by `grimoire-plan` (validation) and `grimoire-execute` (wave construction).

### 4. Extend `§ Pause-point pattern` — `grimoire-plan: final clarity check`

Extend the existing **`grimoire-plan — final clarity check`** bullet to require **DAG presentation** when frontmatter is being emitted. Append (roughly):

> When the planner has decided to emit `depends-on` frontmatter on one or more step files, the final clarity check MUST also surface the full DAG to the user — list each step's `depends-on` set, the computed waves, and any advisory `touches`-overlap warnings — so the user can veto a suspect parallelization before step files are written. If no frontmatter will be emitted, this DAG-presentation requirement does not apply.

Also extend the `grimoire-execute — precondition check (HARD STOP)` bullet by adding a hard-stop case **after** the existing four hard-stop cases:

> - Cyclic `depends-on` graph detected (any cycle in the DAG) → `❌ Page NNN tem dependências cíclicas entre steps. Rode /grimoire-plan NNN para corrigir.` and STOP.
> - `depends-on` reference to a non-existent step number → `❌ Page NNN referencia um step inexistente em depends-on. Rode /grimoire-plan NNN para corrigir.` and STOP.

(The exact wording above is a suggestion — match the style of the existing pt-BR hard-stop messages.)

## Style + constraints

- All conventions content stays in **English**. Operator-facing hard-stop messages in the new pause-point cases use **pt-BR**, matching the existing precedent.
- **Single source of truth.** Do not duplicate any of these rules into SKILL.md bodies. The SKILL.md updates in steps 2 and 3 will only *reference* `§ Parallel execution`.
- Preserve all other sections (`§ TDD`, `§ Commits`, `§ Project context`, `§ Historic`, `§ IDE-aware review`) byte-identical unless an explicit edit above touches them.
- Do not renumber the existing sections; add `§ Parallel execution` between `§ Sub-agent spawning` and `§ Commits`.

## Acceptance

- `GRIMOIRE-CONVENTIONS.md` contains a new `§ Parallel execution` section covering frontmatter schema, DAG construction, waves, worktree contract (with mandatory unconditional teardown), failure semantics, and the back-compat fallback rule.
- `§ Sub-agent spawning` is amended to admit parallel sub-agents inside a wave under the frontmatter-present precondition; legacy strict-serial remains the default.
- `§ .grimoire/ layout` notes that step files MAY carry YAML frontmatter per `§ Parallel execution`.
- `§ Pause-point pattern` `grimoire-plan: final clarity check` requires DAG presentation when frontmatter is being emitted, and `grimoire-execute: precondition check (HARD STOP)` lists the two new hard-stop cases (cycle, dangling `depends-on`).
- No other sections changed.

## Commit

After the edit, commit per `§ Commits`:

```
docs(grimoire): add § Parallel execution and amend supporting sections
```

The commit covers only `GRIMOIRE-CONVENTIONS.md`. Do not bundle other files in this commit.

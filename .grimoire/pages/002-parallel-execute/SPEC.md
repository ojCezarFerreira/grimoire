# 002 — Parallel `grimoire-execute`

## Context

Today `grimoire-execute` runs every step file of a page strictly sequentially: `§ Sub-agent spawning` mandates "never start step N+1 until the sub-agent for step N has fully completed." This is safe but slow when steps are *factually* independent. Page `001-grimoire-note-skill` shipped with five step files; in retrospect, several (e.g. `4-docs-update` and `5-version-bump-and-changelog`) touch disjoint files and could have run in parallel without risk.

This page introduces a **dependency-graph execution model** for `grimoire-execute`:

- `grimoire-plan` emits a DAG declaring which steps depend on which (and which files each step touches), encoded as YAML frontmatter inside each step file.
- `grimoire-execute` computes topologically-sorted **waves**, runs all steps in a wave in parallel via one sub-agent per step, each inside an isolated **git worktree**, and cherry-picks commits back to the main workspace once the wave completes.
- Worktrees are aggressively cleaned up — the maintainer has been bitten by stale worktree clutter before; the contract is **no residual worktrees after any run, success or failure**.
- Pages without dependency frontmatter run *exactly* as they do today (strict serial). Parallelism is strictly opt-in.

## Goals

- Define a step-file YAML frontmatter format declaring `depends-on: [step-numbers]` and `touches: [paths]`.
- Teach `grimoire-plan` to emit that frontmatter when the page's structure benefits from parallelism, to validate the resulting DAG (cycle detection mandatory; `touches`-overlap warning advisory), and to surface the wave plan to the user at the existing final-clarity-check pause-point.
- Teach `grimoire-execute` to detect frontmatter, compute waves via topological sort, and run wave members concurrently — one sub-agent per step, each inside its own git worktree.
- **Worktree lifecycle (load-bearing).** Worktrees are created at wave start under a deterministic path (`.grimoire/bag/worktrees/page-NNN-step-K/`), branched from the main workspace's current HEAD; commits from each successful sibling are cherry-picked back to the main workspace in step-number order once the wave settles; **the worktree directory and its `git worktree` registration are removed unconditionally before the next wave starts, including on failure or hard abort**.
- Partial-wave failure: the failed step's worktree is discarded (no commits land); successful siblings still cherry-pick back; downstream waves whose deps transitively include the failed step are skipped; independent downstream waves still run; HISTORIC stays `[planned]` and the user re-runs `/grimoire-execute NNN` to retry.
- Backward compatibility: any page where no step file carries the new frontmatter executes via the legacy strict-serial code path. No existing page must be re-planned for the new format to ship.
- Amend `GRIMOIRE-CONVENTIONS.md`:
  - `§ Sub-agent spawning` to admit the parallel wave model under an explicit precondition (frontmatter present).
  - A new `§ Parallel execution` section consolidating frontmatter format, DAG semantics, worktree contract, failure handling, and back-compat — as the single source of truth so SKILL.md bodies do not duplicate the rules.
  - `§ .grimoire/ layout` to note step files MAY carry YAML frontmatter.
  - `§ Pause-point pattern` (`grimoire-plan: final clarity check`) to require surfacing the DAG to the user when frontmatter is being emitted.
- Update `skills/grimoire-plan/SKILL.md` and `skills/grimoire-execute/SKILL.md` to reference the new conventions and describe their respective new responsibilities.
- Document the capability in `README.md`, `README.pt-BR.md`, `CLAUDE.md`.
- Bump plugin version (minor: `0.7.0 → 0.8.0`) and add a matching `CHANGELOG.md` entry.

## Non-goals

- **No change to `grimoire-spec`, `grimoire-quick`, `grimoire-init`, `grimoire-note`, `grimoire-know`, or `grimoire-update`.** They don't touch step files or the execution loop.
- **No change to HISTORIC semantics.** Status progression remains `[spec]` → `[planned]` → `[finished]`. On partial-wave failure the page stays `[planned]` (already the natural state) until a re-run completes all steps.
- **No `WAVES.md` / `DAG.md` artifact in v1.** The DAG lives in step files' frontmatter; `grimoire-execute` reconstructs it at runtime. A separate visualization file is a possible v2 (Open Questions).
- **No mid-wave kill.** When a step fails inside a wave, the orchestrator waits for the other in-flight siblings to finish (success or failure) before declaring the wave result. Premature cancellation is out of scope.
- **No resume-from-partial-success state file.** Re-invoking `/grimoire-execute NNN` after a partial failure re-runs *every* step from scratch (in a fresh worktree). It is the user's responsibility to inspect and, if necessary, revert commits from the prior partial run before retrying. The per-step TDD pattern makes most steps idempotent in practice; the brittle cases stay an Open Question.
- **No concurrency cap inside `grimoire-execute`.** Wave members all spawn at once. The runtime's existing sub-agent limits are the de-facto cap.
- **No retroactive migration.** Existing pages (`001-grimoire-note-skill`) stay as-is and continue to execute serially. No auto-emit of frontmatter into legacy step files.
- **`§ TDD` does not apply to this page.** Like other prompt-text pages, the artifacts are markdown/JSON.

## Scope

### Modified files

- `GRIMOIRE-CONVENTIONS.md`:
  - **`§ Sub-agent spawning`** — amended to state that `grimoire-execute` MAY spawn multiple sub-agents in parallel within a single wave when the page's step files carry the new frontmatter; sequencing across waves remains strict; sequencing across steps inside a wave is unconstrained.
  - **New `§ Parallel execution`** — single source of truth for:
    - Step-file frontmatter schema: `depends-on` (list of step numbers; missing/empty = root step), `touches` (list of repo-relative paths; optional but recommended).
    - DAG construction: nodes = step files, edges = `depends-on` references; cycle detection mandatory at plan time and execute time (defense in depth).
    - Wave construction: Kahn-style topological levels; wave K = all steps whose entire `depends-on` set lives in waves `< K`.
    - Worktree contract: location (`.grimoire/bag/worktrees/page-NNN-step-K/`); creation (`git worktree add` from main-workspace HEAD at wave start); per-step sub-agent isolation; commit cherry-pick back to main workspace in step-number order; **unconditional teardown** (`git worktree remove --force` + filesystem cleanup of any residue) before the next wave starts. `.grimoire/bag/worktrees/` MUST be empty at the end of any `grimoire-execute` run, whether successful or aborted, and `git worktree list` MUST show only the main workspace.
    - Failure semantics: failed step → worktree discarded, no commits cherry-picked for that step; siblings' commits still applied; downstream waves with transitive dep on the failed step skipped; HISTORIC stays `[planned]`; user re-runs to retry.
    - Back-compat rule: if no step file in the page carries `depends-on` frontmatter, `grimoire-execute` falls back to strict-serial legacy mode. Detected at execute Phase 1.
  - **`§ .grimoire/ layout`** — add a sentence noting that step files MAY carry YAML frontmatter per `§ Parallel execution`; the canonical body of the step file is still markdown.
  - **`§ Pause-point pattern`** — extend `grimoire-plan: final clarity check` to require surfacing the DAG (waves + step-by-step deps + touches) to the user when frontmatter is being emitted, so the user can veto a suspect parallelization before step files are written.
- `skills/grimoire-plan/SKILL.md`:
  - Phase 3 (Planning) extended: when the step breakdown contains genuinely independent steps, the planner MAY emit the new frontmatter. When all steps are inherently sequential, the planner emits no frontmatter and the page remains a legacy serial page.
  - New explicit substep: build the DAG (cycle check + advisory touches-overlap warning) before writing step files; surface the wave plan to the user at the final-clarity-check pause-point.
  - Step-file writing now includes the YAML frontmatter block when applicable.
  - `[Required Reading]` adds `§ Parallel execution`.
- `skills/grimoire-execute/SKILL.md`:
  - Phase 1 precondition unchanged in spirit, plus a new detection: parse step-file frontmatter; if any step has `depends-on`, use the parallel-wave path; otherwise fall back to legacy serial.
  - Phase 2 (Execution) replaced/extended with the wave loop:
    1. Compute waves (re-validate DAG; hard-stop on cycle).
    2. For each wave: create one worktree per step; spawn one sub-agent per step targeting its worktree; wait for all to complete; cherry-pick successful siblings' commits in step-number order; tear down all worktrees in the wave (try/finally-shaped invariant).
    3. Branch downstream wave eligibility on failure transitivity.
  - Phase 4 finalization unchanged for fully-successful runs. For partial-failure runs: do **not** mark `[finished]`; report finished / failed / skipped steps to the user.
  - `[Required Reading]` adds `§ Parallel execution`.
- `README.md`, `README.pt-BR.md` — add a short paragraph to the `grimoire-execute` description noting the new wave model and the opt-in via frontmatter.
- `CLAUDE.md` — extend the `grimoire-plan` and `grimoire-execute` bullets in "The skills and how they connect" with the new behavior; add a one-line pointer to `§ Parallel execution`.
- `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` — bump `0.7.0` → `0.8.0` in lockstep.
- `CHANGELOG.md` — new `0.8.0` section covering: parallel waves in `grimoire-execute`, dep-graph emission in `grimoire-plan`, frontmatter format addition, worktree-based isolation with mandatory cleanup, back-compat note.

### Untouched

- `skills/grimoire-spec/SKILL.md`, `skills/grimoire-quick/SKILL.md`, `skills/grimoire-init/SKILL.md`, `skills/grimoire-note/SKILL.md`, `skills/grimoire-know/SKILL.md`, `skills/grimoire-update/SKILL.md` — none participate in step-file generation or execution.
- The HISTORIC file format and the existing `[spec] → [planned] → [finished]` lifecycle.
- The page folder layout (`.grimoire/pages/NNN-[page-name]/`).
- Existing pages (`001-grimoire-note-skill`) — they retain serial execution under the back-compat rule.

## Acceptance criteria

- `GRIMOIRE-CONVENTIONS.md` has a new `§ Parallel execution` section that exhaustively documents:
  - the frontmatter schema (`depends-on`, `touches`),
  - DAG construction rules,
  - wave semantics (Kahn-style topological levels),
  - the worktree lifecycle (creation path, branching from current HEAD, cherry-pick in step-number order, **mandatory teardown — `git worktree remove --force` + filesystem residue cleanup — before the next wave and at end of run, regardless of outcome**),
  - failure handling (rollback failed step, keep siblings, skip transitive-downstream waves, HISTORIC stays `[planned]`),
  - back-compat fallback (no `depends-on` anywhere → legacy serial path).
- `GRIMOIRE-CONVENTIONS.md § Sub-agent spawning` is amended to allow parallel sub-agents within a wave under the explicit precondition that frontmatter is present and the worktree contract is honored.
- `GRIMOIRE-CONVENTIONS.md § .grimoire/ layout` notes that step files MAY carry YAML frontmatter per `§ Parallel execution`.
- `GRIMOIRE-CONVENTIONS.md § Pause-point pattern` extends `grimoire-plan: final clarity check` to require DAG presentation when frontmatter is emitted.
- `skills/grimoire-plan/SKILL.md` Phase 3 instructs the planner to emit frontmatter when steps are independent, surface the DAG at the final-clarity-check pause-point, and refuse to emit a cyclic DAG; `[Required Reading]` references `§ Parallel execution`.
- `skills/grimoire-execute/SKILL.md` Phase 1 detects frontmatter presence and selects the wave path or the legacy serial path accordingly; Phase 2 implements the wave loop with explicit worktree teardown after each wave; partial-failure handling matches `§ Parallel execution`; `[Required Reading]` references `§ Parallel execution`.
- A synthetic end-to-end smoke test on a 4-step page where step 1 is the root, steps 2 and 3 depend on step 1 (sibling pair), and step 4 depends on both 2 and 3:
  - `grimoire-execute` runs step 1 alone, then steps 2 and 3 in a single wave (two worktrees), then step 4 alone. Commits land in step-number order in the main branch.
  - After the run, `.grimoire/bag/worktrees/` is empty AND `git worktree list` shows only the main workspace.
  - Page entry in `HISTORIC.md` moves to `[finished]`; commit log contains a `chore: mark page <name> finished` commit.
- Partial-failure smoke test (same 4-step page; the step-3 sub-agent fails):
  - Step 2's commits land; step 3's worktree is discarded with no commits cherry-picked; step 4 is skipped (transitive dep on step 3).
  - `.grimoire/bag/worktrees/` is empty at the end of the (aborted) run; `git worktree list` clean.
  - Page entry in `HISTORIC.md` stays `[planned]`; user is told which steps finished, which failed, and which were skipped, and that re-running `/grimoire-execute NNN` will retry from scratch.
- Back-compat smoke test: re-executing `001-grimoire-note-skill` (no frontmatter on any step file) takes the legacy serial path verbatim — no worktrees created, behavior byte-identical to the current implementation.
- `README.md`, `README.pt-BR.md`, and `CLAUDE.md` all describe the new wave model in their respective `grimoire-execute` / `grimoire-plan` entries.
- `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` versions match at `0.8.0`.
- `CHANGELOG.md` has a `0.8.0` section enumerating the parallel-wave feature, frontmatter format, worktree contract (with explicit cleanup guarantee), and the back-compat statement.

## Constraints

- **Single source of truth.** All shared, load-bearing rules go to `GRIMOIRE-CONVENTIONS.md` (new `§ Parallel execution`, amendments to `§ Sub-agent spawning`, `§ .grimoire/ layout`, `§ Pause-point pattern`). SKILL.md bodies reference the conventions, they never restate them.
- **Worktree cleanup is non-negotiable.** Every `grimoire-execute` run, whether it ends in success, partial failure, or hard error, MUST leave `.grimoire/bag/worktrees/` empty and MUST run `git worktree remove --force` on every worktree it created — followed by a defensive filesystem removal of any leftover directory. The skill body must explicitly enumerate teardown as a try/finally-shaped invariant. The maintainer has been bitten by stale worktree clutter before; this is the dominant safety property of the feature.
- **Pure opt-in back-compat.** Pages without `depends-on` frontmatter behave byte-identically to the current implementation. No silent re-planning, no auto-emit on legacy pages. Detection lives in `grimoire-execute` Phase 1.
- **Atomic commits per step are preserved.** Even when steps run in parallel, each sub-agent makes its own granular commits inside its worktree per `§ TDD` and `§ Commits`. Cherry-pick back to main applies those commits in step-number order — so the main branch history reads as if the wave had run serially, even though the work itself was concurrent.
- **Cycle detection at plan time AND execute time.** A cyclic `depends-on` graph is a hard error in `grimoire-plan` (refuses to write step files) and in `grimoire-execute` Phase 1 (refuses to start the run). It never silently degrades to serial — the user must fix the dependencies.
- **Conventional Commits in English.** Step-level commits remain `feat:`, `fix:`, `test:`, etc. The page-level commit `chore: mark page <name> finished` stays as-is.
- **Page vocabulary.** All references to the Spec → Plan → Execute model use "page".
- **Version bump magnitude.** Minor (`0.7.0` → `0.8.0`). New capability, additive frontmatter format, no breaking change to existing pages — minor matches the precedent of `grimoire-know` (`0.5.0` → `0.6.0`) and `grimoire-note` (`0.6.0` → `0.7.0`).
- **Languages.** SPEC and conventions content in English; operator-facing messages may switch to pt-BR per the existing precedent in `§ Pause-point pattern`.

## Open questions

- **Resume semantics after partial failure.** v1 re-runs every step on re-invocation. A future page may add a per-step success marker (e.g., commit-trailer or sidecar state file) so a retry skips already-completed steps. Out of scope for now.
- **`touches`-overlap policy.** `touches` is recommended but not required. When two parallel siblings declare overlapping `touches`, worktrees make this *technically* safe (each gets its own filesystem) but the cherry-pick may conflict. v1: surface the overlap as a warning at the final-clarity-check pause-point; let the user decide whether to keep them parallel or force a serial edge. v2 could be more opinionated.
- **Concurrency cap.** v1 spawns all wave members at once. If runtime sub-agent limits become a problem, `grimoire-execute` could grow a configurable cap (e.g., max 4 concurrent). Track separately.
- **WAVES.md visualization.** v1 keeps the DAG implicit in step-file frontmatter. If wave plans grow complex, a generated `WAVES.md` rendering the topology at planning time would help the user audit before approving — possible v2.
- **Worktree base ref.** v1 branches each wave's worktrees from the main workspace's HEAD at wave start. If multiple `grimoire-execute` invocations interleave on the same repo (unlikely but possible), branch hygiene gets murkier — currently considered out of scope.
- **Step-number renumbering.** When `grimoire-plan` decides on a different step ordering during re-planning, the `depends-on` references would need to be remapped. v1 assumes step numbers are stable within a page once written; if `grimoire-plan` rewrites the plan, it rewrites the frontmatter consistently too.
- **Step-file breakdown for `grimoire-plan` (this page).** A reasonable split is: (1) new `§ Parallel execution` + `§ Sub-agent spawning` amendment in `GRIMOIRE-CONVENTIONS.md` (plus the small `§ .grimoire/ layout` and `§ Pause-point pattern` extensions); (2) `skills/grimoire-plan/SKILL.md` updates; (3) `skills/grimoire-execute/SKILL.md` updates; (4) docs (`README.md`, `README.pt-BR.md`, `CLAUDE.md`); (5) version bump + `CHANGELOG.md`. Final breakdown is `grimoire-plan`'s call — note especially that step (1) is the natural DAG root and steps (2) and (3) depend on it. Steps (4) and (5) are likely independent siblings, and a good dogfood opportunity for the new format.

## References

- [GRIMOIRE-CONVENTIONS.md](../../../GRIMOIRE-CONVENTIONS.md) — `§ Sub-agent spawning` (amend), `§ .grimoire/ layout` (extend), `§ Pause-point pattern` (extend `grimoire-plan: final clarity check`), `§ Commits` (no change). The new `§ Parallel execution` lands here.
- [skills/grimoire-plan/SKILL.md](../../../skills/grimoire-plan/SKILL.md) — Phase 3 and Phase 4 are the modification targets; the `[Required Reading]` block adds `§ Parallel execution`.
- [skills/grimoire-execute/SKILL.md](../../../skills/grimoire-execute/SKILL.md) — Phase 1 (frontmatter detection + cycle check) and Phase 2 (wave loop, worktrees, cherry-pick, cleanup) are the modification targets; the `[Required Reading]` block adds `§ Parallel execution`.
- [.grimoire/pages/001-grimoire-note-skill/](../../001-grimoire-note-skill/) — example of a 5-step page where steps 4–5 could have parallelized; useful as the back-compat regression test target.
- [.claude-plugin/plugin.json](../../../.claude-plugin/plugin.json), [.claude-plugin/marketplace.json](../../../.claude-plugin/marketplace.json) — version bump targets, lockstep.
- [CHANGELOG.md](../../../CHANGELOG.md) — release notes pattern.
- [CLAUDE.md](../../../CLAUDE.md) — maintainer-facing skill enumeration; updated `grimoire-plan` and `grimoire-execute` bullets.
- `git worktree` documentation (https://git-scm.com/docs/git-worktree) — underlying primitive used for sub-agent isolation; `git worktree add`, `git worktree remove --force`, `git worktree list` are the relevant commands.

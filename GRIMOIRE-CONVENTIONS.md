# Grimoire Conventions

Shared, load-bearing rules for the Grimoire workflow skills (`grimoire-init`, `grimoire-spec`, `grimoire-plan`, `grimoire-execute`, `grimoire-quick`). Each SKILL.md references this file as required reading. Do not duplicate these rules inside skill bodies — update them here.

The long-form pipeline is **Spec → Plan → Execute**: `grimoire-spec` gathers context and writes the page's `SPEC.md`, `grimoire-plan` reads that SPEC and writes the executable step files, `grimoire-execute` runs the step files. `grimoire-quick` is a separate fast-path that bypasses all three.

---

## § TDD

Every code change follows strict Red → Green → Refactor.

**CRITICAL:** NEVER execute the entire test suite. Use test filters (e.g., `--filter` in PHPUnit/Pest, `-k` in pytest, exact file paths in Jest/Vitest, etc.) to run ONLY the tests created or modified in the current step.

1. **Red:** Write or modify the tests for the current step. Run ONLY these tests and verify failure.
2. **Green:** Implement the minimum code needed. Run the same filtered tests again and verify they pass.
3. **Refactor:** Clean up the code while keeping the filtered tests green.

The same loop applies regardless of whether the work comes from `grimoire-plan`, `grimoire-execute`, or `grimoire-quick`.

---

## § Sub-agent spawning

The main context is an orchestrator. Implementation work runs inside sub-agents so the orchestrator's context stays lucid.

- **`grimoire-execute` on a page with one step file:** execute directly inside one sub-agent.
- **`grimoire-execute` on a page with multiple step files and NO `depends-on` frontmatter:** strict serial — spawn one sub-agent **per step file** in numeric order; never start step `N+1` until step `N` has fully completed. This is the default and matches the pre-0.8.0 behavior.
- **`grimoire-execute` on a page where any step file declares `depends-on` frontmatter:** parallel-wave mode per `§ Parallel execution`. Steps inside a single wave MAY be spawned **concurrently** — one sub-agent per step, each operating inside its own git worktree per the worktree contract. Sequencing **across** waves remains strict: wave `K+1` never starts until wave `K` has settled and all of its worktrees have been torn down. Sequencing **inside** a wave is unconstrained.
- **`grimoire-quick`:** spawn a single sub-agent for the entire fix once the user has authorized the inline plan.

Always instruct the sub-agent with the specific step file (or inline plan) it must execute, and require it to follow `§ TDD` and `§ Commits`.

---

## § Parallel execution

`grimoire-execute` MAY run mutually-independent step files of the same page concurrently. This section is the **single source of truth** for the parallel-wave model — SKILL.md bodies reference it; they never restate the rules.

The model is **strictly opt-in**. A page becomes eligible for parallel execution only when at least one of its step files declares `depends-on` in its YAML frontmatter (see schema below). A page where no step file carries `depends-on` runs via the legacy strict-serial path defined in `§ Sub-agent spawning` — byte-identical to the pre-0.8.0 behavior.

### Frontmatter schema

Step files MAY carry a YAML frontmatter block at the very top of the file, delimited by `---` lines:

```
---
depends-on: [<step-numbers>]
touches: [<repo-relative paths>]
---

# Step N — …

(markdown body as before)
```

- `depends-on` — list of step numbers (integers) this step depends on. **Missing or empty means the step is a DAG root** (eligible for wave `0`).
- `touches` — list of repo-relative paths the step will modify. **Optional but recommended.** Used at plan time to surface advisory overlap warnings between parallel siblings; not enforced at execute time.
- The markdown body below the frontmatter is the canonical step content the sub-agent executes; the frontmatter is parsed and stripped by the runtime.

### DAG construction

- **Nodes** are step files (identified by step number).
- **Edges** are `depends-on` references: an edge from step `K` to step `N` exists when step `N` lists `K` in its `depends-on`.
- **Cycle detection is mandatory** at both plan time (refuse to write step files) and execute time (refuse to start the run). Defense in depth — neither side may assume the other validated the graph.
- **Dangling references are a hard error.** A `depends-on` value pointing at a step number that does not exist in the page is a hard error at both plan time and execute time. It is never silently ignored.

### Wave construction (Kahn-style topological levels)

- **Wave `0`** = the set of all steps whose `depends-on` is missing or empty (DAG roots).
- **Wave `K`** = the set of all steps whose entire `depends-on` set lies in waves `< K`.
- Waves are computed deterministically from the DAG at the start of the run.
- All steps in a wave run **concurrently**. Waves run **sequentially** in increasing order (`0`, then `1`, then `2`, …).

### Worktree contract (load-bearing — non-negotiable)

- **Location.** Each step's worktree lives at the deterministic, page- and step-scoped path `.grimoire/bag/worktrees/page-NNN-step-K/`, where `NNN` is the zero-padded page number and `K` is the step number.
- **Creation.** At the start of each wave, the orchestrator runs `git worktree add` for every step in the wave, branching from the main workspace's current `HEAD`. The runtime MAY use detached worktrees or short-lived branches — the only requirement is that each worktree is independently committable.
- **Per-step sub-agent isolation.** Each step's sub-agent works **only inside its own worktree directory**. It must not touch the main workspace, sibling worktrees, or any `.grimoire/` state file outside its worktree.
- **Cherry-pick back to main.** Once the wave settles (every step sub-agent has either succeeded or failed), the orchestrator cherry-picks each **successful** step's commits back to the main workspace **in step-number order** (lowest step number first). The main branch history reads as if the wave had run serially, even though the work itself was concurrent.
- **Unconditional teardown.** Before starting the next wave, **and** at the end of the run regardless of outcome (success, partial failure, or hard abort), the orchestrator runs `git worktree remove --force <path>` for every worktree it created in that wave, followed by a defensive filesystem `rm -rf` on any residual directory under `.grimoire/bag/worktrees/`. **At the end of any `grimoire-execute` run, `.grimoire/bag/worktrees/` MUST be empty AND `git worktree list` MUST show only the main workspace.**
- The skill body must enumerate teardown as a **try/finally-shaped invariant** — teardown runs whether the wave succeeded, failed, partially failed, or the run was aborted. The maintainer has been bitten by stale worktree clutter before; this is the dominant safety property of the feature.

### Failure semantics

- **Failed step → worktree discarded.** No commits from a failed step's worktree are cherry-picked back to the main workspace. The worktree is removed per the teardown rule above.
- **Successful siblings still cherry-pick.** Other steps in the same wave that succeeded **still apply their commits** back to the main workspace, in step-number order, skipping the failed step's slot.
- **Downstream waves are filtered.** Any step in a later wave whose `depends-on` set **transitively** includes the failed step is **skipped**. Independent downstream waves (no transitive dep on the failure) still run.
- **No mid-wave kill.** When one step inside a wave fails, the orchestrator waits for the other in-flight siblings to finish (success or failure) before declaring the wave result. Premature cancellation is out of scope.
- **HISTORIC stays `[planned]`** on partial failure. The page is **not** marked `[finished]` unless every step succeeded. The user re-runs `/grimoire-execute NNN` to retry; the re-run **starts from scratch** — there is no resume-from-partial-success state file in v1.
- **No concurrency cap** inside `grimoire-execute` itself. All wave members are spawned at once. The runtime's existing sub-agent limits are the de-facto cap.

### Back-compat fallback

- If **no step file** in the page carries `depends-on` frontmatter, `grimoire-execute` falls back to the **legacy strict-serial path** defined in `§ Sub-agent spawning`. Behavior is **byte-identical to the pre-0.8.0 implementation**: no worktrees are created, no waves are computed, sub-agents are spawned one at a time in numeric order.
- This detection happens in `grimoire-execute` Phase 1, before any execution begins.
- Existing pages (e.g., `001-grimoire-note-skill`) are unaffected — no auto-migration, no silent re-planning, no auto-emit of frontmatter on legacy step files.

### Commits

Per-step atomic commits and the page-level finalization commit follow `§ Commits`. This section does not restate those rules — it only notes that step-level commits are made by the sub-agent **inside its worktree**, and that cherry-pick-back to the main workspace preserves their atomicity and order.

---

## § Commits

- Make granular, atomic commits as work progresses.
- Commit immediately after each logical step completes (typically end of Green or Refactor).
- Use **Conventional Commits in English**: `feat:`, `fix:`, `test:`, `refactor:`, `chore:`, `style:`, `docs:`.
- One logical change per commit — do not bundle unrelated edits.
- `grimoire-spec` ends its run with two commits: `docs(grimoire): spec page <name>` for the new `SPEC.md` file, followed by `chore: register page <name> in historic` covering the new `HISTORIC.md` entry (and any rotation to `.grimoire/bag/historic/`).
- `grimoire-plan` ends its run with a `chore: mark page <name> planned` commit covering the in-place `HISTORIC.md` status update from `[spec]` to `[planned]`. This commit is separate from the step-file commit(s). `grimoire-plan` **never bootstraps, appends, or rotates** `HISTORIC.md`.
- `grimoire-execute` ends its run with a `chore: mark page <name> finished` commit covering the in-place `HISTORIC.md` status update from `[planned]` to `[finished]`. No files are moved on completion.

---

## § .grimoire/ layout

Every feature, change, or alteration tracked by Grimoire is a **page**. All pages live under `.grimoire/pages/` in the consuming project, regardless of whether they are planned, in progress, or finished. **There is no `.grimoire/finished/` directory** — status lives in `HISTORIC.md`, not in the filesystem.

**Page folder (mandatory for every page):**
- Path: `.grimoire/pages/NNN-[page-name]/` (e.g., `.grimoire/pages/001-user-auth/`).
- `NNN` is incremental and chronological across the whole project (`001`, `002`, `003`, …).
- The folder is **created by `grimoire-spec`** when the page is first specified. `grimoire-plan` and `grimoire-execute` never create page folders — they look up an existing one by number.

**Spec file inside a page folder:**
- `SPEC.md` at the root of the page folder describes the page in detail (Context, Goals, Non-goals, Scope, Acceptance criteria, Constraints, Open questions, References to code).
- Written by `grimoire-spec`; consumed by `grimoire-plan` as the source of truth for what to plan.
- A page without `SPEC.md` is not eligible for planning — `grimoire-plan` hard-stops if it is missing.

**Step files inside a page folder:**
- Always sequential, page-scoped numbering: `1-[step-name].md`, `2-[step-name].md`, `3-[step-name].md`, … (e.g., `1-schema.md`, `2-endpoints.md`, `3-tests.md`).
- A simple page has a single step file (`1-[step-name].md`); a larger page has multiple.
- Written by `grimoire-plan`. `grimoire-plan` chooses how many step files a page needs based on projected context lucidity per step.
- Step files MAY carry YAML frontmatter per `§ Parallel execution` to declare inter-step dependencies and the files they touch. The canonical body of the step file remains markdown; the frontmatter is parsed by `grimoire-plan` (validation) and `grimoire-execute` (wave construction).

**On completion (`grimoire-execute`):**
- The page folder, its `SPEC.md`, and its step files stay in place. **Nothing is moved.**
- The page's entry in `HISTORIC.md` is updated from `[planned]` to `[finished]`. See `§ Historic`.

**Project context file:**
- `.grimoire/PROJECT.md` — persistent project context generated and updated by `grimoire-init`. Loaded by every skill's `[Required Reading]` block. Not subject to `NNN-` numbering.

**Recent execution log:**
- `.grimoire/HISTORIC.md` — recency log and status-of-record for the last 5 pages. Bootstrapped, appended, and rotated by `grimoire-spec`; status-updated to `[planned]` by `grimoire-plan`; status-updated to `[finished]` by `grimoire-execute`. See `§ Historic`.
- `.grimoire/bag/historic/` — archive. When `HISTORIC.md` already has 5 entries and a new page must be added, the file is rotated here with a `HISTORIC-N.md` suffix before a fresh `HISTORIC.md` is started.

---

## § Project context

Every Grimoire skill loads `.grimoire/PROJECT.md` (if present) at the top of its `[Required Reading]` block so the orchestrator — and the sub-agents it spawns — share a baseline understanding of the project.

- **If `.grimoire/PROJECT.md` exists:** read it as project context before doing anything else.
- **If it does not exist:** proceed without it, and at the end of the run remind the user that running `grimoire-init` once will give future Grimoire sessions richer context. This is non-blocking — never refuse to work because `PROJECT.md` is missing.
- **Dual-writer contract.** Only `grimoire-init` and `grimoire-note` may write `PROJECT.md`, and their scopes do not overlap:
  - **`grimoire-init`** is the only skill that may **create** `PROJECT.md`. It is the only skill authorized to write **outside** `## Key Conventions / Constraints` and `## Notes` — `## Purpose`, `## Audience`, `## Tech Stack`, `## Repository Layout`, and `## Current Status` are init-only. It is also the only skill that may **add new sections** to the file.
  - **`grimoire-note`** is authorized to write **only** `## Key Conventions / Constraints` and `## Notes`, incrementally. It never creates the file, never adds sections, and never edits other sections.
  - **All other skills** (`grimoire-spec`, `grimoire-plan`, `grimoire-execute`, `grimoire-quick`, `grimoire-know`, `grimoire-update`) never write `PROJECT.md` at all — not as a primary action, not as a side effect.
- **Update mode:** when `grimoire-init` is invoked and `.grimoire/PROJECT.md` already exists, it reads the existing file first and asks only about deltas (what has changed or is missing), then rewrites the file in place.

---

## § Historic

`.grimoire/HISTORIC.md` is both the recency log and the **status-of-record** for the most recent pages in the project. `.grimoire/bag/historic/` stores rotated snapshots of that log for older context.

**Entry format:**
```
N. **<page-name>** [spec|planned|finished] — <1–2 sentence description of what the page delivers>
```
Newest entry is item `1.`; older entries are renumbered downward. Status is one of `[spec]`, `[planned]`, or `[finished]`. The progression is strictly `[spec]` → `[planned]` → `[finished]`; statuses never move backwards and never skip a step.

**Write responsibilities (split across three skills):**

- **`grimoire-spec` — bootstrap, append, rotate.** Runs after the page folder and `SPEC.md` have been written. This is the **only** skill that creates entries, prepends entries, or rotates the file.
  - If `.grimoire/HISTORIC.md` **does not exist** → create it with the new page as entry `1.` with status `[spec]`.
  - If the entry for this page **already exists** (re-spec) → ensure its status is `[spec]` and refresh the description in place; do not duplicate and do not move its position.
  - If the entry does **not** exist and the file has **fewer than 5 entries** → prepend the new entry at the top with status `[spec]`, renumbering the previous ones.
  - If the entry does **not** exist and the file already has **5 entries** → **rotate first** (see below), then write a brand-new `HISTORIC.md` containing only the new entry as item `1.` with status `[spec]`.
- **`grimoire-plan` — status update only.** Runs after the step files have been written.
  - Find the entry whose `<page-name>` matches the planned page and update its status from `[spec]` to `[planned]` **in place** — do not change its position, do not append a new entry.
  - `grimoire-plan` **never appends new entries and never rotates**. If the matching entry is not in `[spec]` status (missing, `[planned]`, or `[finished]`), hard-stop per `§ Pause-point pattern` — never silently update.
- **`grimoire-execute` — status update only.** Runs after a successful execution of the page.
  - Find the entry whose `<page-name>` matches the executed page and update its status from `[planned]` to `[finished]` **in place** — do not change its position, do not append a new entry.
  - If the matching entry is not found (e.g., it was rotated away while the page was being executed), skip silently. This is non-blocking.
  - `grimoire-execute` **never appends new entries and never rotates**.

**Rotation (only triggered by `grimoire-spec`):**
- Create `.grimoire/bag/historic/` if it does not exist.
- Determine the next suffix: list names in `.grimoire/bag/historic/`, extract the `N` from `HISTORIC-N.md` files, and use `max(N) + 1`. If the folder is empty, use `N = 1`.
- Move `.grimoire/HISTORIC.md` to `.grimoire/bag/historic/HISTORIC-N.md`, then write a fresh `HISTORIC.md` with the new page as entry `1.`.

**How other skills consume it (read-only):**
- If `.grimoire/HISTORIC.md` exists, read it for recent-execution context and current status.
- If older context is relevant to the current task, browse `.grimoire/bag/historic/` in descending suffix order.
- Absence of the file is non-blocking for read-only consumers: proceed without recency context. (`grimoire-plan` and `grimoire-execute`, however, must hard-stop if the page they are asked to operate on has no matching entry — see `§ Pause-point pattern`.)

**Commits:**
- `grimoire-spec` ends with `chore: register page <name> in historic` covering the new entry (and any rotation) — see `§ Commits`.
- `grimoire-plan` ends with `chore: mark page <name> planned` covering the in-place `[spec]` → `[planned]` update — see `§ Commits`.
- `grimoire-execute` ends with `chore: mark page <name> finished` covering the in-place `[planned]` → `[finished]` update — see `§ Commits`.

---

## § IDE-aware review

When a skill must show the user a draft, plan, or other markdown content for review before proceeding (e.g., `PROJECT.md` draft, `SPEC.md` draft, `grimoire-quick` plan, `grimoire-update` changelog), prefer rendered markdown in the user's IDE over inline chat text. Inline chat text is hard to scan and has no markdown rendering; an open file in the IDE has both.

**Detection.** Inspect the runtime context for signals that Claude Code is running inside a desktop IDE extension (VSCode native extension, JetBrains plugin, Cursor, etc.). The system context block usually announces this explicitly (e.g., a `# VSCode Extension Context` section, or wording like *"running inside a VSCode native extension environment"*). When in doubt, assume terminal-only and use the inline fallback — the inline path is always safe.

**IDE-aware flow.**

1. Write the reviewable content to disk so the IDE auto-surfaces it with markdown preview:
   - `grimoire-init` → `.grimoire/PROJECT.md` (final destination).
   - `grimoire-spec` → `.grimoire/pages/NNN-[page-name]/SPEC.md` (create the page folder first if needed; do **not** touch `HISTORIC.md` yet — that still belongs to Phase 6).
   - `grimoire-quick` → `.grimoire/bag/drafts/quick-plan-<epoch>.md` (create the folder if needed; this is a throwaway draft, not a final artifact).
   - `grimoire-update` → `.grimoire/bag/drafts/grimoire-update-changelog-<epoch>.md` (same — throwaway).

   For the two ephemeral drafts above (`quick-plan`, `grimoire-update-changelog`), generate `<epoch>` as a Unix timestamp in seconds (`$(date +%s)`) at the moment of writing and memorize the resulting path for use during cleanup. This defeats the VSCode markdown preview cache, which keys on file path — without the suffix, a second invocation can show the stale preview from the prior run. The final-destination drafts (`PROJECT.md`, `SPEC.md`) keep their canonical names.
2. Tell the user in one short line which file was opened and what answer you're waiting for. Example: `📄 Rascunho aberto em .grimoire/PROJECT.md — revise no editor e diga "ok" para seguir, ou liste mudanças.`
3. PAUSE for the user's confirmation or edits — same semantics as the inline flow.
4. On approval:
   - `grimoire-init` / `grimoire-spec`: the file is already at the final path — proceed straight to the commit phase. **No additional Write step needed.**
   - `grimoire-quick` / `grimoire-update`: delete the draft file from `.grimoire/bag/drafts/` and proceed.
5. On requested edits: edit the file in place and re-pause.
6. On abandonment: delete the file (including the init/spec destination, since it was never approved — leaving an uncommitted, unapproved file pollutes the repo).

**Inline fallback (terminal-only).** When the IDE is not detected, paste the rendered draft inline in chat (legacy behavior). Pause semantics, approval gating, and downstream commit timing are identical to the IDE-aware flow.

**Commit timing.** Files written for review remain uncommitted until the user approves. The existing commit phases in each skill run unchanged AFTER approval — there is no separate "draft commit" and no cleanup commit.

---

## § Pause-point pattern

Grimoire skills must pause and wait for the user at well-defined checkpoints. Never proceed past a pause silently.

- **`grimoire-init` — clarifying questions:** after analyzing the codebase, ask the user only about facts that cannot be inferred from the code (purpose, audience, stage, non-obvious constraints). In update mode, ask only about deltas.
- **`grimoire-init` — draft review:** after composing the proposed `PROJECT.md`, present it for review per `§ IDE-aware review` and PAUSE for the user's confirmation or edits before locking the file in as approved.
- **`grimoire-spec` — clarifying questions:** after analyzing the codebase and the user's request, ask targeted questions about pain, ambiguities, scope, and critical architectural decisions. Answer the user's questions in turn. Iterate until a consensus on what the page must deliver is reached.
- **`grimoire-spec` — draft review:** after composing the proposed `SPEC.md`, present it for review per `§ IDE-aware review` and PAUSE for the user's confirmation or edits before locking the file in as approved. Do not write or touch `HISTORIC.md` yet.
- **`grimoire-plan` — precondition check (HARD STOP):** before doing anything else, resolve the page number to `.grimoire/pages/NNN-*/`. If the folder does not exist, if `SPEC.md` is missing, or if the entry's status in `HISTORIC.md` is not `[spec]`, STOP immediately with a clear message naming the current state and the skill the user should run instead. Never offer to run that skill automatically; never silently fix the state.
  - Page NNN does not exist → `❌ Page NNN não existe. Rode /grimoire-spec primeiro.`
  - `SPEC.md` missing → `❌ Page NNN não tem SPEC.md. Rode /grimoire-spec primeiro.`
  - Status is `[planned]` → `❌ Page NNN já foi planejada.`
  - Status is `[finished]` → `❌ Page NNN já foi finalizada.`
  - Entry missing from `HISTORIC.md` (rotated or never registered) → `❌ Page NNN não está registrada no HISTORIC. Rode /grimoire-spec primeiro.`
- **`grimoire-plan` — unclear requirements:** after the precondition passes and the SPEC has been read, if a critical architectural decision is still ambiguous, ask clarifying questions before writing step files.
- **`grimoire-plan` — final clarity check:** immediately before writing step files in Phase 4, the agent must self-review the planned step breakdown and surface any remaining assumption or open decision. If anything is still unclear (e.g. an architectural choice the agent had to guess, a step boundary the SPEC did not anticipate, a tradeoff with no recorded rationale), PAUSE and ask the user using the available question tooling (e.g. `AskUserQuestion`) before any step file is written. Skip silently only when the plan is fully unambiguous. **When the planner has decided to emit `depends-on` frontmatter on one or more step files**, the final clarity check MUST also surface the full DAG to the user — list each step's `depends-on` set, the computed waves, and any advisory `touches`-overlap warnings between parallel siblings — so the user can veto a suspect parallelization before step files are written. If no frontmatter will be emitted, this DAG-presentation requirement does not apply.
- **`grimoire-execute` — precondition check (HARD STOP):** before doing anything else, resolve the page number to `.grimoire/pages/NNN-*/`. If the folder does not exist, if no step file (`1-*.md`) is present, or if the entry's status in `HISTORIC.md` is not `[planned]`, STOP immediately with a clear message. Never offer to run another skill automatically; never silently fix the state.
  - Page NNN does not exist → `❌ Page NNN não existe.`
  - No step files in the folder → `❌ Page NNN ainda não tem plano. Rode /grimoire-plan NNN primeiro.`
  - Status is `[spec]` → `❌ Page NNN ainda está em spec. Rode /grimoire-plan NNN primeiro.`
  - Status is `[finished]` → `❌ Page NNN já foi finalizada.`
  - Entry missing from `HISTORIC.md` (rotated) → proceed; this matches the existing "skip silently if rotated" rule in `§ Historic` because rotation can happen during long executions.
  - Cyclic `depends-on` graph detected (any cycle in the DAG) → `❌ Page NNN tem dependências cíclicas entre steps. Rode /grimoire-plan NNN para corrigir.`
  - `depends-on` reference to a non-existent step number → `❌ Page NNN referencia um step inexistente em depends-on. Rode /grimoire-plan NNN para corrigir.`
- **`grimoire-quick` — scope gatekeeper:** if the request is too large/complex to fit a quick execution, STOP immediately and tell the user to switch to `grimoire-spec` (the entry point to the long-form Spec → Plan → Execute pipeline) instead of generating a plan.
- **`grimoire-quick` — plan authorization:** after presenting the plan for review per `§ IDE-aware review`, PAUSE and wait for the user's explicit authorization or corrections before spawning the sub-agent. Do not start coding yet.
- **`grimoire-quick` — final clarity check:** immediately before composing and presenting the plan draft (per `§ IDE-aware review`), the agent must self-review the intended fix and surface any remaining assumption. If anything is still unclear (e.g. expected behavior, scope of touched files, error handling, naming), PAUSE and ask the user using the available question tooling (e.g. `AskUserQuestion`) before the draft is written or pasted. Skip silently only when the fix is fully unambiguous.

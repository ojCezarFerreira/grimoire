# Step 4 — Update user-facing and maintainer docs

## Scope

Document the new parallel-wave capability and the opt-in YAML frontmatter in the three docs that already describe `grimoire-plan` and `grimoire-execute`. Pure documentation — no behavioral edits, no version bump (that's Step 5). No code; markdown only.

## Files touched

- `README.md`
- `README.pt-BR.md`
- `CLAUDE.md`

## Precondition

Steps 1, 2, and 3 must be committed first — these docs describe the live behavior introduced in those steps. Writing the docs before the SKILLs are updated would create a docs/behavior gap on `main`.

## Edits

### 1. `README.md`

**In the `## Commands` table** (the row matrix near the top): the existing `/grimoire-plan <NNN>` and `/grimoire-execute <NNN>` rows describe their core action. Either leave the table rows as-is (the per-skill sections below carry the new behavior), or tighten the descriptions if doing so keeps them under one line. Prefer minimal edits to the table.

**In the `### /grimoire-plan` section** (currently a two-paragraph block): after the existing prose, append a short paragraph explaining that when a page's planned step breakdown contains genuinely independent steps, `grimoire-plan` MAY emit YAML frontmatter on the step files (`depends-on`, `touches`) declaring inter-step dependencies — see `GRIMOIRE-CONVENTIONS.md § Parallel execution`. The planner surfaces the resulting wave plan to the user at the final-clarity-check pause-point before any step file is written. Pages whose steps are inherently sequential get no frontmatter and remain legacy strict-serial pages.

**In the `### /grimoire-execute` section** (currently a two-paragraph block): after the existing prose, append a short paragraph explaining that when any step file of the page declares `depends-on` frontmatter, `grimoire-execute` switches from strict-serial to a parallel-wave model: it computes Kahn-style topological waves, runs all steps in a wave concurrently — one sub-agent per step in its own throwaway git worktree under `.grimoire/bag/worktrees/page-NNN-step-K/` — cherry-picks successful siblings back to the main branch in step-number order, and tears down every worktree unconditionally before the next wave and at end of run. Pages with no `depends-on` frontmatter anywhere fall back to the legacy strict-serial path (byte-identical to pre-0.8.0). See `GRIMOIRE-CONVENTIONS.md § Parallel execution` for the full contract.

Keep the paragraphs short; do NOT restate frontmatter syntax, worktree paths, or failure semantics in the README — link/reference the conventions section.

### 2. `README.pt-BR.md`

Mirror the README.md additions in pt-BR. Locate the equivalent `### /grimoire-plan` and `### /grimoire-execute` sections, append matching paragraphs translated to Portuguese-BR. The reference to `GRIMOIRE-CONVENTIONS.md § Parallel execution` stays in English (section name).

### 3. `CLAUDE.md`

**In the `## The skills and how they connect` section**, the existing bullets for `grimoire-plan` and `grimoire-execute` describe core behavior in one paragraph each. Extend each bullet with one short clause about the new behavior:

- `grimoire-plan` bullet: append a sentence noting that the planner MAY emit `depends-on`/`touches` frontmatter on step files when the step breakdown contains genuinely independent steps; the resulting wave plan is surfaced to the user at the final-clarity-check pause-point. Refer the reader to `§ Parallel execution`.
- `grimoire-execute` bullet: append a sentence noting that when step files carry `depends-on` frontmatter, execution switches from strict serial to dependency-graph waves with per-step git-worktree isolation and unconditional worktree teardown; absence of frontmatter falls back to the legacy strict-serial path. Refer the reader to `§ Parallel execution`.

**In the `## Shared conventions` bullet list**, add a new entry between `§ Sub-agent spawning` and `§ Commits`:

> - § Parallel execution (frontmatter schema `depends-on`/`touches`; DAG construction + cycle detection; Kahn-style waves; worktree contract with mandatory unconditional teardown; partial-failure handling; back-compat fallback to legacy serial)

This mirrors the conventions file's section order.

## Style + constraints

- **Reference, don't restate.** All three docs should point readers to `GRIMOIRE-CONVENTIONS.md § Parallel execution` for details — no restatement of the worktree paths, frontmatter schema, or failure semantics.
- Tone: terse, factual, matching the existing docs voice. The two READMEs are user-facing; CLAUDE.md is maintainer-facing.
- Preserve unrelated sections byte-identical.
- Do **NOT** bump versions in this step. The release commit lives in Step 5.

## Acceptance

- `README.md` `### /grimoire-plan` and `### /grimoire-execute` each have a new short paragraph describing the new behavior and pointing to `§ Parallel execution`.
- `README.pt-BR.md` mirrors those additions in pt-BR.
- `CLAUDE.md`:
  - `## The skills and how they connect` bullets for `grimoire-plan` and `grimoire-execute` each gain a one-sentence extension covering the new behavior.
  - `## Shared conventions` bullet list has a new `§ Parallel execution` entry between `§ Sub-agent spawning` and `§ Commits`.
- No other docs touched.
- No version numbers changed.

## Commit

After the edits, commit per `§ Commits`:

```
docs(grimoire): document parallel-wave execution in README and CLAUDE.md
```

The commit covers `README.md`, `README.pt-BR.md`, and `CLAUDE.md`. Do not bundle other files.

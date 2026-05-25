# Step 2 — Update `skills/grimoire-plan/SKILL.md`

## Scope

Teach `grimoire-plan` to emit YAML frontmatter (`depends-on` / `touches`) on step files when the page's structure benefits from parallelism, to build and validate the resulting DAG, and to surface the wave plan at the existing final-clarity-check pause-point. **The shared rules live in `§ Parallel execution` (added in Step 1) — this SKILL.md only references them, it must not restate them.** No code; markdown only.

## Files touched

- `skills/grimoire-plan/SKILL.md` (only this file)

## Precondition

Step 1 must be committed first — this step relies on `§ Parallel execution`, `§ Sub-agent spawning` (amended), `§ .grimoire/ layout` (extended), and `§ Pause-point pattern` (extended) being live in `GRIMOIRE-CONVENTIONS.md`.

## Edits

### 1. `[Required Reading]` — add `§ Parallel execution`

In the first bullet of `[Required Reading]` (which currently lists `§ TDD, § Sub-agent spawning, § Commits, § .grimoire/ layout, § Project context, § Historic, § Pause-point pattern`), insert `§ Parallel execution` so the list reads:

> § TDD, § Sub-agent spawning, **§ Parallel execution**, § Commits, § .grimoire/ layout, § Project context, § Historic, § Pause-point pattern

Order: place `§ Parallel execution` right after `§ Sub-agent spawning` to match the conventions file order.

### 2. Phase 3 — extend with DAG emission

The current Phase 3 ("Planning, Scope & Context Management") talks about step count and per-step names. Extend it without rewriting the existing bullets. Add (roughly, integrated into the existing prose):

> **Inter-step dependencies (opt-in).** When the planned step breakdown contains steps that are **genuinely independent of each other** (different files, no logical ordering needed), the planner MAY emit YAML frontmatter on each step file declaring `depends-on: [<step-numbers>]` and `touches: [<paths>]` per `§ Parallel execution`. This unlocks parallel-wave execution in `grimoire-execute`.
>
> **When NOT to emit frontmatter.** If every step is genuinely sequential — each builds on the prior step's edits in a load-bearing way — the planner SHOULD emit no frontmatter. The page then runs via the legacy strict-serial path, which is the default. Emitting frontmatter where it isn't warranted just adds noise.
>
> **DAG construction and validation (before writing step files).** When frontmatter is going to be emitted, the planner MUST:
> 1. Build the DAG: nodes = step files, edges = `depends-on` references.
> 2. **Detect cycles.** Any cycle is a hard error — the planner MUST NOT write step files with a cyclic DAG. If the natural step breakdown produces a cycle, revise the breakdown until the DAG is acyclic, or abandon frontmatter and fall back to legacy serial.
> 3. **Check for dangling references.** Every step number referenced in any `depends-on` must correspond to an actual planned step. Dangling references are a hard error.
> 4. **Compute waves** (Kahn-style topological levels per `§ Parallel execution`): wave 0 = all roots; wave K = all steps whose `depends-on` lies entirely in waves `< K`.
> 5. **Advisory `touches`-overlap check.** When two sibling steps in the same wave declare overlapping `touches` paths, surface the overlap as a warning (not a hard error) — the user decides at the pause-point whether to keep them parallel or force a serial edge.

### 3. Phase 3 — extend the final-clarity-check pause-point

The current Phase 3 closes with the **final clarity check** pause-point per `§ Pause-point pattern`. Extend that pause-point to require **DAG presentation** when frontmatter will be emitted:

> When `depends-on` frontmatter will be emitted on one or more step files, the final clarity check MUST also present the full DAG to the user before any step file is written:
> - Each step number, its planned name, and its `depends-on` set.
> - The computed waves (e.g., `Wave 0: {1}`, `Wave 1: {2, 3}`, `Wave 2: {4, 5}`).
> - Any advisory `touches`-overlap warnings.
>
> The user MUST be able to veto the parallelization (force serial / change deps / change step boundaries) before any step file is written. If the user vetoes, fall back to legacy serial (no frontmatter) or re-plan per the user's correction. If no frontmatter will be emitted, the existing final-clarity-check semantics apply unchanged.

(The shared rule for this also lives in `§ Pause-point pattern` after Step 1's amendments; this SKILL.md just enacts it.)

### 4. Phase 4 — step-file writing includes frontmatter when applicable

The current Phase 4 says "write each step file as `1-[step-name].md`, `2-[step-name].md`, …". Append a sentence:

> When the DAG approved at the final clarity check requires frontmatter, prepend each step file with a YAML frontmatter block declaring `depends-on` and (recommended) `touches`, per the schema in `§ Parallel execution`. Steps that are DAG roots use `depends-on: []` (or omit it — both are valid). The markdown body of the step file remains the canonical instructions for the sub-agent.

Frontmatter example to embed in the prose (illustrative, not normative — the schema lives in `§ Parallel execution`):

> ```
> ---
> depends-on: [1]
> touches:
>   - skills/grimoire-execute/SKILL.md
> ---
> # Step 3 — Update grimoire-execute …
> ```

## Style + constraints

- **Do not restate the rules of `§ Parallel execution`** inside the SKILL.md body — reference the section. The single-source-of-truth invariant applies here especially.
- Preserve all existing Phase 1, Phase 2, Phase 5, and Phase 6 text byte-identical unless an explicit edit above touches them.
- Preserve the SKILL.md frontmatter (`name`, `description`, `argument-hint`) and the `---` delimiters exactly.
- Preserve `$ARGUMENTS` substitutions verbatim.
- The SKILL.md is in English; messages emitted at runtime MAY remain pt-BR for consistency with the existing hard-stop precedent.

## Acceptance

- `[Required Reading]` lists `§ Parallel execution` between `§ Sub-agent spawning` and `§ Commits`.
- Phase 3 instructs the planner to emit frontmatter when steps are genuinely independent, to build and validate the DAG (cycle + dangling-reference detection, advisory `touches`-overlap), and to compute waves.
- Phase 3's final clarity check requires DAG presentation when frontmatter will be emitted, and supports user veto.
- Phase 4 instructs writing the YAML frontmatter block on each step file when the approved DAG calls for it.
- Phases 1, 2, 5, and 6 are unchanged.
- The SKILL.md never restates the contents of `§ Parallel execution`; it only references the section.

## Commit

After the edit, commit per `§ Commits`:

```
docs(grimoire): teach grimoire-plan to emit dependency frontmatter
```

The commit covers only `skills/grimoire-plan/SKILL.md`.

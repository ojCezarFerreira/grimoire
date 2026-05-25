# Step 5 — Version bump (`0.7.0` → `0.8.0`) + `CHANGELOG.md`

## Scope

Cut the `0.8.0` release: bump `plugin.json` and `marketplace.json` in lockstep, add the matching `CHANGELOG.md` section. **This is the release trigger** — once this commit lands on `main`, the `release.yml` workflow tags and publishes `v0.8.0`, extracting the new CHANGELOG section as the release notes. No code; JSON + markdown only.

## Files touched

- `.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json`
- `CHANGELOG.md`

## Precondition

Steps 1–4 must be committed first. The release workflow validates that the version-bump commit's CHANGELOG section reflects the actual changes on `main`. Bumping before the changes land would publish an empty release.

## Edits

### 1. `.claude-plugin/plugin.json`

Change the `version` field from `"0.7.0"` to `"0.8.0"`. All other fields untouched (`name`, `displayName`, `description`, `author`, `homepage`, `repository`, `license`, `keywords`, `$schema`). The `description` need not change — the feature is documented in CHANGELOG and the READMEs, and the manifest description is intentionally generic.

### 2. `.claude-plugin/marketplace.json`

Change the `plugins[0].version` field from `"0.7.0"` to `"0.8.0"`. All other fields untouched. **Versions in `plugin.json` and `marketplace.json` MUST match** — the release workflow fails the build otherwise.

### 3. `CHANGELOG.md`

Add a new section at the top of the file, **above the existing `## [0.7.0] — 2026-05-23` section**, in the same "Keep a Changelog" format used by prior entries. The date should be today's date (planner: pass through whatever the executing agent's clock reports; do not hard-code).

Section structure (use these subsections; omit any that have no entries):

> ## [0.8.0] — <today's date in YYYY-MM-DD>
>
> ### Added
> - `GRIMOIRE-CONVENTIONS.md` — new `§ Parallel execution` section: opt-in YAML frontmatter (`depends-on`, `touches`) on step files unlocks a dependency-graph execution model for `grimoire-execute`. The orchestrator builds a DAG, runs Kahn-style topological waves, spawns one sub-agent per step inside a deterministic git worktree under `.grimoire/bag/worktrees/page-NNN-step-K/`, cherry-picks successful siblings back to the main workspace in step-number order, and tears down every worktree unconditionally (`git worktree remove --force` + defensive filesystem cleanup) before the next wave and at end of run — `.grimoire/bag/worktrees/` and `git worktree list` are guaranteed clean after every run, success or failure.
>
> ### Changed
> - `GRIMOIRE-CONVENTIONS.md § Sub-agent spawning` amended to admit parallel sub-agents inside a single wave when the page's step files declare `depends-on` frontmatter; sequencing across waves remains strict; legacy strict-serial is the default when no step file carries frontmatter.
> - `GRIMOIRE-CONVENTIONS.md § .grimoire/ layout` notes that step files MAY carry YAML frontmatter per `§ Parallel execution`.
> - `GRIMOIRE-CONVENTIONS.md § Pause-point pattern`:
>   - `grimoire-plan: final clarity check` extended to require DAG presentation (wave list + per-step deps + advisory `touches`-overlap warnings) when frontmatter will be emitted, so the user can veto a suspect parallelization before step files are written.
>   - `grimoire-execute: precondition check (HARD STOP)` gains two new hard-stop cases: cyclic `depends-on` graph and `depends-on` reference to a non-existent step.
> - `skills/grimoire-plan/SKILL.md` Phase 3 extended: when steps are genuinely independent, the planner MAY emit the new frontmatter; builds and validates the DAG (cycle + dangling-reference detection, advisory `touches`-overlap), computes waves, surfaces the wave plan at the final-clarity-check pause-point, and falls back to legacy serial if the user vetoes parallelization. Phase 4 writes the YAML frontmatter block on each step file when the approved DAG calls for it.
> - `skills/grimoire-execute/SKILL.md` Phase 1 detects frontmatter and selects between the legacy strict-serial path and the parallel-wave path. Phase 2 is split into 2A (legacy) and 2B (parallel-wave: worktree creation, concurrent sub-agent spawn, wave-settle wait, cherry-pick in step-number order, unconditional teardown as a try/finally invariant). Phase 4 splits finalization: fully-successful runs mark `[finished]`; partial-failure runs keep HISTORIC at `[planned]` and report finished / failed / skipped steps to the user.
> - `README.md`, `README.pt-BR.md`, `CLAUDE.md` describe the new wave model in their `grimoire-plan` and `grimoire-execute` entries, pointing to `§ Parallel execution` for details.
>
> ### Compatibility
> - **Pure opt-in.** Pages whose step files declare no `depends-on` frontmatter execute byte-identically to pre-0.8.0 — no worktrees created, no behavior change. Existing pages (e.g., `001-grimoire-note-skill`) are unaffected; no auto-migration.

Adjust subsection content to whatever the executing agent actually shipped (the executing agent should re-read the commits from Steps 1–4 before composing this section and prune any line whose claim isn't on `main`).

## Style + constraints

- Match the prose voice of prior CHANGELOG entries (`0.7.0`, `0.6.0`, `0.5.0`). Terse, factual, one entry per logical change.
- "Keep a Changelog" subsection order: `### Added`, `### Changed`, `### Deprecated`, `### Removed`, `### Fixed`, `### Security`. Use only the ones that apply.
- The CHANGELOG section MUST be syntactically a new top-level entry above the existing `## [0.7.0]` section — the release workflow extracts the topmost matching version block.
- Both JSON files MUST end at the **same** version string. CI fails otherwise.
- This is a **minor** bump (`0.7.0 → 0.8.0`) per SPEC § Constraints: new capability, additive frontmatter format, no breaking change to existing pages — matches the precedent of `grimoire-know` (`0.5.0 → 0.6.0`) and `grimoire-note` (`0.6.0 → 0.7.0`).

## Acceptance

- `.claude-plugin/plugin.json` `version` is `"0.8.0"`.
- `.claude-plugin/marketplace.json` `plugins[0].version` is `"0.8.0"`.
- The two JSON files agree on the version string.
- `CHANGELOG.md` has a new `## [0.8.0] — <today>` section at the top, above `## [0.7.0]`, enumerating the parallel-wave feature, the frontmatter schema, the worktree contract (with the explicit unconditional-cleanup guarantee), and the back-compat statement.
- No SKILL.md, conventions, or README content edited in this step — Steps 1–4 already cover those.

## Commit

After the edits, commit per `§ Commits`:

```
chore(release): 0.8.0 — parallel-wave execution
```

The commit covers `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, and `CHANGELOG.md`. This single commit is the release trigger — keep it tight and self-contained.

## Post-step note for the executing agent

After this commit lands, `grimoire-execute` Phase 4 will run and mark page `002-parallel-execute` `[finished]` in `HISTORIC.md` with a separate `chore: mark page 002-parallel-execute finished` commit per `§ Commits`. That HISTORIC commit is the responsibility of `grimoire-execute`, not of this step file.

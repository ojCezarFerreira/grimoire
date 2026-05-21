# Grimoire Conventions

Shared, load-bearing rules for the Grimoire workflow skills (`grimoire-init`, `grimoire-plan`, `grimoire-execute`, `grimoire-quick`). Each SKILL.md references this file as required reading. Do not duplicate these rules inside skill bodies — update them here.

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

- **`grimoire-execute` on a single plan file:** execute directly inside one sub-agent.
- **`grimoire-execute` on a directory of sub-plans:** spawn one sub-agent **per sub-plan**, in strict numeric order. Never start the next sub-plan until the current sub-agent has fully completed its assignment.
- **`grimoire-quick`:** spawn a single sub-agent for the entire fix once the user has authorized the inline plan.

Always instruct the sub-agent with the specific plan file (or inline plan) it must execute, and require it to follow `§ TDD` and `§ Commits`.

---

## § Commits

- Make granular, atomic commits as work progresses.
- Commit immediately after each logical step completes (typically end of Green or Refactor).
- Use **Conventional Commits in English**: `feat:`, `fix:`, `test:`, `refactor:`, `chore:`, `style:`, `docs:`.
- One logical change per commit — do not bundle unrelated edits.
- `grimoire-execute` ends its run with a final `chore: move [plan-name] to finished` commit when the plan file or sub-plan directory is moved into `.grimoire/finished/`.

---

## § .grimoire/ layout

All Grimoire plan files live under `.grimoire/` in the consuming project, with `.grimoire/finished/` as the terminal location once execution succeeds.

**Naming:**
- Single plans: `.grimoire/NNN-[feature-name].md` (e.g., `.grimoire/001-user-auth.md`).
- Split plans (large scope): `.grimoire/NNN-[feature-name]/NNN.X-[subplan-name].md` (e.g., `.grimoire/001-user-auth/001.1-schema.md`, `001.2-endpoints.md`).
- `NNN` is incremental and chronological across the whole project.

**Split decision (used by `grimoire-plan`):**
Generate a single plan when the feature fits comfortably in one execution context without risking lucidity loss. Split into a numbered subfolder of sub-plans when token usage or context degradation becomes a risk during execution.

**On completion (`grimoire-execute`):**
- Single plan file → move the `.md` to `.grimoire/finished/`.
- Sub-plan directory → move the **entire** parent folder into `.grimoire/finished/`, preserving its internal structure.

**Project context file:**
- `.grimoire/PROJECT.md` — persistent project context generated and updated by `grimoire-init`. Loaded by every skill's `[Required Reading]` block. Not subject to `NNN-` numbering and never moved to `.grimoire/finished/`.

**Recent execution log:**
- `.grimoire/HISTORIC.md` — recency log maintained by `grimoire-execute`. Holds the last 5 executions (newest at top), each with a short description. Not subject to `NNN-` and never moved to `.grimoire/finished/`. See `§ Historic`.
- `.grimoire/bag/historic/` — archive. When `HISTORIC.md` is full (5 entries) and a new execution must be recorded, the file is moved here with a `HISTORIC-N.md` suffix before a fresh `HISTORIC.md` is started.

---

## § Project context

Every Grimoire skill loads `.grimoire/PROJECT.md` (if present) at the top of its `[Required Reading]` block so the orchestrator — and the sub-agents it spawns — share a baseline understanding of the project.

- **If `.grimoire/PROJECT.md` exists:** read it as project context before doing anything else.
- **If it does not exist:** proceed without it, and at the end of the run remind the user that running `grimoire-init` once will give future Grimoire sessions richer context. This is non-blocking — never refuse to work because `PROJECT.md` is missing.
- **`grimoire-init` is the only skill that writes `PROJECT.md`.** Other skills must not modify it as a side effect.
- **Update mode:** when `grimoire-init` is invoked and `.grimoire/PROJECT.md` already exists, it reads the existing file first and asks only about deltas (what has changed or is missing), then rewrites the file in place.

---

## § Historic

`.grimoire/HISTORIC.md` is the recency log every Grimoire skill consults to understand what has been executed recently in the project. `.grimoire/bag/historic/` stores rotated snapshots of that log for older context.

- **Who creates it:** bootstrap of `HISTORIC.md` is owned by a future skill. None of the current skills create the file from scratch.
- **Who writes it:** `grimoire-execute` is the only skill that updates `HISTORIC.md`. Every other skill treats it as read-only.
- **Append (inside `grimoire-execute`, after successful finalization of an execution):**
  - If `.grimoire/HISTORIC.md` **does not exist** → skip silently. This step is non-blocking; never refuse an execution because of it. Once the bootstrap skill is added, this path stops being used.
  - If it exists and has **fewer than 5 entries** → prepend the new entry at the top in the form `1. **<plan-name>** — <1–2 sentence description of what was delivered>`, renumbering the previous ones.
  - If it exists and already has **5 entries** → rotate first (see below), then write a brand-new `HISTORIC.md` containing the new entry as item `1.`.
- **Rotation:**
  - Create `.grimoire/bag/historic/` if it does not exist.
  - Determine the next suffix: list names in `.grimoire/bag/historic/`, extract the `N` from `HISTORIC-N.md` files, and use `max(N) + 1`. If the folder is empty, use `N = 1`.
  - Move `.grimoire/HISTORIC.md` to `.grimoire/bag/historic/HISTORIC-N.md`.
- **How other skills consume it (read-only):**
  - If `.grimoire/HISTORIC.md` exists, read it for recent-execution context.
  - If older context is relevant to the current task, browse `.grimoire/bag/historic/` in descending suffix order.
  - Absence of the file is non-blocking: proceed without recency context.
- **Commit:** the append (or append + rotation) inside `grimoire-execute` is committed alongside the `chore: move [plan-name] to finished` commit, or as a separate `chore: update historic log` commit when atomic cohesion is clearer that way — see `§ Commits`.

---

## § Pause-point pattern

Grimoire skills must pause and wait for the user at well-defined checkpoints. Never proceed past a pause silently.

- **`grimoire-init` — clarifying questions:** after analyzing the codebase, ask the user only about facts that cannot be inferred from the code (purpose, audience, stage, non-obvious constraints). In update mode, ask only about deltas.
- **`grimoire-init` — draft review:** after composing the proposed `PROJECT.md`, output it inline and PAUSE for the user's confirmation or edits before writing the file.
- **`grimoire-plan` — unclear requirements:** if any requirement is ambiguous or a critical architectural decision is needed, ask clarifying questions before writing the plan.
- **`grimoire-quick` — scope gatekeeper:** if the request is too large/complex to fit a quick execution, STOP immediately and tell the user to switch to `grimoire-plan` instead of generating a plan.
- **`grimoire-quick` — plan authorization:** after outputting the inline plan in chat, PAUSE and wait for the user's explicit authorization or corrections before spawning the sub-agent. Do not start coding yet.

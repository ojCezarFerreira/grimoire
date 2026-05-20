# Sage Conventions

Shared, load-bearing rules for the Sage workflow skills (`sage-plan`, `sage-execute`, `sage-quick`). Each SKILL.md references this file as required reading. Do not duplicate these rules inside skill bodies — update them here.

---

## § TDD

Every code change follows strict Red → Green → Refactor.

**CRITICAL:** NEVER execute the entire test suite. Use test filters (e.g., `--filter` in PHPUnit/Pest, `-k` in pytest, exact file paths in Jest/Vitest, etc.) to run ONLY the tests created or modified in the current step.

1. **Red:** Write or modify the tests for the current step. Run ONLY these tests and verify failure.
2. **Green:** Implement the minimum code needed. Run the same filtered tests again and verify they pass.
3. **Refactor:** Clean up the code while keeping the filtered tests green.

The same loop applies regardless of whether the work comes from `sage-plan`, `sage-execute`, or `sage-quick`.

---

## § Sub-agent spawning

The main context is an orchestrator. Implementation work runs inside sub-agents so the orchestrator's context stays lucid.

- **`sage-execute` on a single plan file:** execute directly inside one sub-agent.
- **`sage-execute` on a directory of sub-plans:** spawn one sub-agent **per sub-plan**, in strict numeric order. Never start the next sub-plan until the current sub-agent has fully completed its assignment.
- **`sage-quick`:** spawn a single sub-agent for the entire fix once the user has authorized the inline plan.

Always instruct the sub-agent with the specific plan file (or inline plan) it must execute, and require it to follow `§ TDD` and `§ Commits`.

---

## § Commits

- Make granular, atomic commits as work progresses.
- Commit immediately after each logical step completes (typically end of Green or Refactor).
- Use **Conventional Commits in English**: `feat:`, `fix:`, `test:`, `refactor:`, `chore:`, `style:`, `docs:`.
- One logical change per commit — do not bundle unrelated edits.
- `sage-execute` ends its run with a final `chore: move [plan-name] to finished` commit when the plan file or sub-plan directory is moved into `.planning/finished/`.

---

## § .planning/ layout

All Sage plan files live under `.planning/` in the consuming project, with `.planning/finished/` as the terminal location once execution succeeds.

**Naming:**
- Single plans: `.planning/NNN-[feature-name].md` (e.g., `.planning/001-user-auth.md`).
- Split plans (large scope): `.planning/NNN-[feature-name]/NNN.X-[subplan-name].md` (e.g., `.planning/001-user-auth/001.1-schema.md`, `001.2-endpoints.md`).
- `NNN` is incremental and chronological across the whole project.

**Split decision (used by `sage-plan`):**
Generate a single plan when the feature fits comfortably in one execution context without risking lucidity loss. Split into a numbered subfolder of sub-plans when token usage or context degradation becomes a risk during execution.

**On completion (`sage-execute`):**
- Single plan file → move the `.md` to `.planning/finished/`.
- Sub-plan directory → move the **entire** parent folder into `.planning/finished/`, preserving its internal structure.

---

## § Pause-point pattern

Sage skills must pause and wait for the user at well-defined checkpoints. Never proceed past a pause silently.

- **`sage-plan` — unclear requirements:** if any requirement is ambiguous or a critical architectural decision is needed, ask clarifying questions before writing the plan.
- **`sage-quick` — scope gatekeeper:** if the request is too large/complex to fit a quick execution, STOP immediately and tell the user to switch to `sage-plan` instead of generating a plan.
- **`sage-quick` — plan authorization:** after outputting the inline plan in chat, PAUSE and wait for the user's explicit authorization or corrections before spawning the sub-agent. Do not start coding yet.

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

- **`grimoire-execute` on a page with one step file:** execute directly inside one sub-agent.
- **`grimoire-execute` on a page with multiple step files:** spawn one sub-agent **per step file**, in strict numeric order. Never start the next step until the current sub-agent has fully completed its assignment.
- **`grimoire-quick`:** spawn a single sub-agent for the entire fix once the user has authorized the inline plan.

Always instruct the sub-agent with the specific step file (or inline plan) it must execute, and require it to follow `§ TDD` and `§ Commits`.

---

## § Commits

- Make granular, atomic commits as work progresses.
- Commit immediately after each logical step completes (typically end of Green or Refactor).
- Use **Conventional Commits in English**: `feat:`, `fix:`, `test:`, `refactor:`, `chore:`, `style:`, `docs:`.
- One logical change per commit — do not bundle unrelated edits.
- `grimoire-plan` ends its run with a `chore: register page <name> in historic` commit covering the new `HISTORIC.md` entry (and any rotation to `.grimoire/bag/historic/`). This commit is separate from the planning-file commit(s).
- `grimoire-execute` ends its run with a `chore: mark page <name> finished` commit covering the in-place `HISTORIC.md` status update from `[planned]` to `[finished]`. No files are moved on completion.

---

## § .grimoire/ layout

Every feature, change, or alteration tracked by Grimoire is a **page**. All pages live under `.grimoire/pages/` in the consuming project, regardless of whether they are planned, in progress, or finished. **There is no `.grimoire/finished/` directory** — status lives in `HISTORIC.md`, not in the filesystem.

**Page folder (mandatory for every page):**
- Path: `.grimoire/pages/NNN-[page-name]/` (e.g., `.grimoire/pages/001-user-auth/`).
- `NNN` is incremental and chronological across the whole project (`001`, `002`, `003`, …).

**Step files inside a page folder:**
- Always sequential, page-scoped numbering: `1-[step-name].md`, `2-[step-name].md`, `3-[step-name].md`, … (e.g., `1-schema.md`, `2-endpoints.md`, `3-tests.md`).
- A simple page has a single step file (`1-[step-name].md`); a larger page has multiple.
- `grimoire-plan` chooses how many step files a page needs based on projected context lucidity per step — same heuristic as the old single-vs-split decision, but the output shape is always a folder of sequential step files.

**On completion (`grimoire-execute`):**
- The page folder and its step files stay in place. **Nothing is moved.**
- The page's entry in `HISTORIC.md` is updated from `[planned]` to `[finished]`. See `§ Historic`.

**Project context file:**
- `.grimoire/PROJECT.md` — persistent project context generated and updated by `grimoire-init`. Loaded by every skill's `[Required Reading]` block. Not subject to `NNN-` numbering.

**Recent execution log:**
- `.grimoire/HISTORIC.md` — recency log and status-of-record for the last 5 pages. Bootstrapped, appended, and rotated by `grimoire-plan`; status-updated to `[finished]` by `grimoire-execute`. See `§ Historic`.
- `.grimoire/bag/historic/` — archive. When `HISTORIC.md` already has 5 entries and a new page must be added, the file is rotated here with a `HISTORIC-N.md` suffix before a fresh `HISTORIC.md` is started.

---

## § Project context

Every Grimoire skill loads `.grimoire/PROJECT.md` (if present) at the top of its `[Required Reading]` block so the orchestrator — and the sub-agents it spawns — share a baseline understanding of the project.

- **If `.grimoire/PROJECT.md` exists:** read it as project context before doing anything else.
- **If it does not exist:** proceed without it, and at the end of the run remind the user that running `grimoire-init` once will give future Grimoire sessions richer context. This is non-blocking — never refuse to work because `PROJECT.md` is missing.
- **`grimoire-init` is the only skill that writes `PROJECT.md`.** Other skills must not modify it as a side effect.
- **Update mode:** when `grimoire-init` is invoked and `.grimoire/PROJECT.md` already exists, it reads the existing file first and asks only about deltas (what has changed or is missing), then rewrites the file in place.

---

## § Historic

`.grimoire/HISTORIC.md` is both the recency log and the **status-of-record** for the most recent pages in the project. `.grimoire/bag/historic/` stores rotated snapshots of that log for older context.

**Entry format:**
```
N. **<page-name>** [planned|finished] — <1–2 sentence description of what the page delivers>
```
Newest entry is item `1.`; older entries are renumbered downward. Status is one of `[planned]` or `[finished]`.

**Write responsibilities (split across two skills):**

- **`grimoire-plan` — bootstrap, append, rotate.** Runs after the page folder and step files have been written.
  - If `.grimoire/HISTORIC.md` **does not exist** → create it with the new page as entry `1.` with status `[planned]`. (This replaces the previous "future skill owns bootstrap" rule.)
  - If the entry for this page **already exists** (rerun/replan) → ensure its status is `[planned]` and refresh the description in place; do not duplicate and do not move its position.
  - If the entry does **not** exist and the file has **fewer than 5 entries** → prepend the new entry at the top with status `[planned]`, renumbering the previous ones.
  - If the entry does **not** exist and the file already has **5 entries** → **rotate first** (see below), then write a brand-new `HISTORIC.md` containing only the new entry as item `1.` with status `[planned]`.
- **`grimoire-execute` — status update only.** Runs after a successful execution of the page.
  - Find the entry whose `<page-name>` matches the executed page and update its status from `[planned]` to `[finished]` **in place** — do not change its position, do not append a new entry.
  - If the matching entry is not found (e.g., it was rotated away while the page was being executed), skip silently. This is non-blocking.
  - `grimoire-execute` **never appends new entries and never rotates**.

**Rotation (only triggered by `grimoire-plan`):**
- Create `.grimoire/bag/historic/` if it does not exist.
- Determine the next suffix: list names in `.grimoire/bag/historic/`, extract the `N` from `HISTORIC-N.md` files, and use `max(N) + 1`. If the folder is empty, use `N = 1`.
- Move `.grimoire/HISTORIC.md` to `.grimoire/bag/historic/HISTORIC-N.md`, then write a fresh `HISTORIC.md` with the new page as entry `1.`.

**How other skills consume it (read-only):**
- If `.grimoire/HISTORIC.md` exists, read it for recent-execution context and current status.
- If older context is relevant to the current task, browse `.grimoire/bag/historic/` in descending suffix order.
- Absence of the file is non-blocking: proceed without recency context.

**Commits:**
- `grimoire-plan` ends with `chore: register page <name> in historic` covering the new entry (and any rotation) — see `§ Commits`.
- `grimoire-execute` ends with `chore: mark page <name> finished` covering the in-place status update — see `§ Commits`.

---

## § Pause-point pattern

Grimoire skills must pause and wait for the user at well-defined checkpoints. Never proceed past a pause silently.

- **`grimoire-init` — clarifying questions:** after analyzing the codebase, ask the user only about facts that cannot be inferred from the code (purpose, audience, stage, non-obvious constraints). In update mode, ask only about deltas.
- **`grimoire-init` — draft review:** after composing the proposed `PROJECT.md`, output it inline and PAUSE for the user's confirmation or edits before writing the file.
- **`grimoire-plan` — unclear requirements:** if any requirement is ambiguous or a critical architectural decision is needed, ask clarifying questions before writing the plan.
- **`grimoire-quick` — scope gatekeeper:** if the request is too large/complex to fit a quick execution, STOP immediately and tell the user to switch to `grimoire-plan` instead of generating a plan.
- **`grimoire-quick` — plan authorization:** after outputting the inline plan in chat, PAUSE and wait for the user's explicit authorization or corrections before spawning the sub-agent. Do not start coding yet.

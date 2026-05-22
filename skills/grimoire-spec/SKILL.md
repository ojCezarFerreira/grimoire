---
name: grimoire-spec
description: Specify a page with Grimoire — gathers context, writes SPEC.md, and registers the page in HISTORIC.md.
argument-hint: Page request / what you want to build
---

**[Required Reading]**
1. Read `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md`. Its rules (§ Commits, § .grimoire/ layout, § Project context, § Historic, § IDE-aware review, § Pause-point pattern) are load-bearing for this skill. **Note:** this skill is the writer for `HISTORIC.md` (bootstrap + append + rotate); `grimoire-plan` and `grimoire-execute` only update status in place. `§ TDD` does not apply here — this skill produces a specification, not code.
2. If `.grimoire/PROJECT.md` exists in the project root, read it for project context. If it does not exist, proceed without it and suggest the user run `grimoire-init` once this task is complete.
3. If `.grimoire/HISTORIC.md` exists, read it for recent-page context. If older context seems relevant, browse `.grimoire/bag/historic/` (newest suffix first). Missing → proceed without it; this skill will bootstrap it in Phase 6.

**[Objective]**
Turn the user's raw request into a clear, agreed-upon `SPEC.md` for a new page, and register that page in `HISTORIC.md` with status `[spec]`. This is the **first step** of the long-form Spec → Plan → Execute pipeline. `grimoire-plan` will refuse to run on a page that does not have a `SPEC.md` and a matching `[spec]` entry, so this skill is the only legitimate way to open a new page.

User's request:

$ARGUMENTS

**[Phase 1: Codebase Analysis]**
- Analyze the relevant slice of the codebase for the user's request. Read only what is needed to form an informed opinion (existing modules, neighboring patterns, schemas, public APIs, tests in the same area).
- Identify what already exists and could be reused. Surface patterns the new page should follow.
- Collect candidate facts and constraints. Anything you cannot infer goes into Phase 2 as a question. **Do not invent facts.**

**[Phase 2: Clarifying Questions]**
Per `§ Pause-point pattern` (grimoire-spec: clarifying questions), ask the user targeted questions. Iterate until consensus on what the page must deliver. Typical topics:
- The underlying pain or goal — what problem the page solves and for whom.
- Concrete acceptance criteria — how the user will know the page is done.
- Scope boundaries — what is in, what is deliberately out.
- Critical architectural decisions the SPEC must lock in (data model, public interface, integration points, failure semantics).
- Non-obvious constraints (perf budgets, compatibility, security, deadlines).
- Open questions that should remain visible in the SPEC for `grimoire-plan` to address later.

Answer the user's questions in turn. Do not write any file until consensus is reached.

**[Phase 3: NNN Resolution & Page Name]**
- List `.grimoire/pages/` (if it exists). Extract the integer `N` from each `NNN-*` subfolder name. Choose `max(N) + 1`, zero-padded to 3 digits (`NNN`). If the folder is empty or does not exist, use `001`.
- Choose `[page-name]` in short, descriptive kebab-case (2–5 words). Avoid generic names (`fix`, `update`) — the name must read meaningfully on its own in `HISTORIC.md`.
- The final folder will be `.grimoire/pages/NNN-[page-name]/`.

**[Phase 4: Draft Review]**
- Compose the proposed `SPEC.md` using this template:

  ```
  # NNN — <Page Title>

  ## Context
  Why this page exists, what triggered it, what outcome is expected.

  ## Goals
  - Bullet list of what the page must achieve.

  ## Non-goals
  - Bullet list of what the page deliberately does **not** address.

  ## Scope
  Files / modules / surfaces touched. Be specific with paths when known.

  ## Acceptance criteria
  - Observable, testable criteria. The page is done when all of these hold.

  ## Constraints
  - Perf budgets, compatibility, security, deadlines, conventions to honor.

  ## Open questions
  - Anything still ambiguous that `grimoire-plan` should resolve. Empty if none.

  ## References
  - Links / paths to existing code, ADRs, prior pages, external docs.
  ```

- Present the proposed content for review per `§ IDE-aware review`:
  - **IDE detected:** create `.grimoire/pages/NNN-[page-name]/` if it does not exist, then write the draft to `.grimoire/pages/NNN-[page-name]/SPEC.md` so the IDE renders it. Tell the user in one short line that the draft is open in the editor. Do **not** touch `HISTORIC.md` yet — that still belongs to Phase 6.
  - **Terminal-only fallback:** output the proposed content inline in the chat.
- Pause per `§ Pause-point pattern` (grimoire-spec: draft review): wait for the user's confirmation or edits. Do not advance to Phase 5 until approved. If the user abandons the draft, delete the file and the folder (IDE mode only).

**[Phase 5: Finalize SPEC.md]**
- **IDE mode:** the approved file is already at `.grimoire/pages/NNN-[page-name]/SPEC.md`. No additional write is needed.
- **Inline fallback:** create `.grimoire/pages/NNN-[page-name]/` if it does not exist, then write the approved content to `.grimoire/pages/NNN-[page-name]/SPEC.md`.
- Commit per `§ Commits`: `docs(grimoire): spec page <name>`. This commit covers only the new `SPEC.md` (and the folder it lives in).

**[Phase 6: HISTORIC.md Maintenance]**
Run this phase only after `SPEC.md` is committed. Follow `§ Historic` strictly. **You are the only skill that touches `HISTORIC.md` beyond an in-place status flip.**

- **If `.grimoire/HISTORIC.md` does not exist:** create it with the new page as entry `1.`:
  ```
  # HISTORIC

  1. **NNN-[page-name]** [spec] — <1–2 sentence description of what the page delivers>
  ```
- **If the entry for this page already exists** (re-spec): ensure its status is `[spec]` and refresh the description in place. Do not duplicate, do not move it.
- **If the entry does not exist and `HISTORIC.md` has fewer than 5 entries:** prepend the new entry as item `1.` with status `[spec]`, renumbering the previous ones.
- **If the entry does not exist and `HISTORIC.md` already has 5 entries:** rotate first per `§ Historic` — create `.grimoire/bag/historic/` if needed; determine the next suffix `N` as `max(existing HISTORIC-N.md suffix) + 1` (or `1` if empty); move `HISTORIC.md` to `.grimoire/bag/historic/HISTORIC-N.md`; then write a fresh `HISTORIC.md` containing only the new entry as item `1.` with status `[spec]`.

**[Phase 7: Historic Commit]**
- Commit the HISTORIC change per `§ Commits` as a separate commit: `chore: register page <name> in historic`.
- If a rotation also happened, the same commit covers both the rotated `HISTORIC-N.md` file and the fresh `HISTORIC.md`.
- End the run by telling the user the next step: `/grimoire-plan NNN` (using the numeric page identifier just assigned).

---
name: grimoire-know
description: Answer questions about the repository or its application — read-only, with web research and source citations when needed.
argument-hint: Your question about the repo or the app
---

**[Required Reading]**
1. Read `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md`. Only `§ Project context` and `§ Historic` (read-only consumption) apply to this skill. `§ TDD`, `§ Sub-agent spawning` rules for execute/quick, `§ Commits`, `§ IDE-aware review`, and `§ Pause-point pattern` do **not** apply — this skill is pure Q&A and never mutates the repo, makes no commits, and creates no pages.
2. If `.grimoire/PROJECT.md` exists in the project root, read it for project context. If it does not exist, proceed without it and, at the end of the run, suggest the user run `grimoire-init` once to enrich future answers (non-blocking — never refuse to answer because `PROJECT.md` is missing).
3. If `.grimoire/HISTORIC.md` exists, read it for recent-page context. If older context seems relevant to the question, browse `.grimoire/bag/historic/` (newest suffix first). Missing → proceed without it.

**[Objective]**
Answer the following question about the repository or the application it builds, faithfully and read-only:

$ARGUMENTS

**[Hard constraints — read-only]**
- Never use `Write`, `Edit`, `NotebookEdit`, or any file-mutating tool against the consumer project.
- Never run shell commands that mutate state (`git commit`, `git push`, `git checkout -b`, file creation/deletion, package installs, etc.). Read-only `Bash` (`ls`, `cat`, `git log`, `git diff`, `git status`, `find`, `grep`) is allowed.
- Never create a page folder under `.grimoire/pages/` and never write to `.grimoire/HISTORIC.md` or `.grimoire/PROJECT.md`.
- Never make commits.
- When uncertain, say so plainly instead of guessing.

**[Phase 1: Question analysis]**
- Classify whether the question is answerable from the repo alone, requires web research, or both.
- Identify which files or directories are most relevant to the question without reading more than needed.
- Decide upfront whether the sub-agent will need `WebSearch`/`WebFetch`, so it can be briefed accordingly.

**[Phase 2: Investigation]**
- Spawn one general-purpose sub-agent with `Read`, `Grep`, `Glob`, read-only `Bash`, `WebSearch`, and `WebFetch`.
- Brief the sub-agent with a self-contained prompt that includes the user's question, the relevant files/dirs identified in Phase 1, and explicit instructions to:
  - Inspect only the files needed to answer the question.
  - Use `WebSearch`/`WebFetch` when the answer depends on facts outside the repo (third-party libs, standards, recent docs, API behavior).
  - Collect every URL it actually consulted.
  - Flag any claim it is not confident about, instead of guessing.
  - Return a structured response with: (a) the answer, (b) an "uncertain about" list (empty if none), and (c) the source links it used (empty if none).

**[Phase 3: Response]**
Render the sub-agent's findings to the user in chat:
- **Direct answer first.** Lead with the answer to the question — no preamble.
- **Uncertain about.** Include this section only if the sub-agent's "uncertain about" list is non-empty; mark it clearly so the user can verify those claims.
- **References.** Include this section at the end **only when the sub-agent actually consulted the web**, listing each URL on its own line. If no web sources were used, omit the section entirely.
- No file changes, no commits, no `HISTORIC.md` updates.

**[Phase 4: Cleanup]**
Nothing to clean up — this skill is fully ephemeral. If `.grimoire/PROJECT.md` was missing during Phase 1, remind the user that running `grimoire-init` once would enrich future answers. End.

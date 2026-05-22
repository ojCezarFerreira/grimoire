---
name: grimoire-update
description: Update the Grimoire plugin to its latest version — detects installed version, fetches the latest from GitHub, shows the changelog, and guides the user through /plugin marketplace update + /reload-plugins.
argument-hint: (no arguments)
---

**[Required Reading]**
1. Read `${CLAUDE_PLUGIN_ROOT}/.claude-plugin/plugin.json`. This is the source of truth for the **installed** Grimoire version on this machine.
2. Read `${CLAUDE_PLUGIN_ROOT}/GRIMOIRE-CONVENTIONS.md` and load `§ IDE-aware review` — Phase 4 below depends on it to decide how the changelog is shown to the user. Other sections of the conventions do not apply to this skill.
3. If `.grimoire/PROJECT.md` exists in the project root, read it for project context only (tone, audience). Missing → proceed without it; do not suggest `grimoire-init` here — this skill is plugin maintenance, not project bootstrapping.

**[Objective]**
Update the Grimoire plugin installed in this Claude Code session to the latest published version. The skill does NOT modify any file inside the plugin install directory (`~/.claude/plugins/`) — Claude Code owns that. It reads the installed version, compares it to the latest on GitHub, shows the user what changed, and guides them through the two official commands required to apply the update. The skill makes no commits in the consumer project; the only file it may write is the ephemeral changelog draft at `.grimoire/bag/drafts/grimoire-update-changelog.md` when running in IDE mode, which is removed before the skill ends.

**[Phase 1: Read installed version]**
- Open `${CLAUDE_PLUGIN_ROOT}/.claude-plugin/plugin.json` and extract the `version` field.
- Report to the user: `Installed Grimoire version: X.Y.Z`.
- If the file is missing or malformed, stop and tell the user the installed plugin manifest is unreadable — they should reinstall Grimoire via `/plugin install grimoire@grimoire`. Do not proceed.

**[Phase 2: Fetch latest version]**
- `WebFetch` `https://raw.githubusercontent.com/ojCezarFerreira/grimoire/main/.claude-plugin/plugin.json`.
- Extract the `version` field from the response.
- If the request fails (network, repo moved, 404), stop and tell the user the latest version could not be determined and ask them to retry later. Do **not** assume the plugin is up to date and do **not** suggest the update commands.

**[Phase 3: Compare versions]**
- Compare the installed `version` (Phase 1) to the latest `version` (Phase 2) as strings.
- **If equal:** print `Grimoire is already up to date (X.Y.Z). Nothing to do.` and end the skill. Do **not** print any commands.
- **If different:** continue to Phase 4.

**[Phase 4: Show changelog]**
- `WebFetch` `https://raw.githubusercontent.com/ojCezarFerreira/grimoire/main/CHANGELOG.md`.
- Extract every `## [VERSION] — DATE` section strictly between the installed version (exclusive) and the latest version (inclusive). For each, include the `### Added` / `### Changed` / `### Fixed` / `### Removed` bullets verbatim.
- If the installed version is older than every section in the file, include all sections.
- Present the extracted slice for review per `§ IDE-aware review`:
  - **IDE detected:** create `.grimoire/bag/drafts/` if it does not exist, then write the changelog to `.grimoire/bag/drafts/grimoire-update-changelog.md` so the IDE renders it. Tell the user in one short line that the changelog is open in the editor.
  - **Terminal-only fallback:** print the extracted slice to the user inline in markdown — concise, no commentary on top.
- If the changelog cannot be fetched, print a one-line warning (`Changelog not available — proceeding without it.`) and continue. Do not abort.

**[Phase 5: Pause — authorization]**
- Ask the user: `Proceed with the update from X.Y.Z to A.B.C? (yes/no)`.
- Wait for an explicit answer. Anything other than an unambiguous yes ends the skill cordially with no further output. If Phase 4 wrote `.grimoire/bag/drafts/grimoire-update-changelog.md` (IDE mode), delete it before ending. This is the only chance the user has to cancel before the guidance phases begin.

**[Phase 6: Guide marketplace update]**
- Print the following block verbatim:

  > Run this command in Claude Code to pull the new version into your installed plugin:
  >
  > ```
  > /plugin marketplace update grimoire
  > ```
  >
  > This updates the installed plugin in place — no uninstall/reinstall needed.

- Pause and wait for the user to confirm they ran it (`done`, `ok`, or similar). If they report an error, surface it and stop — do not advance to Phase 7.

**[Phase 7: Guide reload]**
- Print the following block verbatim:

  > To activate the new version in this current session without restarting Claude Code, run:
  >
  > ```
  > /reload-plugins
  > ```

- Pause and wait for the user to confirm.

**[Phase 8: Summary]**
- Print a one-line confirmation: `Grimoire updated from X.Y.Z to A.B.C. Skills are now reloaded.`
- Point the user at the full changelog if they want more detail: `Full changelog: https://github.com/ojCezarFerreira/grimoire/blob/main/CHANGELOG.md`.
- If Phase 4 wrote `.grimoire/bag/drafts/grimoire-update-changelog.md` (IDE mode), delete it now. The draft is ephemeral and must not survive past the skill's run.

**[Notes]**
- This skill never spawns sub-agents and never makes commits in the consumer project — the §TDD, §Sub-agent spawning, and §Commits sections of `GRIMOIRE-CONVENTIONS.md` do not apply. The only file it may write in the consumer project is the ephemeral changelog draft at `.grimoire/bag/drafts/grimoire-update-changelog.md` when running in IDE mode (see `§ IDE-aware review`), which is removed before the skill ends. Otherwise it is pure orchestration of two slash commands the user must run themselves.
- This skill always assumes a marketplace install (`/plugin install grimoire@grimoire`). Users who installed via `--plugin-dir` or a direct git URL should run `git pull` in their plugin directory instead of the Phase 6 command — but this skill does not detect that case.

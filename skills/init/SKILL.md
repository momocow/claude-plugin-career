---
name: init
description: Bootstrap the ~/.career configuration and journals directory used by the journal-orchestrator and journal-analyzer agents. Run this once before generating your first journal.
user-invocable: true
---

# Career — initialize

Bootstrap the configuration the `journal-orchestrator` and `journal-analyzer` agents depend on. Idempotent — safe to re-run; nothing existing is overwritten without explicit consent.

## What this creates

- `~/.career/config` — YAML config telling the agents where to write journals.
- `<journals_dir>/` — the directory journals will be written into. Default: `~/.career/journals/`.
- `<journals_dir>/resumes/` — subdirectory where `/career:resume` writes resume drafts.
- **Path-specific permissions** in `~/.claude/settings.json` — so subagents can read/write the journals directory, resume output directory, and session data without interactive prompts (critical for background agents).

## Procedure

1. **Check `~/.career/config`.**
   - If the file already exists:
     - Read it and report the current `journals` value.
     - Verify the journals directory exists and is writable.
     - If everything looks good, report success and exit. Do **not** modify the file.
     - If `journals` is missing or unreadable, ask the user before changing anything.
   - If the file does not exist, proceed.

2. **Pick the journals directory.**
   - Default suggestion: `~/.career/journals/`.
   - If the user has expressed a preference (in this conversation or prior), honor it.
   - Otherwise ask the user: "Where should journals be stored? [default: `~/.career/journals/`]"

3. **Create the journals directory and resumes subdirectory** (with `mkdir -p`). Expand `~`. Verify they're writable.
   - Create `<journals_dir>/` if it doesn't exist.
   - Create `<journals_dir>/resumes/` if it doesn't exist. This is where `/career:resume` writes output.

4. **Offer to initialize it as a git repo.**
   - Ask: "The journals directory can be a git repo so your journal history is versioned. Initialize one now? [y/N]"
   - If yes:
     - Run `git init` in the journals directory.
     - Create a minimal `.gitignore` (e.g., `.DS_Store`, editor swap files).
     - Do **not** make an initial commit unless the user asks — let them stage and commit on their own terms.
   - If no, skip silently.

5. **Write `~/.career/config`** with this minimal content:

   ```yaml
   journals: <chosen path>
   # Optional fields the orchestrator will honor when present:
   # timezone: Asia/Taipei
   # categoryLookbackDays: 30
   ```

   Use the canonical path (with `~` if the user wants it portable across machines, or absolute if they prefer explicit).

6. **Write path-specific permissions to `~/.claude/settings.json`.**

   The plugin ships portable tool permissions (e.g., `Bash(python3 *)`) via its own `settings.json`. But path-specific rules depend on where the user chose to store journals, so they must be written at init time.

   - Read `~/.claude/settings.json`. If it doesn't exist, start with `{}`.
   - Merge the following entries into `permissions.allow` (do **not** replace existing entries — append only, skip duplicates):
     - `Read(<journals_dir_absolute>/*)`
     - `Write(<journals_dir_absolute>/*)`
     - `Read(<journals_dir_absolute>/resumes/*)`
     - `Write(<journals_dir_absolute>/resumes/*)`
     - `Read(<home>/.career/*)`
     - `Read(<home>/.claude/projects/**)`
   - Merge into `sandbox.filesystem.allowWrite` (append, skip duplicates):
     - `<journals_dir_absolute>`
     - `<journals_dir_absolute>/resumes`
   - Write the merged file back. Validate JSON before writing.
   - Report what was added (e.g., "Added 4 permission rules and 1 sandbox path to ~/.claude/settings.json").

   **Why this step exists:** background subagents cannot prompt for permission interactively. Without pre-approved path rules, the orchestrator and analyzer agents will fail silently when running in background. The plugin's `settings.json` handles tool-level permissions (portable); this step handles path-level permissions (user-specific).

7. **Offer to set up daily journal generation.**
   - The AI journal is most useful when it runs automatically at the end of each day.
   - **Important:** `/schedule` creates *remote* agents in Anthropic's cloud. Remote agents **cannot** access local session JSONL files (`~/.claude/projects/`), so `/schedule` will not work for journal generation. The journal pipeline must run locally.
   - Ask: "Want to set up a daily cron job to run `/career:journal yesterday` each morning? [y/N]"
   - If yes:
     - Recommend a sensible default: a **system crontab** entry running at 10:00 local time.
     - Show the exact command: `crontab -e` and add:
       ```
       0 <UTC-hour> * * * cd <journals_parent_dir> && claude -p "/career:journal yesterday"
       ```
       (Convert the user's preferred local time to UTC for the cron expression.)
     - Do **not** run `crontab` for them — show the line and let them add it.
   - If no or unclear, skip silently. The user can always set this up later.

8. **Report back** with:
   - Config file path
   - Journals directory path
   - Whether git was initialized
   - Permissions added to `~/.claude/settings.json` (count and summary)
   - Whether the user opted to set up a cron (and if so, the exact line to add)
   - The next step: e.g., "Run `/career:journal` to generate today's AI journal, or `/career:user-journal` to start a personal entry."

## Constraints

- **Never overwrite an existing `~/.career/config`** without explicit user confirmation. The user may have hand-edited fields the agents don't yet know about.
- **Never delete files or directories.** This skill only creates.
- **Never commit or push** in the journals git repo. Initialization stops at `git init` + `.gitignore`.
- **Be quiet on success.** A short status block is enough; don't dump the full config back at the user unless they ask.

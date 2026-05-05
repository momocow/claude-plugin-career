---
name: init
description: Bootstrap the ~/.career configuration and journals directory used by the /career:journal skill and the journal-analyzer subagent. Run this once before generating your first journal.
user-invocable: true
---

# Career — initialize

Bootstrap the configuration the `/career:journal` skill and the `journal-analyzer` subagent depend on. Idempotent — safe to re-run; nothing existing is overwritten without explicit consent.

## What this creates

- `~/.career/config` — YAML config telling the agents where to write journals and resumes.
- `<journals_dir>/` — the directory journals will be written into. Default: `~/.career/journals/`.
- `<resumes_dir>/` — the directory `/career:resume` writes resume drafts into. Default: sibling of journals (e.g., `~/.career/resumes/` when journals is `~/.career/journals/`). Kept separate so resume drafts don't churn inside a versioned journals git repo.
- `~/.career/redactions.yaml` — empty stub (with explanatory comments) where the user lists internal/sensitive terms (codenames, project names) and their vague replacements. Read by `/career:resume` to scrub the output. Lives outside the journals dir on purpose: the list itself is sensitive and shouldn't be casually committed alongside journals.
- **Path-specific permissions** in `~/.claude/settings.json` — so subagents can read/write the journals directory, resume output directory, and session data without interactive prompts (critical for background agents).

## Procedure

1. **Check `~/.career/config`.**
   - If the file already exists:
     - Read it and report the current `journals` and `resumes` values (note if `resumes` is absent — older configs predate that field and will fall back to the sibling default).
     - Verify the journals directory and resumes directory both exist and are writable. If `resumes` is absent, treat the sibling of journals (e.g., `~/.career/resumes/` when journals is `~/.career/journals/`) as the default and create it if missing.
     - If a legacy `<journals_dir>/resumes/` directory exists from a previous version of the plugin, **warn the user** that resume output now defaults to a sibling directory and that they may want to move existing drafts. Do **not** auto-migrate — leave the old directory alone.
     - If everything looks good, report success and exit. Do **not** modify the file unless `resumes` was missing and you added the default (in which case ask first).
     - If `journals` is missing or unreadable, ask the user before changing anything.
   - If the file does not exist, proceed.

2. **Pick the journals directory.**
   - Default suggestion: `~/.career/journals/`.
   - If the user has expressed a preference (in this conversation or prior), honor it.
   - Otherwise ask the user: "Where should journals be stored? [default: `~/.career/journals/`]"

3. **Pick the resumes directory.**
   - Default suggestion: sibling of the journals directory — e.g., `~/.career/resumes/` when journals is `~/.career/journals/`. Compute it as `<parent of journals_dir>/resumes/`.
   - If the user has expressed a preference, honor it.
   - Otherwise ask the user: "Where should resume drafts be written? [default: `<sibling-default>`]"
   - The sibling default is intentional: journals are often a git repo, and resume drafts (especially JD-tailored ones) churn frequently and shouldn't pollute that history.

4. **Create the journals and resumes directories** (with `mkdir -p`). Expand `~`. Verify they're writable.
   - Create `<journals_dir>/` if it doesn't exist.
   - Create `<resumes_dir>/` if it doesn't exist. This is where `/career:resume` writes output.

5. **Offer to initialize the journals directory as a git repo.**
   - Ask: "The journals directory can be a git repo so your journal history is versioned. Initialize one now? [y/N]"
   - If yes:
     - Run `git init` in the journals directory.
     - Create a minimal `.gitignore` (e.g., `.DS_Store`, editor swap files).
     - Do **not** make an initial commit unless the user asks — let them stage and commit on their own terms.
   - If no, skip silently.
   - Do **not** offer to git-init the resumes directory. Resume drafts are typically ephemeral and JD-specific; versioning them is rarely useful.

6. **Write `~/.career/config`** with this minimal content:

   ```yaml
   journals: <chosen journals path>
   resumes:  <chosen resumes path>
   # Optional fields the /career:journal skill will honor when present:
   # timezone: Asia/Taipei
   # categoryLookbackDays: 30
   ```

   Use the canonical path (with `~` if the user wants it portable across machines, or absolute if they prefer explicit). If the resumes path is the sibling default, still write it explicitly so the config is self-documenting.

7. **Scaffold `~/.career/redactions.yaml`** if it doesn't already exist.

   - If the file exists, leave it alone.
   - If not, write this template:
     ```yaml
     # Sensitive terms to scrub from /career:resume output.
     # Each entry: a term that appears in your journals (codename, internal product, etc.)
     # and the vague replacement to use in resume drafts.
     #
     # Match is case-insensitive on whole words. Substitution applies only to user-facing
     # bullets in the resume — never to the journal files themselves.
     #
     # Example:
     # redactions:
     #   - term: "Project Phoenix"
     #     replacement: "an internal authentication platform"
     #   - term: "atlas-v2"
     #     replacement: "a customer data platform"

     redactions: []
     ```
   - Do **not** ask the user to populate it now. The first `/career:resume` run will surface flagged terms and offer interactive options to add entries.

8. **Write path-specific permissions to `~/.claude/settings.json`.**

   The plugin ships portable tool permissions (e.g., `Bash(python3 *)`) via its own `settings.json`. But path-specific rules depend on where the user chose to store journals and resumes, so they must be written at init time.

   - Read `~/.claude/settings.json`. If it doesn't exist, start with `{}`.
   - Merge the following entries into `permissions.allow` (do **not** replace existing entries — append only, skip duplicates):
     - `Read(<journals_dir_absolute>/*)`
     - `Write(<journals_dir_absolute>/*)`
     - `Read(<resumes_dir_absolute>/*)`
     - `Write(<resumes_dir_absolute>/*)`
     - `Read(<home>/.career/*)`
     - `Write(<home>/.career/redactions.yaml)`
     - `Read(<home>/.claude/projects/**)`
   - Merge into `sandbox.filesystem.allowWrite` (append, skip duplicates):
     - `<journals_dir_absolute>`
     - `<resumes_dir_absolute>`
     - `<home>/.career/redactions.yaml`
   - Write the merged file back. Validate JSON before writing.
   - Report what was added (e.g., "Added 5 permission rules and 3 sandbox paths to ~/.claude/settings.json").

   **Why this step exists:** background subagents cannot prompt for permission interactively. Without pre-approved path rules, the `journal-analyzer` subagent will fail silently when running in background, and the `/career:journal` skill won't be able to write the journal file. The plugin's `settings.json` handles tool-level permissions (portable); this step handles path-level permissions (user-specific). The `Write(redactions.yaml)` rule is needed because the `[a] Add replacements now` path in `/career:resume` appends to the file.

9. **Offer to set up daily journal generation.**
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

10. **Report back** with:
    - Config file path
    - Journals directory path
    - Resumes directory path (and whether it's the sibling default)
    - Redactions file path (and whether it was newly scaffolded or already existed)
    - Whether git was initialized in the journals directory
    - Permissions added to `~/.claude/settings.json` (count and summary)
    - Whether the user opted to set up a cron (and if so, the exact line to add)
    - The next step: e.g., "Run `/career:journal` to generate today's AI journal, or `/career:user-journal` to start a personal entry."

## Constraints

- **Never overwrite an existing `~/.career/config`** without explicit user confirmation. The user may have hand-edited fields the agents don't yet know about.
- **Never delete files or directories.** This skill only creates.
- **Never commit or push** in the journals git repo. Initialization stops at `git init` + `.gitignore`.
- **Be quiet on success.** A short status block is enough; don't dump the full config back at the user unless they ask.

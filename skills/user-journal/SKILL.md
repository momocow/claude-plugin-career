---
name: user-journal
description: Scaffold a personal, user-authored journal file (YYYY-MM-DD.user.md) that sits alongside the AI-generated journal. Use this when the user wants to write their own reflections, corrections, or off-screen accomplishments for a given day.
user-invocable: true
---

# User journal — create or open

Scaffold a `YYYY-MM-DD.user.md` file so the user can record reflections, corrections, and off-screen accomplishments that the AI-generated journal can't capture.

## Why two files?

In this plugin, `YYYY-MM-DD.md` is **AI-owned** and regenerated on every `journal-orchestrator` run — any manual edits to it are lost. `YYYY-MM-DD.user.md` is **user-owned** and the AI (orchestrator, analyzer) will never read or modify it. Downstream tools like resume-gen may consume both files as a pair.

## Procedure

1. **Resolve the journals directory.**
   - Read `~/.career/config` (YAML) and take the `journals:` value. Expand `~`.
   - If the file is missing, fall back to `~/.career/journals/` and warn the user that `/career:init` hasn't been run yet (but proceed).
   - If the directory doesn't exist, create it.

2. **Resolve the target date.**
   - Default: today, in the user's local timezone.
   - If the user passed a date argument (e.g., `/career:user-journal 2026-04-14`), use that. Accept `YYYY-MM-DD`, `today`, or `yesterday`.
   - Reject dates in the future.

3. **Compute the path:** `<journals_dir>/<YYYY-MM-DD>.user.md`.

4. **If the file already exists:**
   - Do **not** overwrite.
   - Report the path to the user and offer to open it (leave opening to the user's editor — don't launch tools automatically).
   - Exit.

5. **If the file does not exist, write a scaffold:**

   ```markdown
   ---
   date: <YYYY-MM-DD>
   authored: user
   ---

   # Journal — <YYYY-MM-DD> (personal)

   ## Reflections
   <!-- Free-form thoughts on the day: what went well, what was hard, what you learned. -->

   ## Corrections to the AI journal
   <!-- If <YYYY-MM-DD>.md got something wrong, missed an accomplishment, or mislabeled a category, note it here.
        Format suggestion:
        - Accomplishment "<original text>" — correction: <what it should say>
        - Missing accomplishment: <bullet in resume style>
        - Category drift: <from> → <to>
   -->

   ## Off-screen accomplishments
   <!-- Work that didn't happen inside a Claude session but belongs on your record.
        Examples: PR reviews, design discussions, mentoring, interviews, talks, decisions made in meetings.
        Keep bullets in resume style: action verb + what + outcome/scope. -->

   ## Other notes
   <!-- Anything else worth remembering — context resume-gen might find useful later. -->
   ```

   Use the `Write` tool directly — no `.tmp` + rename dance.

6. **Report back** with:
   - The file path
   - Whether it was newly created or already existed
   - A short reminder: "Edits here are yours — the AI will not touch this file."

## Constraints

- **Never overwrite** an existing `.user.md`. If the user wants to start over, they should delete or rename it manually.
- **Never modify** the AI-managed `<YYYY-MM-DD>.md` file from this skill. Those two files stay strictly separated.
- **Stay minimal.** The scaffold is a starting shape, not a mandate — the user may delete sections or add their own. Do not enforce structure on re-opens.
- **No date coercion without asking.** If the user passes an ambiguous or invalid date, ask for clarification rather than guessing.

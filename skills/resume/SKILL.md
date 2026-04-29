---
name: resume
description: Generate structured resume content from career journals over a date range. Reads both AI-generated (.md) and user-authored (.user.md) journal files. Clusters accomplishments across days, elevates with XYZ formula, scores for quality, and optionally tailors to a job description.
user-invocable: true
---

# Resume — generate resume content from journals

Parse the user's arguments, then delegate to the `resume-builder` subagent. This skill handles argument parsing, config resolution, JD loading, and output writing. All resume logic lives in the agent.

## Accepted arguments

| Form | Meaning |
|------|---------|
| `YYYY-MM-DD..YYYY-MM-DD` | Date range (required) — which journals to consume |
| `--resume <path>` | Path to your existing resume file (markdown, plain text, or PDF). The builder updates it with journal-derived content rather than generating from scratch. |
| `--jd <path>` | Path to a job description file (plain text or markdown) |
| `--role "Title"` | Current role title, used for framing the professional summary |
| `--company "Company"` | Current company name |
| `--max-bullets <N>` | Max bullets per project section (default 5) |
| `--format markdown\|json` | Output format (default: `markdown`) |

At minimum, a date range must be provided. Everything else is optional.

## Procedure

1. **Parse arguments.**
   - The date range is required. Reject if missing and ask the user.
   - Accept `YYYY-MM-DD..YYYY-MM-DD` as an inclusive range.
   - Also accept shorthands for common ranges:
     - `this-week` — Monday through today
     - `last-week` — previous Monday through Sunday
     - `this-month` — first of the current month through today
     - `last-month` — first through last day of the previous month
     - `this-quarter` — first of the current quarter through today
   - Reject future end-dates. Reject inverted ranges (`end < start`).

2. **Read `~/.career/config`** to get the journals directory.
   - Expand `~` to the user's home directory.
   - If `~/.career/config` is missing, fall back to `~/.career/journals/` and warn the user that `/career:init` hasn't been run yet.
   - Verify the journals directory exists.

3. **Load the existing resume** (if `--resume` was provided).
   - If `--resume` points to a file path, read it.
   - If the file doesn't exist, report the error and ask the user to check the path.
   - Store the resume text for passing to the agent. The resume-builder will parse it to extract existing bullets, skills, structure, and formatting conventions.
   - Supported formats: Markdown (`.md`), plain text (`.txt`), or PDF (`.pdf`). For PDF, use the `Read` tool which handles PDF natively.

4. **Load the job description** (if `--jd` was provided).
   - If `--jd` points to a file path, read it.
   - If the file doesn't exist, report the error and ask the user to check the path.
   - Store the JD text for passing to the agent.

5. **Check for journals in range.**
   - Quick glob to see if any `YYYY-MM-DD.md` files exist in the date range.
   - If none, report: "No journals found for this date range. Run `/career:journal` first to generate them."
   - If some dates are missing within the range, note the gaps but proceed with what's available.

6. **Invoke the `resume-builder` agent** via the Agent tool with a prompt containing:
   - The date range (start and end dates)
   - The journals directory path
   - The existing resume text (if provided)
   - The JD text (if provided, pass the full text)
   - Role and company context (if provided)
   - `maxBulletsPerProject` (from `--max-bullets` or default 5)

7. **Format and write the output.**

   Resolve the output path: `<journals_dir>/resumes/<start>--<end>.md` (or `.json` if `--format json`).
   - Create the `resumes/` subdirectory if it doesn't exist.
   - If the file already exists, ask the user: "A resume for this range already exists at `<path>`. Overwrite? [y/N]"

   **If `--format markdown` (default):** Transform the agent's JSON output into a human-readable Markdown document:

   ```markdown
   ---
   dateRange:
     start: YYYY-MM-DD
     end: YYYY-MM-DD
   generatedAt: <ISO-8601 timestamp>
   journalsRead: { ai: N, user: M }
   accomplishmentsProcessed: N
   clustersFormed: N
   existingResumeProvided: true|false
   jdProvided: true|false

   ---

   # Resume Draft — YYYY-MM-DD to YYYY-MM-DD

   > **This is a draft.** Review, edit, and personalize before using on an actual resume.

   ## Professional Summary

   <summary text>

   ## Key Achievements

   - <top bullet 1>
   - <top bullet 2>
   - <top bullet 3>

   ## Experience

   ### <Project Name> — <start> to <end>

   - <bullet 1>
   - <bullet 2>
   - <bullet 3>

   <details>
   <summary>Derivation details</summary>

   Bullet 1 derived from:
   - [YYYY-MM-DD] Original daily bullet text
   - [YYYY-MM-DD] Another daily bullet text

   </details>

   ### <Project Name 2> — <start> to <end>
   ...

   ## Technical Skills

   **Languages:** TypeScript, Python, Go
   **Frameworks:** React, Next.js, Express
   **Infrastructure:** Docker, GitHub Actions, AWS
   **Databases:** PostgreSQL, Redis
   **Tools:** Claude Code, pnpm

   ## Superseded Bullets

   > Only shown if an existing resume was provided and some bullets were replaced.

   The following existing bullets were replaced by stronger journal-derived versions:

   | Original | Replacement | Reason |
   |----------|-------------|--------|
   | <old bullet> | <new bullet> | <why the new one is stronger> |

   ## JD Analysis

   > Only shown if a job description was provided.

   **Covered requirements:** React, TypeScript, CI/CD
   **Gaps (required):** Kubernetes, GraphQL
   **Gaps (preferred):** Terraform
   **Terminology suggestions:**
   - JD says "CI/CD pipelines" → your bullet says "deployment automation" — consider rewording
   **Seniority alignment:** <assessment>
   ```

   **If `--format json`:** Write the agent's JSON output directly.

   Write atomically (`.tmp` then rename).

8. **Report back** to the user:
   - Output file path
   - Counts: journals read (AI + user), accomplishments processed, clusters formed
   - If existing resume was provided: how many existing bullets were kept, how many were superseded by stronger journal-derived versions, how many new bullets were added
   - If JD was provided: brief coverage summary (N required skills covered, M gaps)
   - Reminder: "This is a draft — review and personalize before use."

## Constraints

- **Never modify journal files.** This skill and its agent are read-only consumers of the journal pipeline's output.
- **Date range is mandatory.** Do not default to "all time" — the user must make a conscious choice about scope.
- **Don't invent a JD.** If no JD is provided, skip tailoring entirely. Do not prompt the user to provide one unless they ask about it.
- **Be honest about gaps.** If journals are missing for dates in the range, report the gaps. If v1 journals limit the quality, say so.
- **Overwrite protection.** Always ask before overwriting an existing resume file.
- **No git operations.** Do not add, commit, or push the resume file. The user manages their own git workflow.

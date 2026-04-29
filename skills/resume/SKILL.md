---
name: resume
description: Generate structured resume content from career journals over a date range. Reads both AI-generated (.md) and user-authored (.user.md) journal files. Clusters accomplishments across days, elevates with XYZ formula, scores for quality, optionally tailors to a job description, and scrubs internal/sensitive terms (codenames, private project names) via configurable redactions with an interactive review gate.
user-invocable: true
---

# Resume — generate resume content from journals

Parse the user's arguments, delegate clustering/elevation/scoring to the `resume-builder` subagent, then post-process the agent's output (redaction substitution, sensitive-term review gate, formatting) before writing. This skill handles argument parsing, config resolution, JD loading, redaction handling, and output writing. All clustering/elevation/JD-tailoring logic lives in the agent.

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
| `--out <path>` | Custom output path. Absolute paths are used as-is; bare filenames or relative paths resolve under `<resumes_dir>`. Defaults to `<start>--<end>.md` (or `.json` for `--format json`). |
| `--redactions <path>` | Path to a redactions file (YAML). Defaults to `~/.career/redactions.yaml`. Sensitive internal terms (codenames, etc.) listed there are replaced with vague descriptions in the output. |

At minimum, a date range must be provided. Everything else is optional.

## Procedure

Steps are ordered for fail-fast: cheap checks and interactive prompts that can abort the run come before expensive operations (PDF reads, agent invocation).

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

2. **Read `~/.career/config`** to get the journals and resumes directories.
   - Expand `~` to the user's home directory.
   - If `~/.career/config` is missing, fall back to `~/.career/journals/` and `~/.career/resumes/` and warn the user that `/career:init` hasn't been run yet.
   - If `journals` is set but `resumes` is missing, default `resumes` to the sibling of journals (e.g., `~/.career/resumes/` when journals is `~/.career/journals/`). Compute as `<parent of journals_dir>/resumes/`.
   - Verify the journals directory exists. The resumes directory will be created on first write if absent.

3. **Resolve the output path** (early — used by step 5's existence check).
   - If `--out` was provided:
     - Absolute path (starts with `/` or `~`) → use as-is (expand `~`).
     - Bare filename or relative path → resolve under `<resumes_dir>` (e.g., `--out my-draft.md` → `<resumes_dir>/my-draft.md`).
     - Do **not** auto-append an extension — respect what the user typed. If the extension and `--format` disagree (e.g., `--out foo.json --format markdown`), print a one-line notice ("note: `--out` extension `.json` doesn't match `--format markdown`; proceeding with markdown content in foo.json") and proceed.
   - Otherwise: `<resumes_dir>/<start>--<end>.md` (or `.json` if `--format json`).
   - Store the resolved path for use in steps 5 and 11.

4. **Check for journals in range** (cheap; do this before expensive loads).
   - Quick glob to see if any `YYYY-MM-DD.md` files exist in the date range.
   - If none, report: "No journals found for this date range. Run `/career:journal` first to generate them." Abort.
   - If some dates are missing within the range, note the gaps but proceed with what's available.

5. **Check output file existence** (early — avoids wasting an agent run if the user declines to overwrite).
   - If the resolved path from step 3 already exists, ask: "A resume already exists at `<path>`. Overwrite? [y/N]"
   - If the user declines, abort. Do **not** invoke the agent.

6. **Load the existing resume** (if `--resume` was provided).
   - If `--resume` points to a file path, read it.
   - If the file doesn't exist, report the error and ask the user to check the path.
   - Store the resume text for passing to the agent. The resume-builder will parse it to extract existing bullets, skills, structure, and formatting conventions.
   - Supported formats: Markdown (`.md`), plain text (`.txt`), or PDF (`.pdf`). For PDF, use the `Read` tool which handles PDF natively.

7. **Load the job description** (if `--jd` was provided).
   - If `--jd` points to a file path, read it.
   - If the file doesn't exist, report the error and ask the user to check the path.
   - Store the JD text for passing to the agent.

8. **Load redactions.**
   - Resolve the redactions path: `--redactions` if provided, else `~/.career/redactions.yaml`.
   - If the file is missing, treat the redaction map as empty (no warning — many users won't need this) and proceed.
   - Parse the YAML. Expected shape:
     ```yaml
     redactions:
       - term: "Project Phoenix"
         replacement: "an internal authentication platform"
       - term: "atlas-v2"
         replacement: "a customer data platform"
     ```
   - Build a substitution map `{ term → replacement }`. Build a list of *known terms* (just the keys) to send to the agent so it doesn't waste cycles flagging terms the user has already classified.
   - Validate that each entry has both `term` and `replacement` (non-empty). Report and skip malformed entries; do not abort.

   **Substitution semantics** (used by step 10a, and reused on every re-application after `[a]`/`[e]`):
   - Build a regex per term: `(?<![\w-])` + `re.escape(term)` + `(?![\w-])`, with `re.IGNORECASE`. The `[\w-]` look-arounds (instead of plain `\b`) keep kebab-case identifiers atomic — `atlas-v2` matches in "Migrated atlas-v2…" but **not** inside "atlas-v2-prod".
   - Replacement is the user's `replacement` string verbatim (no case restoration).
   - When multiple terms could match overlapping regions, apply longer terms first to avoid partial substitution (e.g., redact `Project Phoenix Mobile` before `Project Phoenix`).

9. **Invoke the `resume-builder` agent** via the Agent tool with a prompt containing:
   - The date range (start and end dates)
   - The journals directory path
   - The existing resume text (if provided)
   - The JD text (if provided, pass the full text)
   - Role and company context (if provided)
   - `maxBulletsPerProject` (from `--max-bullets` or default 5)
   - The **known redaction terms** list (just the keys from step 8) — so the agent can skip flagging them.

10. **Apply redactions and run the sensitive-terms review gate.**

    The agent returns a JSON payload that includes `flaggedSensitiveTerms` (terms in bullets that *look* internal but aren't in the redactions list).

    **10a. Apply the substitution map** to every user-facing text field in the JSON, using the regex semantics from step 8. Track per-term replacement counts — these become the `redactions.applied` frontmatter field synthesized in step 11 (the agent does not emit this; the skill computes it).

    **Substituted fields (exhaustive list):**
    - `professionalSummary`
    - `experience[].project` (this renders as a markdown section heading — leaks loudly if missed)
    - `experience[].title`
    - `experience[].company`
    - `experience[].bullets[].text`
    - `keyAchievements[].text`
    - `supersededBullets[].original`
    - `supersededBullets[].replacement`
    - `supersededBullets[].reason`
    - `passthroughSections[].content`
    - `jdAnalysis.terminologyMismatches[].bulletTerm`
    - `flaggedSensitiveTerms[].examples[].preview` (these previews surface in the warning block; redact known terms here too)

    **Never substituted:**
    - `experience[].bullets[].derivedFrom[].text` — provenance must remain verbatim so the user can audit back to the original journal text.
    - Any field outside the list above (counts, scores, dates, IDs, scores, slugs, etc.).

    **10b. Check `flaggedSensitiveTerms`.**
    - **If empty:** proceed to step 11. No prompt.
    - **If non-empty:** show the user a single prompt:
      ```
      Found N potentially sensitive term(s) not in your redactions file:
        1. "<term>"  — in: "<truncated bullet preview>…"
        2. "<term>"  — in: "<truncated bullet preview>…"
        ...

      What would you like to do?
        [a] Add replacements now — I'll prompt for each
        [e] Edit <redactions_path> yourself, then say "done"
        [s] Skip — write as-is (terms remain unredacted, banner added to draft)
        [c] Cancel
      ```
      Accept lenient input: `a`, `[a]`, `add`, "add replacements" all map to `[a]`; same shape for the others. If input is unrecognized, re-ask once before defaulting to `[c] Cancel`.

    **10c. Handle the response.**

    - **`[a]` Add replacements now.** For each flagged term in turn, prompt: "Replacement for `<term>`? (or `skip` to leave unredacted, `cancel` to abort the run)".
      - Replacement text → add `{ term, replacement }` to the in-memory substitution map and queue the entry for the file write.
      - `skip` → leave the term in the residual flagged set.
      - `cancel` → abort the entire run; do not write the output.
      - After the loop, **persist queued entries to `<redactions_path>`** (see "File-write semantics" below).
      - **Re-run step 10a** with the updated map.
      - Compute the *residual flagged set*: flagged terms whose `term` is still NOT in the substitution map. If non-empty, fall through to `[s]` semantics (banner + frontmatter warning).

    - **`[e]` Edit yourself.** Print the absolute path to `<redactions_path>` and wait for the user's reply.
      - Strict proceed signals (case-insensitive): `done`, `continue`, `proceed`, `ok`, `go`, `next`. The signal `cancel` aborts. Anything else: ask once for clarification before treating as `[c] Cancel`.
      - On proceed: re-load `<redactions_path>`, re-build the substitution map, re-run step 10a.
      - Compute the residual flagged set. If non-empty, fall through to `[s]` semantics.

    - **`[s]` Skip.** Mark all currently-flagged terms (or the residual set, when reached via `[a]`/`[e]` fall-through) for inclusion in `sensitiveTermsWarning`. Proceed to step 11.

    - **`[c]` Cancel.** Abort. Do not write the output file.

    **File-write semantics for `[a]`** (appending to `<redactions_path>`):
    - If the file is missing entirely, write a fresh file containing the explanatory comment header from `/career:init`'s template followed by the new entries under `redactions:`.
    - If the file exists: load it (`yaml.safe_load`). For each new entry, dedup against existing entries by case-insensitive `term` match — last-write-wins, with a one-line notice when an existing entry is replaced. Re-serialize with `yaml.safe_dump(default_flow_style=False, sort_keys=False)`. **Comments in the existing file may be lost on re-serialization** — print a one-line notice if the file had any comment lines so the user isn't surprised. (For round-trip comment preservation, install and use `ruamel.yaml`; the skill should not require it.)

    **Why the skill (not the agent) does substitution:** the agent already produced expensive structured output. If the user picks `[a]` or `[e]`, we re-substitute without re-running clustering and elevation. Keeping substitution in the skill makes it cheap and idempotent.

11. **Format and write the output.**

    The output path was already resolved in step 3 and overwrite-confirmed in step 5. Create the parent directory if it doesn't exist.

    **If `--format markdown` (default):** Transform the (already-redacted) JSON from step 10 into a human-readable Markdown document. Render `experience[].title` and `experience[].company` when present (typically when an existing resume baseline was provided); fall back to project-only headings otherwise.

    ```markdown
    ---
    dateRange:
      start: YYYY-MM-DD
      end: YYYY-MM-DD
    generatedAt: <ISO-8601 timestamp>
    journalsRead:
      ai: N
      user: M
    accomplishmentsProcessed: N
    clustersFormed: N
    existingResumeProvided: true|false
    jdProvided: true|false
    redactions:
      applied:                       # term → replacement count; omit the entire `applied:` key if the map is empty
        Project Phoenix: 3
        atlas-v2: 1
      sensitiveTermsWarning:         # only present if non-empty (residual after [a]/[e], or the full flagged set on [s])
        - OneCloud
        - k2-cluster
    ---

    # Resume Draft — YYYY-MM-DD to YYYY-MM-DD

    > **This is a draft.** Review, edit, and personalize before using on an actual resume.

    <!-- Only include this block when sensitiveTermsWarning is non-empty -->
    > ⚠️ **Review before sharing.** The following terms in this draft may be internal/confidential and were **not** auto-redacted because they were not in your redactions file:
    > - `OneCloud` (e.g., in: "Built OneCloud auth flow…")
    > - `k2-cluster` (e.g., in: "Reduced k2-cluster p99 latency…")
    >
    > Add them to `<redactions_path>` and re-run, or hand-edit this draft before sending it anywhere.

    ## Professional Summary

    <summary text>

    ## Key Achievements

    - <top bullet 1>
    - <top bullet 2>
    - <top bullet 3>

    ## Experience

    <!-- When baseline provided title and company -->
    ### <Title> @ <Company> — <Project Name>
    *<start> – <end>*

    <!-- When no baseline (no title/company) -->
    ### <Project Name>
    *<start> – <end>*

    - <bullet 1>
    - <bullet 2>
    - <bullet 3>

    <details>
    <summary>Derivation details</summary>

    Bullet 1 derived from:
    - [YYYY-MM-DD] Original daily bullet text
    - [YYYY-MM-DD] Another daily bullet text

    </details>

    ### <Project Name 2>
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

    **If `--format json`:** Write the **redacted** JSON — i.e., the in-memory object after step 10a, with the skill-synthesized fields nested at `meta.redactions`:
    ```json
    {
      "meta": {
        "...": "...",
        "redactions": {
          "applied": { "Project Phoenix": 3, "atlas-v2": 1 },
          "sensitiveTermsWarning": ["OneCloud", "k2-cluster"]
        }
      },
      "...": "..."
    }
    ```
    Omit `meta.redactions.applied` if the map is empty; omit `meta.redactions.sensitiveTermsWarning` if no flagged terms remain unredacted; omit the entire `meta.redactions` object if both are empty. Do **not** write the agent's pre-redaction output. Do **not** transform to markdown.

    Use the `Write` tool directly — no `.tmp` + rename dance.

12. **Report back** to the user:
    - Output file path
    - Counts: journals read (AI + user), accomplishments processed, clusters formed
    - If existing resume was provided: how many existing bullets were kept, how many were superseded by stronger journal-derived versions, how many new bullets were added
    - If JD was provided: brief coverage summary (N required skills covered, M gaps)
    - Redaction summary: total replacements made (e.g., "Redacted 4 occurrences across 2 terms"). If `sensitiveTermsWarning` is non-empty, surface the count prominently and remind the user the warning is in the file.
    - Reminder: "This is a draft — review and personalize before use."

## Constraints

- **Never modify journal files.** This skill and its agent are read-only consumers of the journal pipeline's output.
- **Date range is mandatory.** Do not default to "all time" — the user must make a conscious choice about scope.
- **Don't invent a JD.** If no JD is provided, skip tailoring entirely. Do not prompt the user to provide one unless they ask about it.
- **Be honest about gaps.** If journals are missing for dates in the range, report the gaps. If v1 journals limit the quality, say so.
- **Overwrite protection.** Ask before overwriting an existing resume file. The check happens **before** invoking the agent (step 5) so a "no" doesn't waste an agent run.
- **No git operations.** Do not add, commit, or push the resume file. The user manages their own git workflow.
- **Redactions apply to JSON output too.** `--format json` writes the *redacted* in-memory object after step 10a — never the agent's raw pre-redaction output.
- **Redactions never apply to `derivedFrom` provenance.** Provenance is the user's audit trail back to the original journal text and must remain verbatim. Substitution applies only to user-facing bullets.
- **Residual flagged terms always surface a warning.** Any flagged term not absorbed into the substitution map — whether the user picked `[s]`, or skipped some during `[a]`, or didn't add them all in `[e]` — populates `sensitiveTermsWarning` and triggers the in-file warning banner. There is no path where flagged terms reach the output silently.
- **Redactions never modify the redactions file without consent.** Only the `[a]` interactive path appends to it, and only after the user provides a replacement.

# career

A Claude Code plugin that turns your Claude session history into a searchable, resume-ready journal — and then generates polished resume content from those journals. Three subagents read your session transcripts and git commits across every project you've worked on, classify the work, and emit daily journals with structured frontmatter. A fourth agent consumes those journals to produce clustered, XYZ-formula resume bullets tailored to specific job descriptions.

## What you get

| Command                              | Purpose                                                               |
| ------------------------------------ | --------------------------------------------------------------------- |
| `/career:init`                       | One-time setup: config file, journals directory, optional git repo    |
| `/career:journal [date\|range]`      | Generate or regenerate the AI daily journal (delegates to orchestrator) |
| `/career:user-journal [date]`        | Scaffold a personal, user-authored journal entry                       |
| `/career:resume range [--resume path] [--jd path] [--out path]` | Generate or update resume content from journals, optionally from an existing resume and/or tailored to a JD; `--out` overrides the default output filename |

Plus three subagents the model calls on your behalf:

- **`journal-orchestrator`** — discovers active projects, fans out to analyzers in deterministic batched waves, normalizes category slugs, and writes an idempotent daily journal
- **`journal-analyzer`** — per-project: reads the project's Claude session JSONL, enriches with `git log`, and emits structured accomplishments with resume signals (STAR components, tech stack, impact metrics, quality tiers)
- **`resume-builder`** — reads journals over a date range, clusters related accomplishments across days, elevates with XYZ formula, scores for quality, and structures into resume sections with optional JD tailoring

## Mental model

```
Claude sessions (~/.claude/projects/)
         │
         ▼
┌─────────────────────────────┐
│  journal-orchestrator        │  invokes, per project, in waves
│  (discovery, batching,       │
│   category review, merge)    │
└─────────────────────────────┘
         │  fan-out
         ▼
┌─────────────────────────────┐
│  journal-analyzer × N        │  JSONL + git commits → structured JSON
│  (v2: + STAR, tech stack,    │    with resume signal enrichment
│   impact, quality tier)      │
└─────────────────────────────┘
         │  aggregate + normalize
         ▼
   <journals_dir>/YYYY-MM-DD.md        ← AI-owned, regenerated each run
   <journals_dir>/YYYY-MM-DD.user.md   ← User-owned, AI never touches
         │
         │  /career:resume (date range + optional JD)
         ▼
┌─────────────────────────────┐
│  resume-builder              │  reads .md + .user.md journals
│  (cluster, XYZ elevate,     │  clusters across days, scores,
│   score, JD tailor)          │  tailors to job descriptions
└─────────────────────────────┘
         │
         ▼
   <resumes_dir>/<range>.md            ← Resume draft output (default sibling of journals)
```

### File ownership

| File                 | Owner   | Who writes it                                          | AI reads it? |
| -------------------- | ------- | ------------------------------------------------------ | ------------ |
| `YYYY-MM-DD.md`      | AI      | `journal-orchestrator` (via `/career:journal`)         | Yes — regenerates in full on each run |
| `YYYY-MM-DD.user.md` | User    | You (scaffold via `/career:user-journal`, edit freely) | **No — strictly off-limits** |

This separation exists because the AI regenerates the `.md` side completely on every run. Trying to keep manual edits in there fights the tool; putting them in `.user.md` removes the conflict entirely. Downstream tools (resume-gen) consume both files as a pair.

## Setup

1. **Install the plugin** via your Claude Code marketplace (see the parent repo's marketplace manifest).
2. **Run `/career:init`** — it creates:
   - `~/.career/config` (YAML)
   - A journals directory (default `~/.career/journals/`)
   - Optionally, a git repo in the journals directory so your history is versioned
   - **Path-specific permissions** in `~/.claude/settings.json` (required for background agents)
3. **Optional: set up a daily cron.** `/career:init` will show you the exact `crontab` line to add. Note: `/schedule` (remote agents) **will not work** — the journal pipeline reads local `~/.claude/projects/` session files that don't exist in the cloud.

## Permission model

Permissions are split into three layers to balance portability with security:

| Layer | File | What it covers | When resolved |
| --- | --- | --- | --- |
| **Plugin** | `plugins/career/settings.json` | Tool-level permissions (e.g., `Bash(python3 *)`, `Bash(git log *)`) — portable across machines | Plugin install |
| **Init** | `~/.claude/settings.json` | Path-specific permissions (e.g., `Write(<journals_dir>/*)`, sandbox allowWrite) — user-specific | `/career:init` |
| **Agent** | `agents/*.md` frontmatter `tools:` | Tool restrictions per agent (analyzer: `Read,Glob,Grep,Bash`; orchestrator: `Agent,Read,Write,Glob,Bash`) | Agent start |

This design exists because **background subagents cannot prompt for permission interactively.** Without pre-approved rules, background agents fail silently. The plugin ships what it can portably; `/career:init` fills in the paths that depend on user config.

## Configuration

`~/.career/config`:

```yaml
journals: ~/.career/journals/   # where journal files are written
resumes:  ~/.career/resumes/    # where /career:resume writes drafts (default: sibling of journals)
# timezone: Asia/Taipei          # for resolving "today" / "yesterday"
# categoryLookbackDays: 30       # how far back to scan for knownCategories
# analyzerBatchSize: 5           # max analyzers per wave (hard-capped at 10)
# maxAnalyzers: 30               # warn-and-confirm threshold for big fan-outs
```

Only `journals:` is required. `resumes:` defaults to the sibling of `journals:` (so `~/.career/journals/` → `~/.career/resumes/`). The split keeps resume drafts — which churn frequently and may be JD-tailored — out of a versioned journals git repo.

## How scaling works

The orchestrator batches analyzers to keep its own context flat and costs predictable:

- **Projects are sorted deterministically** (by encoded dir name) before batching — same input always produces same waves.
- **Default batch size: 5 in parallel**, hard-capped at 10 regardless of config.
- **Between waves**: each wave's outputs are folded into a running aggregate and then dropped from context. Raw analyzer responses do not accumulate.
- **Above 30 candidate projects**: orchestrator stops and asks you to narrow scope. Raise `maxAnalyzers` in config if 30 is too tight.
- **Per-analyzer turn budget: 20.** A pathological analyzer is skipped with reason `exceeded turn budget`; it doesn't retry.
- **Cross-wave category learning**: wave 2+ sees slugs minted in wave 1, reducing drift within a single run.

## Category soft-consistency

To keep slugs reusable across days (essential for downstream resume-gen grouping), the orchestrator:

1. Loads `knownCategories` from the frontmatter of the last 30 AI journals before fan-out (`.user.md` files are ignored).
2. Passes that list to every analyzer. The analyzer strongly prefers known slugs; it only mints new ones when no existing slug fits.
3. **Reviews minted slugs before aggregating.** If a new slug (e.g., `json-web-tokens`) is a reasonable substitute for an existing one (`jwt`), the orchestrator rewrites it across that analyzer's output. Replacements surface in the status block.
4. Surfaces any genuinely new slugs under `newCategories:` in the frontmatter so you can audit drift.

## Activity taxonomy (fixed)

Analyzers classify each session into one or more of these eight slugs (kebab-case, in `activities:`):

`development`, `debugging`, `refactoring`, `code-review`, `learning`, `research`, `planning`, `testing`

Categories (in `categories:`) are free-form slugs within the conventions documented in `agents/journal-analyzer.md` — tech, domain, language, with "specific over generic" as the guiding rule.

## Output format

Each `YYYY-MM-DD.md` is a Markdown doc with YAML frontmatter. The **frontmatter is the machine-readable contract** for resume-gen; the body is for humans. Accomplishments carry `evidence.commits` when git can attribute them, which makes every bullet verifiable.

Example skeleton:

```markdown
---
schemaVersion: 1
date: 2026-04-15
range:
  start: 2026-04-14T16:00:00Z
  end:   2026-04-15T15:59:59Z
activities: [development, debugging]
categories: [auth, jwt, react, testing]
techStack: [react, typescript, jwt, postgres]
newCategories: []
projects:
  - name: acme-web
    cwd: /Users/you/Workspace/acme-web
    sessionsCount: 3
    activities: [development, testing]
    categories: [auth, jwt, react]
accomplishments:
  - text: "Implemented JWT refresh-token rotation in auth/middleware.ts, eliminating silent session expiry on the dashboard"
    project: acme-web
    categories: [auth, jwt]
    evidence:
      filesTouched: [src/auth/middleware.ts]
      commits: [a1b2c3d]
    qualityTier: high
    techStack: [react, typescript, jwt]
    scope: { filesChanged: 4, servicesAffected: 1 }
    context:
      situation: "Dashboard users experienced silent session expiry"
      challenge: "Refresh rotation needed to be atomic and backward-compatible"
    impact:
      metric: "Eliminated 100% of silent session expiry incidents"
      type: reliability
    starComponents:
      situation: "Dashboard users were logged out silently when JWT access tokens expired"
      task: "Implement refresh-token rotation without breaking existing sessions"
      action: "Added atomic token rotation in auth/middleware.ts with backward-compatible migration"
      result: "Zero silent session expiry incidents; all sessions migrated transparently"
---

# Journal — 2026-04-15

## Headline
...
```

## Enriched accomplishment fields

Each accomplishment in the journal frontmatter carries optional resume enrichment fields that power the `resume-builder` agent:

| Field | Type | Purpose |
|-------|------|---------|
| `qualityTier` | `high \| medium \| low` | Prioritization for resume inclusion |
| `techStack` | `string[]` | Technologies inferred from file extensions, commands, categories |
| `scope` | `{ filesChanged, servicesAffected, endpointsBuilt }` | Numeric breadth indicators for proxy metrics |
| `context` | `{ situation, challenge }` | The "why" — problem context and difficulty |
| `impact` | `{ metric, type }` | Candidate metric sentence and category |
| `starComponents` | `{ situation, task, action, result }` | Pre-decomposed STAR for XYZ formula input |

All fields are optional. The analyzer populates them when session data supports it; omits rather than hallucinating.

## Resume generation

The resume pipeline is a **read-only consumer** of the journal pipeline's output. It never modifies journal files.

### How it works

1. **`/career:resume 2026-03-01..2026-04-15`** invokes the `resume-builder` agent with the date range.
2. The agent reads all `.md` and `.user.md` journal files in range.
3. **Existing resume** (optional): If `--resume` is provided, the builder parses it to extract existing bullets, skills, structure, and formatting. New journal-derived content **merges into** the existing resume rather than replacing it.
4. **Clustering**: Related accomplishments across days are grouped (same git branch > file overlap > category proximity > semantic similarity). A feature spanning 5 sessions becomes 1 bullet.
5. **XYZ elevation**: Each cluster is transformed using Google's XYZ formula ("Accomplished [X] as measured by [Y], by doing [Z]"). Falls back to CAR (Challenge-Action-Result) when metrics are unavailable.
6. **Scoring**: Each bullet is scored 1-5 using the "So What?" test — does a hiring manager care?
7. **JD tailoring** (optional): If `--jd` is provided, bullets are scored against the job description, ranked by relevance, and terminology mismatches are flagged. Coverage gaps are identified.
8. Output is written to `<resumes_dir>/` (default: sibling of journals; configurable via `resumes:` in `~/.career/config`).

### Why the resume-builder reads `.user.md`

The orchestrator never reads `.user.md` to prevent feedback loops in journal generation. The resume-builder does read them because:
- It's a terminal consumer — no feedback loop risk.
- User corrections to AI-generated bullets should be honored.
- Off-screen accomplishments (meetings, mentoring, PR reviews) belong on a resume.

### Resume output

The resume draft contains:
- **Professional Summary** — 2-3 sentences synthesized from top bullets
- **Key Achievements** — top 3-5 cross-project highlights
- **Experience** — grouped by project with elevated bullets + derivation provenance
- **Technical Skills** — grouped by category (languages, frameworks, infra, databases, tools)
- **JD Analysis** (if JD provided) — covered/uncovered requirements, terminology suggestions

Each elevated bullet links back to the original daily bullets it was derived from, so you can verify and edit.

## Idempotency

Running `/career:journal` twice for the same date is safe. The orchestrator reads the existing file, deduplicates accomplishments by `(project, text)` exact match, merges evidence additively, and regenerates the body from the merged frontmatter. Counts are recomputed from merged data, not summed.

## Git repo (journals directory)

If the journals directory is a git repo, the orchestrator:

- Writes and regenerates `.md` files as normal
- **Does not** run `git add`, `commit`, `push`, or any other git state change
- Reports repo status (`clean` / `modified (N files)`) in its status block so you can commit on your own terms

You own when your journal history gets recorded.

## Constraints & guarantees

- The orchestrator **never reads `.user.md` files** and **never modifies git state** in the journals repo.
- The analyzer **never reads whole JSONL files into context** — it streams via `Bash + python3`.
- Accomplishment text is authored once by the analyzer and flows through unchanged during journal aggregation. The resume-builder may elevate bullets using XYZ formula but always preserves the originals as `derivedFrom`.
- The resume-builder **reads `.user.md` files** (it's a terminal consumer, no feedback loop) but **never writes to journal files**.
- Resume output goes to `<resumes_dir>/` (default: sibling of journals), separate from daily journals so drafts don't churn inside a versioned journals git repo.

## Troubleshooting

- **"Project count exceeds maxAnalyzers"** — the orchestrator is refusing to fan out to too many projects. Narrow the range, pass a project filter, or raise `maxAnalyzers` in config.
- **A project shows up as skipped with reason `exceeded turn budget`** — the analyzer ran out of its 20-turn budget. Usually means the project has an unusual amount of session data in range. Consider analyzing that project in isolation with a shorter range.
- **Categories are drifting** — check the `newCategories:` field on recent journals. Either the work genuinely covers new territory, or you can hand-edit the frontmatter to collapse synonyms before the next run; the orchestrator will pick up your fix on the next run's `knownCategories` load.

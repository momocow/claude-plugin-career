---
name: journal
description: Generate (or regenerate) the AI daily journal by invoking the journal-orchestrator subagent for a given date or range. Defaults to today. Accepts today, yesterday, YYYY-MM-DD, or YYYY-MM-DD..YYYY-MM-DD.
user-invocable: true
---

# Journal — generate the AI daily journal

Parse the user's time-range argument, then delegate to the `journal-orchestrator` subagent. This skill is a thin wrapper — all real work happens in the orchestrator.

## Accepted arguments

| Form                           | Meaning                                                     |
| ------------------------------ | ----------------------------------------------------------- |
| (none)                         | Today, in the user's local timezone                         |
| `today`                        | Today                                                       |
| `yesterday`                    | Yesterday (the recommended form for scheduled/cron runs)    |
| `YYYY-MM-DD`                   | That single date                                            |
| `YYYY-MM-DD..YYYY-MM-DD`       | Inclusive range; use for weekly/monthly summaries           |

## Procedure

1. **Parse the argument.**
   - If none, treat as `today`.
   - **Resolve the timezone first** using this chain (highest precedence first):
     1. `timezone:` field in `~/.career/config` (a YAML key, optional). Read with a `Bash` one-liner — `python3 -c "import yaml,os; print(yaml.safe_load(open(os.path.expanduser('~/.career/config'))).get('timezone',''))"` — and trim whitespace.
     2. The system's local timezone (e.g., from `$TZ` or the OS — whatever Python's `datetime.now().astimezone()` resolves to).
   - Resolve `today`/`yesterday` against that timezone — not against UTC, and not against any container's environment timezone.
   - Reject future dates, inverted ranges (`end < start`), and malformed input — ask the user to retry rather than guessing.

2. **Compute start/end as ISO-8601 timestamps.**
   - Single date → `[YYYY-MM-DDT00:00:00<resolved-offset>, YYYY-MM-DDT23:59:59<resolved-offset>]` using the timezone resolved in step 1.
   - Range → `[start-date T00:00:00, end-date T23:59:59]` in that timezone.
   - Convert to UTC before passing to the orchestrator — the orchestrator operates on UTC and does not re-read `timezone:` itself.

3. **Invoke `journal-orchestrator`** via the Agent tool with a prompt containing:
   - The computed `start` and `end` ISO-8601 timestamps
   - No project filter (unless the user passed one as extra args — forward it if so)
   - No output-destination override (let the orchestrator resolve via `~/.career/config`)

4. **Relay the orchestrator's status block** back to the user verbatim or lightly formatted. Do not paraphrase counts or drop fields — the status block is the user's audit trail for what ran.

## Constraints

- **Don't read JSONL, project directories, or journal files yourself.** This skill's entire job is argument parsing and delegation. All filesystem work happens inside the orchestrator or the analyzers.
- **Don't invent defaults beyond `today`.** If the user's input is ambiguous, ask.
- **Non-interactive runs** (e.g., when fired by `/schedule`) must not hang on prompts. If the orchestrator hits its `maxAnalyzers` warn-and-confirm threshold and there's no user to confirm, surface the error cleanly so the schedule system logs it — do not auto-proceed.
- **No side effects of your own.** No git ops, no writes outside of what the orchestrator does.

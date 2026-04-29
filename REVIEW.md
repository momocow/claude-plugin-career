# Career Plugin Review: Ambiguities & Optimization Opportunities

> Ported from another session. Decide on each item, then mark Status and apply.

Status legend: ⬜ open · ✅ accepted · ❌ rejected · 🟡 deferred

---

## Critical Ambiguities

### 1. Duplicate step numbering in journal-analyzer ⬜
**Priority:** P0
**Where:** `agents/journal-analyzer.md:125` and `:134`

Both are numbered "Step 8":
- Step 8: "Enrich evidence with git commits"
- Step 8: "Derive categories"

The second should be step 9. An LLM executing this agent could skip or conflate the two.

**Proposed fix:** Renumber the second step 8 → step 9 (and shift any subsequent numbering).

**Decision:**
**Notes:**

---

### 2. Contradictory category specificity guidance between analyzer and orchestrator ⬜
**Priority:** P1
**Where:** `agents/journal-analyzer.md:140` vs `agents/journal-orchestrator.md:135`

Analyzer says: *"Prefer specific over generic: react-hooks beats frontend"*
Orchestrator says: *"Sub-topic absorbed by a broader existing slug — minted react-hooks while react exists → replace with react"*

These push in opposite directions. The analyzer mints `react-hooks` (told to be specific), then the orchestrator collapses it into `react` (told consistency wins). On day 1 with empty `knownCategories`, both `react` and `react-hooks` might be minted by different projects. On day 2, normalization is path-dependent — behavior depends on which slugs landed in `knownCategories` first.

**Proposed fix:** Unify with a decision tree. Example: "First run (empty knownCategories): mint specific slugs freely. Subsequent runs: prefer existing slug unless >50% of work under that slug is genuinely in a sub-domain not covered by the broader term."

**Decision:**
**Notes:**

---

### 3. STAR omission is all-or-nothing — wastes partial structure ⬜
**Priority:** P1
**Where:** `agents/journal-analyzer.md:112`

> "If any single component cannot be extracted with reasonable confidence, omit the entire starComponents object"

An accomplishment with strong situation, task, and action but no clear result loses all three. The resume-builder's XYZ elevation could still use partial STAR (situation → context for [Z], action → [Z], result → [X]). Resume-builder already handles graceful degradation, so partial STAR provides strictly more signal than no STAR.

**Proposed fix:** Allow partial STAR. Add: "Emit whichever STAR components can be extracted confidently. Omit individual fields that can't, rather than the entire object. The resume-builder treats each component independently."

**Decision:**
**Notes:**

---

### 4. Ordering conflict when both score and jdMatchScore exist ⬜
**Priority:** P1
**Where:** `agents/resume-builder.md` step 6c vs step 7b

- Step 7b: bullets are "ordered by score descending"
- Step 6c: "Rank bullets by JD-match score (descending). Top-matching bullets should appear first in the output."

A bullet scoring 5 (quality) but 0.2 (JD-match) competes with one scoring 3 but 0.9 JD-match. Which wins?

**Proposed fix:** Define explicitly: "When JD is provided, sort by `0.6 * normalizedScore + 0.4 * jdMatchScore` descending. Without JD, sort by score alone."

**Decision:**
**Notes:**

---

## Moderate Ambiguities

### 5. Enrichment merge on re-run: existing wins or new wins? ⬜
**Priority:** P2
**Where:** `agents/journal-orchestrator.md:256`

> "prefer the newer context/impact/starComponents if the existing entry lacks them"

What if the existing entry has them and the re-run produces a different (potentially better) version? Spec is silent. Matters when the analyzer is updated or the JSONL is enriched between runs.

**Proposed fix:** "If the existing entry already has the field, keep the existing version. To force re-enrichment, delete the journal file and regenerate."

**Decision:**
**Notes:**

---

### 6. awk mentioned but not in permissions ⬜
**Priority:** P0
**Where:** `agents/journal-orchestrator.md:63`, `settings.json`

> "a Bash one-liner with awk or python3 -c is enough"

`settings.json` only permits `Bash(python3 *)`, not `Bash(awk *)`. In background execution, an awk call would trigger a permission prompt that can't be answered, silently failing.

**Proposed fix:** Either add `Bash(awk *)` to settings.json, or remove awk references and standardize on `python3 -c` (more consistent and portable).

**Decision:**
**Notes:**

---

### 7. Timezone resolution chain is implicit ⬜
**Priority:** P2
**Where:** `skills/journal/SKILL.md:25`, `agents/journal-orchestrator.md:20`, `~/.career/config`

- Skill says "system local date"
- Orchestrator says "user's local timezone"
- Config has optional `timezone:` field (commented out)

Resolution order — config `timezone:` → system timezone → ??? — is never stated. If the system runs in a container with UTC but the user is in Asia/Taipei, journals land on the wrong date.

**Proposed fix:** Document the chain in the orchestrator: "Use timezone from `~/.career/config` if set. Otherwise fall back to the system's local timezone. The skill is responsible for converting to UTC before passing to the orchestrator."

**Decision:**
**Notes:**

---

### 8. Semantic overlap detection is undefined algorithmically ⬜
**Priority:** P2
**Where:** `agents/resume-builder.md:94`

> "flag near-duplicates across existing-resume and ai sources using semantic comparison"

No algorithm specified — LLM judgment per pair. For 50+ accomplishments + 20 existing resume bullets, that's O(n*m) comparisons with non-deterministic results.

**Proposed fix:** Either accept non-determinism and document it, or add a heuristic: "Compare using keyword overlap (>60% shared non-stopword tokens) as a pre-filter, then apply semantic judgment only to pre-filtered pairs."

**Decision:**
**Notes:**

---

## Optimization Opportunities

### 9. Orchestrator model choice ⬜
**Priority:** P3
**Where:** `agents/journal-orchestrator.md` frontmatter

Orchestrator uses `model: opus`, but ~70% of work is mechanical (glob, bash, parse JSON, union arrays, write YAML). Judgment-intensive steps are category normalization (step 5) and narrative synthesis (step 7).

**Proposed fix:** Consider whether `sonnet` could handle this — or document why `opus` is required.

**Decision:**
**Notes:**

---

### 10. Jaccard threshold is a magic number ⬜
**Priority:** P3
**Where:** `agents/resume-builder.md:111`

Jaccard similarity > 0.3, with no justification or sensitivity analysis. Too high → multi-file features won't cluster. Too low → unrelated work on shared utility files clusters spuriously.

**Proposed fix:** "0.3 balances false positives from shared utility files against false negatives from sprawling features. Adjust upward (0.4–0.5) for monorepos with many shared files."

**Decision:**
**Notes:**

---

### 11. Resume-builder 30-turn budget for large ranges ⬜
**Priority:** P3
**Where:** `agents/resume-builder.md:402`

30 turns to load journals, cluster, elevate, score, optionally tailor to JD, and structure output. For 90 days / 100+ accomplishments, this is tight. Current mitigation ("prioritize high-tier and cluster aggressively") is vague.

**Proposed fix:** "Turns 1-5: load and parse. Turns 6-15: cluster. Turns 16-25: elevate and score. Turns 26-30: structure and emit. For ranges >60 days, skip per-bullet STAR decomposition for low-tier bullets and aggregate them into summary lines."

**Decision:**
**Notes:**

---

### 12. Atomic write pattern vs. actual tool behavior ⬜
**Priority:** P3
**Where:** Multiple specs

Specs mention "write atomically (.tmp then rename)" but agents use the `Write` tool, which doesn't support atomic writes. Would need `Write` → `Bash(mv ...)` — easy to forget. This pattern adds complexity without preventing real data loss (Write tool succeeds or fails; partial writes are rare).

**Proposed fix:** Either drop the atomic-write requirement (Write is reliable enough) or add explicit note: "Use Write to create `<path>.tmp`, then Bash with `mv <path>.tmp <path>` to rename."

**Decision:**
**Notes:**

---

### 13. Plugin settings.json could add `Bash(head *)` and `Bash(wc *)` ⬜
**Priority:** P3
**Where:** `settings.json`

Common one-liners for quick file inspection. Currently they'd trigger permission prompts in background mode.

**Proposed fix:** Add `Bash(head *)` and `Bash(wc *)` to permissions.

**Decision:**
**Notes:**

---

## Design Strengths Worth Preserving

- File ownership separation (`.md` vs `.user.md`) is clean and well-enforced across all agents.
- Wave-based fan-out with incremental aggregation is the right architecture for scaling to many projects.
- Enrichment fields being optional with "omit rather than hallucinate" is the correct default.
- The journal as intermediate representation between session analysis and resume generation is a good decoupling point.
- Cross-wave category learning (later waves see earlier waves' established slugs) is a clever detail that reduces drift.

---

## Summary Table (Priority Order)

| # | Priority | Issue | Status |
|---|----------|-------|--------|
| 1 | P0 | Duplicate step numbering | ⬜ |
| 6 | P0 | Missing awk permission | ⬜ |
| 2 | P1 | Category specificity contradiction | ⬜ |
| 3 | P1 | STAR all-or-nothing | ⬜ |
| 4 | P1 | Score vs. JD-match ranking conflict | ⬜ |
| 5 | P2 | Enrichment merge semantics | ⬜ |
| 7 | P2 | Timezone resolution chain | ⬜ |
| 8 | P2 | Semantic overlap algorithm | ⬜ |
| 9 | P3 | Orchestrator model choice | ⬜ |
| 10 | P3 | Jaccard threshold rationale | ⬜ |
| 11 | P3 | Resume-builder budget allocation | ⬜ |
| 12 | P3 | Atomic write pattern | ⬜ |
| 13 | P3 | Add `head`/`wc` to permissions | ⬜ |

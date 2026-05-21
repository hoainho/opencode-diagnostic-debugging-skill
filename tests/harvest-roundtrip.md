# Stage A1 — Harvest Pipeline Round-Trip Verification

> **Purpose**: Verify that the `diagnostic-driven-debugging` skill's Phase 5 postmortems can actually round-trip through `omo-session-distiller` and surface in Phase 0 recall queries.
>
> **This is the highest-stakes test in the production-hardening plan.** If it fails, the skill's entire feedback loop is theater.

---

## Test Date
2026-05-21

## Initial Baseline

```
omo-session-distiller_stats →
  Tier 1 (atoms): 199 files, 0.6 MB
  Tier 2 (archives): 250 files, 3.0 MB
  Atoms by repo: react-debugger-extension 83, playsweeps-web 58, gear-pr-review 22, shared 15, ...
  nano-brain daemon: DOWN (recall uses filesystem grep)
```

## Test Procedure

1. Run `omo-session-distiller_harvest` to discover what it scans
2. Inspect memory dir structure to learn the storage schema
3. Write a synthetic sentinel **solution** file to `~/.config/opencode/memory/solutions/shared/`
4. Query recall for the sentinel phrase → **MISS** (recall searches atoms/, not solutions/)
5. Write a paired sentinel **atom** file to `~/.config/opencode/memory/atoms/shared/`
6. Query recall again → **HIT** ✅

## Critical Findings (rolled into A2-A4 + plan amendments)

### Finding 1 — Auto-harvester scans SESSIONS, not project files

`omo-session-distiller_harvest` operates on opencode session storage (Tier 2 archives derived from session transcripts). It does NOT scan project-local `.sisyphus/postmortems/*.md` files.

**Impact**: the skill's Phase 5 documentation says postmortems live at `.sisyphus/postmortems/YYYY-MM-DD-<slug>.md`. This path is correct for source-of-truth storage and git tracking, but those files are **invisible to the auto-harvester**.

**Required correction (rolls into A2-amendment or new A5)**: Phase 5 needs a documented **manual export step**:
- After writing the postmortem to `.sisyphus/postmortems/`, also write paired atom + solution files to `~/.config/opencode/memory/atoms/<repo>/` and `~/.config/opencode/memory/solutions/<repo>/`
- OR write the postmortem content into the active session conversation, which then auto-harvests on next distillation cycle

### Finding 2 — Memory has TWO stores, not one

- `~/.config/opencode/memory/atoms/<repo>/` — the **searchable index**. recall() greps these files.
- `~/.config/opencode/memory/solutions/<repo>/` — the **rich artifacts**. Atoms reference solutions via `related_solution:` field.

Solutions alone are NOT recall-able. They must be paired with an atom that contains the searchable keywords.

### Finding 3 — recall DOES return numeric scores

Earlier recall test on real query "singular attribution scaleo first-touch guard" returned:
- Hit 1: score 42
- Hit 2: score 8

This **contradicts the T5 review fix** (review-fix commit on PR #4) that removed numeric thresholds from the recall-first-checklist rubric. The T5 reviewer claimed "no numeric score field"; this was wrong. The score is present in the raw recall output.

**Required correction (rolls into A2 amendment)**: recall-first-checklist Step 4 should re-introduce score guidance, but tied to the actual score range (single-digit to ~50 in my observations, depending on tag/keyword match density), not the fabricated 0.0-1.0 range from the original draft.

### Finding 4 — nano-brain daemon down → degraded recall

The recall tool reports "nano-brain daemon down → recall uses filesystem grep". Semantic search is NOT available in this environment. recall is currently a keyword search.

**Impact on skill design**:
- Phase 0 queries must be keyword-rich (already in the checklist — good)
- Score thresholds are token-match counts, not semantic similarity
- Without the daemon, recall surfaces lots of weak matches (score: 5 on tag-only matches) — risk of false-positive Phase 0 hits

**Required correction (rolls into A2 amendment)**: Phase 0 Step 4 rubric should warn about low-score-but-tag-match noise when daemon is down. Add: "If recall reports 'nano-brain daemon down', double the relevance bar — score ≥ 10 minimum to consider Clearly Relevant."

### Finding 5 — `expand` is broken in this environment

`expand` calls returned `ERROR running expand.sh:` for both manually-written atoms AND real session-derived atoms. This is independent of our test (it's a tool-level bug).

**Impact**: the skill's Phase 0 Step 5 ("expand the atom for full archive context if Resolution is too brief") cannot rely on `expand`. Workaround: read the atom file directly via `read` tool, or read the linked archive via the `archive:` field.

**Required correction (rolls into A2 amendment)**: Phase 0 Step 5 should have a fallback: "If `expand` fails, read the atom file directly via the file path shown in the recall result."

### Finding 6 — Schema for searchable atoms

```yaml
---
type: atom
id: atom-YYYY-MM-DD-<shortlabel>
verified_at: YYYY-MM-DD
ticket: <jira-key or "N/A">
tags: [<keyword>, <keyword>, ...]
applies_to: [<repo-name>]
file: <primary file path>
related_solution: <solution-id-if-exists>
---

**Fact**: <one-paragraph problem statement, keyword-rich>
**Evidence**: <observable signal, file:line refs>
**Implication**: <numbered list of consequences>
**Related files**: <bullet list>
```

This is the format the auto-harvester produces from sessions, AND the format that recall can find for manually-written atoms.

**Required correction**: Phase 5 postmortem template (T7) should be updated to emit BOTH:
1. The current `.sisyphus/postmortems/YYYY-MM-DD-<slug>.md` (for git history + human-readable record)
2. A paired atom file at `~/.config/opencode/memory/atoms/<repo>/atom-YYYY-MM-DD-<slug>.md` (for harvester compatibility)
3. Optionally a richer solution file at `~/.config/opencode/memory/solutions/<repo>/solution-YYYY-MM-DD-<slug>.md` (for the deep-dive linked artifact)

The postmortem template should provide an explicit "Export to memory" subsection with the exact commands to copy/transform the postmortem into atom + solution format.

---

## Pass Criterion

**Plan A1 pass criterion**: recall query returns the sentinel atom + expand returns content matching the synthetic postmortem.

### Result

| Sub-criterion | Result |
|---|---|
| Recall returns sentinel atom for query `HARVEST_ROUNDTRIP_TEST_SENTINEL_2026_DIAGNOSTIC_SKILL` | ✅ PASS — 1 hit, top result |
| Recall returns sentinel via paired solution file alone (no atom) | ❌ FAIL — solutions/ are not greppable by recall |
| Expand returns full file content | ❌ FAIL — `expand` tool errors on ALL atoms (env issue, not our test) |
| Auto-harvester scans `.sisyphus/postmortems/*.md` | ❌ FAIL — auto-harvester is session-based, not file-based |

### Overall A1 verdict

**PARTIAL PASS.** The harvest pipeline IS viable, but ONLY via the manual atoms/ path. The plan's original assumption (file-based auto-harvest from `.sisyphus/postmortems/`) is wrong. The skill must be amended to document the manual export step OR shift to writing postmortems within the session conversation.

This is exactly the failure mode the plan's worst-case-recovery clause anticipated:
> "**Worst case**: distiller fundamentally incompatible → skill must drop Phase 0 entirely OR build a custom recall index."

We are NOT in worst case. We are in **Option 1 (path config mismatch)**: the skill can use the distiller's existing memory dir if it writes atoms in the correct format and location. No custom index needed.

## Required follow-up (rolls into A2)

The Stage A2 fast-path stale-check PR will be expanded to also cover the 6 findings above. Specifically, A2 will:

1. Add a **"Manual harvest export"** subsection to `prompts/postmortem-template.md` with the exact atom + solution file format and target paths
2. Revert the T5 "no numeric score field" wording in `references/recall-first-checklist.md` — replace with "scores are token-match counts when nano-brain daemon is down; score ≥ 10 = consider Clearly Relevant, score < 10 = treat as noise unless tag-confirmed"
3. Add a fallback in Phase 0 Step 5 for when `expand` fails
4. Update the existing T9 fast-path scenario in `tests/scenario-bank.md` to reflect these corrections
5. Add A1's findings as a citation in the skill's README

## Cleanup

After this PR merges and the A2 corrections land, delete the sentinel files:

```bash
rm ~/.config/opencode/memory/atoms/shared/atom-2026-05-21-HARVEST-ROUNDTRIP-SENTINEL.md
rm ~/.config/opencode/memory/solutions/shared/solution-2026-05-21-HARVEST-ROUNDTRIP-SENTINEL.md
```

These are diagnostic probes, not durable knowledge.

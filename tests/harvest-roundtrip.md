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

**Required correction (rolls into A2 amendment)**: Phase 0 Step 4 rubric should warn about low-score-but-tag-match noise when daemon is down. Provisional guidance based on a single query sample: "observed score range single-digit to ~50; scores <10 commonly represent tag-only matches with no semantic relevance — treat as noise unless tag-confirmed AND Problem text matches." This is **not** a calibrated cutoff — A2 should phrase it as an observation pending more samples, not a hard threshold rule. After 5+ real recall queries (in Stage B pilot), calibrate to a real range.

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

**FAIL on the documented path; viable alternative path identified.**

The skill's currently-documented Phase 5 mechanism (auto-harvest from `.sisyphus/postmortems/*.md`) is broken — auto-harvester is 100% session-driven, never touches project files. Of 4 pass-sub-criteria, only 1 passed (manual atoms/ recall hit). The 3 other failures are not environmental — they are **skill-design defects** (wrong harvest path documented, wrong storage model assumed, no fallback for `expand`). Only the `expand` env bug is non-design.

**Honest framing**: this is a FAIL with an actionable workaround. Calling it "PARTIAL PASS" would be the kind of softening euphemism the plan's own discipline forbids. The skill IS recoverable via A2 amendments — but recoverable from a FAIL state, not from a partial-success state.

This is exactly the failure mode the plan's worst-case-recovery clause anticipated:
> "**Worst case**: distiller fundamentally incompatible → skill must drop Phase 0 entirely OR build a custom recall index."

We are NOT in worst case. We are in **Option 1 (path config mismatch)**: the skill can use the distiller's existing memory dir if it writes atoms in the correct format and location. No custom index needed.

## Required follow-up

Reviewer correctly flagged that bundling A1's findings into A2 (originally just "fast-path stale-check") would scope-creep A2. Split into two stages:

### A2 (stays narrow, original scope)

Original mandate: fix recall fast-path stale-check (Step 2.5 verify regression test exists+passes). One file edited: `references/recall-first-checklist.md`. Scenario T9 updated in same PR (already in original plan). **No scope change.**

### A5 (NEW stage — added by plan amendment after A1)

Covers the 5 corrective items from A1's findings:

1. Add **"Manual harvest export"** subsection to `prompts/postmortem-template.md` with the exact atom + solution file format + canonical target paths under `~/.config/opencode/memory/`
2. Revert the T5 "no numeric score field" wording in `references/recall-first-checklist.md` — replace with the **observation-not-rule** phrasing above (range observed, daemon-down caveat, calibration pending Stage B)
3. Add a fallback in Phase 0 Step 5 for when `expand` fails (read atom file directly via path)
4. Update the existing T9 fast-path scenario in `tests/scenario-bank.md` to reflect these corrections
5. Add A1's findings as a citation in the skill's README

A5 ships as its own PR titled `a5-harvest-pipeline-corrections-from-a1`. This keeps PR titles meaningful and reverting T5 happens in a clearly-titled commit reviewer can trace.

### Reordering implication

The plan's Stage A gate requires A1-A4 done before Stage B. A5 is added to Stage A (becomes A1-A5 done before Stage B). Plan harness file updated to reflect this in a separate plan-amendment PR.

## Cleanup

**Sentinels deleted IMMEDIATELY after this PR's evidence captured** — NOT after A5 merges. Reviewer correctly noted that holding them in `memory/atoms/shared/` pollutes recall for any unrelated query containing "HARVEST" or "SENTINEL" tokens during the A5 review window. A5 will be authored against the schema documented in Finding 6, not against the live sentinel files.

Deletion command (executed at the time of this PR's commit):

```bash
rm ~/.config/opencode/memory/atoms/shared/atom-2026-05-21-HARVEST-ROUNDTRIP-SENTINEL.md
rm ~/.config/opencode/memory/solutions/shared/solution-2026-05-21-HARVEST-ROUNDTRIP-SENTINEL.md
```

After deletion, recall queries for the sentinel phrase will return 0 hits — that's the expected post-cleanup state. The findings above stand on the captured evidence in this document.

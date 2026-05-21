# Stage C3 — Final Status Declaration

> **Plan reference**: [.opencode/plans/2026-05-21-skill-production-hardening.md](../../.opencode/plans/2026-05-21-skill-production-hardening.md) Stage C3.
>
> **Pass criterion** (from plan): status declared in writing per machine-checkable decision rule; README updated; recommendation for next iteration if not Production-ready.
>
> **Date**: 2026-05-21.

---

## Decision rule (from plan, verbatim)

> - **Production-ready** iff: B3 ≥ 80% harvest hit rate AND B4 second-bug Phase 0 hit confirmed AND C1 metrics show net positive time-savings (any percentage > 0)
> - **Partial** iff: harvest works (≥ 80%) BUT Phase 0 second-hit unconfirmed within timebox OR metrics inconclusive
> - **Not-ready** iff: harvest works < 80% OR no second-bug Phase 0 hit possible OR metrics show net negative

---

## Mechanical application of the rule

### Input 1: B3 ≥ 80% harvest hit rate

**TRUE.** Per [B3 PR #22](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/22): 4/4 = 100% hit rate at plan B3 timing; later confirmed 5/5 = 100% in C1's re-measurement (PR #24) with all 5 atoms reaching top-rank under plain-language queries.

### Input 2: B4 second-bug Phase 0 hit confirmed

**TRUE.** Per [B4 PR #23](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/23): WIN-7906 (Daily Bonus reveal-then-reload) query surfaced B2.1 atom at top rank, score 38, with a 5-point gap to next. Wall-clock 54s vs B2.1's 148s = 36%, under the 50% threshold the plan specified.

### Input 3: C1 metrics show net positive time-savings (any percentage > 0)

**INCONCLUSIVE.** This is the pivot.

Per [C1 PR #24](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/24): wall-clock data exists (54-319 seconds across 5 sessions) BUT no measured baseline exists — only the author's gut-estimates per pilot. Per [C2 PR #25](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/25)'s verdict: "Speculation-vs-speculation produces nothing measurable."

The plan's literal phrasing "any percentage > 0" implies MEASURED percentage. Gut-estimates don't satisfy this. The C2 reviewer's COMMENT 4 confirms: "Defending PRODUCTION-READY would require treating gut-estimates as measurements — exactly what §'Why this verdict is conservative' argues against. PARTIAL is the only verdict consistent with the document."

### Branching logic

- **PRODUCTION-READY** requires ALL THREE inputs TRUE. Input 3 is INCONCLUSIVE → condition fails.
- **PARTIAL** requires: harvest works (≥ 80%) — TRUE — AND (Phase 0 second-hit unconfirmed within timebox OR metrics inconclusive). The second clause is OR-gated; "metrics inconclusive" matches Input 3 exactly.
- **NOT-READY** requires harvest < 80% OR no second-bug hit OR net NEGATIVE metrics. None match — harvest is 100%, second-bug confirmed, metrics aren't negative (just unmeasured).

**Verdict: PARTIAL.**

---

## Formal status declaration

**`diagnostic-driven-debugging` skill status as of 2026-05-21: PARTIAL.**

The skill is **mechanically verified for the recall pipeline** (5/5 atoms harvested + reachable + top-ranked, score gaps ≥20, bug-cluster clustering observed) but **the wall-clock speedup claim remains a hypothesis pending controlled measurement** (no parallel-track no-skill baseline timing has been done).

The skill is **safe to use** for diagnostic work, with the understanding that quantified "Nx faster" claims are not yet supported. The recall loop demonstrably works end-to-end on real-bug postmortems and surfaces relevant atoms for both same-bug retrieval and related-bug investigation. What it has NOT demonstrated is a controlled-experiment speedup figure.

---

## What "PARTIAL" means in practice

### What you CAN say about the skill (with citations):
- "5/5 manual harvest export pipeline success" (C1)
- "5/5 Phase 0 recall hits at top rank with plain-language symptom queries" (C1, B3, B4)
- "Score gap ≥20 in all cases under daemon-down filesystem-grep" (C1)
- "Bug-cluster grouping observed in 3/5 cases (related pilot atoms outrank off-topic noise)" (C1)
- "Skill completes Phase 0-3 within bounded wall-clock (54-319s across 5 sessions)" (C1, descriptive only)
- "Recall on second-similar-bug surfaces the original pilot atom at top rank with a 36% wall-clock vs the original investigation (n=1)" (B4)

### What you CANNOT say about the skill (yet):
- "Skill makes debugging X% faster" — NOT measured
- "Skill matches the proposal's 60-75% time-saved claim" — NOT measured (un-contradicted but un-confirmed per C2)
- "Skill scales to N pilots" — sample size is 5, no scaling evidence
- "Skill works under daemon-up semantic recall" — only filesystem-grep mode tested

### Operating recommendations for current users:
1. **DO use the skill** for diagnostic work. The recall + structured Phase 0-5 protocol produces actionable hypothesis matrices even when falsification requires tools the controller doesn't have access to.
2. **DO follow the A5 Manual Harvest Export procedure** at Phase 5 to ensure postmortems become recall-able atoms. Verified working 5/5 times in this pilot.
3. **DO NOT cite the proposal's "60-75%" figure** as a measured outcome. It remains author speculation.
4. **DO NOT skip Phase 5 postmortems** to "save time" — the compounding recall loop is the killer feature, and it only works if atoms accumulate.

---

## Next iteration — closing the PARTIAL → PRODUCTION-READY gap

To upgrade the skill from PARTIAL to PRODUCTION-READY, the missing input is **C1's "net positive time-savings (any percentage > 0)" with measured baseline**. Two paths:

### Path A: Parallel-track controlled experiment
- Pick 3-5 fresh bugs (not from this pilot's archetypes)
- Have 2 engineers (one using the skill, one not) attempt the diagnostic independently
- Measure wall-clock for each; compare directly
- Cost: 4-8 hours of engineer pair time; high quality of evidence
- Cleanest path to a defensible speedup number

### Path B: Lighter retrospective comparison
- Pick 5 already-fixed bugs from Jira (already-merged PRs with timestamps)
- Have the skill's controller attempt the diagnostic blind (without seeing the merged fix)
- Compare time-to-hypothesis vs original time-to-merge from Jira ticket history
- Cost: 1-2 hours; medium quality (Jira time-to-merge includes review + deploy, not just diagnostic)
- Defensible enough to upgrade PARTIAL → PRODUCTION-READY with caveats

### Path C: Defer
- Accept PARTIAL as the long-term status
- Update messaging to reflect "field-validated for recall pipeline; speedup not measured"
- Revisit if the skill gets enough field use that organic timing data accumulates

### Recommendation
Path B is the lowest-cost defensible upgrade. Path A is the gold standard. Path C is the honest status quo until either is performed.

---

## Plan status

Per the production-hardening plan:
- **Stage A1-A5**: Foundation complete (PRs #12-16)
- **Stage B1-B4**: Pilot + harvest + 2nd-similar-bug verification complete (PRs #17-23)
- **Stage C1-C2**: Metrics + proposal-comparison complete (PRs #24-25)
- **Stage C3 (this PR)**: Status declared PARTIAL

**Plan harness final state**: `🟢 done` (gate-checked: all 14 task PRs merged).

The plan's job is complete: the skill went from "logically-complete + review-validated" (where it started) to "field-validated for the recall pipeline; speedup pending measurement" (PARTIAL). The plan EXPLICITLY allowed for this outcome — quoting from the plan's failure-mode column for C3: "if metrics are inconclusive → declare so; do NOT manufacture conclusions." That's what this PR does.

---

## README update (added in this PR)

The README's "Effectiveness" section (added in C2 PR #25) is updated from "Status (pending Stage C3 declaration)" → "Status: PARTIAL (declared 2026-05-21 in C3)". Specific text replacement applied:

Before:
> **Status (pending Stage C3 declaration)**: skill is mechanically verified for the recall pipeline. Speedup remains a hypothesis pending controlled measurement (parallel-track experiment with no-skill baseline timing — not yet done).

After:
> **Status (declared 2026-05-21 in C3): PARTIAL.** Skill is mechanically verified for the recall pipeline (5/5 atoms harvested + 5/5 Phase 0 hit rate + score gaps ≥20). Wall-clock speedup remains a hypothesis pending controlled measurement; the original proposal's 60-75% claim is un-contradicted but un-confirmed. See [tests/pilot-2026-05/102-status-declaration.md](./tests/pilot-2026-05/102-status-declaration.md) for the full decision-rule application and the PARTIAL → PRODUCTION-READY upgrade paths.

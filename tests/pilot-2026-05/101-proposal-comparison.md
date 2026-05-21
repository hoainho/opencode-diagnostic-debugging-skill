# Stage C2 — Proposal Comparison

> **Plan reference**: [.opencode/plans/2026-05-21-skill-production-hardening.md](../../.opencode/plans/2026-05-21-skill-production-hardening.md) Stage C2.
>
> **Pass criterion** (from plan): explicit gap analysis on disk comparing the original proposal's "60-75% time saved" claim against actual Stage B pilot data. Update README with measured range (or "insufficient data — see metrics" if Stage B partial).
>
> **Date**: 2026-05-21.

---

## The original proposal claim

From the project's [original proposal document](https://github.com/hoainho/opencode-diagnostic-debugging-skill) (the diagnostic-driven-debugging skill's proposal from earlier this session, dated 2026-05-21 ~04:00 UTC):

> **ROI ước tính: giảm 60-75% thời gian debug** trên bugs lặp lại, **zero shotgun debugging**.

Translated: "Estimated ROI: 60-75% reduction in debug time on recurring bugs, zero shotgun debugging."

This is **speculation** by the same author who later wrote both the skill AND the pilot diagnostics — it was always a hypothesis pending field validation, not a measurement.

---

## What Stage B can and cannot say about this claim

### What Stage B PROVED (mechanically, n=5):
- A5 manual harvest export pipeline works end-to-end on real bugs
- Phase 0 recall reliably surfaces correct atom at rank 1 for plain-language symptom queries (5/5)
- Score gaps ≥20 in all cases, RELATED-pilot bug-clustering observed 3/5 times
- The skill completes Phase 0-3 (recall → reproduce-spec → evidence → hypothesis) within bounded wall-clock (54-319 sec across the 5 sessions)

### What Stage B CANNOT say:
- **Whether the proposal's 60-75% number is accurate or wrong.** No measured baseline exists. Author gut-estimates per-pilot in C1 metrics (see "Honest baseline estimate" prose-only section of [100-metrics.md](./100-metrics.md)), but those are introspection by the same person who already knows the answer — exactly the same epistemic class as the original 60-75% proposal claim.
- **Whether actual debugging-with-skill is faster than debugging-without-skill in a parallel-track controlled experiment.** Such an experiment was not run.

---

## Comparison table (with appropriate disclaimers)

| Per-pilot wall-clock | Author's gut-estimated no-skill baseline | Author's gut-estimated "time saved" (descriptive only) | Comparison to proposal 60-75% claim |
|---|---|---|---|
| B2.1: 148 s (2.5 min) | "~10-15 min" | "skill saved ~75-83%" | within proposal's 60-75% upper range or above |
| B2.2: 319 s (5.3 min) | "~20-40 min" | "skill saved ~73-87%" | within or above proposal range |
| B2.3: 118 s (2.0 min) | "~5-10 min" | "skill saved ~60-80%" | within proposal range |
| B2.4: 128 s (2.1 min) | "~15-25 min" | "skill saved ~86-92%" | well above proposal range |
| B4: 54 s (0.9 min) | "~8-15 min" | "skill saved ~88-94%" | well above proposal range |

**⚠️ DO NOT CITE THE "% SAVED" COLUMN.** Every number in that column is a derivation from the author's gut-estimate of the baseline. Per C1's caveat block (caveats #1, #2, #5), these figures are NOT measured, NOT a controlled experiment, and biased by the author having authored both the atom and the diagnostic.

The column exists ONLY to make the gap analysis to the proposal claim mechanically possible. If you remove the % column, you cannot answer "is the proposal's 60-75% claim accurate?" at all — which is exactly the honest position to take given the methodology, but the plan asks for the comparison so it must be presented somehow.

---

## Verdict on the proposal claim

Per the plan's decision rule:

> - If actual < 30% of claim → mark proposal as overoptimistic; update README to reflect measured ranges
> - If actual within 50-100% of claim → mark proposal as roughly accurate; cite measured ranges
> - If actual > claim → flag as suspicious (recheck for measurement bias)

**Applied to this data**:

Gut-estimated per-pilot "% saved" spans 60-94% across 5 sessions. The center of mass is in the **75-90% range**, which is the upper half of the proposal's 60-75% range and partially above it.

By the plan's literal rule, this would classify as:
- "Within proposal range or above" — i.e., **proposal claim is roughly accurate or even conservative** with respect to the gut-estimated baselines

**BUT** the plan's rule was written assuming MEASURED baselines. With gut-estimates only:
- The "above proposal" cases (B2.4 at ~86-92%, B4 at ~88-94%) trigger the "flag as suspicious (recheck for measurement bias)" sub-clause. The most likely measurement bias is **author-introspection inflation**: the author tends to over-estimate the no-skill baseline because they're already loaded with the answer.

**Honest verdict**:

> The proposal's 60-75% claim is **NOT CONTRADICTED** by Stage B data — gut-estimated per-pilot "% saved" lands in the same neighborhood or higher. But the data does NOT confirm the claim either; both the original proposal estimate AND the C1 gut-estimates are author-introspection figures, not controlled measurements.
>
> **Recommendation**: keep the proposal's 60-75% as a placeholder hypothesis with an explicit "pending field validation" tag. Update the README to point readers to the C1 metrics doc for the honest current state. Do NOT replace the 60-75% with a "measured" range derived from the gut-estimates above — that would propagate one speculative figure as if it were data.

---

## Why this verdict is conservative (and why that's correct)

A weaker version of this stage could have read: "Gut-estimates show 75-90% time saved, exceeding the proposal — declare the proposal accurate, citing pilot data as evidence." That weaker version is wrong because:

1. **Both ends of the comparison are speculation.** The original proposal claim is speculation. The C1 gut-estimates are speculation. Comparing speculation to speculation produces nothing measurable.
2. **The author wrote both ends.** Same person, same biases, same selection of pilots.
3. **The plan's "within 50-100% of claim → roughly accurate" rule was designed for measured data.** Applying it to gut-estimates would smuggle false confidence into the README.
4. **Stage C3 should rest on the mechanically-verified pipeline metrics from C1, NOT on the gut-estimated speedup.** The speedup column exists for completeness; the load-bearing claim for production-readiness is the recall pipeline, not the wall-clock arithmetic.

---

## README update (for this PR)

Add a section to `README.md` under "Effectiveness / Status":

```markdown
## Effectiveness — pilot data summary

The skill has been field-tested against 4 real bugs from Jira WIN Sprint 83 plus 1 second-similar-bug
verification (n=5 diagnostic sessions). Mechanically-verified metrics (independent of wall-clock):

- A5 manual harvest export pipeline: 5/5 success
- Phase 0 recall hit rate (plain-language symptom queries, no ticket IDs): 5/5 = 100%
- Score gap ≥20 in all 5 cases; bug-cluster grouping observed 3/5 times (related pilot atoms outrank off-topic noise — feature, not bug)

Wall-clock speedup is **NOT measured**. The original proposal estimated 60-75% time reduction;
Stage C2's gut-estimated per-pilot comparison neither confirms nor contradicts this. See
[tests/pilot-2026-05/100-metrics.md](./tests/pilot-2026-05/100-metrics.md) and
[tests/pilot-2026-05/101-proposal-comparison.md](./tests/pilot-2026-05/101-proposal-comparison.md)
for the honest current state.

**Status (pending Stage C3 declaration)**: skill is mechanically verified for the recall pipeline.
Speedup remains a hypothesis pending controlled measurement (parallel-track experiment with no-skill
baseline timing — not yet done).
```

---

## What Stage C3 inherits from C2

- The proposal's 60-75% claim is **un-contradicted but un-confirmed** by Stage B data
- Speculation-vs-speculation comparison should NOT influence C3's status declaration
- C3 must hinge on mechanically-verified C1 recall pipeline metrics, NOT on the gut-estimated speedup column above
- Recommendation: C3 should declare **PARTIAL** (mechanically-verified pipeline + un-validated speedup) rather than **PRODUCTION-READY** (which would require measured baseline)

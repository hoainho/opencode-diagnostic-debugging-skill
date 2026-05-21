# Stage C1 — Metrics Aggregation (pilot data)

> **Plan reference**: [.opencode/plans/2026-05-21-skill-production-hardening.md](../../.opencode/plans/2026-05-21-skill-production-hardening.md) Stage C1.
>
> **Pass criterion** (from plan): a metrics table on disk with REAL numbers; no fabricated entries; explicit "baseline is gut-estimate, not measured" disclaimer.
>
> **Date**: 2026-05-21.
> **Sample size**: 4 pilots (B2.1-B2.4) + 1 second-similar-bug (B4) = **n=5 total diagnostic sessions**.

---

## Methodology disclaimer (READ FIRST — do not skip)

**Every figure in this document is descriptive only.** Specifically:

1. **Baseline is gut-estimate by the author** (Sisyphus), not a measured control. The author authored both the atom AND the pilot diagnostic; "how long would this have taken without the skill" is introspection by someone who already knows the answer. Real baseline measurement requires a parallel-track experiment (e.g., timing a different engineer or a different agent without skill access on the same bugs) — not done here.
2. **Sample size n=5 is too small for inferential statistics.** No p-values, no confidence intervals, no "X% faster" claims that imply generalization.
3. **Heterogeneous archetypes**: 3 phase-3-partial + 1 trivial-confirmation + 1 second-similar-bug. Averaging across these strata would be misleading; stratified breakdown only.
4. **Daemon-down scoring throughout**: all recall numbers are filesystem-grep token-match counts because nano-brain was down in this environment. Daemon-up semantic scoring will use a different scale and threshold. None of these numbers transfer to daemon-up mode without re-calibration.
5. **Same-author bias for recall** (B4-specific): the author wrote both the atom and the second-similar-bug recall query. Keyword-vocabulary consistency between writer and querier inflates recall scores artificially.
6. **Write-up time excluded**: "diagnostic wall-clock" measures recall + reproduce-spec + evidence + hypothesis formation. It does NOT include writing the trace + postmortem (~10-12 min per pilot). Stage C measures skill-as-diagnostic-aid, not skill-as-document-generator.
7. **No production fix verification**: pilots ran Phase 0-3 only; Phase 4-5 fix-then-verify was skipped per Stage B contract. Whether the hypotheses are actually correct cannot be confirmed without backend assignees running falsification tests.

**With all 7 caveats noted, the data below is the most honest summary the pilot can produce.**

---

## Raw data — diagnostic wall-clocks (no write-up)

| Pilot | Bug | Archetype | Wall-clock start | Wall-clock end | Diagnostic seconds | Hypotheses | A5 recall score |
|---|---|---|---|---|---|---|---|
| B2.1 | WIN-7993 Daily Bonus auto-claim stuck | state-desync, single-repo, P0 | 08:07:28Z | 08:09:56Z | **148 s** | 3 (shared falsification) | 43 → 67 in B3 |
| B2.2 | WIN-7988 BI event LP source missing | data, cross-repo, P0 | 08:19:16Z | 08:24:35Z | **319 s** | 3 (A vs C surface overlap) | 39 → 38 in B3 |
| B2.3 | WIN-7990 Prize Summary Lootbox icon | state-desync, single-repo, P2 (trivial-confirmation) | 08:33:12Z | 08:35:10Z | **118 s** | 1 (trivial) | 47 → 46 in B3 |
| B2.4 | WIN-7518 MFA cooldown not resetting | race + security, cross-system Redis, P1 | 08:40:51Z | 08:42:59Z | **128 s** | 3 (3 Redis layers + config) | 38 → 49 in B3 |
| B4 | WIN-7906 DB reload reward summary | state-desync, second-similar to B2.1, P0 | 08:57:49Z | 08:58:43Z | **54 s** | 1 (atom-seeded) | 76 |

Note: "A5 recall score" first number is the score when the atom was queried with the **author's query at write-time** (lower bound); second number (where shown) is the B3-stage independent query the reviewer also ran (higher bound, because both query and atom contain user-symptom vocabulary by then).

---

## Stratified breakdown by archetype

### Stratum 1 — `phase-3-partial` (full investigation, atom did NOT exist at start)
Pilots: B2.1, B2.2, B2.4 (n=3).
- Wall-clock range: **128–319 seconds** (2.1–5.3 min)
- Median: 148 s (2.5 min)
- Mean: 198 s (3.3 min) — driven up by B2.2's cross-repo outlier
- Hypotheses per pilot: 3 each
- Outcomes: all "3 hypotheses formed + handed off to backend assignee for falsification"

### Stratum 2 — `trivial-confirmation` (overwhelming static evidence, 1 hypothesis)
Pilots: B2.3 (n=1).
- Wall-clock: **118 seconds** (2.0 min)
- Hypotheses: 1
- Outcome: single-symbol fix site identified; B1 stop-rule fired correctly (no swap to backup)

### Stratum 3 — `second-similar-bug` (atom existed; recall + atom-seeded compression)
Pilots: B4 (n=1).
- Wall-clock: **54 seconds** (0.9 min)
- vs related pilot B2.1: **54 / 148 = 36%** (well under plan's 50% threshold)
- Hypotheses: 1 (atom-seeded)
- Outcome: atom loaded the right code region; rg narrowed to exact line; hypothesis formed in ~30s

---

## Cross-stratum observations (descriptive only)

| Observation | Data |
|---|---|
| All-pilot diagnostic wall-clock range | 54 – 319 seconds |
| All-pilot median | 128 s (2.1 min) |
| Cross-repo penalty (B2.2 vs single-repo median) | 319 / 128 = **2.5× longer** for cross-repo bugs |
| Trivial-confirmation savings vs `phase-3-partial` median | 118 / 148 = 80% — meaningful but small (saturated, since both branches read the same files quickly once the area is known) |
| Second-similar-bug savings vs original pilot | 54 / 148 = **36%** (1 sample) — the strongest single signal that recall compounds, but DESCRIPTIVE ONLY |

---

## A5 recall pipeline metrics

| Metric | Value |
|---|---|
| Pilots harvested via A5 manual export | 5/5 (B2.1-B2.4 + B4) |
| Top-rank hit rate when queried with plain-language symptom | **5/5 = 100%** (B3 measured 4/4 = 100% on the original 4; B4 added the 5th case which also hit) |
| Median top score (across all 5 atoms, daemon-down filesystem-grep) | **46** |
| Median gap to next-best (signal isolation) | **28 points** |
| Off-topic rank-2 incidence | 5/5 — rank-2 was unrelated in every case (Apple Pay, reCAPTCHA, React Debugger, npm publish, etc.) |
| Schema compatibility | All 5 atoms use the A5 schema verified in Stage A1 round-trip test |

---

## Honest baseline estimate (gut-estimate, NOT measured)

If a developer (no skill, no atoms, no recall) approached each pilot bug fresh, the author's introspective estimate is:

| Pilot | No-skill baseline (gut) | Skill diagnostic | Implied compression | Confidence in baseline |
|---|---|---|---|---|
| B2.1 | 10-15 min | 2.5 min | ~16-25% (4-6× compression) | Low — pure introspection |
| B2.2 | 20-40 min (cross-repo) | 5.3 min | ~13-27% (4-7× compression) | Low |
| B2.3 | 5-10 min (trivial bug) | 2.0 min | ~20-40% (2.5-5× compression) | Medium — easy to bound |
| B2.4 | 15-25 min (Redis + AVA) | 2.1 min | ~8-14% (7-12× compression) | Low — unfamiliar domain inflates the estimate |
| B4 | 8-15 min (would need B2.1's context anyway) | 0.9 min | ~6-11% (9-17× compression) | Lower — already biased by knowing B2.1 |

**Implied range (gut-estimate)**: 6-27% of baseline time, or 4-17× compression depending on pilot.

**WHY YOU SHOULD NOT CITE THIS RANGE AS A SPEEDUP CLAIM**:
1. Baseline is the author's introspection, NOT a measured control
2. Author wrote the atoms, biasing recall outcomes
3. n=5 across 3 archetypes with different baselines — averaging is misleading
4. The "no-skill" alternative wasn't actually run; nobody timed an engineer ignoring the skill
5. The 60-75% original proposal claim (B2 review feedback) is in the same epistemic class — that's exactly why the proposal claim must be marked "speculation pending field validation"

---

## Recall-pipeline metrics (these ARE mechanically verified)

The following figures DO survive the n=5 caveat because they measure pipeline correctness, not user time:

| Metric | Value | What it proves |
|---|---|---|
| A5 round-trip success rate | 5/5 | Manual harvest export pipeline works on real postmortems |
| Phase 0 hit rate (B3 + B4 combined, plain-language queries) | 5/5 = 100% | Atoms are reachable via natural symptom phrases (no ticket IDs) |
| Score gap ≥20 points in all cases | 5/5 | Strong discrimination from off-topic noise under daemon-down filesystem-grep |
| Cross-pilot atom linkage (B4 surfaced both B2.1 AND B2.3) | 1 case observed | Recall captures bug-cluster relationships, not just single-atom mapping |
| Same-atom drift on re-query (B3 vs original write) | 0 | Recall results are stable across query rephrasings of the same symptom |

These ARE the load-bearing metrics for Stage C3's decision rule. The wall-clock figures are nice-to-have descriptive data; the recall-pipeline figures are the actual production-readiness signal.

---

## What this data lets Stage C3 conclude

**Affirmative**:
- A5 manual harvest export pipeline works correctly on real bugs (verified mechanically, 5/5)
- Phase 0 recall surfaces relevant atoms for both same-atom retrieval (B3) and related-bug recall (B4) under daemon-down scoring
- The skill produced actionable hypothesis matrices for 4/4 phase-3-partial pilots, even where falsification required tools the controller didn't have access to (QA env, Redis CLI, bi-event-worker repo)

**Negative / Inconclusive**:
- Wall-clock speedup is NOT measured (only gut-estimated)
- No production fix was applied; hypothesis correctness is unverified
- Daemon-down scoring; daemon-up generalization is unknown
- Same-author bias is uncorrected
- The skill's behavior on bugs WITHOUT prior atoms (truly novel domain) is sampled only by the 4 phase-3-partial pilots, all of which the author DID have prior domain knowledge of (playsweeps stack)

**For Stage C3**: the production-readiness decision should hinge on the mechanically-verified recall pipeline (positive) AND the absence of measured speedup (cautious framing). See Stage C3 doc for the formal decision per the plan's machine-checkable rule.

---

## Hands-off raw data (for future Stage C re-runs)

If a future evaluator wants to re-compute these figures with different baselines, the raw inputs are:

| Pilot | Trace doc | Postmortem | Atom | A5 score (at write) | A5 score (B3 reviewer re-run) |
|---|---|---|---|---|---|
| B2.1 | tests/pilot-2026-05/01-pilot-WIN-7993-trace.md | .sisyphus/postmortems/2026-05-21-pilot-1-WIN-7993-daily-bonus-auto-claim-stuck.md | atoms/playsweeps-web/atom-2026-05-21-pilot-1-WIN-7993-daily-bonus-auto-claim-stuck.md | 43 | 67 |
| B2.2 | tests/pilot-2026-05/02-pilot-WIN-7988-trace.md | .sisyphus/postmortems/2026-05-21-pilot-2-WIN-7988-bi-event-lp-source-missing.md | atoms/playsweeps-backend/atom-2026-05-21-pilot-2-WIN-7988-bi-event-lp-source-missing.md | 39 | 38 |
| B2.3 | tests/pilot-2026-05/03-pilot-WIN-7990-trace.md | .sisyphus/postmortems/2026-05-21-pilot-3-WIN-7990-prize-summary-lootbox-icon-always.md | atoms/playsweeps-web/atom-2026-05-21-pilot-3-WIN-7990-prize-summary-lootbox-icon-always.md | 47 | 46 |
| B2.4 | tests/pilot-2026-05/04-pilot-WIN-7518-trace.md | .sisyphus/postmortems/2026-05-21-pilot-4-WIN-7518-mfa-cooldown-not-resetting.md | atoms/playsweeps-backend/atom-2026-05-21-pilot-4-WIN-7518-mfa-cooldown-not-resetting.md | 38 | 49 |
| B4 | tests/pilot-2026-05/05-B4-second-similar-bug-WIN-7906.md | .sisyphus/postmortems/2026-05-21-pilot-5-WIN-7906-daily-bonus-reload-summary-not-open.md | atoms/playsweeps-web/atom-2026-05-21-pilot-5-WIN-7906-daily-bonus-reload-summary-not-open.md | 76 | n/a (most recent) |

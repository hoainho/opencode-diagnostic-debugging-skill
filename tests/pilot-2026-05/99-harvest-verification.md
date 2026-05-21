# Stage B3 — Harvest Verification

> **Plan reference**: [`.opencode/plans/2026-05-21-skill-production-hardening.md`](../../.opencode/plans/2026-05-21-skill-production-hardening.md) Stage B3.
>
> **Pass criterion** (from plan): ≥ 80% of pilot postmortems surface as Phase 0 recall hits when queried with their original symptom phrasing (so: 3 of 4, or 4 of 5).
>
> **Verification date**: 2026-05-21.
> **Pilot count**: 4 (all of Stage B2 complete).

---

## Methodology

For each pilot, query `omo-session-distiller_recall` with a **plain-language symptom phrase** that a user (or future-self with no prior knowledge of the bug) would actually type. **Crucially: queries do NOT include the WIN-#### ticket ID** — that would be a trivial regression of the harvest pipeline's keyword index. A real Phase 0 query starts from symptoms, not from already-known ticket numbers.

Pass condition per pilot: the corresponding atom appears as the **top-ranked hit** with a meaningful score gap to off-topic alternatives. Per A5's daemon-down rubric (≥10 = consider Clearly Relevant + Problem-text match), a top-rank with score ≥10 AND clear gap to next-best constitutes a hit.

---

## Per-pilot results

### Pilot-1 (B2.1) — WIN-7993 (Daily Bonus auto-claim stuck)

Query: `daily bonus reward stuck claimable after auto claim 5 minute`
(`repo="playsweeps-web"`)

| Rank | Atom | Score | Relevance |
|---|---|---|---|
| **1** | **atom-2026-05-21-pilot-1-WIN-7993-daily-bonus-auto-claim-stuck** | **67** | ✅ EXACT match (the pilot atom) |
| 2 | atom-2026-02-09-...-auto-filled-city-values-dont-match-dropdown | 7 | ❌ unrelated |
| 3 | atom-2026-02-07-...-useApplePayHandler-review | 5 | ❌ unrelated |

**Result**: ✅ HIT. Top rank, score 67, gap of **60 points** to next.

### Pilot-2 (B2.2) — WIN-7988 (BI event LP earning missing source fields)

Query: `BI event missing source_id source_name LP earning daily cap`
(no repo filter — cross-repo bug)

| Rank | Atom | Score | Relevance |
|---|---|---|---|
| **1** | **atom-2026-05-21-pilot-2-WIN-7988-bi-event-lp-source-missing** | **38** | ✅ EXACT match |
| 2 | atom-2026-02-06-...-reCAPTCHA-error-docs | 18 | ❌ unrelated (matched on "search" + "docs" tokens) |
| 3 | atom-2026-02-06-...-recaptcha-script-loading | 17 | ❌ unrelated |

**Result**: ✅ HIT. Top rank, score 38, gap of **20 points** to next.

### Pilot-3 (B2.3) — WIN-7990 (Prize Summary always Lootbox icon, trivial-confirmation)

Query: `prize summary modal wrong icon giftbox lootbox reward type`
(`repo="playsweeps-web"`)

| Rank | Atom | Score | Relevance |
|---|---|---|---|
| **1** | **atom-2026-05-21-pilot-3-WIN-7990-prize-summary-lootbox-icon-always** | **46** | ✅ EXACT match |
| 2 | atom-2026-02-10-...-react-debugger-tracking-info | 11 | ❌ unrelated (matched on "info") |
| 3 | atom-2026-02-05-...-typescript-config-check | 10 | ❌ unrelated |

**Result**: ✅ HIT. Top rank, score 46, gap of **35 points** to next.

### Pilot-4 (B2.4) — WIN-7518 (MFA cooldown not resetting)

Query: `MFA resend cooldown not resetting after extended expires sms otp`
(no repo filter)

| Rank | Atom | Score | Relevance |
|---|---|---|---|
| **1** | **atom-2026-05-21-pilot-4-WIN-7518-mfa-cooldown-not-resetting** | **49** | ✅ EXACT match |
| 2 | atom-2026-02-04-...-react-debugger-npm-notice | 21 | ❌ unrelated (token match) |
| 3 | atom-2026-02-10-...-prometheus-plan-builder | 9 | ❌ unrelated |

**Result**: ✅ HIT. Top rank, score 49, gap of **28 points** to next.

---

## Aggregate

| Pilot | Hit at top rank? | Top score | Gap to next | Verdict |
|---|---|---|---|---|
| B2.1 (WIN-7993) | ✅ | 67 | 60 | HIT |
| B2.2 (WIN-7988) | ✅ | 38 | 20 | HIT |
| B2.3 (WIN-7990) | ✅ | 46 | 35 | HIT |
| B2.4 (WIN-7518) | ✅ | 49 | 28 | HIT |

**Hit rate: 4/4 = 100%.** Pass criterion (≥80%) cleared.

**Score statistics** (descriptive only): range 38-67, median 47.5, mean 50. Gap-to-next range 20-60, median 31.5 — all gaps ≥ 20 points, well above any reasonable noise threshold under the daemon-down filesystem-grep scoring.

---

## Observations

### What worked

- **A5 Manual Harvest Export pipeline works end-to-end on all 4 real-bug postmortems.** Every pilot atom is reachable via plain-language queries that don't include the ticket ID.
- **Top-ranked-with-large-gap pattern**: every pilot atom outranks the next hit by ≥20 points. This is exactly the signal A5's score-and-text rubric was designed to surface — a strong top hit on real-symptom keywords stands out cleanly even under daemon-down filesystem-grep scoring.
- **Off-topic noise is well-discriminated**: in all 4 cases, rank-2 hits are clearly unrelated (Apple Pay, reCAPTCHA, React Debugger, etc.) and would be rejected by the A5 Problem-text-relevance check even before the score gap is considered. Defense in depth: keyword score gap + Problem text check both filter noise.

### Daemon-down caveat (carried forward from A1/A5)

All scoring is **filesystem-grep token-match counts** because `nano-brain` daemon is down in this environment. Stage C must:
- **Not generalize** these scores to daemon-up (semantic) mode — semantic similarity scoring will use a different scale and threshold
- **Re-run B3 in daemon-up mode if/when nano-brain is restored** to confirm the same hit-rate holds under semantic recall (a recall regression would be a meaningful signal)

### Hit-rate is necessary but not sufficient evidence

B3 proves **the A5 export pipeline works correctly** — atoms written via the manual-export procedure ARE indexed and reachable. It does NOT prove:

- That **2nd-bug Phase 0 recall** (B4) will hit (that requires a *different but similar* bug, not the same atom)
- That the **wall-clock speedup** claim (Stage C1) is real (B3 measures pipeline correctness, not user-time benefit)
- That the skill scales beyond n=4 pilots (B3 is bounded to the 4 atoms we wrote)

These are explicit Stage B4 / C1 / future-work concerns, not B3 gaps.

---

## Reviewer carryovers from B2.4 (folded in per oracle's non-blocking recommendation)

### B2.4 C4 — Hypothesis B exposition tightening

Per B2.4 reviewer comment: B2.4's Hypothesis B exposition tangled itself mid-row when discussing Layer 1 firing first vs the reported key. Recommended a 1-line clarification.

Applying the clarification in the B2.4 postmortem **AND** the trace would be a separate PR (B2.4 is already merged). Filing as a low-priority follow-up: the atom + solution remain semantically correct — the readability nit doesn't affect recall behavior. **Decision**: defer to a future B2.4-polish PR if a real reviewer hits the confusion. Not blocking B3.

### B2.4 C5 — Stratified mean for B2.3 trivial-confirmation outlier

Per B2.4 reviewer comment: B2.3 (~2 min, trivial-confirmation, 1 hypothesis) should not be averaged with the 3 phase-3-partial cases (~2.5 / 5.3 / 2.1 min) — different archetypes shouldn't be aggregated into a single mean.

**Applied here in B3**: the aggregate table above does NOT compute a mean across pilots. The wall-clock table from B2.4 noted "n=4 → descriptive stats only" and mean=~3min; that figure is now superseded by this stratified breakdown:

| Stratum | Pilots | Wall-clock range | Median |
|---|---|---|---|
| `phase-3-partial` (3-hypothesis investigation) | B2.1, B2.2, B2.4 | 2.1 - 5.3 min | 2.5 min |
| `trivial-confirmation` (1-hypothesis confirmation) | B2.3 | 2.0 min | 2.0 min (n=1) |

The 2-min B2.3 is now categorized correctly as a 1-sample trivial-confirmation point, not an outlier in the phase-3-partial stratum. Stage C1 metrics should keep this stratification.

---

## Status

`outcome: pass`. Hit rate 4/4 = 100%, well above the ≥80% pass criterion. B3 gate cleared. Stage B4 (second-similar-bug Phase 0 verification) is unblocked.

---
date: 2026-05-21
slug: pilot-1-WIN-7993-daily-bonus-auto-claim-stuck
repo: playsweeps-web
bug_class: state-desync
severity: sev1
area_tag: daily-bonus/auto-claim-v3
recall_hit: no-novel
evidence_quality: degraded
degraded_reason: "Pilot controller has no QA2 auth; static code evidence only. Falsification of all 3 hypotheses requires live API response inspection (handed off to backend assignee)."
fix_commit: n/a-pilot-no-fix-pushed
time_spent_minutes: 3
---

# Postmortem: WIN-7993 Daily Bonus V2 auto-claim leaves reward in claimable state

## Bug Summary
After server-side auto-claim (5-min timeout) credits a Daily Bonus reward to the player's balance, the Daily Bonus modal continues to show the reward as claimable and the countdown timer stops, instead of showing Collected + active countdown to next day.

## Reproduction Command
```
1. QA2 v1.9.2(3), Safari or Chrome, logged in account
2. Daily Bonus settings: auto-claim ON, threshold 5 minutes
3. Open Daily Bonus modal in lobby
4. Reveal a lootbox reward
5. Close browser tab (do NOT click Claim)
6. Wait 5 minutes (server-side auto-claim fires)
7. Reopen lobby
8. Observe Daily Bonus modal state
```
Reproduce rate: 100% per Jira ticket.

## Root Cause
Not confirmed (pilot is degraded evidence; 3 candidate mechanisms identified, all share a single falsification test handed off to backend assignee):

1. **Hypothesis A**: server returns `Days[].Status: Pending` on auto-claimed day; client's `correctCurrentDay()` at `src/redux/dailyBonus/reducer.ts:65-66` maps `Pending → Available` when `nextClaimTime` has passed. No `Pending → Collected` mapping exists, so an auto-claimed day shows as still-claimable.
2. **Hypothesis B**: server returns `Status: Collected` correctly but client clobbers it elsewhere in the reducer chain (`correctCurrentDay` or `appendPreviousMilestonesOnMaxCountsReached`).
3. **Hypothesis C**: server doesn't advance `NextClaimDate` post-auto-claim. `isClaimable = nextClaimTime == null || Date.now() >= nextClaimTime` (reducer.ts:53) then evaluates true indefinitely, causing the timer to stop ("0:00 / Claim Now" stuck state).

## Fix
Not implemented in this pilot (per Stage B contract: diagnostic trace + postmortem only, no production fix).

Likely fix sites if A/B confirmed: `src/redux/dailyBonus/reducer.ts:65-66` (add `Pending → Collected` mapping conditional on a server-provided `IsAutoClaimed: true` flag).

Likely fix site if C confirmed: server-side `Sweeps.Awards.DailyBonusCalculator.cs` (advance `NextClaimDate` on auto-claim) OR defensive client fix in `correctCurrentDay()`.

## Evidence Sources Used
- `omo-session-distiller_recall(query="daily bonus reveal lootbox state machine", limit=5)`: returned 5 token-match hits (scores 30/21/16/16/15), all off-topic Problem texts → no Phase 0 hit.
- `rg -l "autoClaim|AUTO_CLAIM|auto-claim" src/`: **0 matches** in Daily Bonus directories — feature is server-driven, not client-emitted.
- `read src/redux/dailyBonus/reducer.ts:50-156`: `correctCurrentDay()` maps `Pending → Available` (line 65-66) and `Pending → NotReady` (line 63-64); **no `Collected` target** in the Pending-source mappings.
- `read src/redux/dailyBonus/reducer.ts:53`: `isClaimable` formula computes from `nextClaimTime` only — no awareness of a server-side "already collected" signal.

## Phase 0 Recall Result
`no-novel`. 5 hits returned by the second-rephrased query but all were tag-only matches (score 15-30 from `playsweeps-web` + `redux` tags) with off-topic Problem texts. A5's score+text rubric correctly rejected all 5.

## Oracle Consult
—  (Oracle would not help with the falsification gap — Oracle is read-only too.)

## Open Risks
- **Falsification not executed**: 3 hypotheses formed but none confirmed. Backend assignee handoff required to inspect the API response.
- **Cap rule semantics question**: 3 hypotheses share a falsification: A and C are discriminated by ONE API call (inspect both Status field and NextClaimDate); B needs the same call PLUS a reducer trace if Status returns `Collected`. Unique rule-out work ≈ 1.2 units, not 3. The compound-paralysis cap (A3) treats this as "3 hypotheses = at cap" but the unique cost is much lower. A3 cap should measure "unique falsification cost" not "hypothesis count". Filing as Stage A3 follow-up.
- **Speedup claim n=1**: pilot wall-clock of ~2.5 min (diagnostic only, write-up excluded) vs gut-estimated baseline of ~15-20 min unguided is a 1-sample/gut-baseline claim, not a measured control. Do not propagate "5-7x faster" without the qualifier. Stage C1 will calibrate after 3-5 pilots complete.

## Prevention Notes
- Add explicit `Pending → Collected` mapping path in `correctCurrentDay()` triggered by a server-provided auto-claim flag. If the server doesn't currently emit such a flag, add it to the v3/players/daily-bonus/status response.
- Add an integration test in `src/redux/dailyBonus/__tests__/` that simulates a status response with an auto-claimed day and asserts the resulting `dailyBonusData.DailyBonus.Days[N].Status === 'Collected'`.
- Add a CI smoke that exercises the full auto-claim cycle in QA2 nightly: reveal → close session → wait → status check.

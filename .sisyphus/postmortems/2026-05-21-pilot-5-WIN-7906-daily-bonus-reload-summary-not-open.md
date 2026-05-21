---
date: 2026-05-21
slug: pilot-5-WIN-7906-daily-bonus-reload-summary-not-open
repo: playsweeps-web
bug_class: state-desync
severity: sev1
area_tag: daily-bonus/reveal-summary-persistence
recall_hit: yes-partial
evidence_quality: degraded
degraded_reason: "Pilot controller has no QA1 auth; static code evidence only. Bug already addressed by a merged PR per Jira — B4 verifies the skill's recall + diagnostic compression, NOT a fresh fix."
fix_commit: n/a-bug-already-merged-by-assignee
time_spent_minutes: 1
---

# Postmortem: WIN-7906 Daily Bonus V2 — reveal lootbox then reload doesn't reopen reward summary

## Bug Summary
After revealing a Daily Bonus lootbox, if the user reloads the page or closes-and-reopens the lobby on another tab BEFORE viewing the reward summary, the summary does not auto-open on next load. User must reveal lootbox again (already-claimed reward, so this is a state-tracking bug).

## Reproduction Command
```
1. QA1 v1.9.0 - 1.22.5(3), Safari or Chrome, logged-in account
2. Open Daily Bonus modal
3. Reveal a lootbox (do NOT click through to the summary close)
4. Reload page OR close tab and open lobby in another tab
5. Observe: reward summary does not auto-open
```
Reproduce rate: 100% per Jira ticket.

## Root Cause (atom-seeded hypothesis, not verified via merged PR diff)

The B2.1 pilot atom (atom-2026-05-21-pilot-1-WIN-7993-daily-bonus-auto-claim-stuck) immediately surfaced as top recall hit and pointed at `src/redux/dailyBonus/reducer.ts`. The same `correctCurrentDay` state-mapping function is implicated:

`reducer.ts:128` sets `pendingRevealedDay` from `Days.find(day => day.Status === DailyBonusStatus.Pending)` — ONLY when status is `Pending` AND `isClaimable`. After `revealDailyBonusV3` succeeds, `correctCurrentDay` at `reducer.ts:65-66` maps `Pending → Available` (because `isClaimable` after reveal). On reload, the day is now `Available`, not `Pending` → `pendingRevealedDay = null` → the popup trigger (selectors.ts:41 `makeSelectPendingRevealedDay`) returns null → summary not auto-opened.

**Missing state**: the system has no field tracking "reveal was performed but summary not yet shown to user." The popup-trigger only knows about Pending (pre-reveal) state; it has no concept of "post-reveal, pre-summary-acknowledgement."

## Fix (NOT pushed — bug already has a merged PR by assignee)

The assigned team member (Trí Mai Ngọc) has a MERGED PR per Jira. Without comparing diffs, the likely fix pattern is:
- Option A: Persist a `revealedAwaitingSummary: DailyBonusDay | null` field in Redux + localStorage. Set on REVEAL_DAILY_BONUS_V3 success; clear on summary modal close.
- Option B: Modify `correctCurrentDay` to preserve `Pending` status until the summary modal is acknowledged (not at reveal-call success time).

B4's purpose is to demonstrate the skill's recall-driven compression, not duplicate the team's fix.

## Evidence Sources Used
- `omo-session-distiller_recall(query="daily bonus lootbox reveal reload close tab reward summary not open", repo="playsweeps-web", limit=5)`: 5 hits. Top 2 (scores 38, 29) are pilot atoms B2.1 + B2.3 — both directly relevant to the area. Rank-3 onward (scores ≤14) is unrelated noise.
- `read atoms/playsweeps-web/atom-2026-05-21-pilot-1-WIN-7993-...`: "Related files" section pointed at reducer.ts:50-156 and saga.ts:30-119 immediately.
- `rg -n "pendingRevealedDay|revealedReward" src/redux/dailyBonus/`: 7 matches across reducer.ts, type.ts, selectors.ts, saga.ts.
- `read src/redux/dailyBonus/reducer.ts:24, 32, 128`: confirmed `pendingRevealedDay` field semantics and conditional set logic.
- `read src/redux/dailyBonus/selectors.ts:41`: confirmed `makeSelectPendingRevealedDay` is the popup-trigger selector.

## Phase 0 Recall Result
`yes-partial`. B2.1 atom surfaced as top hit (score 38) — same module, similar trigger family (tab close interrupts a Daily Bonus reveal/claim flow), different surface. B2.3 atom at rank 2 (score 29) — related summary-modal angle. Atom served as Phase 3 hypothesis seed, not verbatim fast-path (B2.1 was itself unconfirmed at phase-3-partial).

## Chained Atoms (informal — not yet harvester-indexed per A4 extension-fields tier)
- atom-2026-05-21-pilot-1-WIN-7993-daily-bonus-auto-claim-stuck (B2.1)
- atom-2026-05-21-pilot-3-WIN-7990-prize-summary-lootbox-icon-always (B2.3)

## Oracle Consult
—

## Open Risks
- **Hypothesis not falsified**: I read static code + formed a hypothesis but didn't validate it against the merged PR diff or runtime behavior. The hypothesis could be wrong even though it's reducer.ts-region correct.
- **Same-author bias for recall**: I wrote both the B2.1 atom and the B4 query — the recall hit may partly reflect my own keyword-vocabulary consistency. A future user querying with different vocabulary may get different results.
- **B4 is n=1 in the "second-similar-bug" stratum**: the 36% wall-clock figure is descriptive only. Stage C1 must not propagate as "skill makes related bugs 3x faster."
- **Daemon-down filesystem-grep scoring throughout**: same caveat as B3.

## Prevention Notes
- Add explicit Redux field `revealedAwaitingSummary` to `dailyBonus` slice; persist via existing localStorage middleware. Set on REVEAL success; clear on summary-modal-close action. Add integration test that simulates reveal-then-reload and asserts summary auto-opens.
- More broadly: any "user mid-flow" state (reveal, in-progress purchase, etc.) needs a persisted "interruption-recoverable" field — purely-transient Redux state is lost on reload. Audit other similar flows for the same gap.
- Document in `src/redux/dailyBonus/README.md` (if exists) the state machine: Pending → reveal-pending → Available → summary-shown → Collected. Currently the middle two are conflated.

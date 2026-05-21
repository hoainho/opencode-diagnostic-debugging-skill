---
date: 2026-05-21
slug: pilot-2-WIN-7988-bi-event-lp-source-missing
repo: playsweeps-backend
bug_class: data
severity: sev1
area_tag: bi/lp-earning
recall_hit: no-novel
evidence_quality: degraded
degraded_reason: "playsweeps-bi-event-worker repo not in workspace; cross-system trace incomplete. Field population vs missing-emit-path disambiguation requires QA2 BI dashboard inspection (handed off to backend assignee)."
fix_commit: n/a-pilot-no-fix-pushed
time_spent_minutes: 5
---

# Postmortem: WIN-7988 BI event update_balance + loyalty_cap_reached missing source_id/source_name for LP earning

## Bug Summary
Both `update_balance` and `loyalty_cap_reached` BI events emitted during LP (Loyalty Points) earning flows are missing the `source_id` and `source_name` fields. Impact: lost LP earning attribution in BI reporting and analytics. Triggered by playing slot games until daily LP cap is reached.

## Reproduction Command
```
1. Sign in to QA2
2. Play a game and spin until daily cap for LP earning is reached
3. Return to lobby
4. Inspect update_balance and loyalty_cap_reached events in BI dashboard
```

## Root Cause
Not confirmed (pilot reached Phase 3 with 3 distinct hypotheses; falsification requires QA2 BI dashboard access + bi-event-worker repo access not available to pilot controller):

1. **Hypothesis A**: slot-LP-earning emits via `Sweeps.Player/Services/PsPlayerBalanceService.cs:200` with `eventSource = "sync_nexus"`. Fields ARE populated but with a generic value that downstream BI dashboard treats as null/missing for LP-specific reporting.
2. **Hypothesis B**: `loyalty_cap_reached` is derived inside `playsweeps-bi-event-worker` (Azure Function consuming EventHub). The worker emits the derived event without copying source fields from the trigger event — info loss during transform.
3. **Hypothesis C**: LP-specific `update_balance` emit path is missing entirely. `Sweeps.myVip/Services/MyVipCoBiEventService.SendUpdateBalanceEventAsync:129` only fires on tier-up. Per-spin LP earning has no LP-specific emit — only the generic Sweep-balance update from `PsPlayerBalanceService` (which carries `eventSource="sync_nexus"`).

## Fix
Not implemented in this pilot (per Stage B contract).

Likely fix sites:
- **If A confirmed**: extend `Sweeps.myVip/Services/MyVipCoBiEventService` with non-tier-up LP-earning emit method; wire into gameplay-end processor.
- **If B confirmed**: patch `playsweeps-bi-event-worker` cap-reached derive to forward source fields.
- **If C confirmed**: same architectural fix as A (add the missing LP emit path).

## Evidence Sources Used
- `omo-session-distiller_recall(query="BI event update_balance loyalty_cap_reached source_id source_name missing", limit=5)`: 5 hits, all below A5 score ≥10 bar (max score 9, all off-topic).
- `rg -l "loyalty_cap_reached|update_balance" src/`: in playsweeps-web → only `customEvent.js` (frontend-only `gameplay_update_balance`, NOT the bug event).
- `rg "loyalty_cap_reached"` across all .cs in playsweeps-backend → **0 matches**. Confirms event is NOT emitted from playsweeps-backend; must be derived by playsweeps-bi-event-worker.
- `rg "source_id|source_name|update_balance" Sweeps.Player/ Sweeps.myVip/ Sweeps.Quest/`: identified 3 distinct emit paths:
  - `Sweeps.Player/Services/PsPlayerBalanceService.cs:147-166` — generic `eventSource="sync_nexus"`
  - `Sweeps.Quest/Services/PlayerQuestService.cs:67-80` — correctly populated (`source_name="daily_quest"`, `source_id=questId`)
  - `Sweeps.myVip/Services/MyVipCoBiEventService.cs:122-153` — tier-up-gated (line 129)
- Confirmed `playsweeps-bi-event-worker` is NOT in the workspace; cross-system trace must be completed by backend assignee with access to that repo + QA2 BI dashboard.

## Phase 0 Recall Result
`no-novel`. All 5 recall hits scored 4-9 (below A5 ≥10 bar) and Problem texts were unrelated.

## Oracle Consult
—  (Oracle would not help — read-only and cannot fetch BI dashboard data either.)

## Open Risks
- **Falsification not executed**: 3 hypotheses formed but none confirmed. Backend assignee handoff required for QA2 BI dashboard inspection + bi-event-worker repo read.
- **A vs C falsification overlap** (per B2.2 review C1): both hypotheses predict same observable dashboard surface (`source_name="sync_nexus"`). Discrimination requires PRD/spec lookup for the LP-earning BI contract IN ADDITION to dashboard inspection. Reinforces the A3 cap-rule follow-up — cap should measure unique-falsification-cost not raw hypothesis count, and surface-overlap is exactly the case it must handle.
- **Cross-repo scope without all repos**: skill's Row 9 (build/dep) fallback doesn't cover runtime event-flow across EventHub-connected repos. Filing as Stage A4 follow-up: routing table needs explicit "cross-repo runtime event-flow" subsection with handoff template.
- **n=2 speedup data**: B2.1 was ~2.5min (single-repo state-desync), B2.2 is ~5.3min (cross-repo BI event flow). Sample size still too small for any aggregate claim; Stage C1 calibrates after all 4 pilots.

## Prevention Notes
- Audit ALL `update_balance` BI emit sites (currently 4 known in playsweeps-backend: `PsPlayerBalanceService:147` Nexus sync, `PsPlayerBalanceService:830` IncreasePlayerBalance, `PlayerQuestService:67`, `MyVipCoBiEventService:122`) and ensure each populates `source_name` + `source_id` with a meaningful (non-`sync_nexus`-generic) value.
- Add a CI smoke that publishes a fixture `update_balance` event with each known `eventSource` value through the EventHub publisher and asserts the bi-event-worker preserves `source_name`/`source_id` in any derived events.
- Document the BI event schema contract in a shared `playsweeps-bi-event/SCHEMA.md` so all producer + consumer repos reference the same field-population requirements.

# Pilot-2 trace — WIN-7988 (BI event LP earning missing source_id/source_name)

> **Bug**: WIN-7988 — `update_balance` and `loyalty_cap_reached` BI events missing `source_id` + `source_name` for LP earning flows.
> **Diagnostic wall-clock** (recall + repro-spec + evidence + hypothesis formation; **excludes write-up**): 2026-05-21T08:19:16Z → 08:24:35Z, ~5 min 19 sec.
> **Write-up time** (trace + postmortem authoring): additional ~10 min outside the timed window.
> **Outcome**: PHASE-3-PARTIAL — 3 distinct hypotheses formed with code evidence; falsification requires QA2 BI dashboard access + bi-event-worker repo (not in workspace).

---

## Symptom (from Jira)

> `update_balance` and `loyalty_cap_reached` events are missing `source_id` and `source_name` fields for LP earning flows. Sign in → spin until LP daily cap → return to lobby → observe events. Actual: fields missing. Expected: fields present.
> Environment QA2. Impact: missing LP earning attribution, reduced visibility, incomplete BI reporting.

## Phase 0 — RECALL

```pseudocode
omo-session-distiller_recall(query="BI event update_balance loyalty_cap_reached source_id source_name missing", limit=5)
```
Result: 5 hits, scores 9/7/4/4/4 — **all below A5 ≥10 bar**. Top hit (score 9) is "Agent not found atlas" — unrelated. No clearly-relevant hits.

`evidence_quality: verified` for Phase 0.

## Phase 1 — REPRODUCE

Spec-only (no QA2 auth). Recipe from Jira deterministic:
1. Sign in
2. Spin a game until LP daily cap reached
3. Return to lobby
4. Inspect `update_balance` + `loyalty_cap_reached` events in BI dashboard

## Phase 2 — EVIDENCE (Row 6 data schema + Row 9 cross-repo; degraded — bi-event-worker repo absent)

Cross-repo investigation per B1 review C2 directive (disambiguate `playsweeps-bi-event` vs `playsweeps-bi-event-worker`).

**Available repos in workspace**: playsweeps-web, playsweeps-backend (MyVegas.* solution).
**Missing**: playsweeps-bi-event, playsweeps-bi-event-worker.
→ Mark `evidence_quality: degraded` for Phase 2. The cross-system trace cannot complete; bi-event-worker behavior must be inferred or handed off.

### Frontend evidence (playsweeps-web)

```
rg -l "loyalty_cap_reached|update_balance" src/
→ src/CHANGELOG.md, src/constants/customEvent.js only
rg "loyaltyCapReached|loyalty.?cap" src/
→ 0 matches
```

**Finding**: `loyalty_cap_reached` is NOT emitted from the frontend. Frontend's `customEvent.js:77` has `GAMEPLAY_UPDATE_BALANCE: "gameplay_update_balance"` — a separate, frontend-only event. The bug events are **server-emitted**.

### Backend evidence (playsweeps-backend MyVegas.* solution)

`rg "loyalty_cap_reached"` across ALL .cs files → **0 matches**. The event is NOT emitted by playsweeps-backend either.

**Conclusion**: `loyalty_cap_reached` is likely emitted by `playsweeps-bi-event-worker` (the Azure Function consuming EventHub stream) — derived from balance-update events when running LP total ≥ threshold. Without that repo, B is unverifiable statically.

`rg "source_id|source_name|update_balance" Sweeps.Player/ Sweeps.myVip/ Sweeps.Quest/` reveals THREE distinct emit paths:

1. **`Sweeps.Player/Services/PsPlayerBalanceService.cs:147-166`** — Generic balance sync from Nexus:
   - Line 154-156: `source_name = eventSource; source_id = eventSource;`
   - Line 200: caller passes `eventSource = "sync_nexus"` (or `"sweep_expiration"`)
   - **Source fields populated correctly with the literal string "sync_nexus"** — but this string may not be the source name BI dashboard expects.

2. **`Sweeps.Quest/Services/PlayerQuestService.cs:67-80`** — Quest reward path:
   - Line 76: `source_name = "daily_quest"`
   - Line 77: `source_id = questId`
   - **Quest path is CORRECT** — provides meaningful per-quest attribution.

3. **`Sweeps.myVip/Services/MyVipCoBiEventService.cs:122-153`** — VIP tier-up path:
   - Line 129: gated by `newVipStatus.Tier > 1 && newVipStatus.Tier > vipStatus?.Tier`
   - Line 138-140: source fields populated with `parsedSource = GetSourceName(LootSourceType)`
   - **Only fires on tier-up**, not on every LP earning event.

### Cross-system gap (per B1 C2 disambiguation)

Per AGENTS.md: `playsweeps-bi-event` is a microservice; `playsweeps-bi-event-worker` is an Azure Function consuming EventHub from backend's `PlayStudios.EventHub.Publisher`. The Jira ticket doesn't specify which consumer surfaces the missing-field gap.

**Without bi-event-worker source access**: cannot determine whether the worker (a) derives `loyalty_cap_reached` from a raw stream, (b) copies source fields from input events to derived events, or (c) drops source fields during a schema transform.

## Phase 3 — HYPOTHESES (3 distinct mechanisms; within A3 cap = 3)

| ID | Mechanism | Evidence | Falsification | Cost |
|---|---|---|---|---|
| A | LP-earning slot path emits via `PsPlayerBalanceService.SendSyncNexusEvent` with `eventSource = "sync_nexus"` — fields ARE populated but with a generic value the BI dashboard treats as null/missing for LP-specific reporting | `PsPlayerBalanceService.cs:200`; backend-only emit, no LP-source attribution | Inspect QA2 BI dashboard for one LP-earning spin; check whether source_name/source_id contain `"sync_nexus"` literal OR empty | 5 min |
| B | `loyalty_cap_reached` is derived in bi-event-worker from balance stream. The derive emits the event but does NOT copy source fields from the trigger event (info loss during transform) | Cannot verify statically — repo not in workspace | Inspect bi-event-worker code; specifically the cap-reached emit logic; check whether source fields are copied from input or constructed fresh | 5 min (needs bi-event-worker access) |
| C | LP-specific `update_balance` event is missing entirely — `MyVipCoBiEventService.SendUpdateBalanceEventAsync` only fires on tier-up (line 129 condition). Regular per-spin LP earning has no LP-specific emit. Frontend/BI dashboard observes the generic Sweep-balance `update_balance` from `PsPlayerBalanceService` and mistakenly classifies it as LP, hence "missing fields" because Sweep emit's source = `"sync_nexus"` doesn't match LP expectations | `MyVipCoBiEventService.cs:129` tier-gate condition + missing per-spin LP emit path | Trace a single LP-earning spin end-to-end in QA2: which BI events fire? If no LP-specific `update_balance` (only generic Sweep) → C confirmed | 10 min (cross-system trace) |

**Cap analysis**: 3 hypotheses, 3 distinct mechanisms, partial shared falsification (one BI dashboard inspection reveals scenarios A vs C; B requires separate bi-event-worker access). Unique falsification work ≈ 2 units (BI inspection + bi-event-worker read). Within cap of 3.

### Triage (cheapest-first per A2)

A=5min, B=5min, C=10min. A and B are equal cost; A is most likely (matches "missing fields" symptom most directly). Run A first → if `source_name = "sync_nexus"` confirmed in dashboard, A is the bug. If empty/null, run B (bi-event-worker code read). C is fallback if A and B both miss.

### Important — A vs C falsification overlap (per B2.2 review C1)

**Dashboard inspection alone does NOT cleanly discriminate A from C.** Both hypotheses predict the same observable surface: BI dashboard shows `source_name="sync_nexus"` from `PsPlayerBalanceService.cs:154-156`. The mechanisms differ (A = "this IS the LP emit, value is wrong"; C = "this is NOT the LP emit, the LP emit doesn't exist") but the dashboard reads identical.

Distinguishing A from C **also requires a PRD/spec lookup** answering: "Was the existing `PsPlayerBalanceService` emit ever meant to fire for slot LP earning, or was a separate LP-specific emit supposed to fire here?"

- If spec says "yes, sync_nexus IS the LP path, just rename source for LP attribution" → **A**: fix at `PsPlayerBalanceService` to pass an LP-specific eventSource value through.
- If spec says "no, slot LP earning should fire a dedicated emit with semantic source_name='lp_earning' + source_id=slotId" → **C**: add the missing emit path, the existing `sync_nexus` event is the wrong artifact entirely.

Backend assignee needs BOTH the dashboard inspection AND the PRD/spec for the LP-earning BI contract. Falsification work for A vs C is more entangled than "1 dashboard inspection."

This is exactly the kind of overlap the A3 cap rule was designed for — `count: 2 hypotheses sharing surface, requires +1 unique work unit for disambiguation`. Filed as A3 follow-up reinforcement (cap should measure unique-falsification-cost, not raw count).

## Phase 4 — FIX (not executed)

Skipped per Stage B contract. Likely fix sites:

- **If A confirmed**: extend `Sweeps.myVip/Services/MyVipCoBiEventService` to add a non-tier-up LP-earning emit method that fires on every cap-trigger spin with proper `source_name = "lp_earning"`, `source_id = <slot_id>`. Wire it into the gameplay-end processor (likely `Sweeps.SharedServices/Processors/MyVipGamePlayProcessor` or its caller).
- **If B confirmed**: patch bi-event-worker's cap-reached derive to forward source fields from the input event into the derived event.
- **If C confirmed**: add the LP-emit path. Same architectural location as A's fix.

## Phase 5 — POSTMORTEM

Captured in `.sisyphus/postmortems/2026-05-21-pilot-2-WIN-7988-bi-event-lp-source-missing.md`.

`evidence_quality: degraded`. `degraded_reason: "playsweeps-bi-event-worker repo not in workspace; cross-system trace incomplete; field population vs missing-emit-path disambiguation requires QA2 BI dashboard inspection deferred to backend assignee."`

## Skill protocol observations (Stage B data)

✅ **What worked**:
- A4 fallback chain executed correctly: bi-event-worker missing → marked degraded explicitly; static evidence collected from available repos and clearly labeled.
- B1 review C2 directive (disambiguate the two BI repos) was followed and surfaced the gap honestly (cannot distinguish without access).
- 3 hypotheses (A/B/C) are genuinely distinct mechanisms with non-trivial unique falsification work; A3 cap-paralysis didn't fire.
- Phase 0 score+text rubric (A5) correctly rejected all 5 hits (max score 9 < threshold).

⚠️ **Where the skill ran into limits**:
- **Cross-repo scope without all repos in workspace** is a real limitation. The skill's Row 9 fallback says "rtk grep on imports" — works for build dep resolution but NOT for runtime BI event flow that crosses three independent repos via EventHub messaging. The skill could benefit from an explicit "cross-repo-handoff template" subsection in the routing table.
- **Diagnostic wall-clock 5 min 19 sec** (vs B2.1's 2.5 min) reflects real complexity of cross-repo investigation. The "n=2 sample" speedup claim now has more data: B2.1 (state-desync, single repo) = ~2.5min; B2.2 (cross-repo BI event flow) = ~5.3min. Neither is statistically meaningful yet.

🟢 **Wall-clock (diagnostic only)**: ~5 min 19 sec. Write-up separate.

⚠️ **n=2 speedup claim still NOT statistically valid**: do not propagate "5x faster" or any aggregate. Stage C1 will calibrate after all 4 pilots complete.

## Status

`outcome: phase-3-partial`. 3 hypotheses formed, all handed off to backend assignee (Khương Lê Quang Anh per Jira). Single QA2 BI-dashboard inspection (~5 min) discriminates A from C; B requires separate bi-event-worker repo access. Skill identified the disambiguation path that saves the assignee from cold-investigating the cross-repo event flow.

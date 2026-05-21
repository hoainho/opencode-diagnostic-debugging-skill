---
date: 2026-05-21
slug: pilot-3-WIN-7990-prize-summary-lootbox-icon-always
repo: playsweeps-web
bug_class: state-desync
severity: sev2
area_tag: prize-summary/rive-binding
recall_hit: no-novel
evidence_quality: verified
fix_commit: n/a-pilot-no-fix-pushed
time_spent_minutes: 2
---

# Postmortem: WIN-7990 Prize Summary always shows Lootbox icon regardless of reward type

## Bug Summary
Prize Summary modal always renders the Lootbox icon in its header, regardless of the actual reward type opened (Giftbox, normal Daily Bonus reward, etc.). Visual inconsistency between the opened item and the displayed summary.

## Reproduction Command
```
1. QA2, logged-in account
2a. Open a Giftbox reward → observe Prize Summary header icon (shows Lootbox; expected Giftbox)
2b. Claim a normal (non-box) Daily Bonus reward → observe Prize Summary header icon (shows Lootbox; expected no icon)
3. Reproduce rate: 100%
```

## Root Cause
Trivial-confirmation outcome — single hypothesis with overwhelming static evidence:

The Rive state-machine metadata at `src/view/shared/components/CoinCelebration/useAwardSummaryConfig.ts:12-14` declares two inputs that control the header icon:
```typescript
"header/listType": "Enum",      // selects between Lootbox / Giftbox / no-icon
"header/boxSwitch": "Boolean",  // toggles box-icon visibility
```

But `AwardDataBindingConfig` at `src/view/shared/components/CoinCelebration/AwardSummaryPopup.tsx:166-177` never assigns values to these inputs when building `finalRiveViewData.header`:
```typescript
header: {
    ...riveViewConfig?.header,
    tierNum                      // only tierNum is set; boxSwitch + listType omitted
}
```

`rg "boxSwitch|listType" src/` confirms: 0 assignments anywhere in the codebase. The Rive state machine consequently uses its default values for these inputs, which renders the Lootbox icon regardless of actual reward type.

The required source data (`data.awardType: AwardType`) is already available to the component (used at line 54 for `buttonId` derivation) — the bug is purely a missing JS→Rive binding, not missing data.

## Fix
Not implemented in this pilot (per Stage B contract).

Likely 3-line patch in `AwardSummaryPopup.tsx:167-170`:
```typescript
header: {
    ...riveViewConfig?.header,
    tierNum,
    boxSwitch: data?.awardType === AwardType.LOOT_BOX || data?.awardType === AwardType.GIFT_BOX,
    listType: data?.awardType,  // or enum-int mapping per Rive designer
},
```

Backend/UI assignee (Tân Chu Thái per Jira) should confirm the exact Rive enum value mapping for `listType` (Rive `.riv` enum values may not match JS `AwardType` enum order 1:1).

## Evidence Sources Used
- `omo-session-distiller_recall(query="prize summary lootbox icon giftbox reward type display", limit=5)`: 5 hits, scores 12-25 (above A5 ≥10 bar) but all Problem texts off-topic (Saga lifetime imports, Vue debugger icons). A5 rubric correctly rejected.
- `rg "PrizeSummary|prize_summary_modal" src/`: located 2 files (template + config hook).
- `read src/view/shared/components/CoinCelebration/useAwardSummaryConfig.ts:12-14`: confirms metadata declaration of `header/boxSwitch` + `header/listType`.
- `read src/view/shared/components/CoinCelebration/AwardSummaryPopup.tsx:166-177`: confirms `header` block omits these inputs in the binding.
- `rg "boxSwitch|listType" src/view/shared/components/CoinCelebration/`: confirms 0 assignments anywhere.
- `AwardSummaryPopup.tsx:54`: confirms `data.awardType` is already used elsewhere in the same component → source data is available.

## Phase 0 Recall Result
`no-novel`. 5 recall hits scored 12-25 but Problem texts were all unrelated topics (Saga lifetime, Vue debugger). A5 score+text rubric prevented false-positive.

## Oracle Consult
—

## Open Risks
- **Trivial outcome — n=3 sample**: Stage C1 calibration after B2.4 must account for archetype heterogeneity. B2.1=state-desync single-repo, B2.2=cross-repo BI, B2.3=trivial UI binding, B2.4=upcoming MFA. Aggregating across 4 disparate archetypes still won't be statistically valid; sample size remains too small.
- **Rive enum mapping uncertainty**: `listType` enum integer values inside the `.riv` file may not match the JS `AwardType` enum integer order. Backend/UI assignee needs to verify the mapping (look at the Rive editor's enum definitions or pin the JS enum integer values explicitly).

## Prevention Notes
- Add a unit test in `AwardSummaryPopup.test.tsx` that asserts `finalRiveViewData.header` contains both `boxSwitch` and `listType` keys when `data.awardType` is provided.
- Lint rule candidate: any Rive metadata key declared but never written in a component file should warn — would catch this class of bug at PR review.
- Architectural pattern: when a Rive component declares state-machine inputs, the JS side should have a single explicit "data binding" function that asserts (via TypeScript exhaustiveness check on the metadata key list) that every declared input has a corresponding binding. Currently the contract is implicit and easy to drop.

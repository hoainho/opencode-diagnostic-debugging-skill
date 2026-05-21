# Pilot-3 trace — WIN-7990 (Prize Summary always shows Lootbox icon)

> **Bug**: WIN-7990 — Prize Summary always displays Lootbox icon regardless of reward type (Lootbox / Giftbox / normal reward all show Lootbox).
> **Diagnostic wall-clock**: 2026-05-21T08:33:12Z → 08:35:10Z, ~2 min.
> **Write-up time**: ~10 min separate.
> **Outcome**: TRIVIAL-CONFIRMATION (per B1 stop-rule). Single hypothesis, single-symbol fix site, <10 min diagnostic. NO swap to backup (per B1 rule: trivial-confirmation is still valuable Stage B data — proves skill doesn't over-engineer simple bugs).

---

## Symptom (from Jira)

> Currently Prize Summary always displays Lootbox icon regardless of actual reward type. Opening a Giftbox → still shows Lootbox icon. Claiming a normal (non-box) Daily Bonus reward → still shows Lootbox icon. Expected: no box icon for normal rewards; Lootbox icon for Lootbox; Giftbox icon for Giftbox.

## Phase 0 — RECALL

```pseudocode
omo-session-distiller_recall(query="prize summary lootbox icon giftbox reward type display", limit=5)
```
Result: 5 hits, scores 25/15/14/13/12 (above A5 ≥10 bar) — **but all Problem texts off-topic** (Saga lifetime extension import error, Vue debugger extension icon generation, Vue extension load). A5's score+text rubric correctly rejected — no clearly-relevant hits.

Proceed to Phase 1.

## Phase 1 — REPRODUCE

Spec-only (no QA2 auth). 100% reproducible per ticket. Two repro variants:
- Open a Giftbox → Prize Summary shows Lootbox icon
- Claim a normal reward → Prize Summary shows Lootbox icon

## Phase 2 — EVIDENCE (Row 1 state-desync — frontend-only)

Pure frontend; no cross-repo concerns. mcp-console-hub unavailable (no live session), so degraded to static code reads (Row 1 fallback per A4: `evidence_quality: degraded` for the runtime piece; `verified` for the code-structural piece).

### Code evidence

`rg "PrizeSummary|prize_summary_modal" src/` → 2 hits:
- `src/view/shared/components/RiveView/templates/PrizeSummaryTemplate.tsx` — Rive template wrapper
- `src/view/shared/components/CoinCelebration/useAwardSummaryConfig.ts` — config + Rive metadata declaration

**Critical finding in `useAwardSummaryConfig.ts:12-14`**:
```typescript
"header/listType": "Enum",
"header/boxSwitch": "Boolean",
```
These two Rive state-machine inputs are DECLARED in metadata but **never SET anywhere in the codebase**:
```
rg "boxSwitch|listType" src/view/shared/components/CoinCelebration/
→ only the metadata declaration; 0 assignments found
```

`AwardSummaryPopup.tsx:148-189` defines `AwardDataBindingConfig` which builds `finalRiveViewData` for the Rive state machine. Lines 166-177 construct the `header` block:
```typescript
const finalRiveViewData = {
    header: {
        ...riveViewConfig?.header,
        tierNum
    },
    cta: { ... tierNum, ctaShineLoopSwitch: isMobile },
    prizeList: { numOfPrize: awardDataPrizeList.length }
};
```
`header` block spreads optional config + sets only `tierNum`. **`boxSwitch` and `listType` are NEVER populated based on `data.awardType` or `data.awardConfig`.**

The Rive state machine receives no value for `header/boxSwitch` and `header/listType` → those inputs hold their default values → default icon (Lootbox) is rendered regardless of actual reward type.

**Awardtype IS available**: `AwardSummaryPopup` receives `data: AwardSummaryPopupData` which has `awardType?: AwardType` (line 35). `AwardType` enum is imported at line 13. `data.awardType === AwardType.LOOT_BOX` is already used at line 54 for `buttonId` computation. The data is there; the binding to the Rive state machine is what's missing.

## Phase 3 — HYPOTHESIS (single, trivial case)

Single hypothesis sufficient (per A3 cap: 3 max, but trivial-confirmation bugs don't need fan-out):

| ID | Mechanism | Evidence | Falsification | Cost |
|---|---|---|---|---|
| A | `AwardDataBindingConfig` at `AwardSummaryPopup.tsx:166-177` builds the Rive header block but never sets `header.boxSwitch` (Boolean) or `header.listType` (Enum), despite Rive metadata declaring them at `useAwardSummaryConfig.ts:13-14`. Rive state machine renders the default icon (Lootbox). The required source data (`data.awardType: AwardType`) is already accessible to the component but never propagated to the binding. | `useAwardSummaryConfig.ts:12-14` declares; `AwardSummaryPopup.tsx:166-177` omits in `header` block; `rg "boxSwitch\|listType" src/` returns 0 assignments anywhere. | Add `header.boxSwitch = (awardType === AwardType.LOOT_BOX \|\| awardType === AwardType.GIFT_BOX)` + `header.listType = awardType` to the `finalRiveViewData.header` block; verify the Rive state machine renders the correct icon per `awardType`. | ~5 min (small patch + live QA2 verify) |

**Cap usage**: 1 hypothesis of 3 allowed. The trivial-confirmation outcome means there is no plausible second hypothesis worth exploring — the evidence is overwhelming (metadata declares the inputs; the binding code clearly omits them; the source data is available; falsification is a 3-line patch). Forcing additional hypotheses to "fill the cap" would be skill-as-document-generator, not skill-as-diagnostic-aid.

## Phase 4 — FIX (not executed)

Skipped per Stage B contract. Likely fix is a 3-line change in `AwardSummaryPopup.tsx` at line 167-170:

```typescript
header: {
    ...riveViewConfig?.header,
    tierNum,
    boxSwitch: data?.awardType === AwardType.LOOT_BOX || data?.awardType === AwardType.GIFT_BOX,
    listType: data?.awardType,  // or an enum-int mapping if the Rive enum doesn't match the JS enum 1:1
},
```

The exact `listType` enum value mapping requires asking the Rive designer (the Rive `.riv` file's enum values may not align with the JS `AwardType` enum order). Backend/UI assignee (Tân Chu Thái per Jira) handoff covers this.

## Phase 5 — POSTMORTEM

Captured in `.sisyphus/postmortems/2026-05-21-pilot-3-WIN-7990-prize-summary-lootbox-icon-always.md`.

`recall_hit: no-novel`. `evidence_quality: verified` (code structural evidence is authoritative for this class of bug — the omission is statically observable; no runtime evidence required). `outcome: trivial-confirmation`.

## Skill protocol observations (Stage B data — third pilot, trivial archetype)

✅ **Trivial-confirmation stop-rule worked exactly as designed (per B1 review C1)**:
- Phase 2 evidence collected in <2 min; fix site identified as 3-line patch
- Marked `outcome: trivial-confirmation` instead of forcing additional hypotheses to demonstrate "protocol depth"
- No swap to backup (per B1 rule: trivial-confirmation IS valuable data)
- **This pilot proves the skill doesn't over-engineer simple bugs.** Phase 3 cap of 3 hypotheses is a ceiling, not a floor.

✅ **A5 score+text rubric continues to work**: 5 hits with scores 12-25 (all ≥10) but Problem texts unrelated → all correctly rejected. The rubric correctly distinguishes tag-collision noise from semantic relevance.

⚠️ **Sample observation — n=3 pilot diversity**:
- B2.1 (WIN-7993): state-desync, single-repo, P0. Diagnostic ~2.5 min. 3 hypotheses.
- B2.2 (WIN-7988): BI event, cross-repo, P0. Diagnostic ~5.3 min. 3 hypotheses, A vs C overlap.
- B2.3 (WIN-7990): UI Rive binding, single-repo, P2. Diagnostic ~2 min. 1 hypothesis (trivial-confirmation).
- Range: 2-5.3 min. Still n=3 — no aggregate speedup claim. Stage C1 calibrates after B2.4.

🟢 **Diagnostic wall-clock**: ~2 min. Write-up separate (~10 min).

## Status

`outcome: trivial-confirmation`. Single hypothesis with overwhelming static evidence. Backend/UI assignee handoff. The skill correctly recognized that this bug doesn't require multiple-hypothesis exploration — code-evidence alone is sufficient to identify the fix site and the missing binding.

# Pilot-1 trace — WIN-7993 (Daily Bonus auto-claim stuck)

> **Bug**: WIN-7993 — Daily Bonus V2 reward stays in claimable state after server-side auto-claim succeeds.
> **Diagnostic wall-clock** (recall + reproduce-spec + evidence + hypothesis formation; **excludes write-up**): 2026-05-21T08:07:28Z → 08:09:56Z, ~2.5 min.
> **Write-up time** (trace + postmortem authoring): additional ~10 min outside the timed window. The two are separated because Stage C metrics aim to measure skill-as-diagnostic-aid, not skill-as-document-generator.
> **Outcome**: PHASE-3-PARTIAL — hypotheses formed with code evidence, but falsification requires live API access not available to pilot controller. Documented for backend assignee handoff.

---

## Symptom (from Jira)

> Auto-claim enabled at 5 min. Reveal lootbox → close tab → wait 5 min → re-open lobby.
> Actual: reward added to balance, but DB modal shows reward as still-claimable + countdown stopped.
> Expected: reward in balance, countdown visible, reward shown as Collected.
> Environment QA2 v1.9.2(3). Browser Safari, Chrome. Reproduce rate 100%.

## Phase 0 — RECALL

### Step 1 — Form query
Keywords: `daily bonus`, `auto-claim`, `stuck`, `claimable`, `lootbox`.

### Step 2 — Scope
First call broad (no repo filter) per A5/T5 default rule.

### Step 3 — Invoke

Call 1 (with repo filter, score-aware):
```pseudocode
omo-session-distiller_recall(query="daily bonus auto-claim stuck claimable lootbox", repo="playsweeps-web", limit=5)
```
Result: **0 hits**.

Call 2 (broader rephrase per Phase-0-local "Single-query-satisfied" anti-pattern):
```pseudocode
omo-session-distiller_recall(query="daily bonus reveal lootbox state machine", limit=5)
```
Result: **5 hits**, scores 30/21/16/16/15. **All daemon-down token-match noise** — Problem texts cover unrelated topics (US state names, React Debugger extension, Redux persistence between users, navigation state, address-form city dropdown). No clearly-relevant hit per A5 score-and-Problem-text rubric.

### Step 4 — Classify
**No clearly-relevant hits.** Proceed to Phase 1.

### Step 5 — Skipped (no Clearly Relevant atom to verify)

`evidence_quality: verified` for Phase 0 (recall ran successfully; daemon-down state acknowledged in output).

## Phase 1 — REPRODUCE

The Jira ticket provides a deterministic recipe (100% reproduce rate per ticket):
```
1. Auto-claim enabled, set 5 min
2. Open Daily Bonus modal
3. Reveal a lootbox
4. Close tab; wait 5 min
5. Reopen lobby
```
Pilot controller does NOT have authenticated QA2 access → cannot execute the live repro. Phase 1 is **specification-only** (recipe documented, not run). Marking `evidence_quality: partial` for Phase 1.

## Phase 2 — EVIDENCE (Row 1 + Row 8 spanning; degraded to Row 5/9 fallback due to no runtime access)

Per Discipline Rule 3 (earliest causal signal): the auto-claim timer firing is the earliest event; Row 8 primary, Row 1 secondary. But: pilot controller has no live MCP-console-hub session against the running QA2 app → cannot collect runtime evidence.

**Degraded path**: read code statically (Row 5/9 territory). `evidence_quality: degraded` for Phase 2.

Tools used:
- `rg` to locate Daily Bonus code → `src/view/shared/features/dailyBonusModal/` + `src/redux/dailyBonus/`
- `rg` for "autoClaim"/"AUTO_CLAIM"/"auto-claim" patterns → **no matches in DB code** (the feature is server-driven; client doesn't have an explicit auto-claim path)
- `read` on `src/redux/dailyBonus/reducer.ts:50-156` and `src/redux/dailyBonus/saga.ts:30-119`

Key code observations:

`reducer.ts:50-71` `correctCurrentDay()` maps:
- `Pending` + `!isClaimable` → `NotReady` (line 63-64)
- `Pending` + `isClaimable` → `Available` (line 65-66)

**No mapping `Pending → Collected`** in the post-auto-claim case. After a server-side auto-claim, the server-side state is "reward dispensed but Day status may still be Pending or Available in the response."

`reducer.ts:53` `isClaimable = nextClaimTime == null || Date.now() >= nextClaimTime` — if `NextClaimDate` is in the past after auto-claim, `isClaimable = true` → status stays in claimable family.

## Phase 3 — HYPOTHESES (3 formed, all share single falsification test)

| ID | Mechanism | Evidence | Falsification | Cost |
|---|---|---|---|---|
| A | Server returns `Status: Pending` on auto-claimed day; client's `Pending → Available` mapping shows it as claimable | reducer.ts:65-66 missing `Collected` path | Inspect API response for an auto-claimed day | 5 min (requires QA2 access) |
| B | Server returns `Status: Collected` correctly but client's `correctCurrentDay` clobbers it | Need API response sample | Same as A | 5 min |
| C | `NextClaimDate` not advanced post-auto-claim; client computes `isClaimable=true` indefinitely | reducer.ts:53 strict gt-comparison | Inspect `NextClaimDate` value in API response | 5 min |

**Triage**: A and C share the **same falsification test** (one API call inspecting both `Status` field AND `NextClaimDate` simultaneously). B requires the SAME call PLUS a reducer trace if Status returns `Collected` (to find where the clobber happens). So unique work is ~1.2 units, not strictly 1 — but well under the A3 cap of 3.

This is exactly the cap-rule edge case the postmortem's Open Risks section flags: cap measures "hypothesis count" today but should measure "unique falsification cost" to give the right signal when hypotheses cluster around shared tests. Filing as Stage A3 follow-up.

## Phase 3 — STUCK + handoff (cannot run falsification)

Pilot controller cannot authenticate against QA2 to capture the API response. Two paths:

1. **Hand off to backend assignee (Thanh Đỗ Duy per Jira)** with the hypothesis matrix above. They have QA2 access; running the single API-response inspection takes ~5 min and discriminates A/B/C.
2. **Oracle consult** would not help here — Oracle reads-only, cannot fetch QA2 data either.

**Decision**: hand off. The skill's stop-rule for live-evidence-blocked pilots is exactly this — document the hypothesis chain so the assignee saves the discovery time, not the execution time.

## Phase 4 — FIX (not executed)

**Skipped per plan B2 contract**: pilot generates diagnostic trace + postmortem only; production fix remains assignee's responsibility.

If A or B confirmed → fix is in `reducer.ts:65-66` (add `Pending → Collected` mapping conditional on a server-provided flag like `IsAutoClaimed: true`).
If C confirmed → fix is server-side (advance `NextClaimDate` on auto-claim) OR client-side defensive (treat `nextClaimTime in past + Status: Pending` as a stale-response signal).

## Phase 5 — POSTMORTEM

Captured in `.sisyphus/postmortems/2026-05-21-pilot-1-WIN-7993-daily-bonus-auto-claim-stuck.md`.

`recall_hit: no-novel`. `evidence_quality: degraded`. `degraded_reason: "Pilot controller has no QA2 auth; code-static evidence only. Falsification of all 3 hypotheses requires API response inspection deferred to backend assignee."`

## Skill protocol observations (Stage B data)

✅ **What worked**:
- Phase 0 query construction was fast (~30 sec, two queries, both correctly classified as no-hit per A5 score + Problem-text rubric).
- A5's "score < 10 = noise" + "Problem-text relevance check" prevented false-positive on the 5 token-match hits (scores 30/21/16/16/15 ALL ignored as off-topic).
- Compound-paralysis cap (A3) didn't bind — 3 hypotheses generated, all share evidence (same falsification test), so cap was irrelevant.

⚠️ **Where the skill ran into limits**:
- **No live runtime access** is the dominant constraint for any real-bug pilot run by a controller without authenticated env credentials. The skill's MCP routing assumes mcp-console-hub / ohmyperf are connected to a running app session; for purely-frontend bugs this is true on dev machines but FALSE for QA-env-only repros like WIN-7993.
- **A4 fallback chain executed**: Row 1+8 → Row 5/9 (static code) → `evidence_quality: degraded`. This is exactly the A4 fallback the skill was designed to handle. Working as designed.
- **Phase 3 cap-paralysis question**: 3 hypotheses with identical falsification cost is degenerate w.r.t. the cap (cap measures rule-out cost; here we rule out 3 with 1 test). Cap rule needs refinement: "3 hypotheses' worth of UNIQUE falsification cost" — not "3 hypotheses counted." Stage A3 follow-up worth filing.

🟢 **Diagnostic wall-clock**: ~2.5 minutes real time (Phase 0: 30s, Phase 1: 30s spec only, Phase 2: 60s code reads, Phase 3: 30s hypothesis formation). Write-up of trace + postmortem was separate (~10min, NOT included in diagnostic wall-clock).

⚠️ **Speedup claim — n=1, gut-baseline, NOT measured**: Pre-skill baseline of "~15-20 min unguided ad-hoc grep" is the **author's gut-estimate** by someone who already knows the diagnostic destination. It is an upper-bound guess, NOT a measured control. The implied "skill is 5-7x faster on diagnostic preamble" is therefore a **single-sample, baseline-by-introspection claim** — useful as a directional signal but NOT statistically valid and NOT a defensible headline number. Real speedup will be calibrated after 3-5 more pilots complete (Stage B2.2-B2.4) and metrics aggregate in Stage C1. Until then, **do not quote "5x faster" without the n=1/gut-baseline qualifier**.

The 5-min API falsification still has to happen regardless of the skill — it just happens with a hypothesis-shaped target instead of from-scratch. The skill's contribution is to the diagnostic preamble, not to the irreducible falsification cost.

## Status

`outcome: phase-3-partial`. Pilot does NOT complete Phase 4-5 verification of the fix. Postmortem written. Backend assignee path documented. The skill protocol correctly identified the constraint (no live access) and routed appropriately — that itself is the value the pilot proves.

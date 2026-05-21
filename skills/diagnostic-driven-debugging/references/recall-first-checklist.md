# Phase 0 — Recall-First Checklist

> The single most cost-effective phase in the skill. A 5-second query that, when it hits, can replace 30+ minutes of investigation. **Non-skippable.**

This file is the operational guide for invoking `omo-session-distiller_recall` at the start of every debug session. The output of Phase 0 determines whether you can fast-path to Phase 4 (apply known fix) or must continue to Phase 1 (full investigation).

---

## Why Phase 0 Comes First

LLM agents have **no persistent memory between sessions** by default. Without recall, you re-debug the same class of bug every time you encounter it — even if past-you already solved it last week. `omo-session-distiller` harvests prior sessions into searchable knowledge atoms; `recall` searches them.

The compound effect is the entire point: every postmortem (Phase 5) you write becomes a future Phase 0 hit. Skip Phase 0 → break the loop → debug from zero forever.

---

## The Checklist

### Step 1 — Form the query (one chance to get keywords right)

The query is a natural-language description of the bug. Aim for **3-7 keywords** that capture the bug's distinguishing features:

- ✅ **Good**: `"tournament leaderboard renders empty after pod switch"`
- ✅ **Good**: `"refresh token returns 401 only on Safari"`
- ✅ **Good**: `"webpack module not found after lockfile bump"`
- ❌ **Too vague**: `"bug in tournament"` (will match anything)
- ❌ **Too narrow**: `"line 142 of tournament.ts undefined"` (recall won't have line numbers as keywords)
- ❌ **Includes session context**: `"the bug I just saw"` (recall is keyword-based, not session-aware)

**Construction rules**:
1. Include the **affected feature/module** name (`tournament`, `leaderboard`, `MFA`, `Skrill`)
2. Include the **symptom verb** (`renders empty`, `returns 401`, `module not found`, `hangs after`)
3. Include a **triggering condition** if known (`after pod switch`, `on Safari`, `only on staging`)
4. Avoid filler words ("the", "a", "issue", "problem", "thing")
5. Avoid file paths and line numbers
6. Avoid implementation details you haven't verified are causal

### Step 2 — Choose the scope

`omo-session-distiller_recall` accepts an optional `repo` parameter. **Default rule: start broad, narrow only on re-query.**

- **First Phase 0 query**: omit `repo` to search all atoms (broad sweep — you may discover the same bug in a sibling repo)
- **If too many irrelevant hits** OR **the first query returned 0 hits and you're confident which repo it's in**: re-query with `repo="<repo-slug>"` (narrow sweep)
- **Cross-cutting bugs that touch multiple repos**: stay broad; never filter

### Step 3 — Invoke

This is an MCP tool invocation (not Python). The agent passes structured arguments to the `omo-session-distiller_recall` tool. Shown below in kwarg-style pseudocode:

```pseudocode
omo-session-distiller_recall(
    query="tournament leaderboard renders empty after pod switch",
    # repo: omit on first query (broad sweep). Add for re-query if needed.
    limit=5   # 5 is usually enough; 3 if you trust the query
)
```

### Step 4 — Interpret the response

The tool returns ranked atom hits, each with a Problem statement and a Resolution. There is **no numeric score field** — ranking is positional only. **The first hit is not guaranteed to be the best match**; always read the Problem text of the top 2-3 hits before deciding. Use this rubric:

| Relevance | Indicator | Action |
|---|---|---|
| **Clearly relevant** (tentative — confirm in Step 5) | Top hit's Problem text describes a symptom that closely matches yours, with overlap in affected feature + symptom verb + (ideally) triggering condition | Read the full Resolution. **Do not fast-path yet** — proceed to Step 5 to verify the prior fix is still in code. Step 5 confirms or demotes this classification. |
| **Partially relevant** | Hit's Problem touches the feature but the symptom or condition differs | Read it as context, but **do not apply blindly**. Use it to inform Phase 3 hypothesis generation. Continue to Phase 1. |
| **No clearly-relevant hits** | All hits are on different features or unrelated symptoms | Proceed to Phase 1. Note in your Phase 5 postmortem that recall produced no useful hits (so future queries can learn from this gap). |

### Step 5 — Verify the prior fix is still present (confirms Clearly-Relevant tentative)

This step **finalizes the Step 4 classification**. Tentative "Clearly Relevant" hits become either confirmed (fast-path) or demoted to "Partially Relevant" (continue to Phase 1) based on whether the prior fix is still in the code.

If you got a Clearly Relevant hit and intend to fast-path:

1. **`expand` the atom** to get the full archive context if Resolution is too brief:
   ```pseudocode
   omo-session-distiller_expand(atom_id="<atom-id-or-slug>", around=2)
   ```
2. **Verify the previously-applied fix is still in the code**. Bug regressions happen — the prior fix may have been refactored away. Use `lsp_find_references` or `rtk grep` on the key symbols from the Resolution.
3. **Verify the prior fix's regression test exists AND currently passes.** The fix's presence in code is necessary but not sufficient — the test that proved it works must also be present and green. Steps:
   - From the atom's Resolution, identify the test name(s) added when the original fix landed (atoms typically cite test paths like `tests/integration/foo.test.ts` or test names like `it('does X correctly')`)
   - `rtk grep` for the test name(s) — confirm the file/case exists
   - Run the test (or check CI for its last green status) — confirm it passes RIGHT NOW, not historically
   - If test missing → the prior fix's invariant is no longer protected; the atom's resolution is **suspect**. Demote classification to Partially Relevant. Continue to Phase 1 — do NOT fast-path. The atom may be misdiagnosed or its fix may have rotted; treating it as Clearly Relevant could propagate a wrong fix.
   - If test exists but currently fails → there IS a regression at the same site, but the atom's prescribed fix may not address this variant. Demote to Partially Relevant; use atom as Phase 3 hypothesis seed.
   - If test exists AND passes → proceed to step 4.
4. If the fix is **still present** AND its regression test exists+passes (step 3 green): **demote the classification to Partially Relevant**. This is NOT the same bug — the prior fix didn't cover this case (otherwise the test would catch it). The atom remains useful as a Phase 3 hypothesis seed. Continue to Phase 1.
5. If the fix is **gone**: classification stays Clearly Relevant — that's likely the bug. Re-apply with appropriate adaptation, then continue to Phase 5 (still write the postmortem; this regression is itself worth recording).

**Why step 3 matters**: without it, the fast-path propagates wrong fixes faster. Step 2 alone only checks "is the patch in code?" but a patch can be present yet unprotected — if the regression test was deleted or skipped, future-self has no signal that the fix actually works. Step 3 closes this loophole.

---

## Examples

### Example 1 — Clear hit, fast-path

**Symptom**: "Pod-2 tournament tab shows empty leaderboard after I switch from Pod-1."

**Phase 0** (broad sweep first per Step 2 rule):
```pseudocode
omo-session-distiller_recall(
    query="tournament leaderboard empty after pod switch"
    # no repo filter on first query
)
```

**Hypothetical response**:
> Hit 1 (atom-2026-03-15-tournament-saga-race):
> - Problem: "Tournament leaderboard renders empty when pod context changes mid-fetch. The SET_POD action dispatches before FETCH_LEADERBOARD completes, so the reducer overwrites with stale data."
> - Resolution: "Added `take('FETCH_LEADERBOARD_SUCCESS')` blocking effect in tournamentSaga.ts before SET_POD dispatch. Commit a1b2c3d."

**Action**: Verify `take('FETCH_LEADERBOARD_SUCCESS')` is still in tournamentSaga.ts. If yes → not the same bug, continue to Phase 1. If no (refactored away) → restore it with adaptation, run Phase 1 repro, write Phase 5 postmortem referencing atom-2026-03-15.

### Example 2 — Partial hit, informs Phase 3

**Symptom**: "Tournament prize calculation off-by-one for high-roller users."

**Phase 0** (broad sweep first; this example narrowed because the broad query returned 30+ unrelated hits):
```pseudocode
# First (broad) call returned too many irrelevant hits → re-query narrowed:
omo-session-distiller_recall(
    query="tournament prize calculation off-by-one",
    repo="playsweeps-backend"
)
```

**Hypothetical response**:
> Hit 1 (atom-2026-01-22-prize-rounding):
> - Problem: "Prize amounts off by 1 cent due to floating-point rounding in PrizeCalculator.cs."
> - Resolution: "Switched from `double` to `decimal` for currency math."

**Action**: Different symptom (rounding vs. off-by-one in count). The atom is *related* (prize calc bug) but not the same. Read for context (knowing currency math has prior precision issues), then proceed to Phase 1 expecting either a different bug or a regression of that earlier fix.

### Example 3 — No hit, continue to Phase 1

**Symptom**: "Skrill webhook handler fails to process refunds on Tuesdays."

**Phase 0**:
```pseudocode
omo-session-distiller_recall(
    query="Skrill webhook refund Tuesday"
)
```

**Hypothetical response**: 0 hits.

**Action**: Note "Phase 0 recall: no hits" in the eventual Phase 5 postmortem. Proceed to Phase 1.

---

## Escape Hatch — When omo-session-distiller Is Empty

If `omo-session-distiller_stats` shows that no sessions have been harvested for this repo / workspace yet:

1. **Do not block the debug session** waiting for harvest. Phase 0 yields "no hits" — that's a valid outcome.
2. **Note it explicitly** in the Phase 5 postmortem: "Phase 0 skipped: distiller had no atoms for repo X."
3. **Consider running** `omo-session-distiller_harvest()` (synchronously, or in background) at the START of the session. After it finishes, re-run Phase 0. The harvest may surface relevant atoms from sessions you forgot you had.

The escape hatch exists so a missing memory layer doesn't paralyze the protocol — but the absence is itself data worth recording.

---

## Phase-0-Local Anti-Patterns

> These are **Phase 0 local** — they do not appear in the master `references/anti-patterns.md` list. The master file's AP-5 ("Skip Phase 0") is the global counterpart; the two below specialize it.

- **AP-5 (Skip Phase 0)** — see master list. "This bug is obviously new, no point checking." Bug claims of novelty are wrong roughly 30% of the time. The 5-second cost is always worth paying.
- **Phase-0-local: Over-narrow query** — including line numbers, exact error text byte-for-byte, or session-specific context. Recall is keyword-based; surface-level descriptions match better than transcripts.
- **Phase-0-local: Single-query-satisfied** — if the first query returns 0 hits, try ONE rephrasing before declaring "no hits." Different vocabulary (e.g. "leaderboard empty" vs "tournament rows missing") can hit different atoms.

---

## When Phase 0 Saves Time vs. Costs Time

| Scenario | Phase 0 outcome | Net time saved |
|---|---|---|
| Bug previously solved by current agent | Clearly relevant hit | 25-60 minutes |
| Bug previously solved by a teammate (atom harvested) | Clearly relevant hit | 30+ minutes + spec context |
| Similar bug class, different surface | Partially relevant hit | 5-15 minutes (hypothesis seed) |
| Truly novel bug | No hits | -5 seconds (the cost of the query itself) |
| Repo has no harvested atoms | Empty result | -5 seconds + harvest decision |

Worst case: 5 seconds spent. Best case: 60 minutes saved. The expected value is overwhelmingly positive — which is why Phase 0 is non-skippable.

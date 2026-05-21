# T8 — Dry-Run Test Trace

> **Purpose**: Apply `diagnostic-driven-debugging` end-to-end (Phase 0 → Phase 5) to a fabricated but representative bug, narrating every decision the skill imposes. This catches design flaws BEFORE shipping the skill to real bugs. **Every first version has flaws — the goal of T8 is to find them.**
>
> Defects discovered here roll into T9 fix tasks. The skill is not "done" until a second dry-run (T10) executes cleanly with no protocol crashes.

---

## Test Scenario

**Repo**: `playsweeps-web` (React 18, Redux-Saga, MUI 6, Webpack)

**Bug report** (verbatim from a fictional Slack message):

> "Hey team — Tournament leaderboard is showing wrong values for me. I'm in Pod 2 looking at the Daily Tournament, but the rows look like Pod 1's data. If I refresh the page it's fine for a few seconds then goes weird again. Started after the v1.10 staging deploy. Repro on Chrome 124 macOS."

**User-visible symptom**: Pod 2 tournament shows Pod 1's leaderboard data, intermittently, on Chrome 124 macOS, staging environment, started v1.10.

---

## Phase 0 — RECALL

### Step 1 — Form the query

Per `recall-first-checklist.md` Step 1:
- Affected feature: `tournament leaderboard`
- Symptom verb: `shows wrong / Pod 1 data when in Pod 2`
- Triggering condition: `intermittent, after v1.10 deploy`

Constructed query: `"tournament leaderboard wrong pod data intermittent"`

### Step 2 — Choose the scope

Per fixed default rule (Step 2): start broad — no `repo` filter on first query.

### Step 3 — Invoke

```pseudocode
omo-session-distiller_recall(
    query="tournament leaderboard wrong pod data intermittent",
    limit=5
)
```

### Step 4 — Interpret response

**Hypothetical response** (the test imagines a recall hit):

> Hit 1 (atom-2026-03-15-tournament-saga-race):
> - **Problem**: "Tournament leaderboard renders empty/stale when pod context changes mid-fetch. The SET_POD action dispatches before FETCH_LEADERBOARD completes; the in-flight Pod-1 SUCCESS payload writes into the cleared-for-Pod-2 slice."
> - **Resolution**: "Added `take('FETCH_LEADERBOARD_SUCCESS')` blocking effect in tournamentSaga.ts before SET_POD dispatch. Commit a1b2c3d."

Per rubric: **Clearly relevant** — same feature, same symptom class (wrong/stale data after pod context change). Top hit's Problem text overlaps strongly with the current symptom.

### Step 5 — Verify prior fix is still present

Per checklist Step 5, must verify the fix is still in the code before fast-pathing.

Action: `lsp_find_references` on the `tournamentSaga.ts` file, search for `take('FETCH_LEADERBOARD_SUCCESS')`.

**Hypothetical result**: the `take()` call is **still present** at the expected location.

→ Per Step 5 rule: "If the fix is **still present**: this is NOT the same bug — the prior fix didn't cover this case. Continue to Phase 1."

**Phase 0 outcome**: Partial hit (clear atom, but prior fix is intact → bug is a related-but-different mechanism). Proceed to Phase 1 with the atom as Phase 3 hypothesis seed.

> 🚨 **DEFECT-1 found**: The `recall-first-checklist.md` Step 4 rubric maps "clearly relevant" to "may fast-path to Phase 4". But Step 5 says "if prior fix is still present, continue to Phase 1." These two instructions create an ambiguity: am I at "clearly relevant" or "partially relevant" once Step 5 returns "fix still present"?
>
> **Proposed fix (rolls to T9)**: Add a Step 4.5 in `recall-first-checklist.md`: "Tentative classification — confirm in Step 5." Then Step 5 makes the final classification: if prior fix gone → fast-path (Clearly Relevant); if prior fix present → demote to Partially Relevant and proceed to Phase 1.

---

## Phase 1 — REPRODUCE

Per SKILL.md Phase 1: a single command that triggers the bug 100% of the time.

The report says "intermittent" + "Started after v1.10 staging deploy". Per MCP routing table Row 8 (race/timing): intermittent IS the bug.

Constructed repro (hypothetical):
```
1. Open https://staging.playsweeps/tournaments (loads in Pod 1 by default)
2. Wait for first leaderboard load to complete (~800-1500ms variable)
3. Click "Pod 2" tab IMMEDIATELY after the page becomes interactive
4. Click "Pod 1" tab → click "Pod 2" again (alternating, 3-5 toggles within ~3s)
5. Observe: 40-60% of toggle cycles end with Pod 2 view showing Pod 1's rows
```

Repro is non-deterministic (40-60%) but reliable in distribution. Per Phase 1 guidance: "Intermittent? Find a minimum repro that fails ≥ 50%; if truly random, treat it as the bug to debug."

40-60% straddles the 50% threshold. **Decision**: proceed; this is reliable-enough to investigate.

> 🚨 **DEFECT-2 found**: SKILL.md Phase 1 says "≥ 50%" but my real repro is 40-60%. The threshold creates a discrete cliff: 49% says "not reliable enough" but 51% says "go ahead", when in practice these are indistinguishable noise. Also, the 50% number is unsourced (cf. T4 review Comment on "30 tool calls" being unsourced).
>
> **Proposed fix (rolls to T9)**: Soften Phase 1 wording. Replace "fails ≥ 50%" with "fails often enough that you can collect Phase 2 evidence within ~5 repro attempts." Removes the spurious precision and ties the threshold to the next action.

---

## Phase 2 — EVIDENCE

Per MCP routing table:
- Symptom = "wrong / stale data after rapid pod switching" → **Row 1 (frontend state desync)** AND **Row 8 (race/timing)**.

Per Discipline Rule 3 (after T3 fixes): "if symptom truly spans rows, route to the row whose primary tool produces the earliest causal signal in the event timeline." Row 8's `get_timeline` produces the earliest causal signal (event ordering), so Row 8 is primary; Row 1 is secondary.

**Dispatch in parallel** (per Phase 2 guidance "in parallel where possible"):

```pseudocode
# Primary (Row 8): event timeline
mcp-console-hub_get_timeline(limit=200)

# Primary supporting (Row 8 strategy): variance signal
ohmyperf_measure(url="https://staging/tournaments", runs=10)

# Secondary (Row 1): state slice during the wrong-render window
mcp-console-hub_get_redux_key(keyPath="tournament.leaderboard.items")
mcp-console-hub_get_redux_key(keyPath="tournament.activePodId")
mcp-console-hub_query_console(level="error", since=<repro-start-ts>)
```

**Hypothetical evidence bundle** (cite specific field/value per Discipline Rule 2):
- `get_timeline`: showed action ordering on a wrong-render cycle —
  - t=1580ms: `SET_POD` payload=`{podId: 2}`
  - t=1582ms: `FETCH_LEADERBOARD_REQUEST` payload=`{podId: 2}`
  - t=1640ms: `SET_POD` payload=`{podId: 1}` (user toggled back)
  - t=1642ms: `FETCH_LEADERBOARD_REQUEST` payload=`{podId: 1}` SUPPRESSED (dedupe middleware — identical action shape to in-flight Pod-2 request because dedupe keys by action.type, not payload)
  - t=1700ms: `SET_POD` payload=`{podId: 2}` (user toggles forward again)
  - t=1702ms: `FETCH_LEADERBOARD_REQUEST` payload=`{podId: 2}` SUPPRESSED again
  - t=2390ms: `FETCH_LEADERBOARD_SUCCESS` payload=`{podId: 1, items: [Pod 1 rows]}` (the FIRST Pod-1 request from t=1642 finally returns)
  - t=2392ms: leaderboard reducer writes Pod-1 rows; `activePodId` is now 2 → wrong-data render.
- `ohmyperf_measure` runs=10: LCP variance 1.2s–2.4s; INP p95 = 380ms. Consistent with race window opening on fast network.
- `get_redux_key(tournament.leaderboard.items)` at t=2400ms: `[Pod-1 rows]`. `get_redux_key(tournament.activePodId)`: `2`. State mismatch confirmed.
- `query_console`: no errors. Bug is silent state corruption.

Files implicated:
- `src/redux/middleware/dedupe.ts` — suppression logic (dedupe by `action.type` ignoring payload)
- `src/redux/sagas/tournament.ts:142` — the existing `take('FETCH_LEADERBOARD_SUCCESS')` fix from the prior atom; verify it actually fires when SET_POD repeats
- `src/redux/reducers/leaderboard.ts:88` — the "if pod matches" guard

---

## Phase 3 — HYPOTHESIS

Per SKILL.md Phase 3: ≥ 2 hypotheses, each with mechanism + supporting evidence + falsification test.

### Hypothesis A: Dedupe middleware suppresses Pod-2 retry, then in-flight Pod-1 SUCCESS lands in Pod-2's slice

- **Mechanism**: After the user toggles Pod 2 → Pod 1 → Pod 2 rapidly, dedupe middleware (keyed by `action.type` only) sees identical `FETCH_LEADERBOARD_REQUEST` types and suppresses the second and third. The first request (which was for Pod 1, dispatched at t=1642) completes ~750ms later with Pod-1 data. The reducer writes those rows even though `activePodId` is now Pod 2.
- **Supporting evidence**: timeline shows `REQUEST SUPPRESSED` at t=1642 and t=1702; only one `SUCCESS` at t=2390 with `podId: 1`; `activePodId === 2` at SUCCESS time.
- **Falsification test**: Patch dedupe middleware to key by `(action.type, action.payload)` instead of `action.type` only. Re-run the repro. If wrong-render rate drops to ~0%, hypothesis confirmed.

### Hypothesis B: The reducer's "if pod matches" guard at leaderboard.ts:88 is bypassed

- **Mechanism**: The guard is supposed to drop SUCCESS payloads when `state.activePodId !== payload.podId`. If the guard is missing, malformed, or checking the wrong field, Pod-1 SUCCESS gets written into Pod-2 state.
- **Supporting evidence**: state mismatch confirmed at t=2400ms (activePodId=2, items=Pod-1 rows). If the guard worked, the items would have been dropped.
- **Falsification test**: `lsp_goto_definition` on leaderboard.ts:88; read the actual guard logic. If guard correctly checks `state.activePodId === payload.podId`, this hypothesis falsifies — the bug is upstream (Hypothesis A). If guard is missing/wrong, this hypothesis confirms.

### Hypothesis C: The take('FETCH_LEADERBOARD_SUCCESS') blocker (prior fix) doesn't handle re-entrant SET_POD

- **Mechanism**: The prior fix wraps SET_POD in a `take()` for the in-flight fetch. But if SET_POD fires WHILE the take is blocking (rapid toggle), the blocker only handles the first SET_POD; the second one slips past.
- **Supporting evidence**: Multiple SET_POD events in the timeline (1580, 1640, 1700) during a single fetch window.
- **Falsification test**: Read `tournamentSaga.ts:142-148`. If the take is in a `takeLatest` wrapper, re-entrance is handled and this hypothesis falsifies. If it's in `takeEvery` or a bare `yield take()`, this hypothesis is in play.

### Triage order

Per the implicit cost-vs-likelihood logic the skill teaches:
- B is cheapest (single LSP lookup, ~5s) → run first
- C is also cheap (single file read, ~10s) → run second
- A is most expensive (patch + redeploy + 10-repro statistical run) → run last

**B → C → A.**

> 🚨 **DEFECT-3 found**: SKILL.md Phase 3 says "test top hypothesis FIRST" — implying rank-by-likelihood. But the right triage is cost-vs-likelihood: cheap-falsification-first, *especially* when cheap tests rule out hypotheses for free. The skill should make this explicit.
>
> **Proposed fix (rolls to T9)**: Update SKILL.md Phase 3 box: "Test top hypothesis FIRST" → "Run the cheapest falsification test first, unless one hypothesis is overwhelmingly likely. The cheapest test that can rule out a hypothesis pays for itself even if it falsifies the most likely guess." Also mirror this in `prompts/oracle-root-cause.md` (which already does have the "TRIAGE ORDER" output with cost annotation — good).

---

## Phase 4 — FIX

Imagine I run B first: read leaderboard.ts:88. **Hypothetical result**: guard is correctly written but checks `payload.podId === state.podId` (which doesn't exist; should be `state.activePodId`). Hypothesis B confirms — the guard has a subtle field-name bug that makes it always pass through.

But wait — Hypothesis A is also true (dedupe middleware does suppress retries). The two hypotheses are not mutually exclusive; they're both root causes. Fixing only one would leave the bug:
- Fix only B (guard) → Pod-2 retry still suppressed; user sees stale Pod-1 data until they navigate away
- Fix only A (dedupe) → guard still buggy; if anything else writes Pod-1 SUCCESS into Pod-2 state, bug recurs

**Decision**: fix both, as one logical change ("close the cross-pod data leak"). Per Phase 4 hard rule: "minimal, root-cause-targeted, no incidental refactor."

> 🚨 **DEFECT-4 found**: SKILL.md Phase 3 says "CANNOT proceed to fix until ONE hypothesis is confirmed." But what about cases where TWO hypotheses are confirmed and both are causal (compound bugs)? The skill provides no guidance for compound root causes — defaulting to "test then fix one at a time" risks shipping a partial fix that masks the second cause.
>
> **Proposed fix (rolls to T9)**: Update Phase 3 box: "CANNOT proceed to fix until you've confirmed at least one hypothesis AND ruled out other still-plausible ones (especially other hypotheses sharing evidence with the confirmed one). Compound root causes exist; fixing only one of two causes is AP-8 with extra steps."

### Fix plan

- `src/redux/reducers/leaderboard.ts:88`:
  - Replace `payload.podId === state.podId` with `payload.podId === state.activePodId`
- `src/redux/middleware/dedupe.ts:N`:
  - Key dedupe by `(action.type, JSON.stringify(action.payload))` instead of `action.type` alone — at minimum for `FETCH_*_REQUEST` actions
- Regression test: `tests/integration/tournament-pod-rapid-toggle.test.ts`
  - RED first: drive 5 rapid pod toggles, assert final render's data matches final `activePodId`. Test fails on current code.
  - GREEN after fix: same test passes.

### Verification

- Re-run repro from Phase 1: wrong-render rate after fix → ~0% over 20 toggle cycles
- `lsp_diagnostics` clean on the 2 changed files
- Regression test passes

---

## Phase 5 — POSTMORTEM

Per `prompts/postmortem-template.md`, write `.sisyphus/postmortems/2026-05-21-tournament-pod-data-leak-on-rapid-toggle.md`:

```yaml
---
date: 2026-05-21
slug: tournament-pod-data-leak-on-rapid-toggle
repo: playsweeps-web
bug_class: race
recall_hit: yes-partial
fix_commit: <sha>
time_spent_minutes: 35
---
```

Body would cite:
- Bug Summary: "Pod 2 tournament leaderboard displays Pod 1 data intermittently after rapid Pod 1↔2 toggling. Two root causes."
- Root Cause: compound — dedupe middleware suppressed Pod-2 retries by action.type only, AND leaderboard reducer's pod-match guard checked the wrong field name (`state.podId` instead of `state.activePodId`)
- Evidence Sources Used: `get_timeline` (action ordering with timestamps + SUPPRESSED markers), `get_redux_key` (final state mismatch confirmed), `lsp_goto_definition` (revealed wrong field name)
- Phase 0 Recall Result: "Partial hit on atom-2026-03-15-tournament-saga-race. Same feature, related but distinct mechanism (prior atom was first-toggle race; this is rapid-retoggle dedupe + guard bug). Prior fix still present and functioning for its scenario."
- Prevention Notes:
  - "Dedupe middleware should key by action shape, not just action.type, for fetch actions."
  - "When adding reducer guards that compare to state fields, lint-check that the field name exists on the state shape (TypeScript could enforce this if we typed state slices more strictly)."
  - "Add fuzz-test for rapid context-switch (pod, user, tenant) toggling across all paginated data screens."

> ✅ Postmortem fits within 25-line target. Frontmatter fields all applicable. `recall_hit: yes-partial` correctly captures the "atom found, partial reuse" case.

---

## Defects Found (Summary — rolls into T9)

| ID | Phase | File | Defect | Severity |
|---|---|---|---|---|
| DEFECT-1 | Phase 0 | `references/recall-first-checklist.md` | Step 4 rubric and Step 5 verification create classification ambiguity (clearly-relevant + prior-fix-present is undefined) | Major — controller can stall in Phase 0 |
| DEFECT-2 | Phase 1 | `SKILL.md` | "Fails ≥ 50%" threshold is unsourced and creates a fake cliff | Minor — wording |
| DEFECT-3 | Phase 3 | `SKILL.md` | "Test top hypothesis FIRST" should be "cheapest falsification test first unless one hypothesis is overwhelmingly likely" | Major — wrong triage doubles average debug time |
| DEFECT-4 | Phase 3→4 | `SKILL.md` | No guidance for compound root causes; current wording suggests fix-then-test-next, which can mask one of two causes | Major — risks AP-8 (symptom fix) on compound bugs |

**Verdict**: The skill executes the dry-run cleanly end-to-end with 4 defects discovered. None are protocol-fatal; all are fixable in T9 with text edits to `SKILL.md` and `recall-first-checklist.md`.

The dry-run **also validated**:
- Phase 0 → Phase 5 sequencing works
- MCP routing table successfully resolved a spanning symptom (Row 1 + Row 8) via Discipline Rule 3's tie-breaker
- The "cite specific field/value" rule in Discipline Rule 2 produced a postmortem with actionable evidence
- The Phase 5 postmortem template's `recall_hit: yes-partial` enum value covers the "atom found, partial reuse" case correctly — fits the design
- The skill correctly funnels rapid-toggle / intermittent symptoms to Row 8 (race), which only exists because of T3 review Comment 1 (coverage gap fix) — validates that the T3 fix was the right call

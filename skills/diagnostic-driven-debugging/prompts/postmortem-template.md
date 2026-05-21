# Phase 5 — Postmortem Template

> **Used in**: Phase 5 (POSTMORTEM), after a fix is verified by re-running the Phase 1 reproduction.
> **Where it's saved**: `.sisyphus/postmortems/YYYY-MM-DD-<slug>.md` (project-local, committed with the fix).
> **Why it exists**: This file is the closing of the feedback loop. It becomes the source material that `omo-session-distiller` harvests into atoms — atoms that future Phase 0 RECALL queries will hit. **A debug session that skips this step ends with the bug fixed but the system unimproved.**

---

## Hard Rules for Postmortems

1. **Non-skippable.** Every debug session that reaches Phase 4 must produce one of these. Even when Phase 0 hit immediately and Phase 4 took 2 minutes — write it. The compound effect is the point.
2. **2 minutes is the target, 5 minutes is the ceiling.** If you spend longer, you're rationalizing past investigation, not documenting it.
3. **≤ 25 lines of actual content** (excluding section headers). Brevity is what makes future-you actually read it.
4. **Cite specific field/value, not the tool name.** Postmortems with "I ran `get_redux_state`" are useless. Postmortems with "`tournament.leaderboard.items.length === 0 at t=1620ms`" are gold.
5. **Sections are fixed.** Don't add or rename — the harvester expects this shape. Leave a section empty (with a `—`) if it truly doesn't apply; never delete it.

---

## The Template (copy this verbatim and fill it in)

```markdown
---
date: YYYY-MM-DD
slug: <kebab-case-bug-name>
repo: <repo-slug or "cross-repo">
bug_class: <one of: state-desync | network | perf | runtime-exception | type | data | library-quirk | race | concurrency | build | env-config | regression | security | other>
severity: <sev1 | sev2 | sev3>  # sev1 = prod outage / data loss / security breach; sev2 = degraded UX or partial outage; sev3 = minor / cosmetic
area_tag: <optional, free-form, e.g. "tournaments/leaderboard" or "auth/refresh-token">
recall_hit: <yes-fast-path | yes-partial | no-novel | distiller-empty>
evidence_quality: <verified | partial | degraded>  # A4: provenance signal — verified = primary MCP succeeded; partial = had to reason from code or sampled data instead of runtime evidence; degraded = primary MCP unavailable, used generic substitute (loss of structure/queryability)
degraded_reason: <required only if evidence_quality is "degraded" or "partial"; one-line explanation, e.g., "primary mcp-console-hub_get_redux_state unavailable; used chrome-devtools_evaluate_script instead">
fix_commit: <sha or "uncommitted">
time_spent_minutes: <integer>
---

# Postmortem: <one-line bug summary>

## Bug Summary
<One sentence — what was broken and for whom. Will be the atom's Problem field.>

## Reproduction Command
```
<the exact command, URL, click sequence, or test that triggers the bug 100% of the time>
```

## Root Cause
<One paragraph (3-5 sentences) — the structural reason the bug exists. NOT the symptom. NOT the fix. The mechanism. Cite file:line for the cause site.>

## Fix
- File: <path>:<line(s)>
  - <one line describing the change>
- File: <path>:<line(s)>
  - <one line describing the change>

## Evidence Sources Used
<Which MCPs / tools produced the breakthrough. Quote the specific field/value, not just the tool name.>
- `<mcp-tool-name>`: <specific finding — e.g. `tournament.leaderboard.items.length === 0 at t=1620ms while FETCH_LEADERBOARD_SUCCESS fired at t=2390ms`>
- `<another-tool>`: <finding>

## Phase 0 Recall Result
<If recall_hit=yes-fast-path: cite the atom-id you fast-pathed from.>
<If recall_hit=yes-partial: cite the atom + what it added vs. what was different.>
<If recall_hit=no-novel: "No relevant atoms in distiller. This postmortem is the first record of this bug class.">
<If recall_hit=distiller-empty: "Distiller had no atoms for this repo. Consider harvesting.">

## Oracle Consult
<!-- If you invoked the oracle in Phase 3, fill this section. Otherwise write "—" as the section body. -->
<If you invoked the oracle in Phase 3:>
<- Atom/session ID of the consult>
<- Which Oracle hypothesis was the breakthrough (A/B/C from triage order)>
<If no oracle: write "—">

## Open Risks
<!-- Filled when the Phase 3 compound-paralysis cap fired (3-hypothesis rule-out budget exhausted before full disambiguation) OR when the controller is shipping a fix with known residual uncertainty. Write "—" if none. -->
<If cap fired: list each still-plausible hypothesis that could NOT be ruled out within budget, with:>
<- Hypothesis statement (1 sentence, mechanism + evidence)>
<- Falsification test that was too expensive to run NOW (e.g., "requires staging deploy + 24h soak")>
<- Triggering signal to watch for (what would indicate this hypothesis is the actual cause)>
<- Recommended follow-up window (next sprint? after observability lands? when?)>
<If no cap fired and no residual uncertainty: write "—">

## Prevention Notes
<1-3 bullet points. Things future-you should check to prevent this bug class. Examples:>
- <"When dispatching SET_POD, always block on the in-flight FETCH_LEADERBOARD before mutating pod state.">
- <"Lint rule candidate: warn on `dispatch(SET_*)` calls not preceded by `take(*_SUCCESS)` in saga files.">
- <"Add an integration test that switches pods mid-fetch — currently uncovered.">
```

---

## Worked Example

```markdown
---
date: 2026-03-15
slug: tournament-leaderboard-empty-after-pod-switch
repo: playsweeps-web
bug_class: race
severity: sev2
area_tag: tournaments/leaderboard
recall_hit: no-novel
evidence_quality: verified
fix_commit: a1b2c3d
time_spent_minutes: 42
---

# Postmortem: Tournament leaderboard renders empty when user switches pods mid-fetch

## Bug Summary
Switching from Pod-1 to Pod-2 while the leaderboard is still loading leaves the list visibly empty until next page refresh.

## Reproduction Command
```
1. Open https://staging.playsweeps/tournaments?pod=1
2. Within 500ms of page load (before leaderboard finishes), click "Pod 2" tab
3. Observe: leaderboard area shows "No players" state. Expected: Pod-2 leaderboard.
```

## Root Cause
`tournamentSaga.ts:142` dispatches `SET_POD` immediately on the tab click, which triggers the leaderboard reducer to clear its items. Meanwhile the in-flight `FETCH_LEADERBOARD_REQUEST` for Pod-1 completes ~800ms later with `FETCH_LEADERBOARD_SUCCESS` carrying Pod-1's data, which writes back into a state slice that's now associated with Pod-2 — the reducer's "if pod matches" guard at `leaderboard.ts:88` was added for a different scenario and doesn't cover the clear-then-stale-success race. The Pod-2 fetch eventually fires but its dispatch is suppressed by the `dedupe` middleware because the symbol-equal request was already in flight.

## Fix
- File: `src/redux/sagas/tournament.ts:142-148`
  - Wrap `put(SET_POD)` in a `take('FETCH_LEADERBOARD_SUCCESS')` blocker; cancel in-flight fetch first via `put(CANCEL_LEADERBOARD_FETCH)`.
- File: `src/redux/middleware/dedupe.ts:31`
  - Add `CANCEL_LEADERBOARD_FETCH` to the dedupe-reset action list so the subsequent Pod-2 fetch isn't suppressed.

## Evidence Sources Used
- `mcp-console-hub_get_redux_key(keyPath="tournament.leaderboard.items")`: returned `[]` at t=1620ms, then `[Pod-1 rows]` at t=2390ms — proving the SUCCESS payload wrote into the cleared-for-Pod-2 slice.
- `mcp-console-hub_get_timeline(limit=50)`: showed action ordering `SET_POD@1580 → FETCH_LEADERBOARD_REQUEST@1582 → FETCH_LEADERBOARD_SUCCESS@2390 → (no second REQUEST for Pod-2)`.
- `lsp_find_references(filePath="src/redux/middleware/dedupe.ts", line=31, character=10)`: revealed `dedupe-reset` action list was last modified for an unrelated bug 4 sprints ago, never touched for tournament context.

## Phase 0 Recall Result
No relevant atoms in distiller. This postmortem is the first record of this bug class.

## Oracle Consult
—

## Open Risks
—  (Phase 3 disambiguated within budget; no residual uncertainty.)

## Prevention Notes
- When dispatching context-switch actions (SET_POD, SET_USER, SET_TENANT) in saga files, always cancel and await in-flight context-dependent fetches before the dispatch.
- Lint rule candidate: warn on `put(SET_*)` in sagas that's not preceded by a `take(*_SUCCESS)` or `cancel()` of the matching pending task.
- Add integration test in `tests/integration/tournament-pod-switch.test.ts` that switches pods mid-fetch — currently uncovered.
```

---

## Anti-Patterns Specific to Postmortem Writing

> See `references/anti-patterns.md` AP-6 for the global "skip Phase 5" anti-pattern. Below are the writing-specific variants:

### PP-A — "I'll write it later"
**Behavior**: Marking the task done, intending to return after the next task to write the postmortem.
**Why it's blocked**: You will not return. The context that makes the postmortem useful — exact MCP outputs, exact thoughts at the time of breakthrough — evaporates within minutes. The 2-minute version, right now, is the only version that exists.
**Correct alternative**: Write it before marking the task complete. Treat it as Phase 5 of the protocol, not paperwork.

### PP-B — Postmortem-as-narrative
**Behavior**: Writing in story form — "First I tried X, then I noticed Y, then I realized Z..."
**Why it's blocked**: Narrative form buries the structured fields the harvester needs. Future-you searching for "leaderboard empty after pod switch" needs the bug summary, root cause, and fix — not a chronology.
**Correct alternative**: Fill the fixed sections. The chronology of your investigation is irrelevant; only the conclusions are.

### PP-C — "Defensive code" as the fix
**Behavior**: Documenting a fix that adds `?? []` / `try/catch` / null guards around the symptom site without addressing the structural cause.
**Why it's blocked**: This is AP-8 (fix symptom not root cause) preserved in the historical record. The bug will recur and the postmortem will mislead future-you into thinking it was solved.
**Correct alternative**: If your Fix section reads as a *guard*, *coerce*, *catch-and-default*, or "added retry" without explaining the underlying invariant that was violated, you have not found root cause. Loop back to Phase 3 — re-state hypotheses, the previous ones were incomplete. Don't ship the postmortem with a symptom-fix.

### PP-D — Citing tool names without quoting evidence
**Behavior**: "Evidence Sources Used: `mcp-console-hub_get_redux_state`, `lsp_diagnostics`."
**Why it's blocked**: The tool name says nothing about what the tool revealed. Future-you reading this atom has no idea what to look for.
**Correct alternative**: Quote the specific field/value/output that produced the breakthrough. The harvester turns these quotes into searchable atom contents.

### PP-E — Over-engineering the postmortem
**Behavior**: Writing 200 lines, adding "lessons learned" essays, philosophizing about the codebase, suggesting architectural overhauls.
**Why it's blocked**: Postmortems that take >5 minutes to write get skipped next time. Postmortems that take 30 minutes to read get skipped on recall. Brevity is the entire mechanism.
**Correct alternative**: Stay within the 25-line target. Prevention Notes is for 1-3 bullets, not essays. If you have a refactor idea, file it as a separate task.

---

## Frontmatter Field Reference

| Field | Required | Type | Purpose |
|---|---|---|---|
| `date` | yes | `YYYY-MM-DD` | The day the bug was FIXED (not discovered) |
| `slug` | yes | kebab-case string ≤ 6 words | Filename + atom search keyword |
| `repo` | yes | string (slug or `"cross-repo"`) | Scope for `recall(repo=...)` filter |
| `bug_class` | yes | enum (see template) | Coarse classifier for routing-table reverse lookup |
| `severity` | yes | enum `sev1` / `sev2` / `sev3` | Prioritization signal for recall: sev1 atoms surface first |
| `area_tag` | no | string (`module/feature` form) | Narrower than bug_class — used to disambiguate when class is too coarse |
| `recall_hit` | yes | enum (see template) | Tracks the recall loop's hit rate over time |
| `evidence_quality` | yes | enum `verified` / `partial` / `degraded` | A4 provenance signal — `verified` (primary MCP succeeded), `partial` (reasoned from code/samples instead of runtime), `degraded` (primary MCP unavailable, used generic substitute). Future Phase 0 recall flags degraded atoms as lower-confidence. |
| `degraded_reason` | conditional | string (≤80 chars) | Required iff `evidence_quality ∈ {partial, degraded}`. One-line explanation. Enables future recall to understand WHY the evidence was weak. |
| `fix_commit` | yes | SHA or `"uncommitted"` | Audit trail back to the code change |
| `time_spent_minutes` | yes | integer | Cost data — feeds the "is this skill saving time?" metric |

The harvester reads these fields verbatim. Do not rename, do not skip required, do not add fields outside this set (extensions break the index).

---

## Filename Convention

```
.sisyphus/postmortems/YYYY-MM-DD-<slug>.md
```

- `YYYY-MM-DD`: the date the bug was fixed (not discovered)
- `<slug>`: kebab-case, ≤ 6 words, descriptive of the bug class. Examples:
  - `tournament-leaderboard-empty-after-pod-switch`
  - `skrill-webhook-refund-fails-tuesday`
  - `webpack-module-not-found-after-lockfile-bump`
  - `mfa-token-expires-on-safari-only`

Slugs are part of the searchable atom. Make them keyword-rich; the same words you'd type into Phase 0 recall.

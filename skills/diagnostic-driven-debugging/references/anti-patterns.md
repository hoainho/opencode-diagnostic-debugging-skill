# Anti-Patterns Reference

> The 6-phase pipeline blocks shotgun debugging by construction. But LLM agents under context pressure rationalize their way around any process. This file enumerates the rationalizations and the correct alternatives. **Read this before every debug session.**

Adapted from obra/superpowers' Red Flag + Excuse→Reality format, with opencode-specific additions.

---

## The 10 Anti-Patterns

### AP-1 — "Try a fix and see"

**Behavior**: Edit a file mid-investigation to test a guess. Wait for the result. If it works, declare done. If not, try another guess.

**Why it's blocked**: This is the canonical shotgun-debugging loop. Every "try" is paid for in lost time AND in subtle wrong fixes that look right because they masked the symptom.

**Correct alternative**: Stay in Phase 3. State the hypothesis. Design the **falsification test** (what would prove this wrong?). Run it. Only then, if confirmed, edit in Phase 4.

---

### AP-2 — Vague hypothesis

**Behavior**: "I think it might be a race condition" or "probably something to do with the auth flow". No mechanism, no file:line, no falsification test.

**Why it's blocked**: A vague hypothesis cannot be tested. Without a test, the next step is inevitably AP-1.

**Correct alternative**: Force concrete shape:
- **Mechanism**: "When pod-switch fires, `tournamentSaga` dispatches `SET_POD` before `FETCH_LEADERBOARD` completes, so the leaderboard reducer overwrites with stale data."
- **Evidence**: `src/redux/sagas/tournament.ts:142`, `src/redux/reducers/leaderboard.ts:88`
- **Falsification test**: "Add a temporary `await delay(500)` before `SET_POD` dispatch — if the bug disappears, race confirmed. If still broken, race is not the cause."

---

### AP-3 — console.log spam

**Behavior**: Sprinkle `console.log()` / `print()` across suspected functions, run, scroll output, repeat.

**Why it's blocked**: Wastes time (rebuild between iterations), pollutes the codebase, and skips the *right* tool. Opencode has `mcp-console-hub` and `mcp-console-hub_get_redux_state` for exactly this purpose.

**Correct alternative**: Use the routing table. For frontend runtime, `mcp-console-hub` queries the actual running browser without code edits. For backend, structured logs at boundaries — not prints inside hot paths.

---

### AP-4 — Refactor-while-debugging

**Behavior**: While investigating, "clean up" surrounding code because "it'll be easier to fix once it's readable".

**Why it's blocked**: Changes the surface area of the bug. Now you can't tell whether the refactor introduced new issues, fixed the original by accident, or both. Phase 4 demands a MINIMAL fix.

**Correct alternative**: Note the refactor opportunity in a separate task / TODO. Touch only the root-cause line(s) in this session.

---

### AP-5 — Skip Phase 0

**Behavior**: "This bug is obviously new, no point checking recall." Jump straight to Phase 1.

**Why it's blocked**: You don't know it's new until you've checked. `omo-session-distiller_recall` is a 5-second query. The skip is pure availability bias.

**Correct alternative**: Always run Phase 0. Even when you're 99% sure it's novel — the 1% case where recall hits saves 30+ minutes.

---

### AP-6 — Skip Phase 5

**Behavior**: "Bug fixed. I'm done. The postmortem is paperwork."

**Why it's blocked**: The postmortem is the **feedback loop** that makes future Phase 0 hits work. Skipping it means future-you will debug the same bug from scratch. The compound effect across a project's lifetime is enormous.

**Correct alternative**: 2-minute postmortem from the template. Fill the 8 required fields. Commit it. Move on. Future-you will thank you.

---

### AP-7 — Diagnostic silencing ("fix" the message, not the bug)

**Behavior**: Applied to the *symptom diagnostic* rather than its cause. Includes:
- Type suppression: `as any`, `@ts-ignore`, `@ts-expect-error`, `# type: ignore`
- Lint suppression: `eslint-disable`, `# noqa`, `@SuppressWarnings`
- Silent error swallowing: empty `catch (e) {}`, `Promise.catch(() => {})`, `except: pass`, `_ = await foo();` to discard rejections
- Console silencing: `console.error = () => {}`, log-level downgrades during debug

**Why it's blocked**: The type/lint/error message is often the bug's voice. Silencing it ships the bug to production wearing a green CI badge. Empty catches are particularly insidious — they convert a loud failure into a silent state corruption that surfaces three layers downstream as a mystery.

**Correct alternative**: Treat the diagnostic as evidence in Phase 2. The type error tells you something about types or assumptions. The exception you're tempted to swallow names the boundary condition you haven't handled. Fix the assumption / handle the boundary properly.

---

### AP-8 — Fix the symptom, not the root cause (includes the "partial-fix" compound-cause variant)

**Behavior**: Two failure modes covered here:
- *Symptom variant*: "Tournament shows empty? I'll add a defensive `?? []` here." Bug stops appearing. Done.
- *Partial-fix / compound variant*: Phase 3 confirmed hypothesis A is a real cause and the test passes — but two hypotheses shared evidence. You ship fix A without confirming whether B was also causal. The bug appears to recede; weeks later it returns from cause B that was always there. (This is the compound-root-cause case SKILL.md Phase 3 now gates against.)

**Why it's blocked**: Both variants ship "less than the whole cause." The symptom variant fixes nothing structural; the partial-fix variant fixes one strand of a compound cause. Both produce the same outcome — the bug returns at a slightly different surface, and now two debugging sessions accumulate to chase one root cause that should have been finished in one.

**Correct alternative**: In Phase 3:
- For the symptom variant — ask "why is this list empty?" until the answer is structural ("the fetch saga doesn't await its dependency"), not surface ("the variable is undefined"). Fix at the structural layer.
- For the partial-fix variant — after confirming hypothesis A, explicitly verify each other hypothesis that shared evidence: either falsify it (the confirmed A subsumes its evidence) or confirm it as a co-cause and fix both atomically. Per SKILL.md Phase 3 gate: "confirmed at least one hypothesis AND ruled out other still-plausible ones sharing evidence."

---

### AP-9 — Read the implementation, not the evidence

**Behavior**: Open the suspected file. Read top to bottom. Reason about what *might* go wrong. Form a hypothesis without ever observing the bug's actual data.

**Why it's blocked**: You can't out-reason a running system. Most non-trivial bugs involve runtime data (network response, state at a specific moment, race between events) that's invisible from static code review.

**Correct alternative**: Phase 1 produces a reproduction. Phase 2 collects evidence from that reproduction (MCP queries, state dumps, network traces). Reason about the **observed**, not the imagined.

**Sub-case — git archaeology**: A close cousin is "let me check `git blame` / `git log -S` instead of running the repro." Reading history is evidence about *past* code, not *current* runtime. Useful in Phase 3 to enrich a hypothesis (the commit that introduced the regression), useless as a substitute for the live observation that Phase 2 demands. Always run the repro first.

**See also**: AP-1 (try-a-fix-and-see) — the *action-first* variant of the same evidence-bypass. AP-9 fails by being too static; AP-1 fails by being too kinetic. Both skip Phase 2.

---

### AP-10 — Trust your previous fix

**Behavior**: "I fixed this 2 weeks ago. The bug shouldn't be back. Something else is going on."

**Why it's blocked**: Bugs *do* come back — usually because the original fix was AP-8 (symptom only), or because a refactor stripped the defensive code. Trusting the prior fix is how regressions survive.

**Correct alternative**: Phase 0 recall will show your prior atom. Read it. **Verify the prior fix is still in the code** (`grep` / `lsp_find_references`). If it's gone, that's the bug. If it's there, the bug class is different — proceed to Phase 1.

---

### AP-11 — Disable the failing test

**Behavior**: `it.skip()`, `xit()`, `@pytest.mark.skip`, `@Ignore`, `[Fact(Skip="flaky")]`, or commenting out the assertion to make CI green. Often justified as "I'll come back to it" or "it's flaky anyway."

**Why it's blocked**: A failing test is a *running reproduction* — exactly the Phase 1 artifact this skill spends effort to construct. Skipping it discards the most expensive piece of debugging evidence the project has, and the "come back to it" almost never happens. Disabled tests accumulate into a pool of unmonitored debt that hides real regressions.

**Correct alternative**:
- If the test fails reliably → it found the bug; the bug is now your Phase 1 artifact. Investigate immediately.
- If the test is flaky (passes/fails non-deterministically) → that flake IS the bug. Route to Row 8 of the MCP routing table (race/timing/intermittent). Do NOT skip; treat reproducing the flake reliably as a P1 task.
- If the test is testing the wrong behavior and you intend to remove it → delete it in a separate, explicit commit that explains why the behavior is no longer required. NEVER silently skip.

The only legitimate uses of `skip` are: (a) the feature is intentionally not yet implemented (TDD red), tagged with the task that will implement it; (b) a known environment-specific skip (e.g. "skip on Windows"), explicitly documented with the platform reason. Neither is debugging.

---

## Excuse → Reality Table

| Excuse | Reality |
|---|---|
| "Phase 0 is overhead for a 2-minute bug." | Then it stays overhead. The compound benefit comes from the 30-minute bugs where recall saves you 25 of those minutes. |
| "Two hypotheses is overkill, I know what it is." | If you knew, the bug would already be fixed. Two hypotheses force you to articulate the alternatives — which is when the second-best one turns out to be correct. |
| "I'll write the postmortem after I finish other tasks." | You won't. You'll context-switch and forget. The 2-minute version, right now, is the only version that actually exists. |
| "The MCP routing table is for when I don't know what tool to use. I know this one." | "I know this one" is the rationalization that produces console.log spam. Use the primary tool listed in the table. It's faster than your habit. |
| "The fix is obviously correct, I don't need a falsification test." | Then designing the test costs 30 seconds and you lose nothing. Every "obviously correct" fix that bypassed a falsification test is a story I've heard before — usually told as a postmortem of a production incident. |
| "I'll just add error handling here to mask the issue while I figure out the real cause." | The error handling will outlive the investigation by years. It's now permanent technical debt with a comment saying "TODO: revisit". You will not revisit. |
| "The bug only happens sometimes — probably just a transient, ignore it." | Transients are bugs whose reproduction you haven't found yet. Find the conditions that trigger them. Otherwise they accumulate into "the system is flaky" tribal knowledge. |

---

## Red Flags Checklist (run mentally before claiming a fix is done)

Tick any that apply — each one means STOP and reassess:

- [ ] I edited code before Phase 3 confirmed a hypothesis
- [ ] My hypothesis has no falsification test
- [ ] I cannot point to a specific file:line for the root cause
- [ ] The fix introduces `as any` / `@ts-ignore` / similar
- [ ] The fix is "defensive" code added in case the bug recurs
- [ ] The reproduction from Phase 1 wasn't re-run after the fix
- [ ] I haven't written the postmortem
- [ ] I touched files outside what the root cause requires
- [ ] The session has run > 30 tool calls without a confirmed Phase 3 hypothesis (observable proxy for "wandering in shotgun mode")
- [ ] I've revised the hypothesis ≥ 3 times without collecting new Phase 2 evidence in between (observable proxy for "fitting hypothesis to bias rather than data")
- [ ] I disabled, skipped, or weakened a test as part of the fix (AP-11)

If any box ticks → loop back to the appropriate phase. The fix is not done until the checklist clears.

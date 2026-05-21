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

### AP-7 — Type suppression to "fix" the bug

**Behavior**: `as any`, `@ts-ignore`, `@ts-expect-error`, `# type: ignore`, `eslint-disable` — applied to the error itself rather than the cause.

**Why it's blocked**: The type / lint error is often the bug's voice. Silencing it ships the bug to production wearing a green CI badge.

**Correct alternative**: Treat the diagnostic as evidence in Phase 2. The type error tells you something about types or assumptions. Fix the assumption.

---

### AP-8 — Fix the symptom, not the root cause

**Behavior**: "Tournament shows empty? I'll add a defensive `?? []` here." Bug stops appearing. Done.

**Why it's blocked**: The empty list had a cause (race, wrong dispatch order, missing fetch). The `?? []` masks it. It reappears elsewhere — same root cause, different surface — and now you have two places to investigate next time.

**Correct alternative**: In Phase 3, ask "why is this list empty?" until the answer is structural ("the fetch saga doesn't await its dependency"), not surface ("the variable is undefined"). Fix at the structural layer.

---

### AP-9 — Read the implementation, not the evidence

**Behavior**: Open the suspected file. Read top to bottom. Reason about what *might* go wrong. Form a hypothesis without ever observing the bug's actual data.

**Why it's blocked**: You can't out-reason a running system. Most non-trivial bugs involve runtime data (network response, state at a specific moment, race between events) that's invisible from static code review.

**Correct alternative**: Phase 1 produces a reproduction. Phase 2 collects evidence from that reproduction (MCP queries, state dumps, network traces). Reason about the **observed**, not the imagined.

---

### AP-10 — Trust your previous fix

**Behavior**: "I fixed this 2 weeks ago. The bug shouldn't be back. Something else is going on."

**Why it's blocked**: Bugs *do* come back — usually because the original fix was AP-8 (symptom only), or because a refactor stripped the defensive code. Trusting the prior fix is how regressions survive.

**Correct alternative**: Phase 0 recall will show your prior atom. Read it. **Verify the prior fix is still in the code** (`grep` / `lsp_find_references`). If it's gone, that's the bug. If it's there, the bug class is different — proceed to Phase 1.

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
- [ ] I'm "tired and want to be done" (the most reliable rationalization detector)

If any box ticks → loop back to the appropriate phase. The fix is not done until the checklist clears.

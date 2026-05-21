---
name: diagnostic-driven-debugging
description: "MUST USE for any non-trivial bug, crash, regression, perf drop, intermittent failure, race condition, deadlock, memory leak, hang, or 'why is X not working' investigation. Enforces a 6-phase root-cause-first protocol (Phase 0 RECALL + 5 investigation/fix phases): Phase 0 RECALL (search past session atoms via omo-session-distiller before investigating), Phase 1 REPRODUCE (deterministic trigger), Phase 2 EVIDENCE (MCP routing table picks the right tool per bug class — mcp-console-hub for Redux/network/runtime, ohmyperf for perf regression, database-inspector for data bugs, lsp_diagnostics for type bugs, grep_app for OSS pattern, context7 for library quirks), Phase 3 HYPOTHESIS (ranked, with falsification test each — no 'try a fix and see' allowed), Phase 4 FIX (minimal, root-cause-targeted), Phase 5 POSTMORTEM (capture for future recall hits). MUST USE when: user says 'debug this', 'fix this bug', 'why is X broken', 'tại sao X không chạy', 'X bị lỗi', 'regression after Y', 'broke after deploy', 'intermittent', 'flaky', 'đôi khi lỗi', 'performance regression', 'chậm hơn trước', 'crash on Z', 'TypeError', 'null pointer', 'stack trace', 'stuck on', 'doesn't work as expected', 'memory leak', 'hangs', 'race condition', 'deadlock', 'API returns 500', 'state is wrong', 'render is broken'. Also triggers AFTER subagent-driven-development returns SPEC_FAIL twice for the same task (escalation), or AFTER pr-code-reviewer flags a suspected bug. DO NOT use for typos, missing imports, lint errors with obvious cause, or 'add feature X' requests — see 'Don't use it for' section. When in doubt about whether to use this skill for a non-trivial bug, USE IT — every 'I'll just try this and see' is paid for in lost hours."
compatibility: "OpenCode"
metadata:
  version: "0.1.0"
  source: "Adapted from github.com/obra/superpowers (skills/systematic-debugging) — Jesse Vincent"
  ports: ["systematic-debugging: Reproduce → Diagnose/Hypothesize → Fix → Verify"]
  extensions: ["Phase 0 RECALL via omo-session-distiller (new)", "MCP routing table for Phase 2 EVIDENCE (new)", "Phase 5 POSTMORTEM feedback loop (new)", "oracle consult on Phase 3 stuck (new)"]
---

# Diagnostic-Driven Debugging

## What This Skill Does

You become a **diagnostician**. You do NOT touch production code until you have:
1. Checked whether this bug was already solved in a past session (Phase 0)
2. Reproduced the bug deterministically (Phase 1)
3. Collected evidence from the RIGHT MCP for the bug class (Phase 2)
4. Formed ≥ 2 falsifiable hypotheses (Phase 3)

Only then do you fix (Phase 4), and you ALWAYS write a postmortem (Phase 5) so future bug encounters get a Phase 0 hit instead of an investigation.

This skill exists because LLM agents debug poorly by default. Without structure, the default behavior is **shotgun debugging** — guess, edit, retry. The 6-phase gate (a memory-backed RECALL phase plus 5 investigation/fix phases) makes guessing structurally impossible.

---

## The 6-Phase Pipeline (Phase 0 RECALL + 5 investigation/fix phases)

```
┌─────────────────────────────────────────────────────────────┐
│  PHASE 0 — RECALL    "Have we debugged this before?"        │
│                                                             │
│  • Query omo-session-distiller_recall with bug keywords     │
│  • Top hit looks clearly relevant → read atom resolution →  │
│    may skip to Phase 4 (still write Phase 5 postmortem)     │
│  • Top hit partially relevant → read, but verify against     │
│    current symptoms before applying any of its conclusions  │
│  • No clearly-relevant hits → proceed to Phase 1            │
│                                                             │
│  Reference: references/recall-first-checklist.md            │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 1 — REPRODUCE   Deterministic trigger required       │
│                                                             │
│  • A single command, URL, click sequence, or test that       │
│    triggers the bug 100% of the time                        │
│  • Cannot reproduce → bug does not yet exist for you →       │
│    request more context from user (this is a NEEDS_CONTEXT  │
│    return, not a debugging session)                         │
│  • Intermittent? Find a minimum repro that fails often       │
│    enough to collect Phase 2 evidence within ~5 attempts;    │
│    if truly random, treat the randomness itself as the bug   │
│                                                             │
│  Output: a runnable repro committed to memory                │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 2 — EVIDENCE    MCP routing → multi-source data      │
│                                                             │
│  • Classify bug from symptom (see MCP routing table)        │
│  • Dispatch MCP queries IN PARALLEL where possible          │
│  • Read full output — never skim                            │
│  • Output: evidence bundle (raw outputs + file:line refs)   │
│                                                             │
│  Reference: references/mcp-routing-table.md                 │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 3 — HYPOTHESIS  Ranked + falsifiable                 │
│                                                             │
│  • State ≥ 2 hypotheses, each with:                          │
│      mechanism (HOW the bug arises)                         │
│      supporting evidence (file:line)                        │
│      falsification test (what would prove this wrong?)      │
│      cost estimate (sec / min / requires-deploy)            │
│  • Run the CHEAPEST falsification test first, unless one    │
│    hypothesis is overwhelmingly likely. Cheap tests pay     │
│    for themselves even if they falsify the leading guess.   │
│  • All hypotheses falsified → consult oracle (template)     │
│  • CANNOT proceed to fix until you've confirmed at least    │
│    one hypothesis AND ruled out other still-plausible ones  │
│    sharing evidence with the confirmed one. COMPOUND root   │
│    causes exist; fixing only one of two is AP-8 with extra  │
│    steps. See anti-patterns AP-8.                           │
│  • CAP: spend at most 3 hypotheses' worth of rule-out cost  │
│    (1 hypothesis-cost = its falsification test cost). If    │
│    you can't disambiguate within that budget, ship the      │
│    highest-confidence single fix AND flag residual          │
│    ambiguity in Phase 5 'Open Risks' subsection. Cap        │
│    prevents analysis paralysis from forever-rule-out loops. │
│                                                             │
│  Reference: prompts/oracle-root-cause.md                    │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 4 — FIX         Minimal, root-cause-targeted          │
│                                                             │
│  • Change ONLY what the confirmed root cause requires       │
│  • NO incidental refactor, NO cleanup, NO style fixes       │
│  • Reproduce command from Phase 1 must now succeed          │
│  • Regression test added (RED first, then GREEN — see TDD)  │
│  • lsp_diagnostics clean on every changed file              │
│                                                             │
│  Anti-patterns blocked: see references/anti-patterns.md     │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  PHASE 5 — POSTMORTEM  Capture for future recall hits        │
│                                                             │
│  • Write to .sisyphus/postmortems/YYYY-MM-DD-<slug>.md      │
│  • Format optimized for omo-session-distiller harvesting    │
│  • Closes the loop: this bug → atom → recall next time      │
│                                                             │
│  Reference: prompts/postmortem-template.md                  │
└─────────────────────────────────────────────────────────────┘
```

---

## When to Use This Skill

**Use it for**:
- Crashes (uncaught exceptions, segfaults, panics)
- Wrong output / wrong state / wrong render
- Regressions ("worked yesterday")
- Performance drops (LCP, INP, TTFB regressions)
- Intermittent / flaky failures
- Library-quirk bugs ("docs say X but it does Y")
- Data bugs (nulls where there shouldn't be, RI violations)

**Don't use it for**:
- Typos / missing imports → just fix them
- Lint errors with obvious cause → just fix them
- "Add feature X" requests → not a bug, use `subagent-driven-development`
- Unknown behavior you haven't tried to understand yet → read code first, then if confused, use this skill

---

## Controller Hard Rules

1. **No code edits before Phase 4.** Investigation is read-only. If you find yourself editing a `.ts` / `.py` / `.cs` file during Phase 0–3, STOP — that's shotgun debugging.
2. **Phase 0 is non-skippable** unless the bug is trivial enough to skip this whole skill. If you're using this skill, you must check recall first.
3. **No fix without a confirmed hypothesis.** "I think it might be X, let me try" is forbidden. State the hypothesis, design the falsification test, run it, THEN fix.
4. **Phase 5 is non-skippable.** Every debug session ends with a postmortem. Even if recall hit immediately and the fix took 2 minutes — write the postmortem. The compound effect is the point.
5. **The fix touches ONLY root cause.** Side cleanups go in a separate task.
6. **Reproduce command from Phase 1 is the verification.** If it now succeeds, the bug is gone. If it still fails, your hypothesis was wrong — back to Phase 3.

---

## Composes With

- **`omo-session-distiller`** — Phase 0 reads atoms; Phase 5 writes postmortems that future harvests turn into atoms
- **`subagent-driven-development`** — when `SPEC_FAIL` recurs, controller invokes this skill before retrying
- **`pr-code-reviewer`** — when reviewer flags a suspected bug, hand off to this skill
- **`oracle` agent** — consulted in Phase 3 when all hypotheses falsify
- **`mcp-console-hub`** — primary evidence source for frontend runtime bugs
- **`ohmyperf`** — primary evidence source for perf regressions
- **`database-inspector`** — primary evidence source for data-layer bugs

---

## Files in This Skill

> Note: this PR (T2) ships only `SKILL.md` and `skill.json`. Files marked _(T3-T7)_ land in subsequent PRs per the plan in `.opencode/plans/2026-05-21-diagnostic-driven-debugging.md`. Links below resolve as those PRs merge.

- [SKILL.md](./SKILL.md) — this file (6-phase protocol, hard rules) — _T2_
- [skill.json](./skill.json) — registry metadata — _T2_
- [references/mcp-routing-table.md](./references/mcp-routing-table.md) — symptom → bug class → MCP — _T3_
- [references/anti-patterns.md](./references/anti-patterns.md) — 8+ blocked anti-patterns — _T4_
- [references/recall-first-checklist.md](./references/recall-first-checklist.md) — Phase 0 procedure — _T5_
- [prompts/oracle-root-cause.md](./prompts/oracle-root-cause.md) — Phase 3 fallback template — _T6_
- [prompts/postmortem-template.md](./prompts/postmortem-template.md) — Phase 5 capture template — _T7_

---

## Attribution

Protocol structure adapted from [obra/superpowers](https://github.com/obra/superpowers) `skills/systematic-debugging/` by Jesse Vincent. Extensions (Phase 0 RECALL, MCP routing table, Phase 5 POSTMORTEM, opencode tool integration) are new and built specifically for opencode's runtime.

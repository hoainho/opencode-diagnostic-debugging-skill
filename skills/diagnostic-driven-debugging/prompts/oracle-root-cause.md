# Oracle Root-Cause Consultation Prompt Template

> **Used in**: Phase 3 (HYPOTHESIS), when all your hypotheses have been falsified and you need a high-IQ second opinion.
> **Caller**: the controller agent running this skill.
> **Callee**: the `oracle` subagent (read-only, expensive, smart).
> **Substitute** `<<<PLACEHOLDER>>>` with concrete values before dispatch. The oracle subagent **does NOT** read this file — you inject the substituted text into the `prompt=` parameter of `task()`.

---

## When to Use This Template

Invoke this template ONLY when:

1. You completed Phase 0 (recall) — no actionable hit
2. You completed Phase 1 (repro) — bug triggers reliably
3. You completed Phase 2 (evidence) — collected via the routing table
4. You completed Phase 3 (hypothesis) — formed ≥ 2 hypotheses
5. **Every hypothesis was falsified by its test**, OR all hypotheses fit the evidence equally well and you cannot discriminate between them

If any of 1-4 is incomplete, do it first. Oracle is the most expensive call in the pipeline — don't burn it on questions you haven't earned the right to ask.

---

## Dispatch Pattern

```python
task(
    subagent_type="oracle",
    load_skills=[],
    run_in_background=True,  # oracle takes minutes; do non-overlapping work meanwhile
    description="Root-cause consult: <one-line bug summary>",
    prompt="""
<paste the template below, with <<<PLACEHOLDER>>> values substituted>
"""
)
```

After dispatch, end your response and wait for the `<system-reminder>`. Do NOT poll. Do NOT start fix attempts based on hypotheses Oracle is currently evaluating.

---

## The Template

```
[ROLE]
You are a senior debugging consultant. The caller has exhausted their own hypotheses. Read the evidence below, then produce a ranked list of root-cause candidates with falsification tests. You write ZERO code — your job is hypothesis generation and triage only.

[BUG SUMMARY]
<<<ONE_PARAGRAPH_DESCRIBING_THE_BUG>>>

User-visible symptom: <<<SYMPTOM_VERB_AND_OBJECT>>>
Affected feature/module: <<<MODULE_NAME>>>
Triggering condition (if any): <<<CONDITION_OR_'unconditional'>>>
Repro frequency: <<<100% | 50% | intermittent — describe>>>

[PHASE 1 — REPRODUCTION]
The bug is triggered by:
<<<EXACT_COMMAND_URL_CLICK_SEQUENCE_OR_TEST>>>

Expected behavior: <<<WHAT_SHOULD_HAPPEN>>>
Actual behavior: <<<WHAT_HAPPENS_INSTEAD>>>

[PHASE 2 — EVIDENCE BUNDLE]
Collected from the routing table (relevant excerpts only; cite specific field/value, not whole tool dumps).

<<<EVIDENCE_FROM_PRIMARY_MCP>>>
(e.g. for a frontend state bug:
  - mcp-console-hub_get_redux_key(keyPath="tournament.leaderboard.items"):
    returned `[]` at t=1620ms, then `[{id: 1, ...}, ...]` at t=2400ms
  - mcp-console-hub_get_timeline(limit=50):
    SET_POD dispatched at t=1580ms
    FETCH_LEADERBOARD_REQUEST at t=1582ms
    FETCH_LEADERBOARD_SUCCESS at t=2390ms
    UI re-render at t=1620ms ← bug visible here
)

<<<EVIDENCE_FROM_SECONDARY_MCP_IF_ANY>>>
(e.g. lsp_find_references on the suspect symbol, network responses, perf trace insights)

Files implicated by the evidence (file:line):
- <<<FILE_PATH>>>:<<<LINE>>> — <<<one-line role>>>
- <<<FILE_PATH>>>:<<<LINE>>> — <<<one-line role>>>

[PHASE 3 — HYPOTHESES TRIED AND FALSIFIED]
I formed the following hypotheses and tested each. All falsified. Listed in the order I tried them.

Hypothesis 1: <<<MECHANISM_DESCRIBED_IN_ONE_SENTENCE>>>
  - Supporting evidence at time of formulation: <<<EVIDENCE_REFS>>>
  - Falsification test I ran: <<<TEST_DESCRIPTION>>>
  - Result: <<<WHAT_THE_TEST_SHOWED>>>
  - Why this falsifies the hypothesis: <<<REASONING>>>

Hypothesis 2: <<<MECHANISM>>>
  - Supporting evidence: <<<...>>>
  - Falsification test: <<<...>>>
  - Result: <<<...>>>
  - Why falsified: <<<...>>>

(Add more if you tried more)

[CONTEXT THAT MIGHT MATTER]
Repo / codebase: <<<REPO_NAME>>> (<<<LANGUAGE_STACK>>>)
Relevant conventions: <<<e.g. "Redux-Saga for async, MUI 6 components, AVA flags via src/redux/ava/">>>
Recent changes I'm aware of: <<<COMMIT_SHAS_OR_'none'>>>
Phase 0 recall result: <<<ATOM_REFERENCE_IF_PARTIAL_HIT_OR_'no hits'>>>

[WHAT I NEED FROM YOU]
1. **Ranked root-cause candidates**: 2-4 hypotheses I haven't tried (or have tried badly), each with:
   - Mechanism: HOW the bug arises, in 1-2 sentences
   - Evidence supporting this hypothesis (cite from the bundle above)
   - **Falsification test**: a concrete observable check (specific MCP call, specific file:line to inspect, specific repro variation) that would distinguish this hypothesis from the others
   - Cost estimate: time to run the falsification test (seconds / minutes / requires-deploy)
2. **Triage order**: which falsification test should I run first, and why? (Usually: lowest-cost-first if they're equally likely; highest-likelihood-first if costs are comparable.)
3. **What you'd look at that I might have missed**: 1-3 specific files / fields / queries I should pull into Phase 2 if my current evidence is too narrow.

[CONSTRAINTS]
- Do NOT propose fixes. Hypothesis + test only.
- Do NOT propose "rewrite this module" or other large refactors. Bugfix scope.
- If the evidence is insufficient to form new hypotheses, say so explicitly and list the 2-3 evidence gaps that would unblock you.
- If one of my falsified hypotheses was actually correct and I mis-falsified it, flag it with the corrected falsification test.

[OUTPUT FORMAT]
```
HYPOTHESIS A: <mechanism>
  Evidence: <pointers>
  Falsification: <concrete test>
  Cost: <estimate>

HYPOTHESIS B: ...

HYPOTHESIS C: ...

TRIAGE ORDER: A → B → C, because <reason>

EVIDENCE GAPS (if any): ...

POSSIBLE MIS-FALSIFICATION (if any): ...
```
```

---

## Post-Consultation Workflow

After Oracle returns:

1. **Pick the first hypothesis** per Oracle's TRIAGE ORDER.
2. **Run its falsification test**. Record the result.
3. **If confirmed** → proceed to Phase 4 (FIX).
4. **If falsified** → move to the next hypothesis in the order. Add the result to your Phase 3 log.
5. **If all of Oracle's hypotheses falsify** → this is rare. Consider:
   - Collecting more evidence (Oracle may have flagged gaps)
   - Re-running Phase 0 with refined keywords (your understanding has deepened)
   - Escalating to the user (Oracle's escalation is the last in-protocol step before "I cannot diagnose this autonomously")

6. **Phase 5 postmortem MUST cite** the Oracle consultation if it produced the breakthrough — future Phase 0 recall benefits from knowing which bug classes needed Oracle.

---

## Anti-Patterns Specific to This Template

- **Premature oracle**: invoking before Phase 2 evidence is collected. Oracle gives bad hypotheses when fed bad evidence — garbage in, garbage out.
- **Vague placeholders**: leaving `<<<PLACEHOLDER>>>` filled with "TODO" or hand-waving. Substitute every placeholder with concrete substance or remove the section.
- **Asking for the fix**: violates the consult-only contract. Oracle's hypothesis is a search direction, not a patch.
- **Ignoring the triage order**: running hypotheses in the order they appear rather than the order Oracle ranked them. The ranking encodes cost × likelihood reasoning.
- **Skipping the postmortem citation**: even when Oracle solved it, you must write Phase 5. The atom that future-you will recall must mention "Oracle hypothesis B was the breakthrough" so future-you knows when to escalate sooner.

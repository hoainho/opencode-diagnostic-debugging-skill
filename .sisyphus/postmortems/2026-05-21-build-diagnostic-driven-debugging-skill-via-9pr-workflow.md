---
date: 2026-05-21
slug: build-diagnostic-driven-debugging-skill-via-9pr-workflow
repo: opencode-diagnostic-debugging-skill
bug_class: other
severity: sev3
area_tag: meta/skill-build-workflow
recall_hit: no-novel
fix_commit: post-merge-of-pr9
time_spent_minutes: 180
---

# Postmortem: Built `diagnostic-driven-debugging` skill end-to-end via 9-PR oracle-reviewed workflow

## Bug Summary

Not a bug in the traditional sense — this is a build-session postmortem capturing the workflow learnings from building the skill itself. Future-me running a similar multi-PR skill build should recall this atom.

## Reproduction Command

```
1. Receive user request: "implement skill X, test it, fix bugs, ship via PR-per-phase with reviewer approval"
2. Read the approved proposal
3. Init repo + harness file at .opencode/plans/YYYY-MM-DD-<slug>.md
4. Execute T1-Tn: one task = one branch = one PR = one reviewer round
5. End with Phase 5 postmortem (THIS FILE)
```

## Root Cause

Not applicable — this is a workflow postmortem, not a defect-fix postmortem. But three workflow-level "root causes" worth recording:

1. **Stale-CDN review fetches**: First oracle re-review fetched files via `raw.githubusercontent.com` which caches branch tips aggressively. Returned false-negative REQUEST_CHANGES on already-fixed content. **Structural cause**: review tooling defaulted to the cached endpoint. **Fix**: switch to GitHub Contents API (`/repos/{}/contents/{path}?ref={branch}`) which is uncached.
2. **Phase 8 nearly skipped**: build session almost ended with todo `[in_progress]` and harness file showing stale `[ ]` for tasks T2-T10. **Structural cause**: treating Phase 8 as a closing chat rather than a structured task. **Fix**: future plans must make Post-Mortem a `[ ]` task in the harness so it's gated by the same status-transition discipline as code tasks.
3. **AP-8 cross-reference drift**: SKILL.md Phase 3 was modified in T9 to reference AP-8 for compound root causes, but AP-8 in references/anti-patterns.md only covered the symptom variant. T9 reviewer caught the drift; closed in T10 PR. **Structural cause**: cross-file references weren't checked during T9 fix. **Fix**: when adding a cross-ref, fetch the target file and verify the reference is honored.

## Fix

- Workflow protocol (PR #1 onward): use GitHub Contents API for review fetches, not raw.githubusercontent.com
- Harness file (this PR): all 10 task statuses synced to `[x]`, Post-Mortem section filled, Decisions Log + Open Questions + Cross-Task Notes appended with session learnings
- AP-8 (`skills/diagnostic-driven-debugging/references/anti-patterns.md`): expanded to enumerate symptom variant + partial-fix/compound variant; quoted SKILL.md Phase 3 gate verbatim to prevent future drift (committed in T10 PR #9)

## Evidence Sources Used

- `omo-session-distiller_recall`: 0 prior atoms for "skill build via multi-PR workflow"
- `gh pr view`: ran on every PR to verify merge status, comment count, review state
- `gh api repos/.../contents/...?ref=`: GitHub Contents API used as uncached source-of-truth for review fetches after PR #1's stale-CDN incident
- `task(subagent_type="oracle", run_in_background=true)`: 14 review invocations across 9 PRs, fetching files via Contents API and validating against acceptance criteria
- Visual inspection of ASCII box widths in SKILL.md (T9 review): `sed -n '70,95p' SKILL.md` to confirm right-edge alignment after edits

## Phase 0 Recall Result

No relevant atoms in distiller — first record of "skill build via 9-PR oracle-reviewed workflow" for this repo class.

## Oracle Consult

—  (no escalation needed; oracle was the substitute REVIEWER for every PR, not consulted for any single hard problem)

## Prevention Notes

- **Cache discipline**: when fetching branch-tip content for review automation, always use GitHub Contents API or `git ls-remote` SHA verification. Never trust raw.githubusercontent.com for "is the latest commit content correct?" queries.
- **Phase 8 as gated task**: future plan templates should make Post-Mortem a `[ ]` task in the harness, not a free-form trailing section. Forces status-transition discipline.
- **Cross-ref integrity check**: when a skill file references another by name (AP-X, Phase N, prompts/foo.md), the writing pass should immediately verify the target exists and says what the reference claims. Cheaper than catching it in review.
- **Architecturally interesting**: the skill being built and the workflow building it are isomorphic — both are "controller dispatches reviewable units, each merged only after acceptance criteria pass". This is the subagent-driven-development pattern recursing on itself. Worth featuring as a worked example if I ever build a `building-skills` meta-skill.

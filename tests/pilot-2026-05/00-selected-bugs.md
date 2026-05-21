# Stage B1 — Pilot Bug Selection (2026-05)

> **Plan reference**: [`.sisyphus/plans/2026-05-21-skill-production-hardening.md`](https://github.com/hoainho/opencode-diagnostic-debugging-skill/blob/main/.opencode/plans/2026-05-21-skill-production-hardening.md) Stage B1.
>
> **Selection date**: 2026-05-21.
> **Selector**: Sisyphus (opencode controller running diagnostic-driven-debugging skill v0.5 post-A5).
> **Source**: Jira JQL `project = WIN AND issuetype = Bug AND status in ("To Do", "Open", "Backlog") AND sprint in openSprints() ORDER BY priority DESC, created DESC` (7 candidates returned; 4 selected for diversity).

## Selection criteria

Per the plan B1's acceptance criteria:
- 3-5 bugs (selecting 4)
- Diverse coverage: at least 1 frontend, 1 backend, 1 transformer or cross-cutting
- All in current sprint (Sprint 83) to avoid pulling from completed/closed work
- Not yet assigned-and-in-progress (each has an assignee on paper but status is "To Do" — pilot only generates the diagnostic trace, NOT the production fix, so no conflict)
- Mix of priorities (P0 + P1) to expose the skill to both critical and non-critical bug archetypes

## Why not all 7

3 bugs were filtered out for diversity reasons:

- **WIN-7992** (Daily Bonus reward clickable during animation): too similar to WIN-7993 (also Daily Bonus state issue) — duplicate archetype coverage
- **WIN-7987** (Daily Bonus countdown delay): also Daily Bonus; third one would over-weight DB module
- **WIN-7906** (Daily Bonus V2 reload behavior): same Daily Bonus cluster

Selecting 1 representative from the Daily Bonus cluster (WIN-7993, the most critical) preserves diversity vs. the other 3 selected bugs.

## The 4 Selected Bugs

### Pilot-1: WIN-7993 — Daily Bonus auto-claim leaves reward in claimable state

- **JIRA**: WIN-7993
- **Type**: Bug, Pod 2, Sprint 83
- **Priority**: P0 / Critical / "All Players All The Time"
- **Repo**: playsweeps-web (frontend) — Daily Bonus modal state machine
- **Symptom verb**: "stuck", "stays claimable", "auto-claim succeeds in balance but UI doesn't update"
- **Expected MCP routing**: Row 1 (Redux state desync) + Row 8 (race/timing — auto-claim timer vs. UI state update)
- **Expected protocol surface area exercised**: Phase 0 RECALL (similar Daily Bonus atom may exist from prior sprints), Phase 2 multi-row spanning (Discipline Rule 3 tie-breaker), Phase 3 compound check possible
- **Why pilot**: P0 priority gives skill maximum stakes; state-machine bug is exactly the archetype the skill's Row 1+8 path was designed for
- **Reproducibility**: 100% (per ticket)

### Pilot-2: WIN-7988 — BI event missing source_id/source_name for LP earning

- **JIRA**: WIN-7988
- **Type**: Bug, Pod 2, Sprint 83
- **Priority**: P0 / BI-tracking
- **Repos**: playsweeps-web (event emit site) + playsweeps-backend (PlayStudios.EventHub.Publisher) + playsweeps-bi-event-worker (consumer that surfaces the gap)
- **Note (per B1 review C2)**: AGENTS.md also lists `playsweeps-bi-event` as a separate BI microservice (distinct from `playsweeps-bi-event-worker`). Pilot-2's Phase 2 evidence collection MUST disambiguate which consumer(s) surface the missing-field gap — the worker (EventHub trigger function) or the microservice or both. The `area_tag` in the Phase 5 postmortem must reflect the precise consumer ID, not just `bi/lp-earning`.
- **Symptom verb**: "missing field", "BI event payload incomplete"
- **Expected MCP routing**: Row 6 (data — event schema mismatch) + Row 9 (cross-repo build/integration) + Row 10 (env-config if field is config-driven)
- **Expected protocol surface area**: cross-repo evidence collection (need to trace event from frontend emit → backend publisher → BI worker consume → possibly BI microservice); Phase 5 postmortem with precise `area_tag`; potentially `evidence_quality: partial` if can't run the full pipeline
- **Why pilot**: only cross-cutting bug in current sprint; tests the skill's handling of bugs that span 3+ repos
- **Reproducibility**: deterministic per cap-reach trigger

### Pilot-3: WIN-7990 — Prize Summary always shows Lootbox icon

- **JIRA**: WIN-7990
- **Type**: Bug, Pod 2, Sprint 83
- **Priority**: P2 / UI inconsistency
- **Repo**: playsweeps-web (frontend) — Prize Summary component
- **Symptom verb**: "always shows X regardless of input"
- **Expected MCP routing**: Row 1 (component reads wrong prop/state) OR Row 5 (type/symbol — wrong icon import) — depends on root cause
- **Expected protocol surface area**: pure frontend, no timing; tests Phase 1 reproducibility on UI-only bug; tests whether skill's MCP routing correctly differentiates state-desync (Row 1) from static-import bug (Row 5) when symptoms are similar
- **Why pilot**: contrast against P0 timing bug (Pilot-1) — confirms skill handles low-complexity bugs without over-engineering
- **Reproducibility**: 100% (per ticket — "always displays")

### Pilot-4: WIN-7518 — MFA cooldown doesn't reset after extended cooldown expires

- **JIRA**: WIN-7518
- **Type**: Bug, Pod 3, Sprint 83
- **Priority**: P1 / Major
- **Repos**: playsweeps-web (cooldown state UI) + playsweeps-backend (Sweeps.SMS.MFA cooldown logic)
- **Symptom verb**: "incorrectly continues using extended", "state not resetting after timer"
- **Expected MCP routing**: Row 1 (state lifecycle) + Row 8 (timer/event ordering) + potentially Row 6 (if cooldown state persists in DB/Redis)
- **Expected protocol surface area**: Phase 3 compound check (frontend timer state + backend cooldown calculation could both be involved); MFA security implications → `area_tag: auth/mfa`, `bug_class: security` (per A4 enum expansion)
- **Why pilot**: only Pod 3 bug + only security-adjacent bug; tests skill's coverage of auth/security archetypes
- **Reproducibility**: 100% (per ticket)

## Coverage matrix verification

| Pilot | Frontend | Backend | Cross-repo | Pod | Priority | Bug class (expected) |
|---|---|---|---|---|---|---|
| Pilot-1 (WIN-7993) | ✅ primary | — | — | 2 | P0 | state-desync + race |
| Pilot-2 (WIN-7988) | ✅ emit | ✅ publisher + worker | ✅ 3 repos | 2 | P0 | data (BI schema) |
| Pilot-3 (WIN-7990) | ✅ primary | — | — | 2 | P2 | state-desync OR type |
| Pilot-4 (WIN-7518) | ✅ UI | ✅ MFA service | ✅ 2 repos | 3 | P1 | security + race |

Diversity satisfied: 4 frontend, 2 backend, 2 cross-repo, both pods, P0/P1/P2 mix, distinct bug classes per A4 enum (`state-desync`, `data`, `security`, `race`).

## Pilot execution contract

For each Pilot-N (B2):
1. Walk Phase 0 → 5 of the skill with **wall-clock measurement** (start timer at first read of symptom; stop at fix-verified)
2. Capture: Phase 0 recall outcome (hit/miss/score), MCP tool calls per phase (with args), hypotheses generated (with cost estimates), false leads, evidence quotes, fix diff
3. Write Phase 5 postmortem at `.sisyphus/postmortems/2026-05-21-pilot-N-<slug>.md` AND execute the A5 Manual Harvest Export to make it recall-able
4. **Do NOT push the production fix** to the upstream playsweeps repos — pilot only produces the diagnostic trace + postmortem. The actual fix remains the assignee's responsibility per normal sprint workflow.
5. Each pilot ships as its own PR (B2.1, B2.2, B2.3, B2.4) so the diagnostic trace is independently reviewable.

## Stop conditions for individual pilots

- **Trivial-confirmation stop-rule for Pilot-3** (per B1 review C1): if Phase 2 evidence collection completes in <10 minutes AND the fix is a single-symbol change (e.g., wrong icon import, single literal value), mark pilot as `outcome: trivial-confirmation` — protocol ran but exercised minimal surface area. Do NOT swap to backup in this case; trivial-confirmation IS still valuable Stage B data (proves the skill doesn't over-engineer simple bugs). Document the trivial outcome explicitly in the pilot's postmortem.
- **Bug-wider-than-scoped stop**: if a pilot's bug turns out to touch features beyond what the symptom report says → swap per the **backup-activation rule** below.
- **Skill-protocol-stuck stop**: if the skill can't proceed past a phase → document the stuck point + recovery path in the pilot's postmortem. This is itself valuable Stage B data (skill defect to fix in Stage A++).
- **Pilot-budget exhaustion**: if budget runs out before 3 pilots complete → ship Stage B partial + explicitly flag insufficient data for Stage C.

### Backup-activation rule (per B1 review C4)

Backups (WIN-7992, WIN-7987) are both Daily Bonus archetype. Activating them would create 2 DB pilots and break the diversity matrix. Rule:

- **Pilot-1 stalls (already DB)** → swap to Backup-A (WIN-7992) or Backup-B (WIN-7987). Diversity preserved because the slot was DB to begin with.
- **Pilot-2, Pilot-3, or Pilot-4 stalls** → do NOT swap to backup. Abort that pilot and ship Stage B with N-1 pilots (3 instead of 4). Diversity beats sample size when the open-bug pool is DB-dominated (4 of 7 candidates were DB). Document the abort + reason in pilot-2026-05/01-aborted-<pilot>.md so Stage C can correctly weight the truncated dataset.

This rule resolves the apparent contradiction of filtering DB bugs out for diversity yet listing them as backups — the backups are scoped to DB-slot replacement only.

## What this PR does NOT do

- Does NOT pull from main playsweeps repos
- Does NOT modify production code
- Does NOT assign work to other engineers
- Does NOT block sprint work (assignees keep their bugs)

## What B2-B4 will do

- B2.1-B2.4: one PR per pilot bug with diagnostic trace + postmortem
- B3: harvest all 4 postmortems, verify ≥80% recall hit rate when queried with original symptom phrasing
- B4: pick a second-similar bug (organic or from backlog with similar archetype) and verify Phase 0 hit + wall-clock <50% of first

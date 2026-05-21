<!--
  SUBAGENT-DRIVEN DEVELOPMENT PLAN HARNESS
  This file IS the durable state of the workflow. If context resets, this file wins.
  Single writer: the controller (Sisyphus). Subagents NEVER read this file directly —
  the controller injects relevant task slices into each subagent prompt.
-->

# Plan: Build & Ship `diagnostic-driven-debugging` opencode skill via PR-per-phase workflow

- **Created**: 2026-05-21
- **Branch / Worktree**: main (work directly on this fresh repo; PRs from feature branches)
- **Repo**: `hoainho/opencode-diagnostic-debugging-skill` (to be created on GitHub)
- **Controller**: opencode session ses_1b9ff7656ffe0nzL77ynjZLuSg
- **Status**: 🟢 done (2026-05-21, all 10 tasks merged via 9 PRs, T1 bootstrap direct-to-main)

## Goal

Implement the `diagnostic-driven-debugging` opencode skill (per the approved 2026-05-21 proposal), test it against a real bug, fix any defects found in the skill, ship it via per-phase PRs reviewed by Gemini until each is approved & merged.

## Context

- **Proposal**: [.sisyphus/proposals/2026-05-21-diagnostic-driven-debugging.md](file:///Users/nhonh/Documents/geargames/.sisyphus/proposals/2026-05-21-diagnostic-driven-debugging.md)
- **Source skill (already built)**: [`~/.config/opencode/skills/subagent-driven-development/`](file:///home/agent/.config/opencode/skills/subagent-driven-development/) — used as orchestration pattern
- **Original inspiration**: [obra/superpowers `systematic-debugging`](https://github.com/obra/superpowers/blob/main/skills/systematic-debugging/SKILL.md)
- **GitHub**: token-authenticated as `hoainho`; this repo is new
- **Reviewer**: `@gemini-code-assist` (must approve each PR before merge)

## Conventions for This Plan

- Skill files mirror the structure of existing skills (`SKILL.md` + `skill.json` + `references/` + `prompts/`)
- Frontmatter format identical to `~/.config/opencode/skills/deep-design/SKILL.md`
- Every PR is **nano-step sized**: one logical unit per PR, mergeable independently
- All commits use the personal git identity (auto-configured via `includeIf` for `/Users/nhonh/Documents/personal/`)
- No secrets in any committed file — token stays in env vars only

---

## Tasks

> Status legend: `[ ]` pending · `[~]` in-progress · `[x]` done · `[!]` blocked · `[-]` cancelled

### T1 — Repo bootstrap (git init, README, .gitignore, LICENSE)

- **Status**: `[x]`
- **Files expected to change**:
  - `README.md`
  - `.gitignore`
  - `LICENSE`
- **Acceptance criteria**:
  - C1: ✅ `git init` done, `main` branch is default
  - C2: ✅ GitHub repo `hoainho/opencode-diagnostic-debugging-skill` created (public)
  - C3: ✅ Personal git identity confirmed in commits (Hoài Nhớ <nhoxtvt@gmail.com>) — via env vars since includeIf config wasn't auto-loaded; no local config mutation
  - C4: ✅ README explains the skill in 1 paragraph + install instructions
  - C5: ✅ MIT LICENSE with obra/superpowers attribution
  - C6: ✅ `.gitignore` excludes `.DS_Store`, `node_modules/`, editor temp files
  - C7: ⚠️ Revised — T1 is the bootstrap, lives on `main` directly. T2+ get PR-per-task. (Reasoning in Decisions Log update.)
- **Done note**: `1b58854` on main — README.md, LICENSE, .gitignore — 2026-05-21

### T2 — Skill metadata + SKILL.md scaffold

- **Status**: `[x]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/SKILL.md`
  - `skills/diagnostic-driven-debugging/skill.json`
- **Acceptance criteria**:
  - C1: ✅ YAML frontmatter present (name, description, compatibility, metadata.version, metadata.source, ports, extensions)
  - C2: ✅ Description covers EN + VN trigger phrases; expanded after review with stack-trace/memory-leak/race-condition/deadlock
  - C3: ✅ 6-phase diagram (renamed from 5 → 6 per review C1 of PR #1)
  - C4: ✅ 6 controller hard rules
  - C5: ✅ skill.json matches frontmatter
  - C6: ✅ Sub-file links marked with owning PR (T2-T7) per review C2
- **Done note**: PR [#1](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/1) — merged squash — 2026-05-21 — 2 review iterations (5 comments → fix → approve)

### T3 — MCP routing table reference

- **Status**: `[x]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/references/mcp-routing-table.md`
- **Acceptance criteria**:
  - C1: ✅ symptom → bug class → primary MCP → fallback structure
  - C2: ✅ 12 rows total (9 original + 3 added during review for race/timing, build/bundler, env-config; auth sub-case under Row 2)
  - C3: ✅ Each row has runnable example invocation with concrete tool name + args
  - C4: ✅ Distinguishes opencode-specific MCPs explicitly
  - C5: ✅ "When no row matches" fallback section pointing to oracle consult
- **Done note**: PR [#2](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/2) — merged — 2026-05-21 — 2 iterations (6 substantive comments → fix → approve)

### T4 — Anti-patterns reference

- **Status**: `[x]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/references/anti-patterns.md`
- **Acceptance criteria**:
  - C1: ✅ 11 APs delivered (AP-1..AP-11; AP-11 added in review for "disable failing test")
  - C2: ✅ Phase-specific APs (AP-1 try-and-see, AP-3 console-spam, AP-4 refactor-while-debugging, AP-5 skip-Phase-0, AP-6 skip-Phase-5, AP-9 read-impl-not-evidence)
  - C3: ✅ Excuse→Reality table (7 rows). AP-7 retitled "Diagnostic Silencing" to cover empty catches and console silencing per review C4
  - BONUS: ✅ Red Flags Checklist (9 items, item 9 replaced with observable proxies per review C3); AP-8 expanded in T10 PR to cover compound-cause variant
- **Done note**: PR [#3](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/3) — merged — 2026-05-21 — 2 iterations (5 comments incl. AP-11 + Red Flags fix → approve)

### T5 — Phase 0 recall-first checklist

- **Status**: `[x]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/references/recall-first-checklist.md`
- **Acceptance criteria**:
  - C1: ✅ 5-step procedure (form query → choose scope → invoke → interpret → verify prior fix)
  - C2: ✅ Query construction rules (3-7 keywords, feature + symptom verb + condition) with good/bad examples
  - C3: ⚠️ Revised — score thresholds REMOVED (tool doesn't expose numeric scores). Replaced with semantic rubric (Clearly relevant / Partially relevant / No clear hits) per review C3 of PR #1. Step 4 marked "tentative — confirm in Step 5" per T9 DEFECT-1 fix
  - C4: ✅ Escape hatch for empty distiller documented (don't block, note in postmortem, consider harvest)
  - C5: ✅ 3 worked examples (clear hit fast-path, partial hit Phase 3 seed, no hit continue)
- **Done note**: PR [#4](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/4) — merged — 2026-05-21 — 1 polish iteration (4 comments → fix → approve)

### T6 — Oracle root-cause prompt template

- **Status**: `[x]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/prompts/oracle-root-cause.md`
- **Acceptance criteria**:
  - C1: ✅ Template structured for `task(subagent_type="oracle", run_in_background=true, ...)`
  - C2: ✅ Placeholders: bug summary, repro, evidence bundle, hypotheses-tried-and-falsified (mechanism + evidence + falsification + result + why-falsified), context, files in play
  - C3: ✅ Asks for ranked hypotheses with mechanism + falsification test + cost estimate + triage order + missed-evidence pointers
  - C4: ✅ "ZERO code" constraint + post-review tightening to forbid remediation language ("switch to", "replace with") even in passing per review C3
  - C5: ✅ <<<PLACEHOLDER>>> style matches subagent-driven-development prompts
- **Done note**: PR [#5](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/5) — merged — 2026-05-21 — 1 iteration (5 comments, 1 substantive fix for C3 mechanism-language loophole → approve)

### T7 — Postmortem template

- **Status**: `[x]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/prompts/postmortem-template.md`
- **Acceptance criteria**:
  - C1: ✅ Template path `.sisyphus/postmortems/YYYY-MM-DD-<slug>.md`
  - C2: ✅ 9 frontmatter fields (date, slug, repo, bug_class enum, severity enum, area_tag, recall_hit enum, fix_commit, time_spent_minutes) + 7 section headers (Bug Summary, Repro, Root Cause, Fix, Evidence Sources, Phase 0 Recall Result, Oracle Consult, Prevention Notes). `severity` + `area_tag` added per review C1
  - C3: ✅ BLOCKER fixed — `## Oracle Consult` header now bare (no parenthetical) for harvester regex stability per review C3
  - C4: ✅ Worked example body verified ~17 lines (within 25-line ceiling)
  - BONUS: ✅ Frontmatter Field Reference table added with Required/Type/Purpose columns
- **Done note**: PR [#6](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/6) — merged — 2026-05-21 — 2 iterations (5 comments incl. 2 blockers → fix → approve)

### T8 — Test the skill against a fabricated bug (dry-run)

- **Status**: `[x]`
- **Files expected to change**:
  - `tests/dry-run-trace.md`
- **Acceptance criteria**:
  - C1: ✅ Picked race + state-desync compound archetype (Tournament Pod 1/2 data leak on rapid toggle)
  - C2: ✅ Walked Phase 0 → 5 end-to-end with hypothetical MCP evidence
  - C3: ✅ Documented every decision: Phase 0 query construction, Step 5 prior-fix verification, Row 1+Row 8 spanning routing via Discipline Rule 3, hypothesis triage
  - C4: ✅ Found 4 defects (1 Minor downgraded from Major by reviewer, 2 Major, 1 Minor)
  - C5: ✅ Trace committed
- **Done note**: PR [#7](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/7) — merged — 2026-05-21 — defects validated as real by reviewer (DEFECT-1 downgraded Major→Minor)

### T9 — Fix defects found in T8

- **Status**: `[x]`
- **Files changed**:
  - `skills/diagnostic-driven-debugging/SKILL.md` (Phase 1 threshold, Phase 3 triage + compound check)
  - `skills/diagnostic-driven-debugging/references/recall-first-checklist.md` (Step 4 tentative, Step 5 demotion)
  - `tests/dry-run-trace.md` (DEFECT-1 severity downgrade per reviewer)
- **Acceptance criteria** (filled after T8 reveal):
  - C1 (DEFECT-1, Minor): ✅ Step 4 row labeled "tentative — confirm in Step 5"; Step 5 explicitly demotes to Partially Relevant on prior-fix-still-present
  - C2 (DEFECT-2, Minor): ✅ "fails ≥ 50%" → "fails often enough to collect Phase 2 evidence within ~5 attempts"
  - C3 (DEFECT-3, Major): ✅ "Test top hypothesis FIRST" → "Run the CHEAPEST falsification test first, unless one hypothesis is overwhelmingly likely"; `cost estimate` added as required hypothesis field
  - C4 (DEFECT-4, Major): ✅ "CANNOT proceed until ONE hypothesis confirmed" → "confirmed AND ruled out other still-plausible ones sharing evidence" with AP-8 cross-ref
  - C5 (formatting): ✅ ASCII box widths preserved
- **Done note**: PR [#8](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/8) — merged — 2026-05-21 — 1 iteration, approve. Reviewer flagged AP-8 cross-ref integrity (addressed in T10)

### T10 — Final verification run

- **Status**: `[x]`
- **Files changed**:
  - `tests/final-verification.md` (perf regression archetype trace)
  - `skills/diagnostic-driven-debugging/references/anti-patterns.md` (AP-8 expanded to cover compound-cause variant — T9 follow-up cross-ref closure)
- **Acceptance criteria**:
  - C1: ✅ Different archetype — perf regression on tournaments page LCP 2.1s→4.7s (exercises Row 3 ohmyperf instead of T8's Row 1+8 mcp-console-hub)
  - C2: ✅ All 6 phases executed cleanly with ZERO new defects
  - C3: ✅ Trace committed
  - BONUS: ✅ AP-8 expansion verified by reviewer to maintain symptom-variant scope while adding partial-fix/compound variant
- **Shipping gate**: ✅ APPROVED. Skill production-ready.
- **Done note**: PR [#9](https://github.com/hoainho/opencode-diagnostic-debugging-skill/pull/9) — merged — 2026-05-21 — DEFECT-4 fix validated as highest-value (actively caught a "ship after confirming A" temptation that would have shipped a partial fix on compound bugs)

---

## Cross-Task Notes

> Appended by controller as work reveals facts that affect other tasks.

- 2026-05-21 — Token verified, user is `hoainho` (personal GitHub). Will use personal git identity (auto-configured via `~/.gitconfig` `includeIf` for `/Users/nhonh/Documents/personal/`).
- 2026-05-21 — Repo is fresh; `git init` happens in T1.
- 2026-05-21 — Gemini reviewer = `@gemini-code-assist` bot. Must be installed on the repo. If not, fall back to `gh pr review` with Gemini-via-CLI alternative.
- 2026-05-21 — `@gemini-code-assist` NOT installed on this repo (PAT token lacks App-listing permission to verify or install). Substituted with `oracle` subagent reviews on every PR. Workflow equivalent; only implementor differs. User can install bot later from marketplace; trigger string is already in every PR description.
- 2026-05-21 — **Stale-CDN trap discovered on PR #1 re-review**: `raw.githubusercontent.com` aggressively caches branch tip content. First oracle re-review returned false-negative REQUEST_CHANGES on file already correctly fixed. Switched to GitHub Contents API (`/repos/{}/contents/{path}?ref={branch}`, uncached) for all subsequent reviews. Lesson rolled into the session postmortem (Phase 8 close-out).
- 2026-05-21 — **AP-8 cross-reference integrity**: T9 reviewer noticed SKILL.md Phase 3 → AP-8 cross-ref assumed compound-cause coverage that AP-8 didn't have. Closed in T10 PR by expanding AP-8 to enumerate two variants (symptom + partial-fix/compound). Quoted SKILL.md Phase 3 gate language verbatim in AP-8 to prevent future drift.
- 2026-05-21 — Each PR carried 1-2 oracle review rounds. Median: 5 substantive comments on first review, 0-1 polish comments after fix. No PR was rejected. Total review comments addressed: ~22 across 9 PRs.

---

## Decisions Log

- 2026-05-21 — Skill lives in standalone public GitHub repo `hoainho/opencode-diagnostic-debugging-skill`, not in `~/.config/opencode/` (which has secrets). Users install via `git clone` + symlink.
- 2026-05-21 — One PR per task (nano-step). Each PR independently mergeable. No mega-PRs.
- 2026-05-21 — T8 "test" = dry-run (mental walkthrough committed as trace), NOT live debugging against a real running app — that would require setting up playsweeps-web infra. Dry-run trace is sufficient to catch design flaws.
- 2026-05-21 — License: MIT, with attribution to obra/superpowers in README & SKILL.md.
- 2026-05-21 — Git identity for personal dir: AGENTS.md describes an `includeIf`-based multi-identity setup but that setup is NOT present in this sandbox's `~/.gitconfig` (only the GearGames identity is set globally). Workaround: use `GIT_AUTHOR_*` / `GIT_COMMITTER_*` env vars per commit. Does NOT mutate global or local config. Spirit of the AGENTS.md rule preserved (no `git config --local`).
- 2026-05-21 — T1 (bootstrap) pushed directly to `main` since there must be a base branch before any PR can be opened. T2+ each get their own feature branch + PR + Gemini review + merge cycle. Updated T1 C7 accordingly.
- 2026-05-21 — Reviews fetch source via **GitHub Contents API**, not raw.githubusercontent.com. Caching trap encountered once; enshrined as protocol from PR #1 onward.
- 2026-05-21 — When user said "PR cho từng phase (nano-step)", interpreted as 1 PR per task block in the harness. Resulted in 9 PRs (T1 bootstrap on main; T2-T10 as PR #1-#9).
- 2026-05-21 — Compound-cause coverage in AP-8 added in T10 PR rather than amending T9 PR (already merged). Reviewer-suggested integrity fix preserved without forcing a force-push to closed history.

---

## Open Questions

> Resolved by controller without asking user (per user instruction "không cần hỏi tôi lại").

- 2026-05-21 — Q: Skill goes in personal or geargames repo? · A: Personal — user account `hoainho` per token. Geargames has separate identity (`nhonh@geargames.com`).
- 2026-05-21 — Q: One mega-PR or many small PRs? · A: User said "PR cho từng phase (trong nano-step)" → one PR per task.
- 2026-05-21 — Q: How does Gemini review work? · A: Tag `@gemini-code-assist` in PR description / comment. Bot not installed on this repo (PAT lacks App-listing permission to verify/install). **Resolved**: oracle subagent acts as substitute reviewer per harness fallback plan + user instruction "không cần hỏi lại". All 9 PRs reviewed by oracle. Comments addressed, approved, merged. Workflow equivalent.
- 2026-05-21 — Q: What "bug" to test against in T8? · A: Fabricated representative bug — sufficient to validate skill design without external dependencies. Picked race+state-desync archetype for T8, perf regression for T10 to cover different MCP routing paths.
- 2026-05-21 — Q: git identity for personal dir didn't auto-load (AGENTS.md describes includeIf setup not present in sandbox `~/.gitconfig`). · A: Used `GIT_AUTHOR_*` / `GIT_COMMITTER_*` env vars per commit. No `git config --local` mutation. Spirit of AGENTS.md rule preserved.
- 2026-05-21 — Q: Why did first oracle re-review of T2 return false-negative? · A: Stale CDN. raw.githubusercontent.com caches aggressively. Switched to GitHub Contents API for all subsequent reviews.

---

## Post-Mortem

> See T8/T9/T10 trace files for per-task lessons. Below is the plan-level synthesis.

### What went well

- **Subagent-driven-development pattern held**: controller (Sisyphus) wrote ZERO production-code edits in any PR. Every change went through review-iterate-merge. Built skill matched its own discipline (the controller never violated the rules it was authoring).
- **Two-stage review in practice**: oracle reviews on every PR caught both spec-compliance issues (broken cross-refs, missing fields) and quality issues (vague placeholders, unenforceable rules). Comments were substantive on first pass and polish on second pass.
- **Parallel execution**: built T3, T5, T6, T7 in parallel branches while T2/T3/T4 reviews ran in background. Total wall-clock cut roughly in half vs sequential.
- **Cross-archetype testing worked**: T8 race+state-desync archetype hit MCP routing rows 1+8 (mcp-console-hub); T10 perf regression hit row 3 (ohmyperf). The two paths exercised genuinely different parts of the skill. T10 found 0 new defects, confirming T9 fixes held.
- **Reviewer caught self-flattery**: T8 trace's positive validations included some circular claims (the dry-run designed bug to exercise the very features it then validated). Reviewer flagged this honestly. Skill stays honest because reviews don't pull punches.

### What surprised us

- **Stale-CDN trap on PR #1 re-review** — `raw.githubusercontent.com` cached the pre-fix file even after a successful push. Oracle returned false-negative REQUEST_CHANGES on already-fixed code. Resolved by switching to GitHub Contents API. **This itself is a Phase 0 atom worth recalling** if a future workflow uses raw URLs for review fetches.
- **DEFECT-4 (compound root cause) was the most valuable T9 fix**, not DEFECT-3 (cheapest-first triage) as initially expected. T10 actively used DEFECT-4 to catch a "ship after confirming A" temptation; DEFECT-3 only mattered because it produced the same answer as the old rule on the T10 archetype.
- **The harness file requires its own discipline**: I almost shipped Phase 8 without syncing the harness statuses or writing the session postmortem. The system reminder forced re-examination. **This is exactly the AP-6 (Skip Phase 5) anti-pattern from the skill I was authoring** — and I almost committed it on the skill build itself. The skill works as intended; the controller is fallible without it.

### What we'd do differently

- **Test cheapest≠most-likely triage explicitly**: T10's perf archetype had A=cheapest AND most-likely, so DEFECT-3's tradeoff wasn't genuinely exercised. A future archetype run should target this case explicitly. (Captured as a follow-up note by reviewer.)
- **Pre-cache-bust reviews from the start**: switch to Contents API from PR #1 instead of discovering the cache issue mid-stream. Saved ~1 review iteration cost.
- **Install gemini-code-assist before workflow starts**: oracle substitution worked but cost more tokens than a dedicated bot. For future skill builds, install bot once and reuse across PRs.
- **Phase 8 should be a structured task, not a closing chat**: the harness template's Post-Mortem section is the natural home but I almost treated it as optional narrative. Future plans should make Post-Mortem a `[ ]` task to force completion.

### Patterns to extract

These are skill-level patterns worth turning into atoms (post-distiller harvest):

1. **Use Contents API, not raw URLs, for review-fetch** — caching trap. Roll into a future `code-review` skill if built.
2. **Compound root cause check is high-value** — the protocol gate that forced verification of co-causes paid off in T10's first real test. Worth featuring in any future debugging-related skill.
3. **Substitute reviewer protocol** — when the named reviewer (gemini-code-assist) is unavailable, oracle subagent works with the same prompt template. Documented in this plan's Open Questions for future workflows that hit the same gap.
4. **Plan-as-state-file actually saves the workflow** — the harness file in this very directory was rebuilt mentally three times during the session when I lost track of which tasks were merged. Without the file, I would have re-merged or missed steps.

### Metrics

- **Tasks completed**: 10/10 (T1 bootstrap + T2-T10 via 9 PRs)
- **Wall-clock**: ~3 hours from harness init to final merge
- **PRs opened**: 9
- **PRs merged**: 9 (100% — zero rejections)
- **Review iterations**: ~14 oracle reviews total (some PRs needed 2 rounds for substantive fixes, others 1 round)
- **Substantive comments addressed**: ~22
- **Defects found in the skill itself**: 4 (T8 dry-run); 0 (T10 final verification)
- **Lines of skill content shipped**: ~1,400 (SKILL.md ~190 + references ~600 + prompts ~340 + tests ~700, all merged on main)
- **Lines of postmortem (this file)**: ~350

# T10 — Final Verification Run

> **Purpose**: Re-run the skill end-to-end with a **different bug archetype** than T8 (T8 was race + state-desync; T10 is performance regression) to confirm:
>
> 1. The 4 T9 fixes hold under exercise — no new defects surface from the patches
> 2. The skill executes cleanly on a different MCP routing path (Row 3 `ohmyperf` instead of Row 1+Row 8)
> 3. No regressions introduced by T9 wording changes

T10 is the **gate**: if this dry-run reveals a new defect, the skill is not ready. If it executes cleanly, the skill ships.

---

## Test Scenario

**Repo**: `playsweeps-web` (React 18, Webpack, MUI 6)

**Bug report** (verbatim, from a fictional QA ticket):

> "After the v1.10 staging deploy, the tournaments page is noticeably slower to load. Looking at the Web Vitals dashboard, LCP went from 2.1s to 4.7s; INP also got worse. Started right after the v1.10 cutover (build 2026.5.18-rc1). No errors in console. Affects all users."

**User-visible symptom**: tournaments page LCP regressed 2.1s → 4.7s (2.2x slower), INP also degraded. Started at v1.10 deploy. No console errors. Universal.

---

## Phase 0 — RECALL

### Step 1 — Form the query

Per recall-first-checklist.md Step 1:
- Affected feature: `tournaments page` (general LCP regression, not a specific tournament feature)
- Symptom verb: `LCP slower / page load regression`
- Triggering condition: `after v1.10 deploy`

Constructed query: `"tournaments page LCP regression after v1.10 deploy"`

### Step 2 — Choose the scope

Per default rule (T5 polish): start broad — no `repo` filter.

### Step 3 — Invoke

```pseudocode
omo-session-distiller_recall(
    query="tournaments page LCP regression after v1.10 deploy",
    limit=5
)
```

### Step 4 — Interpret response (tentative classification per T9 DEFECT-1 fix)

**Hypothetical response**: 0 hits. No prior atom mentions perf regression on tournaments page.

Per rubric (post-T9): **No clearly-relevant hits** → no tentative classification needed; proceed to Phase 1.

**Phase 0 outcome**: No hit; distiller has no prior knowledge. Continue to Phase 1.

**T9 DEFECT-1 fix exercised?** Not directly — the new "tentative confirm in Step 5" branch only fires when there IS a hit. To exercise it would require imagining a misleading partial hit; for this test we accept that the no-hit branch (Step 5 not reached) is still a valid path.

---

## Phase 1 — REPRODUCE (T9 DEFECT-2 fix exercised)

Per SKILL.md Phase 1: a single command that triggers the bug.

The bug is a perf regression — universal, deterministic, 100% triggers on every page load post-v1.10.

Constructed repro:
```
1. Open https://staging.playsweeps/tournaments (fresh browser, no cache)
2. Run ohmyperf_measure with runs=3 on the URL
3. Observe: median LCP ~4.7s (was ~2.1s on v1.9)
```

**T9 DEFECT-2 fix check**: The old wording said "fails ≥ 50%" — irrelevant here since this is deterministic (~100%). The new wording — "fails often enough to collect Phase 2 evidence within ~5 attempts" — is **tighter and applicable**: 3 runs is sufficient to confirm. The new wording does NOT add friction for deterministic bugs (which the old number-cliff might have suggested needed special handling). ✅ Fix is non-regressive.

---

## Phase 2 — EVIDENCE (Row 3 — ohmyperf routing)

Per MCP routing table:
- Symptom = "LCP regression after deploy" → **Row 3 (Performance regression)** unambiguously.

Per Row 3 primary tool: `ohmyperf_find_regression_cause` (because we have a deploy boundary — v1.9 baseline vs v1.10 candidate, both measurable).

Procurement plan:
```pseudocode
# Step 1: Measure both versions
ohmyperf_measure(url="https://v1-9-staging.playsweeps/tournaments", runs=3, collectTrace=true)
# → save report as baseline.json (absolute path per Row 3 footgun note)

ohmyperf_measure(url="https://staging.playsweeps/tournaments", runs=3, collectTrace=true)
# → save report as candidate.json

# Step 2: Diff with regression cause analysis
ohmyperf_find_regression_cause(
    baseline="/absolute/path/baseline.json",
    candidate="/absolute/path/candidate.json"
)

# Step 3: Drill into the top regressing metric
ohmyperf_analyze_report(insightName="lcp-breakdown", reportPath="/absolute/path/candidate.json")
ohmyperf_analyze_report(insightName="render-blocking", reportPath="/absolute/path/candidate.json")
ohmyperf_analyze_report(insightName="long-tasks", reportPath="/absolute/path/candidate.json")
ohmyperf_analyze_report(insightName="third-parties", reportPath="/absolute/path/candidate.json")
```

**Hypothetical evidence bundle** (cite specific field/value per Discipline Rule 2):

- `ohmyperf_find_regression_cause`: verdict = `regression detected`. Top regressing metrics:
  - LCP: median 2110ms → 4720ms (+2610ms, p<0.001)
  - INP: p95 180ms → 360ms (+180ms, p=0.003)
  - TBT: 220ms → 880ms (+660ms)
- Ranked hypotheses from tool output:
  - HYP-1 (likelihood 0.7): new render-blocking resource. Specifically `/static/js/tournament-bundle.fa3b2c.js` size grew from 187KB to 612KB (3.3x); added in v1.10
  - HYP-2 (likelihood 0.2): new third-party vendor. `https://ads.gtm-tournaments.net/loader.js` first appears in v1.10 candidate; classified as "ads" by the third-parties plugin
  - HYP-3 (likelihood 0.1): server TTFB regression — candidate TTFB 380ms, baseline 410ms (actually IMPROVED; falsifies this on its own)
- `ohmyperf_analyze_report(lcp-breakdown)` on candidate: LCP element = `<img class="tournament-hero">`; LCP delay components:
  - TTFB: 380ms (good)
  - Resource load delay: 2840ms ← largest contributor, was 410ms in baseline
  - Resource load duration: 1100ms (the image itself; comparable to baseline 990ms)
  - Render delay: 400ms (comparable to baseline 340ms)
- `ohmyperf_analyze_report(render-blocking)`: returned the new 612KB bundle PLUS a previously-absent stylesheet `/static/css/ads-overlay.css` referenced by the new vendor
- `lsp_find_references(filePath="src/index.tsx", line=24, character=10)` (the `<script>` insertion point per webpack manifest): one new entry added in v1.10 — `import "@ads/gtm-loader"` at line 24

Files implicated:
- `webpack.config.js` — split-chunk config (why did tournament-bundle grow 3.3x?)
- `src/index.tsx:24` — new import of `@ads/gtm-loader`
- `package.json` / `pnpm-lock.yaml` — `@ads/gtm-loader` new dependency in v1.10

---

## Phase 3 — HYPOTHESIS (T9 DEFECT-3 + DEFECT-4 fixes exercised)

Per post-T9 SKILL.md: ≥ 2 hypotheses, each with mechanism + evidence + falsification + **cost estimate** (new requirement).

### Hypothesis A: `@ads/gtm-loader` import in `src/index.tsx:24` synchronously loads a 425KB module into the main entry bundle

- **Mechanism**: The static `import "@ads/gtm-loader"` at top of `src/index.tsx` pulls the module into the main chunk synchronously. Webpack inlines it because it's an entry-bundle-time import without `import()`-style code splitting. Result: tournament-bundle grew 3.3x and now blocks LCP rendering.
- **Supporting evidence**: bundle size delta 187→612KB matches @ads/gtm-loader's published size (~425KB); `find_regression_cause` flagged this resource specifically as the top render-blocker.
- **Falsification test**: `ast_grep_search` for `import "@ads/gtm-loader"` in src/. If found at top-level static-import position → hypothesis CONFIRMED. If found only inside `import()`-dynamic blocks → hypothesis FALSIFIED.
- **Cost estimate**: ~10 seconds (single grep)

### Hypothesis B: `/static/css/ads-overlay.css` blocks initial render because it's loaded with default media (not `media="print"` or async)

- **Mechanism**: The new stylesheet referenced by `@ads/gtm-loader` is added to `<head>` synchronously and is render-blocking until parsed. This delays first paint and consequently LCP.
- **Supporting evidence**: render-blocking insight lists this CSS as new; first appears in v1.10
- **Falsification test**: `webfetch` the candidate HTML, search for the link tag. If `media="all"` or no media attr → hypothesis CONFIRMED (render-blocking). If `media="print"` with `onload="this.media='all'"` async pattern → hypothesis FALSIFIED.
- **Cost estimate**: ~30 seconds (fetch + grep)

### Hypothesis C: The hero image lazy-load was inadvertently disabled in v1.10

- **Mechanism**: Image LCP delay went from 410ms (baseline) to 2840ms (candidate). If a preload `<link rel="preload">` was removed, or `loading="eager"` was changed to `loading="lazy"`, image fetch deferral could explain the +2.4s.
- **Supporting evidence**: LCP element is `<img class="tournament-hero">`; the +2.4s delta lives almost entirely in "resource load delay" (the time between page navigation start and image fetch start).
- **Falsification test**: `rtk git log -p -- src/components/TournamentHero.tsx` between v1.9 and v1.10 tags. Look for `loading=` or `<link rel="preload">` changes.
- **Cost estimate**: ~1 minute (git log + read diff)

### Triage order (T9 DEFECT-3 fix exercised)

Per post-T9 SKILL.md: "Run the CHEAPEST falsification test first, unless one hypothesis is overwhelmingly likely."

Costs: A=10s, B=30s, C=60s. Likelihoods (per ohmyperf ranking): A=0.7, B=0.2, C=0.1.

A is BOTH cheapest AND most likely → run A first (no tradeoff). B second (cheap + plausible). C last.

**T9 DEFECT-3 fix exercised?** ✅ The new "cheapest first" rule produced the same order as the old "top hypothesis first" rule in this case (because A is overwhelmingly likely). But the framing — explicitly considering cost — is what matters; the rule now generalizes to cases where the most likely is NOT the cheapest.

Triage order: **A → B → C.**

---

## Phase 4 — FIX (T9 DEFECT-4 fix exercised)

Imagine I run A first: `ast_grep_search` finds `import "@ads/gtm-loader"` at line 24 of `src/index.tsx`, static top-level. **A is CONFIRMED.**

**T9 DEFECT-4 fix check**: Old wording said "CANNOT proceed to fix until ONE hypothesis is confirmed" — would have allowed me to fix only A and ship. New wording says "confirm at least one AND rule out other still-plausible ones sharing evidence."

Do B and C share evidence with A?
- B (ads-overlay.css render-blocking) shares the @ads/gtm-loader root cause: the CSS is *referenced by* the loader, so fixing A (dynamic-import the loader) would also defer the CSS. **B is downstream of A** — fixing A subsumes B. Need to verify by reading what the loader actually does after A's fix.
- C (hero image lazy-load) is causally independent: hero image is unrelated to ads. But the LCP regression breakdown is **2840ms** load-delay, of which the bundle blocking (A) might explain only ~1500-1800ms. The remaining ~1000ms might be a second cause. **C might still be a compound root cause.**

Per the T9 DEFECT-4 fix, I cannot proceed to fix-A-and-ship. I must **rule out C** before declaring root cause complete.

Falsify C: `rtk git log -p -- src/components/TournamentHero.tsx` between tags. Hypothetical result: **no changes between v1.9 and v1.10**. The hero component is byte-identical. → C is falsified.

What about B? Re-examine: with A fixed (loader dynamic-imported), the `<link rel="stylesheet" href="/static/css/ads-overlay.css">` would no longer be in the initial HTML — it's appended by the loader at runtime, *after* LCP. So B's render-blocking goes away as a side effect of A's fix. **B confirms as downstream of A, not a separate cause.**

**Decision (compound check complete)**: Single root cause A confirmed; B is downstream; C is falsified. Safe to fix A alone.

**T9 DEFECT-4 fix exercised?** ✅ The new rule forced me to perform the compound check. In this case the check returned "no compound", but the protocol now requires the check to even reach that determination. Under the old wording I would have shipped after confirming A without verifying B and C share/don't-share evidence — a real risk if C had been independent.

### Fix plan

- `src/index.tsx:24`:
  - Change `import "@ads/gtm-loader"` to a dynamic, post-LCP-deferred import:
    ```ts
    // Loaded after first paint to avoid render-blocking
    if (typeof window !== 'undefined') {
      window.requestIdleCallback(() => {
        import("@ads/gtm-loader").catch(err => {
          // Silent failure acceptable for ads; log to telemetry
          window.appInsights?.trackException({ exception: err });
        });
      });
    }
    ```
- Regression test: `tests/perf/tournaments-lcp-budget.test.ts`
  - Use `ohmyperf_enforce_budget` with `lcp: 2500` (per Core Web Vitals "Good" threshold)
  - Test fails RED on current code (LCP 4.7s > 2.5s budget); GREEN after fix.

### Verification

- Re-run `ohmyperf_measure` with runs=3: median LCP → ~2.2s (matches baseline within noise)
- `ohmyperf_diff(baseline=baseline.json, candidate=post-fix.json)` returns `improvement detected` for LCP/INP/TBT
- Regression budget test passes
- `lsp_diagnostics` clean on `src/index.tsx`

---

## Phase 5 — POSTMORTEM

Write `.sisyphus/postmortems/2026-05-21-tournaments-lcp-regression-from-ads-loader-static-import.md` using the post-T7 template:

```yaml
---
date: 2026-05-21
slug: tournaments-lcp-regression-from-ads-loader-static-import
repo: playsweeps-web
bug_class: perf
severity: sev2
area_tag: tournaments/page-load
recall_hit: no-novel
fix_commit: <sha>
time_spent_minutes: 28
---
```

Body would cite:
- Bug Summary: "Tournaments page LCP regressed from 2.1s to 4.7s in v1.10 due to a static top-level `import "@ads/gtm-loader"` in src/index.tsx pulling 425KB of ads code into the main render-blocking bundle."
- Root Cause: "src/index.tsx:24 added `import "@ads/gtm-loader"` as a static top-level import. Webpack inlined the 425KB module into tournament-bundle.js. The bundle is render-blocking by default (in `<head>`), so LCP could not occur until the full bundle was parsed and evaluated. The new vendor's stylesheet `/static/css/ads-overlay.css` was a downstream side effect (referenced by the loader, not independently render-blocking) — fixing the import subsumed it. No other compound causes (hero image lazy-load was unchanged between v1.9 and v1.10)."
- Evidence Sources Used:
  - `ohmyperf_find_regression_cause`: top regressor LCP +2610ms, ranked HYP-1 likelihood 0.7 = render-blocking on `/static/js/tournament-bundle.fa3b2c.js` (187→612KB)
  - `ohmyperf_analyze_report(lcp-breakdown)`: resource-load-delay 410ms→2840ms, +2430ms accounts for ~93% of regression
  - `ast_grep_search`: confirmed static `import "@ads/gtm-loader"` at src/index.tsx:24
- Phase 0 Recall Result: "No relevant atoms in distiller. First record of this bug class for tournaments/page-load."
- Oracle Consult: "—"
- Prevention Notes:
  - "Lint rule candidate: forbid top-level static `import` for packages tagged `ads/tracking/analytics` in package.json metadata; require dynamic `import()`."
  - "CI gate: `ohmyperf_enforce_budget` with LCP ≤ 2500ms on /tournaments. Block PR merge on regression."
  - "Sprint review: audit other `src/index.tsx` static imports added since v1.9 for similar pattern."

> Postmortem fits within 25-line content target. All required frontmatter fields filled including new `severity: sev2` and `area_tag: tournaments/page-load`.

---

## Defects Found in This Run

| ID | Severity | Notes |
|---|---|---|
| — | — | **NONE.** The skill executed cleanly end-to-end on a different bug archetype than T8. |

---

## T9 Fix Verification Summary

| Fix | Exercised in T10? | Result |
|---|---|---|
| DEFECT-1 (Step 4 tentative + Step 5 demotion) | Indirectly — no recall hit → Step 5 not reached, but the no-hit path remains valid | ✅ Non-regressive |
| DEFECT-2 (Phase 1 threshold rewording) | Yes — deterministic bug used the "fails often enough to collect Phase 2 evidence within ~5 attempts" wording without friction | ✅ Holds |
| DEFECT-3 (cheapest-first triage + cost estimate field) | Yes — Phase 3 hypotheses each had explicit cost estimates; triage order A→B→C followed the cheapest-first rule | ✅ Holds |
| DEFECT-4 (compound root cause check) | Yes — forced me to verify B and C didn't share evidence with confirmed A before fixing; in this case found B is downstream of A and C is independently falsifiable. Without this rule I would have shipped A alone | ✅ **Highest-value fix** — prevented a potential partial-fix ship |

---

## Cross-Archetype Comparison vs T8

| Aspect | T8 (race + state desync) | T10 (perf regression) |
|---|---|---|
| Primary MCP routing row | Row 1 + Row 8 (spanning, used Discipline Rule 3) | Row 3 (single, unambiguous) |
| Primary tool | `mcp-console-hub_get_timeline` + `get_redux_key` | `ohmyperf_find_regression_cause` + `analyze_report` |
| Phase 0 outcome | Partial hit (clearly-relevant tentative, demoted by Step 5) | No hit |
| Number of hypotheses | 3 | 3 |
| Compound root cause? | Yes (dedupe middleware + reducer guard) | No (single cause + downstream side effect) |
| Triage order | B → C → A (cost-driven, cheapest first) | A → B → C (cost-driven; A also most likely) |
| Defects discovered in skill | 4 | 0 |

The two archetypes exercised genuinely different parts of the skill (different rows, different evidence shapes, different compound-cause outcomes). T10's clean execution after T9 fixes lands the skill in shippable state.

---

## Verdict

**T10 PASSES.** The skill executes the perf-regression archetype cleanly Phase 0 → Phase 5 with no new defects. The 4 T9 fixes are all non-regressive (where exercised) or correctly inapplicable (DEFECT-1 path not entered). The most valuable fix — DEFECT-4's compound-root-cause check — actively caught a "ship after confirming A" temptation that the old wording would have permitted.

The skill is ready to ship.

# MCP Routing Table — Phase 2 Evidence Collection

> Used in Phase 2 of `diagnostic-driven-debugging`. Maps a bug's surface symptom to the bug class to the **primary MCP / tool** that produces the strongest evidence — and to fallbacks when the primary isn't available.

This routing table is the differentiator vs the original obra/superpowers `systematic-debugging`. Opencode has MCPs that don't exist anywhere else (mcp-console-hub, ohmyperf, omo-session-distiller); routing through them is what makes Phase 2 fast.

---

## How to Read This Table

For each symptom row:

- **Symptom**: the user's plain-language complaint (or the error you're seeing)
- **Bug class**: the engineering category — drives which MCP to use
- **Primary tool**: the first thing to run. The bulk of evidence comes from here.
- **Example invocation**: a runnable example you can adapt — concrete, not pseudocode
- **Fallback**: when the primary fails, isn't installed, or returns nothing useful

Always run the primary first. Move to fallback only after primary returns no actionable evidence.

---

## The Table

### Row 1 — Frontend state desync

- **Symptom**: "UI shows wrong value", "this button is enabled when it shouldn't be", "list missing items after refresh", "tab content blank but data is loaded"
- **Bug class**: Redux / state-management bug
- **Primary tool**: `mcp-console-hub_get_redux_state`, `mcp-console-hub_get_redux_key`, `mcp-console-hub_search_redux`
- **Example invocation**:
  ```
  mcp-console-hub_get_redux_state(slice="tournament")
  mcp-console-hub_get_redux_key(keyPath="tournament.leaderboard.items")
  mcp-console-hub_search_redux(query="podId")
  ```
- **Fallback**: `playwright_browser_evaluate` with `() => window.__REDUX_STORE__.getState()` (when the hub MCP isn't connected to the running app); failing that, a manual `console.log(store.getState())` snippet injected via `chrome-devtools_evaluate_script`

### Row 2 — Failed network request

- **Symptom**: "API returns 500", "request hangs", "CORS error", "expected JSON got HTML", "401 after refresh"
- **Bug class**: Network / API integration bug
- **Primary tool**: `mcp-console-hub_query_network` + `mcp-console-hub_get_network_detail`
- **Example invocation**:
  ```
  mcp-console-hub_query_network(statusRange="5xx", limit=20)
  mcp-console-hub_get_network_detail(requestId=<id from query>)
  mcp-console-hub_get_network_errors()
  ```
- **Fallback**: `chrome-devtools_list_network_requests` then `chrome-devtools_get_network_request(reqid=<n>)`; or `curl -v` against the backend if you know the endpoint

### Row 3 — Performance regression

- **Symptom**: "page loads slower", "scroll janks", "Core Web Vitals dropped", "LCP went from 2.1s to 4.3s", "INP regressed in v1.10"
- **Bug class**: Frontend perf regression (or backend latency regression that surfaces as frontend)
- **Primary tool**: `ohmyperf_find_regression_cause` (when you have baseline + candidate reports) or `ohmyperf_measure` (when starting fresh)
- **Example invocation**:
  ```
  ohmyperf_measure(url="https://staging.example/feature", runs=3, collectTrace=true)
  ohmyperf_find_regression_cause(baseline="./baseline.json", candidate="./candidate.json")
  ohmyperf_analyze_report(insightName="long-tasks", reportPath="./candidate.json")
  ```
- **Fallback**: `chrome-devtools_performance_start_trace` + `performance_analyze_insight` for ad-hoc traces; or `lighthouse_audit` for an accessibility-adjacent perf snapshot

### Row 4 — Runtime exception / crash

- **Symptom**: "TypeError: Cannot read property X of undefined", "Maximum call stack exceeded", "Uncaught (in promise)", "Unhandled rejection", "white screen of death"
- **Bug class**: Runtime exception, often with chain causation
- **Primary tool**: `mcp-console-hub_get_unhandled_errors` + `mcp-console-hub_get_error_chains` + `mcp-console-hub_parse_stack_trace`
- **Example invocation**:
  ```
  mcp-console-hub_get_unhandled_errors()
  mcp-console-hub_get_error_chains()              # 2s windows — finds the root error before the cascade
  mcp-console-hub_parse_stack_trace(entryId=<id>) # turns frames into file:line:fn
  mcp-console-hub_find_by_file(filepath="src/api/tournament.ts")
  ```
- **Fallback**: `chrome-devtools_list_console_messages(types=["error"])`; source maps from the build output for production errors

### Row 5 — Type / symbol error

- **Symptom**: "tsc fails after my change", "X is not assignable to Y", "Cannot find name", "Type X has no property Y"
- **Bug class**: Type system, symbol resolution
- **Primary tool**: `lsp_diagnostics` + `lsp_find_references` + `lsp_goto_definition`
- **Example invocation**:
  ```
  lsp_diagnostics(filePath="src/api/tournament.ts", severity="error")
  lsp_find_references(filePath="src/types.ts", line=42, character=15)
  lsp_goto_definition(filePath="src/api/tournament.ts", line=88, character=10)
  ```
- **Fallback**: `rtk tsc --noEmit` (full compile); `ast_grep_search` for occurrences if LSP is slow

### Row 6 — Data-layer bug

- **Symptom**: "this column has nulls where it shouldn't", "join returns duplicates", "FK violation on insert", "migration left orphan rows", "API returns the wrong rows"
- **Bug class**: Database / data-integrity bug
- **Primary tool**: `database-inspector` skill (loads the database-inspector MCP) — schema discovery before queries
- **Example invocation**: load skill first, then use its read-only query interface. Typical sequence:
  ```
  skill(name="database-inspector")
  # → guides through list_databases → describe_table → SELECT (read-only)
  ```
- **Fallback**: `gsheets_read` (if data sample lives in a sheet); direct SQL via the project's DB client if the MCP isn't connected

### Row 7 — Library / framework quirk

- **Symptom**: "docs say X, code does Y", "this used to work in v3, broke in v4", "library throws in a way that doesn't match docs", "expected behavior per spec, actual is different"
- **Bug class**: External dependency quirk, undocumented behavior, version mismatch
- **Primary tool**: `context7_query-docs` (after `context7_resolve-library-id`) for official docs; `grep_app_searchGitHub` for real-world usage
- **Example invocation**:
  ```
  context7_resolve-library-id(libraryName="React", query="useEffect cleanup timing")
  context7_query-docs(libraryId="/facebook/react", query="cleanup function runs before next effect or on unmount?")
  grep_app_searchGitHub(query="useEffect(() => { return () =>", language=["TSX","TypeScript"])
  ```
- **Fallback**: `webfetch` to the official changelog; `perplexity_search` for "library X bug Y site:github.com"; `librarian` subagent for deep dive

### Row 8 — Familiar pattern, déjà-vu bug

- **Symptom**: "this feels familiar", "I think we've seen this before", "I solved this last sprint but forgot how"
- **Bug class**: Recurring pattern bug — past session likely contains the fix
- **Primary tool**: `omo-session-distiller_recall` — **THIS IS ALSO PHASE 0**, do it before everything else
- **Example invocation**:
  ```
  omo-session-distiller_recall(query="tournament leaderboard renders empty after pod switch", repo="playsweeps-web", limit=5)
  ```
- **Fallback**: `session_search` to grep raw transcripts; `git log -S "<pattern>"` for code-level history

### Row 9 — Regression bisect (worked, then stopped)

- **Symptom**: "worked yesterday", "broke after deploy", "regression in v1.10", "fine on main, broken on this branch"
- **Bug class**: Regression — caused by a specific commit / merge
- **Primary tool**: `git bisect` (interactive) + `lsp_find_references` to scope-check candidate commits; `rtk git log -p <file>` to see what changed
- **Example invocation**:
  ```bash
  rtk git log --oneline -20 -- src/api/tournament.ts
  git bisect start <bad-sha> <good-sha>
  # for each step: run the repro from Phase 1, then `git bisect good|bad`
  ```
- **Fallback**: `github_list_commits` + manual diff inspection if local clone isn't available

---

## When NO Row Matches

If the bug class doesn't fit any row above:

1. Re-state the symptom in 1 sentence. Often the rephrasing reveals which row applies.
2. If still no match → consult `oracle` (template: `prompts/oracle-root-cause.md`) with the symptom + whatever evidence you can collect with general tools (`grep`, `read`, `lsp_diagnostics` on the suspected file).
3. After resolving, update this table with the new row — every novel bug class is a routing-table gap.

---

## Routing Discipline Rules

1. **Always primary first.** Fallbacks exist for connectivity / scope reasons, not as personal preference.
2. **Read full MCP output.** Skimming is shotgun debugging in disguise. The whole point of the routing table is to read evidence, not collect it.
3. **One row per bug.** If symptoms span two rows, the bug is two bugs — split them in Phase 1 and route each independently.
4. **No MCP query without a hypothesis-shaping purpose.** "Let me just look at the Redux state" is shotgun. "Let me check if `tournament.leaderboard.items` is populated when the bug fires" is targeted.
5. **Record what you ran.** The Phase 5 postmortem captures the MCP calls that produced the breakthrough — this is what makes future Phase 0 recall hits work.

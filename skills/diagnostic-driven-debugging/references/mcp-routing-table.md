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
- **Sub-case — Auth / session (401, 403, RBAC denial, token-refresh issues)**: the diagnostic path differs — inspect cookies & storage first:
  ```
  mcp-console-hub_get_cookies(search="auth")
  mcp-console-hub_read_storage(type="all")
  mcp-console-hub_read_storage_key(storageType="localStorage", key="access_token")
  ```
  Decode the JWT (head/payload, NOT the signature) and confirm `exp`, `iss`, scopes match what the API expects. Then return to the primary tools above.
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
  > Footgun: both reports must be on a filesystem path the MCP server can read (not necessarily the agent's CWD). Use absolute paths if unsure.
- **Fallback**: `chrome-devtools_performance_start_trace` + `chrome-devtools_performance_analyze_insight` for ad-hoc traces; or `chrome-devtools_lighthouse_audit` for an accessibility-adjacent perf snapshot

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
- **Fallback**: `chrome-devtools_list_console_messages(types=["error"])`. For production errors with minified frames: download the sourcemap (via `webfetch` from the JS bundle's `//# sourceMappingURL=` comment), then re-run `mcp-console-hub_parse_stack_trace` once the symbolicated stack is in the hub.

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
  > Indexing convention: `line` is **1-based**, `character` is **0-based**. Off-by-one is the most common copy-paste bug here.
- **Fallback**: `rtk tsc --noEmit` (full compile); `ast_grep_search` for occurrences if LSP is slow

### Row 6 — Data-layer bug

- **Symptom**: "this column has nulls where it shouldn't", "join returns duplicates", "FK violation on insert", "migration left orphan rows", "API returns the wrong rows"
- **Bug class**: Database / data-integrity bug
- **Primary tool**: The `database-inspector` skill (which is a workflow guide; once loaded, it tells you which MCP database tools are available in your current opencode session and the exact tool names to call).
- **Example invocation**: load the skill first; it walks you through the canonical sequence — list databases → describe target table → run a read-only SELECT.
  ```
  skill(name="database-inspector")
  ```
- **Fallback**: For SQL Server (the playsweeps stack), `sqlcmd -S <server> -d <db> -Q "<query>"` or Azure Data Studio against the read replica. For Cosmos, the Azure Portal Data Explorer. If the data also lives in a Google Sheet sample → `gsheets_read`.

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

### Row 8 — Race condition / async ordering / intermittent flake

- **Symptom**: "fails maybe 1 in 5", "flaky test", "works on slow network, fails on fast", "depends on order of events", "happens only after a refresh", "Promise resolves out of order"
- **Bug class**: Concurrency / ordering bug
- **Primary tool**: `ohmyperf_measure` with `runs=N` (statistical signal across runs) for browser scenarios; `mcp-console-hub_get_timeline` for event ordering; `playwright_*_browser_evaluate` with repeated runs for deterministic-ish repro
- **Example invocation**:
  ```
  ohmyperf_measure(url="https://staging/feature", runs=10)  # variance reveals races
  mcp-console-hub_get_timeline(limit=200)                   # chronological event view
  mcp-console-hub_query_console(level="error", since=<timestamp-before-repro>)
  ```
  > Strategy: don't try to "fix the race". Identify which two events must be ordered, then enforce the ordering at the source (await the prerequisite saga, debounce the trigger, or move state init upstream).
- **Fallback**: Add temporary logging at the suspected race boundaries with monotonic timestamps; or use `playwright_*_browser_run_code_unsafe` to inject a synchronization probe.

### Row 9 — Build / bundler / dependency-resolution failure

- **Symptom**: "webpack: Module not found", "Cannot resolve X", "Wrangler deploy fails", "MSBuild error CS0246", "vite: failed to resolve import", "package version mismatch", "peer dep warning escalated to error"
- **Bug class**: Build-time / bundler / package-management bug (distinct from runtime Row 4 and types Row 5)
- **Primary tool**: `rtk err <build-cmd>` (captures build error tail) + `ast_grep_search` for the import statement that fails to resolve + `context7_query-docs` for the offending package
- **Example invocation**:
  ```
  rtk err pnpm build
  ast_grep_search(pattern='import $X from "@foo/bar"', lang="typescript")
  context7_resolve-library-id(libraryName="@foo/bar", query="export path changes in v2")
  ```
- **Fallback**: Inspect `package.json` resolutions + lockfile (`rtk grep '"@foo/bar"' pnpm-lock.yaml`); compare to a known-good branch via `git diff main -- package.json pnpm-lock.yaml`.

### Row 10 — Environment / config drift

- **Symptom**: "works locally, fails on staging", "missing env var", "wrong AVA flag", "different behavior in CI vs dev", "Azure App Settings out of sync", "feature flag returns different value than expected"
- **Bug class**: Environment / configuration / feature-flag bug
- **Primary tool**: Compare configuration sources side-by-side. For env vars: read the local `.env.example` + the actual staging env (Azure Portal / `az` CLI / k8s secret manifest). For AVA flags (`playsweeps`): `mcp-console-hub_get_redux_key(keyPath="ava.flags")` in the broken env vs the working env.
- **Example invocation**:
  ```
  mcp-console-hub_get_redux_key(keyPath="ava.flags.tournamentsEnabled")
  mcp-console-hub_read_storage_key(storageType="localStorage", key="ava-overrides")
  webfetch(url="https://config-service.staging/flags?user=...", format="text")
  ```
  > Bisect strategy: produce a diff of the resolved config between the working and broken environment. The first non-trivial differing key is almost always the cause.
- **Fallback**: For SQL Server/Azure: query the `[dbo].[Configuration]` table directly via the database-inspector path. For k8s deployments: `kubectl get configmap -o yaml` + `kubectl get secret -o yaml` (decoded).

### Row 11 — Familiar pattern, déjà-vu bug

- **Symptom**: "this feels familiar", "I think we've seen this before", "I solved this last sprint but forgot how"
- **Bug class**: Recurring pattern bug — past session likely contains the fix
- **Primary tool**: `omo-session-distiller_recall`. Note: this is the Phase 0 tool. If you're hitting this row in Phase 2, you either skipped Phase 0 (loop back!) or your Phase 0 query was too narrow. **Phase 2 use is a deepening query**: more specific, informed by the evidence you just collected.
- **Example invocation**:
  ```
  # Phase 0 query (broad, symptom-only):
  omo-session-distiller_recall(query="leaderboard empty after pod switch", limit=5)

  # Phase 2 deepening query (specific, with the evidence):
  omo-session-distiller_recall(query="tournamentSaga SET_POD before FETCH_LEADERBOARD race", repo="playsweeps-web", limit=5)
  ```
- **Fallback**: `session_search` to grep raw transcripts; `git log -S "<pattern>"` for code-level history

### Row 12 — Regression bisect (worked, then stopped)

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
2. **Cite specific field/value, not the tool name.** Skimming is shotgun debugging in disguise. When you reference an MCP result in Phase 3 or 5, you must quote a concrete field/value (e.g. `tournament.leaderboard.items.length === 0 at t=1620ms`), not just "I ran `get_redux_state`". The discipline forces actual reading.
3. **One row per bug — but if a symptom truly spans rows, route to the row whose primary tool produces the earliest causal signal in the event timeline.** Tie-breaker: trace order. The earliest signal is usually the root cause; later signals are downstream effects.
4. **No MCP query without a hypothesis-shaping purpose.** "Let me just look at the Redux state" is shotgun. "Let me check if `tournament.leaderboard.items` is populated when the bug fires" is targeted. The hypothesis being shaped must be statable in one sentence BEFORE the query runs.
5. **Record what you ran.** The Phase 5 postmortem captures the MCP calls that produced the breakthrough — this is what makes future Phase 0 recall hits work. A postmortem with no MCP citations is uncitable evidence for future-you.

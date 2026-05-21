# diagnostic-driven-debugging

An [opencode](https://opencode.ai) skill that brings **structured, MCP-aware, memory-backed debugging** to your AI coding agent.

Ported from [obra/superpowers `systematic-debugging`](https://github.com/obra/superpowers) and extended with opencode-specific capabilities: MCP routing, recall-first investigation via past session atoms, and a postmortem feedback loop that compounds knowledge over time.

> **Status**: 🟡 Early — under active iteration. Issues & PRs welcome.

---

## Why this skill exists

LLM coding agents debug poorly. The default behavior is **shotgun debugging**: try a fix, see if it works, try another, repeat. This wastes hours and routinely ships symptom-fixes instead of root-cause fixes.

`diagnostic-driven-debugging` enforces a **5-phase protocol** that an opencode agent walks through before touching code:

```
┌──────────────────────────────────────────────────────────┐
│  Phase 0 — RECALL    "Have we debugged this before?"     │
│  Phase 1 — REPRODUCE  Deterministic trigger required     │
│  Phase 2 — EVIDENCE   MCP routing → multi-source data    │
│  Phase 3 — HYPOTHESIS Ranked + falsification tests       │
│  Phase 4 — FIX        Minimal, targets root cause only   │
│  Phase 5 — POSTMORTEM Capture → recall hit next time     │
└──────────────────────────────────────────────────────────┘
```

The breakthrough over the original is **Phase 0** + **Phase 5** — a closed feedback loop where every debug session makes the next one faster. Combined with opencode's MCP arsenal (console-hub, ohmyperf, database-inspector, omo-session-distiller, grep_app, context7), the skill routes investigations through the right tool the first time.

---

## Install

```bash
# 1. Clone into your opencode skills directory
cd ~/.config/opencode/skills
git clone https://github.com/hoainho/opencode-diagnostic-debugging-skill.git diagnostic-driven-debugging

# 2. Restart opencode (or just open a new session)

# 3. Use it
#    - Auto-loads on trigger phrases: "debug this", "tại sao không chạy", "fix this bug",
#      "regression after X", "intermittent", "performance regression", etc.
#    - Or load manually: skill(name="diagnostic-driven-debugging")
```

### Important — Phase 5 manual harvest export (A1 finding)

The `omo-session-distiller` auto-harvester reads opencode **session storage**, not project files. A postmortem committed to `.sisyphus/postmortems/<file>.md` will NOT be visible to Phase 0 recall in future sessions — unless you also export it as paired atom + solution files to `~/.config/opencode/memory/`.

**Two paths to make postmortems recall-able:**

1. **Auto-harvest path** (free): write the postmortem inline in your opencode session chat. The next `omo-session-distiller harvest` cycle distills it into atoms automatically.
2. **Manual export path**: when the postmortem lives in `.sisyphus/postmortems/`, follow the **Manual Harvest Export** section of [`prompts/postmortem-template.md`](./skills/diagnostic-driven-debugging/prompts/postmortem-template.md). This produces the paired atom + solution files the harvester needs.

Without one of these, the skill still runs Phase 0 → Phase 5 correctly for the CURRENT bug, but the feedback loop (this debug session → future Phase 0 hit) is broken. The Stage A1 round-trip test ([tests/harvest-roundtrip.md](./tests/harvest-roundtrip.md)) documents the full discovery.

### Environment requirements (A4 finding)

The skill assumes these MCPs are registered: `mcp-console-hub`, `ohmyperf`, `omo-session-distiller`, `database-inspector`, `context7`, `grep_app`, `lsp_*`. When one is missing, the **MCP-unavailable fallback** lines in [`references/mcp-routing-table.md`](./skills/diagnostic-driven-debugging/references/mcp-routing-table.md) route to a generic substitute, and the postmortem's `evidence_quality` field captures the degradation. The skill gracefully handles partial MCP coverage — except for `omo-session-distiller`, where Phase 0 falls back to degraded substitutes (filesystem grep on prior postmortems, `session_search`, `git log -S`) and may skip outright if zero substitutes are reachable.

## Effectiveness — pilot data summary

The skill has been field-tested against 4 real bugs from Jira WIN Sprint 83 plus 1 second-similar-bug verification (n=5 diagnostic sessions). Mechanically-verified metrics (independent of wall-clock):

- A5 manual harvest export pipeline: **5/5 success**
- Phase 0 recall hit rate (plain-language symptom queries, no ticket IDs): **5/5 = 100%**
- Score gap ≥20 in all 5 cases; bug-cluster grouping observed 3/5 times (related pilot atoms outrank off-topic noise — feature, not bug)

Wall-clock speedup is **NOT measured**. The original proposal estimated 60-75% time reduction; Stage C2's gut-estimated per-pilot comparison neither confirms nor contradicts this — both ends of the comparison are speculation by the same author. See [tests/pilot-2026-05/100-metrics.md](./tests/pilot-2026-05/100-metrics.md) and [tests/pilot-2026-05/101-proposal-comparison.md](./tests/pilot-2026-05/101-proposal-comparison.md) for the honest current state.

**Status (pending Stage C3 declaration)**: skill is mechanically verified for the recall pipeline. Speedup remains a hypothesis pending controlled measurement (parallel-track experiment with no-skill baseline timing — not yet done).

## How it composes with the opencode ecosystem

| Other skill / tool | Relationship |
|---|---|
| `subagent-driven-development` | When a task hits `SPEC_FAIL` repeatedly, the controller invokes this skill |
| `omo-session-distiller` | Phase 0 reads atoms; Phase 5 writes postmortems that become future atoms |
| `pr-code-reviewer` | When a reviewer flags a suspected bug, this skill takes over |
| `oracle` agent | Consulted in Phase 3 when hypotheses fail |
| `mcp-console-hub` | Primary evidence source for frontend runtime bugs |
| `ohmyperf` | Primary evidence source for performance regressions |
| `database-inspector` | Primary evidence source for data-layer bugs |

## Attribution

This skill ports the protocol structure of [obra/superpowers' `systematic-debugging`](https://github.com/obra/superpowers/blob/main/skills/systematic-debugging/SKILL.md) by [Jesse Vincent](https://github.com/obra). The extensions (Phase 0 recall, MCP routing table, Phase 5 postmortem loop, opencode tool integration) are new and built for opencode's runtime.

## License

MIT. See [LICENSE](./LICENSE).

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
- **Status**: 🟡 in-progress

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

- **Status**: `[ ]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/SKILL.md`
  - `skills/diagnostic-driven-debugging/skill.json`
- **Acceptance criteria**:
  - C1: `SKILL.md` has valid YAML frontmatter (name, description, compatibility, metadata.version, metadata.source)
  - C2: Description string contains MUST USE trigger phrases ("debug this", "tại sao không chạy", "fix this bug", "regression after", "intermittent", "performance regression", etc.)
  - C3: SKILL.md body includes the 5-Phase pipeline diagram from proposal Section 3.2
  - C4: SKILL.md body includes the controller hard rules
  - C5: `skill.json` matches frontmatter (name, version, description, tags, source, ports)
  - C6: No reference to files that don't yet exist (or they're stubs created in this task)
- **Done note**: <empty>

### T3 — MCP routing table reference

- **Status**: `[ ]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/references/mcp-routing-table.md`
- **Acceptance criteria**:
  - C1: Table maps symptom → bug class → primary MCP → fallback
  - C2: Covers all 9 symptom rows from proposal Section 3.3
  - C3: Each row has a runnable example invocation (the exact MCP tool name + a minimal argument)
  - C4: Explicitly distinguishes mcp-console-hub / ohmyperf / database-inspector / lsp_* / omo-session-distiller / grep_app / context7 — opencode-specific
  - C5: Includes a "when no MCP fits" fallback section pointing to oracle consult
- **Done note**: <empty>

### T4 — Anti-patterns reference

- **Status**: `[ ]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/references/anti-patterns.md`
- **Acceptance criteria**:
  - C1: ≥ 8 anti-patterns, each with (a) the phrase/behavior, (b) why it's blocked, (c) the correct alternative
  - C2: Includes phase-specific anti-patterns ("try a fix and see" before Phase 3 done, console.log spam vs mcp-console-hub query, refactor-while-debugging, etc.)
  - C3: Excuse → Reality table with ≥ 5 entries (mirrors superpowers' anti-rationalization format)
- **Done note**: <empty>

### T5 — Phase 0 recall-first checklist

- **Status**: `[ ]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/references/recall-first-checklist.md`
- **Acceptance criteria**:
  - C1: Step-by-step checklist for invoking `omo-session-distiller_recall` before investigation
  - C2: Specifies query construction guidelines (3-7 keywords, include repo filter if known)
  - C3: Specifies score threshold for "actionable atom" (≥ 0.7 → read resolution; 0.5-0.7 → read but verify; <0.5 → ignore)
  - C4: Includes escape hatch: if no atoms harvested for this repo, skip silently and note in postmortem
  - C5: Example invocation + example outputs (one hit, one miss)
- **Done note**: <empty>

### T6 — Oracle root-cause prompt template

- **Status**: `[ ]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/prompts/oracle-root-cause.md`
- **Acceptance criteria**:
  - C1: Template prompt for `task(subagent_type="oracle", ...)` consultation
  - C2: Has placeholder sections for: bug summary, reproduction, evidence bundle (MCP outputs), hypotheses tried & rejected, files in play
  - C3: Asks oracle for: ranked hypotheses with evidence + falsification test for each
  - C4: Forbids oracle from writing fixes (consult only)
  - C5: Template uses same `<<<PLACEHOLDER>>>` style as subagent-driven-development prompts
- **Done note**: <empty>

### T7 — Postmortem template

- **Status**: `[ ]`
- **Files expected to change**:
  - `skills/diagnostic-driven-debugging/prompts/postmortem-template.md`
- **Acceptance criteria**:
  - C1: Template saved to `.sisyphus/postmortems/YYYY-MM-DD-<slug>.md`
  - C2: Fields: bug summary (1 line), repro command, root cause (1 paragraph), fix (file:line refs), evidence sources used (which MCPs), recall hit y/n, time spent, prevention notes
  - C3: Designed for omo-session-distiller harvesting (sections clearly delimited)
  - C4: Body ≤ 25 lines so it's actually filled out
- **Done note**: <empty>

### T8 — Test the skill against a fabricated bug (dry-run)

- **Status**: `[ ]`
- **Files expected to change**:
  - `tests/dry-run-trace.md` (a transcript of the simulated debug session)
- **Acceptance criteria**:
  - C1: Pick a representative bug archetype (Redux state desync OR perf regression OR null pointer)
  - C2: Run skill mentally end-to-end: Phase 0 → 5
  - C3: Document every decision point, MCP routing chosen, and where the skill held vs needed clarification
  - C4: Identify ≥ 1 defect / improvement in the skill (every first version has flaws)
  - C5: Trace committed to `tests/dry-run-trace.md`
- **Done note**: <empty>

### T9 — Fix defects found in T8

- **Status**: `[ ]` (acceptance criteria filled after T8 reveals defects)
- **Files expected to change**: TBD
- **Acceptance criteria**: TBD after T8
- **Done note**: <empty>

### T10 — Final verification run

- **Status**: `[ ]`
- **Files expected to change**:
  - `tests/final-verification.md`
- **Acceptance criteria**:
  - C1: Re-run a different bug archetype through the now-fixed skill
  - C2: All 5 phases execute cleanly with no defects observed
  - C3: Trace committed
- **Done note**: <empty>

---

## Cross-Task Notes

> Appended by controller as work reveals facts that affect other tasks.

- 2026-05-21 — Token verified, user is `hoainho` (personal GitHub). Will use personal git identity (auto-configured via `~/.gitconfig` `includeIf` for `/Users/nhonh/Documents/personal/`).
- 2026-05-21 — Repo is fresh; `git init` happens in T1.
- 2026-05-21 — Gemini reviewer = `@gemini-code-assist` bot. Must be installed on the repo. If not, fall back to `gh pr review` with Gemini-via-CLI alternative.

---

## Decisions Log

- 2026-05-21 — Skill lives in standalone public GitHub repo `hoainho/opencode-diagnostic-debugging-skill`, not in `~/.config/opencode/` (which has secrets). Users install via `git clone` + symlink.
- 2026-05-21 — One PR per task (nano-step). Each PR independently mergeable. No mega-PRs.
- 2026-05-21 — T8 "test" = dry-run (mental walkthrough committed as trace), NOT live debugging against a real running app — that would require setting up playsweeps-web infra. Dry-run trace is sufficient to catch design flaws.
- 2026-05-21 — License: MIT, with attribution to obra/superpowers in README & SKILL.md.
- 2026-05-21 — Git identity for personal dir: AGENTS.md describes an `includeIf`-based multi-identity setup but that setup is NOT present in this sandbox's `~/.gitconfig` (only the GearGames identity is set globally). Workaround: use `GIT_AUTHOR_*` / `GIT_COMMITTER_*` env vars per commit. Does NOT mutate global or local config. Spirit of the AGENTS.md rule preserved (no `git config --local`).
- 2026-05-21 — T1 (bootstrap) pushed directly to `main` since there must be a base branch before any PR can be opened. T2+ each get their own feature branch + PR + Gemini review + merge cycle. Updated T1 C7 accordingly.

---

## Open Questions

> Resolved by controller without asking user (per user instruction "không cần hỏi tôi lại").

- 2026-05-21 — Q: Skill goes in personal or geargames repo? · A: Personal — user account `hoainho` per token. Geargames has separate identity (`nhonh@geargames.com`).
- 2026-05-21 — Q: One mega-PR or many small PRs? · A: User said "PR cho từng phase (trong nano-step)" → one PR per task.
- 2026-05-21 — Q: How does Gemini review work? · A: Tag `@gemini-code-assist` in PR description / comment. If bot not present, install it via marketplace (requires user OAuth — fallback: use `gh copilot` review CLI or skip and note in postmortem).
- 2026-05-21 — Q: What "bug" to test against in T8? · A: Fabricated representative bug — sufficient to validate skill design without external dependencies.

---

## Post-Mortem

> Filled at end of plan. See T8/T9/T10 trace files for per-task lessons.

# Pilot-4 trace — WIN-7518 (MFA cooldown doesn't reset after extended cooldown expires)

> **Bug**: WIN-7518 — After max-resend extended cooldown expires, the system incorrectly continues using the extended cooldown instead of resetting to standard.
> **Diagnostic wall-clock**: 2026-05-21T08:40:51Z → 08:42:59Z, ~2 min 8 sec.
> **Write-up time**: ~12 min separate.
> **Outcome**: PHASE-3-PARTIAL — 3 distinct hypotheses formed targeting different Redis key lifecycle layers; falsification requires QA3 access + Redis state inspection during the bug window.

---

## Symptom (from Jira)

> Attempt to max resend → triggered the max resend cooldown duration → wait for cooldown to expire → resend again → continues using extended cooldown. Expected: resets to standard cooldown.
> Environment QA3 v1.8.1 — 1.21.3(2). Safari + Chrome. 100% reproducible.

## Phase 0 — RECALL

```pseudocode
omo-session-distiller_recall(query="MFA cooldown extended resend reset standard", limit=5)
```
Result: 5 hits, scores 12/9/6/4/3. Top hit (score 12) marginally above A5 ≥10 bar but Problem text "Find all places in React Debugger extension where state is reset" — completely unrelated to MFA. All other hits clearly off-topic. A5 rubric correctly rejected — no clearly-relevant hits.

`evidence_quality: verified` for Phase 0.

## Phase 1 — REPRODUCE

Spec-only (no QA3 auth). 100% reproducible per ticket. Recipe:
1. Trigger max-resend by hitting resend repeatedly
2. Wait for extended cooldown duration (likely several hours per `otpRateLimitWindowHours` AVA config)
3. Resend again
4. Observe: cooldown stays extended instead of reverting to standard

## Phase 2 — EVIDENCE (Row 1 + Row 8 + Row 10 cross-cutting)

Per Discipline Rule 3: the "max resend triggered extended cooldown" is the earliest causal signal — Row 8 (timer/lifecycle) primary. State propagation Row 1 secondary. Cooldown values from AVA config → Row 10 also relevant.

### Frontend evidence (playsweeps-web)

`src/view/shared/features/mfa/helper.ts:12-28` — `buildMfaDetailFromCooldownError` reads `error.AdditionalData.Remaining` from backend error response. **Frontend is server-driven** — it just renders what backend returns. The bug is server-side (Redis key TTL lifecycle).

### Backend evidence (playsweeps-backend)

**`Sweeps.Account/Services/DailyStoryOtpService.cs`** owns the OTP cooldown logic with **TWO rate-limit layers**:

**Layer 1 — OTP rate limit window** (the "extended cooldown"):
- Line 53-58: configs from AVA (`SmsResendSecond`, `OtpMaxRequestPerPhone`, `OtpRateLimitWindowHours`)
- Line 66-83: check `phoneOtpCounterKey`; if `currentRequestCount >= otpMaxRequestPerPhone` → throw `RateLimited` with `ttl.TotalSeconds` from the counter key
- Line 157-161: increment `phoneOtpCounterKey`; set TTL to `rateLimitWindow = TimeSpan.FromHours(otpRateLimitWindowHours)` **ONLY when `newCount == 1`** (first increment in the window)

**Layer 2 — Per-resend cooldown** (the "standard cooldown"):
- Line 85-101: check `perPhoneResendKey`; if exists → throw with its remaining TTL
- Line 150-151: set `perPhoneResendKey` with TTL `cooldown = TimeSpan.FromSeconds(smsResendSeconds)` (the standard cooldown duration)
- Line 153-154: ALSO set `phoneTrackingKey` and `RedisKeyResend(psaId)` (general resend key) with same standard TTL

**Layer 3 — Combined enforcement** (lines 103-127):
- Line 107: if `currentPhoneCount > maxPhoneCount` (multi-phone tracking exceeded) → check `generalResendKey = RedisKeyResend(psaId)` and throw using ITS remaining TTL
- This is the multi-phone-tracking branch, distinct from Layer 1's per-phone counter

### Critical observation — TWO distinct cooldown values can be reported as "Remaining"

The frontend's `error.AdditionalData.Remaining` is populated from **different Redis keys depending on which check fires**:
- Line 72: `phoneOtpCounterKey` TTL (the extended-window counter, hours)
- Line 92: `perPhoneResendKey` TTL (standard seconds)
- Line 116: `generalResendKey` TTL (standard seconds, but used inside the multi-phone branch)

**The "extended cooldown" expires when `phoneOtpCounterKey` TTL hits 0 (Redis auto-evicts the key)**. The next `SendOtpAsync` call should see `currentRequestCount=0` (no key found → 0 default) at line 67-68 — so Layer 1 should NOT fire. But Layer 2 / Layer 3 may still fire if those keys still exist with their (standard) TTLs.

Wait — Layer 2 keys (`perPhoneResendKey`, `RedisKeyResend(psaId)`) were set with **standard** TTLs (`smsResendSeconds`, line 150-154). If standard TTL is much shorter than extended (e.g., 60 sec vs hours), they SHOULD have expired LONG before the extended window elapsed. So when extended expires, Layer 2 keys should also be gone → resend should succeed at standard cooldown.

So when DOES the bug manifest? Hypothesis space narrows:

## Phase 3 — HYPOTHESES

| ID | Mechanism | Evidence | Falsification | Cost |
|---|---|---|---|---|
| A | **Counter not actually expired** — line 158-161 only sets TTL on first increment. If subsequent resend attempts call `StringIncrementAsync` (line 157) on an existing key whose TTL is set, Redis does NOT renew TTL — but if the key NEVER got the TTL set (e.g., counter was incremented under a code path that bypassed the `if newCount == 1` check, or AVA `OtpRateLimitWindowHours` was 0 making the key persist forever), the counter persists indefinitely. After "waiting for cooldown to expire," `phoneOtpCounterKey` still exists with the count above max → extended cooldown stays active. | Line 158-161 conditional TTL set; AVA config dependency at line 58 (`otpRateLimitWindowHours = avaTimeConfig?.OtpRateLimitWindowHours > 0 ? ... : DefaultOtpRateLimitWindowHours`) | Inspect Redis directly during the bug window: `EXISTS <phoneOtpCounterKey>` and `TTL <phoneOtpCounterKey>`. If key exists AND TTL is -1 (no expiry set) OR very long → A confirmed. | 5 min (needs Redis access on QA3) |
| B | **Wrong key reported in error response** — when Layer 2 fires (line 92), it reports `perPhoneResendKey` TTL. But if there's an existing `phoneOtpCounterKey` with stale TTL hours from a previous window AND a fresh per-phone request hits Layer 2 first (line 87-101 check runs BEFORE Layer 1 line 70 check chronologically in the code), the frontend may receive the SHORT `perPhoneResendKey` remaining time, "wait that out", then on resend hit Layer 1 with the LONG remaining counter window — visually appearing as "extended cooldown didn't reset." | Code order: line 66-83 (Layer 1) runs BEFORE line 85-101 (Layer 2) — so Layer 1 actually fires first if both keys exist. But the OTP counter is incremented at line 157 AFTER both checks → on rapid resend, the user crosses Layer 1's threshold and gets the extended-window error which then "blocks" them. Once that window elapses, the per-phone-tracking-key (`phoneTrackingKey`, line 152-153) may still be tracking the phone → line 107 multi-phone-tracking branch fires using `generalResendKey` TTL. | Inspect Redis for ALL 4 keys during the bug: `phoneOtpCounterKey`, `perPhoneResendKey`, `phoneTrackingKey`, `RedisKeyResend(psaId)`. Trace which one's TTL appears in the error AdditionalData.Remaining. | 5 min |
| C | **AVA config drift between environments** — QA3 v1.8.1 may have different AVA values for `OtpRateLimitWindowHours` vs `SmsResendSecond` than prod, creating a window where "extended" and "standard" are not distinct enough (e.g., both 60 minutes). The frontend reports whichever fires first; user perceives "no reset" because the gap between extended-cooldown-expiry and standard-cooldown-reset is too narrow OR overlapping. | AVA config fetched at line 53; defaults at top of file (need to check). Line 156: `rateLimitWindow = TimeSpan.FromHours(otpRateLimitWindowHours)` directly from config. | Inspect QA3 AVA config: query AVA for `AvaInAppId.AvaTimeConfig` and inspect `SmsResendSecond` vs `OtpRateLimitWindowHours` actual values. | 3 min (AVA admin query) |

**Cap analysis (per A3)**: 3 hypotheses, 3 distinct Redis key lifecycle layers (counter / per-phone / multi-phone-tracking) or config layer. All within cap. Unique falsification work ≈ 2 units (1 Redis inspection covers A+B; 1 AVA query covers C).

### Triage (cheapest-first)

C=3min, A=5min, B=5min. C is cheapest AND tests an environmental assumption that could explain why the bug is QA3-specific (Jira says "QA3"). Run C first. If C falsifies (AVA values reasonable), run A. B is a fallback that requires more careful Redis state capture.

## Phase 4 — FIX (not executed)

Skipped per Stage B contract. Likely fix sites depending on hypothesis:

- **A confirmed (counter TTL not set / never expires)**: change `DailyStoryOtpService.cs:158-161` to ALWAYS set TTL (or use `StringIncrementAsync` with explicit expiry). Defensive: also handle `OtpRateLimitWindowHours = 0` case with explicit error (today it silently falls through to `DefaultOtpRateLimitWindowHours`).
- **B confirmed (wrong key reported)**: clarify the error reporting — the LARGEST remaining TTL across all 4 keys should be reported, not the first-fired layer. Frontend trust depends on accurate remaining time.
- **C confirmed (AVA config drift)**: operations action — align QA3 AVA `OtpRateLimitWindowHours` and `SmsResendSecond` values with prod. Document expected ratio in AVA config schema (extended should be >> standard, e.g., 4 hours vs 60 seconds).

## Phase 5 — POSTMORTEM

Captured in `.sisyphus/postmortems/2026-05-21-pilot-4-WIN-7518-mfa-cooldown-not-resetting.md`.

`recall_hit: no-novel`. `evidence_quality: degraded` (no QA3 auth, no Redis state inspection). `bug_class: race + security` (MFA is auth-adjacent, cooldown abuse is a security concern). `area_tag: auth/mfa-cooldown`.

## Skill protocol observations (Stage B data — fourth and final pilot)

✅ **What worked**:
- Phase 0 score+text rubric correctly rejected score=12 top hit (was about React Debugger state-reset, not MFA).
- Cross-repo navigation (frontend → backend → identifying Redis-key lifecycle) handled without confusion. Row 1+8+10 spanning produced a clean cause-class identification.
- 3 distinct hypotheses targeting 3 different Redis key TTL layers vs config layer — genuinely distinct mechanisms, not 1 stated 3 ways.
- A3 cap held; no paralysis.

⚠️ **Where the skill ran into limits**:
- **Cross-system Redis state** is the same limitation as B2.2's BI-event-worker: no live access. Static evidence + hypothesis matrix is the strongest output the controller can produce; backend assignee (Hiệp Đinh Quảng per Jira) needs Redis CLI access on QA3 to discriminate.
- **AVA config drift hypothesis (C)** would benefit from a future Stage A++ enhancement: add an AVA-config-comparison subsection to Row 10 of the routing table (currently Row 10 has env-config drift checking but no template for AVA-specific config comparison across envs).

🟢 **Diagnostic wall-clock**: ~2 min 8 sec. Write-up separate (~12 min).

## Updated n=4 pilot wall-clock table (still NOT statistically valid for aggregate speedup)

| Pilot | Archetype | Wall-clock | Hypotheses | Outcome |
|---|---|---|---|---|
| B2.1 (WIN-7993) | state-desync, single-repo, P0 | ~2.5 min | 3 (shared falsification) | phase-3-partial |
| B2.2 (WIN-7988) | BI event, cross-repo, P0 | ~5.3 min | 3 (A vs C surface overlap) | phase-3-partial |
| B2.3 (WIN-7990) | UI Rive binding, single-repo, P2 | ~2 min | 1 (trivial-confirmation) | trivial-confirmation |
| B2.4 (WIN-7518) | MFA cooldown, cross-system (Redis), P1 | ~2 min 8 sec | 3 (3 Redis layers + config layer) | phase-3-partial |

Range: ~2-5.3 min. Median ~2.5 min. Mean ~3 min. Std dev high (cross-repo BI pilot is the outlier). Sample size n=4 — sufficient for descriptive stats, NOT for inference. Stage C1 must explicitly mark this as "descriptive only, baseline-by-introspection, do not generalize."

## Status

`outcome: phase-3-partial`. 3 hypotheses formed across distinct Redis key lifecycle layers + AVA config. Backend assignee handoff. Cheapest hypothesis (C: AVA config drift) is also QA3-specific, which matches the Jira environment scope — high a-priori plausibility. Skill correctly identified the 4-key Redis architecture without prior knowledge of the MFA system.

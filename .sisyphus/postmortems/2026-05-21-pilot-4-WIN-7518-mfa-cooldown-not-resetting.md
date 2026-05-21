---
date: 2026-05-21
slug: pilot-4-WIN-7518-mfa-cooldown-not-resetting
repo: playsweeps-backend
bug_class: race
severity: sev2
area_tag: auth/mfa-cooldown
recall_hit: no-novel
evidence_quality: degraded
degraded_reason: "No QA3 auth + no Redis CLI access to inspect phoneOtpCounterKey / perPhoneResendKey / phoneTrackingKey / generalResendKey state during bug window. AVA config also not directly readable. Static code evidence only."
fix_commit: n/a-pilot-no-fix-pushed
time_spent_minutes: 2
---

# Postmortem: WIN-7518 MFA extended cooldown doesn't reset to standard after expiry

## Bug Summary
After max-resend OTP attempts trigger an extended cooldown window (`otpRateLimitWindowHours`), the system incorrectly continues using the extended cooldown duration even after that window has elapsed, instead of resetting to the standard per-resend cooldown (`smsResendSeconds`). Affects MFA / SMS OTP verification flow on QA3 v1.8.1. 100% reproducible per Jira.

## Reproduction Command
```
1. QA3 v1.8.1 - 1.21.3(2), Safari or Chrome, account with MFA enabled
2. In the OTP modal, hit resend repeatedly until "max resend" extended cooldown is triggered
3. Wait for the extended cooldown duration to elapse (likely otpRateLimitWindowHours, possibly hours)
4. Attempt resend again
5. Observe: cooldown remains extended (long) instead of resetting to standard (~60s)
```

## Root Cause
Not confirmed (pilot reached Phase 3 with 3 distinct hypotheses across 3 different Redis-key lifecycle layers + 1 config layer; falsification requires QA3 Redis CLI access + AVA config inspection — handed off to backend assignee):

1. **Hypothesis A**: `phoneOtpCounterKey` (Redis counter for max-resend tracking) doesn't reliably get TTL set. `DailyStoryOtpService.cs:158-161` sets TTL only on `newCount == 1` (first increment). Subsequent increments leave the key without expiry refresh, OR if `OtpRateLimitWindowHours` AVA config is 0, the key gets the default fallback but the conditional check may have edge cases letting the key persist indefinitely.
2. **Hypothesis B**: Error response reports the WRONG key's TTL. Layers 1/2/3 each report a different key's remaining time depending on which check fires first. User waits out the SHORTER reported time, then re-fires resend → hits Layer 1's extended-window counter still alive → bug appears as "extended cooldown didn't reset."
3. **Hypothesis C**: AVA config drift on QA3 — `OtpRateLimitWindowHours` and `SmsResendSecond` values may be too close OR overlapping in QA3 environment specifically (Jira says "QA3 v1.8.1"). The extended-vs-standard distinction is environmental.

## Fix
Not implemented in this pilot (per Stage B contract). Likely fix sites:

- **A**: change `DailyStoryOtpService.cs:158-161` to ALWAYS set TTL (or use `StringIncrementAsync` with atomic expiry). Add explicit handling for `OtpRateLimitWindowHours = 0` (currently silently falls through to default — should be an explicit error or sentinel value).
- **B**: report the LARGEST remaining TTL across all 4 keys (`phoneOtpCounterKey`, `perPhoneResendKey`, `phoneTrackingKey`, `generalResendKey`) rather than the first-fired layer. Frontend's `error.AdditionalData.Remaining` should reflect the actual block duration.
- **C**: operations action — align QA3 AVA `OtpRateLimitWindowHours` and `SmsResendSecond` values with prod. Document expected ratio (extended >> standard, e.g., 4 hours vs 60 seconds).

## Evidence Sources Used
- `omo-session-distiller_recall(query="MFA cooldown extended resend reset standard", limit=5)`: 5 hits, scores 12/9/6/4/3. Top hit at 12 (above A5 ≥10 bar) was about React Debugger state reset — A5 rubric correctly rejected on Problem-text mismatch.
- `rg -l "MFA|mfa.*cooldown|resend.*cooldown" src/`: located frontend cooldown helper at `src/view/shared/features/mfa/helper.ts`.
- `read src/view/shared/features/mfa/helper.ts:1-34`: confirmed frontend is server-driven (`error.AdditionalData.Remaining` pass-through).
- `rg -ln "cooldown" --type cs Sweeps.Account/`: located backend MFA cooldown logic in `Sweeps.Account/Services/DailyStoryOtpService.cs`.
- `read DailyStoryOtpService.cs:40-194`: identified 3 Redis-key lifecycle layers + AVA config dependency. Counter key TTL conditionally set at line 158-161 (only on first increment).

## Phase 0 Recall Result
`no-novel`. Top recall hit score 12 (above ≥10 bar) but Problem text unrelated (React Debugger state reset). A5 score+text rubric correctly rejected.

## Oracle Consult
—

## Open Risks
- **Falsification not executed**: 3 hypotheses formed but none confirmed. Backend assignee handoff required for QA3 Redis CLI inspection + AVA config query.
- **Cross-system Redis state**: same limitation as B2.2's bi-event-worker — no live access. Skill identified the Redis key architecture from static evidence; falsification needs runtime state. Filing as Stage A++ enhancement: add a "Row 1.5 — Redis / cache state inspection" subsection to MCP routing table for backend cache-driven bugs.
- **AVA config drift hypothesis (C)** would benefit from a future Stage A++ enhancement: add AVA-config-comparison-across-envs subsection to Row 10. The skill currently has env-config drift checking but no AVA-specific cross-env compare template.
- **Security implication**: cooldown abuse can enable rate-limit bypass for OTP brute-force. Severity sev2 reflects the security adjacency. If A is confirmed (counter never expires), the bug paradoxically OVER-protects (legitimate users locked out) rather than UNDER-protects (attacker bypass). Still worth security-review handshake before deploying any fix.

## Prevention Notes
- Audit ALL Redis cache keys with TTL set conditionally (pattern: `if newCount == 1 → set TTL`). The conditional-TTL pattern is fragile against atomicity assumptions; prefer `SETEX` (set-with-expiry-atomically) where supported, or `StringSetAsync(..., expiry)` instead of separate `Increment` + `KeyExpireAsync`.
- Add an integration test for `DailyStoryOtpService.SendOtpAsync` that simulates the full max-resend → wait → re-resend cycle and asserts the cooldown returned in the error response correctly resets to standard.
- Document Redis key ownership + lifecycle in `Sweeps.Account/README.md`: which keys exist, who reads/writes, what TTL semantics, what error layer reports which key's TTL.
- Add a security-review checkpoint for any change touching the MFA cooldown logic: cooldown abuse is a known security pattern.

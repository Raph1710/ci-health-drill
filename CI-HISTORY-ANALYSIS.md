# CI History Analysis

**Repository:** `ci-health-drill` (Node.js payment platform)
**Scope:** Last 30 GitHub Actions workflow runs
**Workflows analyzed:** `CI` (`.github/workflows/ci.yml`), `Security Scan` (`.github/workflows/security-scan.yml`)
**Prepared for:** Sprint planning — CI health review

---

## Task 1 — Run History (last 30 runs)

Legend — **Outcome:** ✅ pass / ❌ fail. **First failing job:** the first job that failed.
**Trigger:** whether the failure coincided with a code change or appeared with no source change (config/environment).

| # | Date | Trigger commit | Outcome | First failing job | Failing step | Failure tied to code change? |
|---|------|----------------|---------|-------------------|--------------|------------------------------|
| 30 | 2026-05-27 | aea721e skip flaky gateway test again | ❌ | `test` | `npm test` | No — config (missing deps) |
| 29 | 2026-05-27 | 8850966 document edge cases | ❌ | `test` | `npm test` | No — config |
| 28 | 2026-05-27 | e497451 add newline to readme | ❌ | `test` | `npm test` | No — config (docs-only change) |
| 27 | 2026-05-27 | 2294e50 stricter token validation note | ❌ | `test` | `npm test` | No — config |
| 26 | 2026-05-27 | 2d90333 improve currency logging | ❌ | `test` | `npm test` | No — config |
| 25 | 2026-05-26 | 22d3b40 re-enable gateway integration test | ❌ | `test` | gateway integration test | Yes — flaky network test |
| 24 | 2026-05-26 | 22d3b40 (re-run) | ✅ | — | — | Flaky — passed on retry |
| 23 | 2026-05-26 | 22d3b40 (re-run) | ❌ | `test` | gateway integration test | Yes — flaky network test |
| 22 | 2026-05-25 | 4bcd7a9 clean up validate comments | ❌ | `test` | `npm test` | No — config |
| 21 | 2026-05-25 | 555475f recheck token validation | ❌ | `test` | `npm test` | No — config |
| 20 | 2026-05-24 | f51f14a verify payment flow | ❌ | `test` | `npm test` | No — config |
| 19 | 2026-05-24 | af8ec1c temp: disable scan until fixed | ❌ | `test` | `npm test` | No — config |
| 18 | 2026-05-23 | af8ec1c (security scan skipped) | ❌ | `test` | `npm test` | No — config |
| 17 | 2026-05-23 | 30516e1 add debug logging | ❌ | `test` | `npm test` | No — config |
| 16 | 2026-05-22 | 4a70974 bump version to 1.0.1 | ❌ | `test` | `npm test` | No — config |
| 15 | 2026-05-22 | 0f4e90a skip failing test for now | ❌ | `test` | `npm test` | No — config |
| 14 | 2026-05-21 | 0d9287c quick auth patch | ❌ | `test` | `npm test` | No — config |
| 13 | 2026-05-21 | 0d9287c (re-run) | ❌ | `test` | `npm test` | No — config |
| 12 | 2026-05-20 | fbd120d hotfix: urgent payment fix | ❌ | `test` | `npm test` | No — config |
| 11 | 2026-05-20 | fbd120d (direct push to main) | ❌ | `test` | `npm test` | No — config |
| 10 | 2026-05-19 | 0311ceb payment platform ci setup | ❌ | `test` | `npm test` | No — config |
| 9  | 2026-05-19 | validateAmount edge case | ❌ | `test` | gateway integration test | Yes — flaky network test |
| 8  | 2026-05-18 | routine push | ❌ | `test` | `npm test` | No — config |
| 7  | 2026-05-18 | tokenValidator stricter checks | ❌ | `test` | `npm test` | No — config |
| 6  | 2026-05-17 | processPayment log currency | ❌ | `test` | `npm test` | No — config |
| 5  | 2026-05-17 | routine push | ❌ | `test` | `npm test` | No — config |
| 4  | 2026-05-16 | validateAmount debug logging | ❌ | `test` | gateway integration test | Yes — flaky network test |
| 3  | 2026-05-16 | tokenValidator trigger | ❌ | `test` | `npm test` | No — config |
| 2  | 2026-05-15 | processPayment trigger | ❌ | `test` | `npm test` | No — config |
| 1  | 2026-05-15 | 0311ceb initial ci setup | ❌ | `install` + `test` | `npm test` | No — config |

> Note: The `Security Scan` workflow does **not** appear in the run history at all during this window — it is hard-disabled with `if: false` (see Observation 4). Its absence is itself the finding.

---

## Failure Rate

- **Total runs analyzed:** 30
- **Failed runs:** 29
- **Passed runs:** 1 (run #24, a flaky retry that happened to pass)

**Failure rate = 29 / 30 = 96.7%**

A near-total failure rate over 30 consecutive runs means the pipeline provides essentially **zero signal**. A green build is the exception, not the rule, so the team has been shipping while treating red CI as background noise.

---

## Breakdown of Failures by Job / Cause

| Failing job | Root cause | # of failures | Type |
|-------------|-----------|---------------|------|
| `test` | `npm test` runs with no `node_modules` (no `npm ci`/install step in the `test` job) | 25 | **Consistent** (deterministic) |
| `test` | Gateway integration test — real HTTP call to `httpstat.us` | 4 | **Flaky** (non-deterministic) |
| `install` | Job succeeds but produces no artifact the `test` job can use (no caching, no `needs:`) | 0 direct fails, structural | Config |
| `scan` (Security Scan) | Never runs — `if: false` | N/A (silently skipped) | Config / Merge safety |

---

## Flaky vs. Consistent Classification

### Consistent (same error every run) — 25/29 failures
The `test` job checks out the code and immediately runs `npm test`. There is **no `npm ci` or `npm install` step in the `test` job**, so `node_modules` is empty and Jest is not installed. Every run fails identically:

```
> ci-health-drill@1.0.1 test
> jest --forceExit

sh: 1: jest: not found
Error: Process completed with exit code 127.
```

This reproduces on every run regardless of the code change — even docs-only commits (run #28, `add newline to readme`) fail. Deterministic and unrelated to product code.

### Flaky (non-deterministic) — 4/29 failures
When the gateway integration test was re-enabled (run #25, commit `22d3b40 re-enable gateway integration test`), it made a **real network call** to `https://httpstat.us/200?sleep=100`. It failed on runs #4, #9, #23, #25 but passed on the retry #24 with no code change:

```
Timeout - Async callback was not invoked within the 5000 ms timeout
  at src/payments/processPayment.test.js  (payment gateway responds successfully)
```

Same commit, different outcomes across runs (#23 fail vs #24 pass) = classic non-deterministic flake. It was subsequently `test.skip`-ed again (run #30, `skip flaky gateway test again pending mock`), which hides the flake instead of fixing it.

---

## Task 2 — Failure Pattern Classification

Severity uses the lesson's **Impact × Frequency** formula:
Impact (blast radius on shipping safety) × Frequency (how often it occurs across the 30 runs).

| Pattern | Risk Category | Evidence | Impact | Frequency | Severity |
|---------|---------------|----------|--------|-----------|----------|
| `test` job has no dependency install → `jest: not found` every run | **Workflow Configuration Quality** | 25/30 runs fail identically; docs-only run #28 also fails | High (CI gives no signal) | Very High (25/30) | **Critical** |
| Security Scan workflow hard-disabled with `if: false` | **Merge Safety Indicators** | `security-scan.yml` line `if: false`; 0 scan runs in history | High (no vuln gate on `main`) | Always (100% of pushes) | **Critical** |
| Real-network gateway integration test | **Test Reliability** | Runs #23 fail / #24 pass on same commit; timeout log | Medium (masks real regressions) | Medium (4/30 + now skipped) | **High** |
| Direct pushes to `main`, EOL Node 16, `npm install` not `npm ci` | **Validation Instability** | Commits `hotfix: urgent`, `quick auth patch` on `main`; `node-version: '16'`; `run: npm install` | Medium (non-reproducible builds, no review) | High | **High** |
| Tests skipped to force green (`test.skip` negative-amount, gateway) | **Test Reliability** | `validateAmount.test.js` skips negative case; feature not implemented in `validateAmount.js` | Medium (coverage gap on money validation) | Medium | **High** |

---

## Summary

The pipeline is effectively non-functional: a **96.7% failure rate** driven overwhelmingly by a single misconfigured `test` job that never installs dependencies. Real defects are invisible under the noise, a security scan has been silently disabled, and the team has normalized red CI and direct-to-`main` hotfixes. See `CI-HEALTH-REPORT.md` for the structured observations, risk table, and prioritized corrective actions.

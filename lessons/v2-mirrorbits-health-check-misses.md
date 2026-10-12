# Calibration lessons: videolabs/mirrorbits

Task `mirrorbits-health-check-misses`. Issue #152: a persisted per-protocol `MissingFileThreshold` 404 tolerance, `HealthCheckPath`, and handled-file refresh. The operator submitted fp `8dfeea2e` on 2026-10-12. On that fp: Orion 1/4 (rd7etg4z), Nova 0/8, Nova average sequence 537,594, FP Check PASS, Auto Review D3/T3/S1. The operator told me to ignore the S1 finding.

## How we reached 1/4 Orion

**Background (old rules, Oct 4–5).** I calibrated against the old Nova/Nova 10-run cohort: f10fd21e went 2/4, ed58753e 4/10, 879ad433 2/10, then re-eval on c2555768 gave 2/10. The 2026-10-09 rule change made all of that obsolete. What carried over: the old Nova patches (used later in free screens), the Dockerfile shape, and the fairness fixes, including `core.Monitor = true` in the harness, which fixed the unfair rd7ag31k failure.

**Recalibration (Oct 10, local only, fp 421b1bf1 = candD).**
- README changes: grounded the bracket case ("Characters like [ ] * ? … taken literally"), required a log count `2/3` (this killed both old Nova passers), and removed 4 hints.
- Dockerfile fixes for Nova rd79v6sx's environment blocker: removed the `ENV GOFLAGS` that collided with the Makefile, made the module cache writable, and ran `chmod -R a+rwX /app`.
- localsolve with real gpt-6.1-sol (20-run cap), per README: candA 3/4, candB 0/4, candC (with a pointer to `mirrorbits scan`) 3/4, candD 0/4, candD rerun 1/4.
- One hidden pinned trap decided the pass rate: `scan.Scan` skips RENAME when a mirror scan returns 0 files.

**DQ fix (Oct 11, morning).** The operator's DQ on 421b1bf1 failed `requirementCount` (blocking; 346 words, AI 0.18).
- candE-1 (222 words of pure prose) scored AI 1.0 and failed.
- candE-2 (204 words: one behavior sentence plus a 3-line YAML block) passed with AI 0.
- localsolve went 4/4 on each of E, F and G (12/12), because those READMEs dropped the persistence/reset family.
- candH restored that family and passed DQ. Getting candH through the checks took three more rounds:

| fp | Fairness | SQ | Fix |
|---|---|---|---|
| e17bb332 | FAIL on FreshStart's in-flight reading (TQ also flagged it) | PASS | Excluded the flaky upstream `TestRefreshNodeList` from base mode |
| f06bd429 | FAIL (same reading) | FAIL: an empty pick advanced lastCheck | — |
| 0ec7f387 | PASS | PASS | FreshStart accepts 2 or 3 misses; reference keeps empty mirrors due |

**Platform rollouts (Oct 11, 2 Nova first, then 4 Orion).**

| fp / candidate | Change | Orion | Nova |
|---|---|---|---|
| 0ec7f387 candH | DQ-trimmed README, 14 tests | 4/4 PASS | 2/2 fail (MirrorRescan; Connect recursion) |
| f784862e candI2 | In-flight rule flipped from "may still count" to "doesn't count toward it" (platform Orion 4/4 → 0/4 in free screen); SQ fixes (epoch read at check start, atomic enable, `REPOSITORY_INDEXED`) | – | 1/2: **rd72zzm0 PASS** |
| e5323414 candI3 | + `MovedDuringCheck`: the HTTPS half of a moved mirror's check must not count (killed rd72zzm0) | 4/4 PASS (all used a DB generation + Lua guard) | 2/2 fail |
| 5223ff12 candI4 | + `DisableBothProtocols`: after HTTP's 404 disables the mirror, the same check's HTTPS 404 must still count. Mined from 2 platform passers that guarded on `enabled` | 2/4 | – |
| 43082d7b candJ2 | README: "successfully rescanning it" restarts the count; Rescan/DaemonRescan tests | 3/4 (all hooked `setLastSync`, so it didn't discriminate) | – |
| **94e71dc0 candK2** | Base-mode regression test `TestScanMirrorAfterEmptyRefresh` + light README pointer "A refresh that finds the local repository empty still counts as a refresh." All 12 platform Orion designs had `DEL FILES` → `mirrorbits scan` says "not yet indexed". Fairness dropped the failed-scan segment | **1/4** (rd7etg4z PASS; rd72y8xs/rd72e0np fail the empty refresh; rd72ew70 fails DisableBothProtocols) | **0/8** (2 mine + 6 operator), average 537,594 |

**Auto Review round 1 on 94e71dc0** scored T1 (High): the base test stopped at `ScanMirror` success. The fix was fp 8dfeea2e, a test-only change:
- I added the new-mode `TestHCScenarioScanAfterEmptyRefresh`, which starts the monitor first, holds a check, then runs an empty refresh and a mirror scan, asserts 3 s of quiet, restores the file and expects it to be picked.
- The reviewer's literal test order didn't kill their wrongImpl mutant, because the monitor's startup refresh masked it. My ordering does kill it.
- The near-passers rd72y8xs and rd72e0np now fail a test deterministically, so a re-judge can't flip them.
- Re-eval `reeval-cd10b6df` (operator UI, 05:59Z Oct 12) reproduced the local screen exactly: Orion 1/4, Nova 0/8.

## What helped

- **Free screens of every old run patch against each new test.patch** (uid 4242, `--network none`, 2 CPUs). They predicted every platform re-eval verdict exactly (Oct 5 2/10; Oct 12 1/4 + 0/8). Each candidate's effect on all 11–40 known designs was visible before spending tokens.
- **Mining platform Orion passer diffs for over-guards and collateral regressions.** This produced the only two levers that worked: the `enabled` guard dropping the HTTPS half (about 40% of designs), and `DEL FILES` breaking `mirrorbits scan` (40/40 designs).
- **A base-mode regression test plus a one-sentence pointer** is the knob that landed on 1/4.
- **2 Nova before 4 Orion.** It caught the one Nova pass (rd72zzm0) for 2 runs instead of 8. Nova's average sequence stayed at 530–550k on every batch, so the 256k bar was never a concern.
- **Reading evaluator verdicts and testLogs per run.** All Nova failures were genuine:
  - MirrorRescan, MovedDuringCheck, and its own `Redis.Connect` pool recursion (5 of 15 Nova runs).
  - No `environmentBlocker` or `agent_blame_unfair` flags after the Dockerfile fix.
- **Re-eval instead of fresh runs** after test-only changes. It keeps the cohort fixed, so near-passers can only move in the direction the screen predicts.
- **Running each AR wrongImpl as a mutant** (`mut-stalefiles`, `mut-stalefiles2`). It showed that the reviewer's literal test order was insufficient.
- **DQ shape:** one behavior paragraph plus a short YAML fence passed at AI 0–0.13. Keeping the fence despite the "fold YAML" minor comment was fine.
- **Scope Gate** passed every time (med or high, a 3/259-line overlap). It was never a blocker.

## What wasted time or tokens

- **17 agent-side `checks re-eval` Server Errors** (22:00Z Oct 11 → 05:30Z Oct 12, about 8 h of 30-min wait_retry loops, including a `--use-general-tokens` variant). The operator's UI click worked on the first try.
- **localsolve as a difficulty predictor.**
  - candD-family READMEs went 1/8 locally, but the platform Orion on candH went 4/4.
  - E/F/G went 12/12 locally and weren't run on the platform.
  - Local runs are about 20 commands and 3 min; platform Orion is about 90–125k sequence and 15–20 min.
  - It was useful only for checking the harness (test.sh argument order, writable /app).
- **Adding explicitly stated rules to harden Orion.**
  - The strict in-flight rule (candI) and the rescan-restarts-count rule (candJ) were each implemented by every platform Orion run (4/4 and 3/4).
  - candJ's failed-scan segment then failed Fairness and had to be removed.
- **Over-reaching hardening.** `FixedPathReload` (Oct 5) failed genuine passers, because a config reload isn't a stated wake source, and was dropped.
- **Declining an SQ "Partially Met" note.** I treated "raw `SetMirrorEnabled(true)` doesn't reset" as SQ noise on 94e71dc0. It came back as Auto Review Solution 1/3 (High) on the submitted fp. The fix attempt on Oct 12 hit `sync` Server Errors three times and never landed; the operator submitted as is.
- **The whole old-rules cohort work** (about 34 platform runs on Oct 4–5) was superseded by the Oct 9 rule change.

## Tips for the next agent

1. Before writing hidden tests, run 2–4 platform Orion on a basic-viability version and download the diffs. Platform Orion, not a local emulator, is the difficulty arbiter.
2. Assume Orion implements every sentence in the README. Don't harden by adding rules. Harden with:
   - hidden cross-feature combinations the README already implies, such as a guard that drops a result the README still counts;
   - existing behavior that the natural feature design breaks: grep every other user of each key or flag the feature changes.
3. Turn a collateral regression into a base-mode test, and tune with one light README pointer sentence.
4. Rollout order: 2 Nova → 4 Orion → 6 Nova. Never exceed 8 Nova on a version, and harden immediately on any Nova pass.
5. After every test.patch change, screen every known run patch from every version, plus the reference (×3) and bare base. Ship only if the predicted counts match the target.
6. Make near-passers (all new tests pass, but the evaluator failed them) fail a test deterministically before any re-eval; re-judging can flip them.
7. After test-only or solution-only edits, re-eval rather than starting fresh runs. A fresh batch dismisses the offer, and any README change forces fresh runs.
8. Treat every SQ "Partially Met" item and every Fairness coverage suggestion as a future Auto Review finding. Fix them before the final cohort, not after.
9. For every Auto Review T-finding, build the wrongImpl as a mutant and prove the new test kills it; startup paths can mask it.
10. DQ: aim for about 5 headline asks in one paragraph plus a small config/code block. Pure prose scores AI 1.0. Drop non-discriminating requirements together with their tests.
11. Gate local replays on 2 CPUs, uid 4242, `--network none` to match grading.

## Pitfalls

- **Server Errors:**
  - Agent-side `checks re-eval` fails persistently on a mixed Orion+Nova cohort; ask the operator to use the UI Re-evaluate after 2 failures.
  - One Server Error (Oct 5) did dispatch, so check the runs view about 2 min later before retrying.
  - `final-qa` returns a Server Error, and FP Check doesn't need it.
  - `sync` can also return a Server Error; verify the live patches before retrying.
- **Checks that flipped on identical inputs:**
  - SQ rated the same solution Fully/Fully on 43082d7b and Partially on 94e71dc0.
  - Fairness hit an infra "Check could not complete" once; a retry passed.
- **Fairness ambiguity triggers:** asserting exact log counts (use `>= 2`); request order on dual-protocol mirrors (assert the protocol where the request is received, and comment it); asserting cases the README doesn't define (failed scans).
- **Upstream flaky tests** (`daemon::TestRefreshNodeList`) fail Flakiness. Exclude them in the base-mode regex.
- **Upstream redigomock tests pin exact Redis commands** (`TestSetMirrorEnabled`). They constrain where reference fixes can go.
- **Environment blockers:** `ENV GOFLAGS` broke the repo Makefile; Go caches and `/app` must be writable for non-root solvers; harborSafe rejects `apt-get update`; rebuildSafe rejects ARG URLs.
- **localsolve** calls `test.sh <mode> --output_path X` and needs a writable /app. Otherwise the batch dies with 0 requests.
- **`olympus workflow local --full` stops on the docs lint of AGENTS.md headings.** Run the Docker four-state verify directly.
- **Harness defaults matter:** the zero-valued `core.Monitor` made a production-correct design fail unfairly.
- **/tmp tmpfs filled to 100% from harness snapshots.** Prune old directories.

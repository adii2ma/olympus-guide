# Calibration lessons: bokuweb/docx-rs

Task `docx-rs-read-footnotes`. The final version is fp `dd4e5024`, with 12 hidden tests and a 179-word README. Result: Orion 1/4 (`rd7dt9et`), Nova 0/8 with an average sequence of 943K, FP Check PASS at 2026-10-12 06:24:40Z, and Auto Review agents 3 / description 3 / solution 3 / tests 2.

## How we reached 1/4 Orion

Context: the task was first calibrated under the old 4+4+2 rules (2026-10-03/04, four cohorts, 32 "Nova" runs). The pass-rate rules changed on 2026-10-09. On live v2 (fp `2a29ed33`), Orion `rd7cxp97` and Vega `rd76khpq` both passed and Nova `rd70zxy9` failed, so the strong models found it too easy.

1. **cand2 → cand3 → cand4 (local only, 2026-10-10).** cand2 added `read_xml` parity and pictures in notes, and removed the hint "Same for files docx-rs wrote itself." The Claude Vega emulator went 1/8 and real gpt-6.1-sol (`~/olympus-localsolve`) went 2/2, so cand2 alone didn't slow Orion. cand3 added a test case for a file's link inside a content control in a note, and local gpt-6.1-sol went 1/4. Review then showed that one of those failures, and one more, came from test brittleness: the hidden-test walker had `_ => {}`, so it never saw a solver's new public `StructuredDataTagChild::Hyperlink` variant. cand4 fixed the walker, and the corrected local result was 2/4.
2. **DQ requirement count forced a trim (cand4 → cand5).** cand4 synced, then DQ failed 3 of 3 attempts. One attempt passed the AI check (0.444) but hit the new blocking `requirementCount` gate. The run stopped at the operator's 3-attempt DQ cap. cand5 cut the tests from 17 to 13 to 11 (dropping the ones no strong patch ever failed) and the README to 132 words (c5-32, AI 0.139), across 32 DQ reports.
3. **Vega 4 on fp `52849271` = 0/4**, plus local gpt-6.1-sol 0/4. Every failure was genuine, but all 8 strong patches kept the base bug where read hyperlinks are saved with `Target=""`. The trimmed README no longer named that case, so this was one cluster.
4. **Eased with a symptom sentence in the opening (c6-2, fp `22c2bd33`).** The added sentence: "when I save a .docx I've read its hyperlinks point nowhere", plus "ids clash". Local gpt-6.1-sol went 2/4 twice. **Platform Orion went 4/4** (`rd74h0wm`, `rd70fqj1`, `rd7c4vyy`, `rd785f83`), so platform Orion is easier than local.
5. **Hardened (c7 → c7d, fp `9c8aa07d`).** Two probe rounds (21 scenarios) found no split. Probe round 3 plus a static-review subagent found two discriminators:
   - the "moved" case: a note copied between read files keeps its own link when ids clash;
   - note text inside a block content control is lost (the solver only reads `w:p` at depth 1).

   verifyFairness failed twice: a table in an endnote is outside the repo's note model, and asserting flattened text imposes a "flattening policy". c7d fixed this by asserting only that the text survives in the saved part. The screen projected 1/4. Re-eval returned a Server Error twice, so fresh runs were used: **Orion b2 0/4, then b3 1/4** (`rd719t8n`).
6. **"Fair task" criterion failed.** The evaluator on `rd74fxmd` set `agentBlameUnfair` because the README didn't cover the moved case. c8-1 stated it in the README, but local went 0/4, all on the no-notes JSON rule. c8-2 clarified that rule ("down to what `docx.json()` prints") and local went 2/4. Fairness then flipped to FAIL on the identical content-control test. c9 fixed it by wrapping the second paragraph of the README's example footnote in `<w:sdt>`, with no new sentence. All 7 quality checks passed at fp `87945caa`. **Orion c9 went 4/4** (`rd71w00j`, `rd70w375`, `rd7a6w99`, `rd7fzh42`).
7. **Test-only hardening found by static review (c10, fp `781a9602`).** Rerunning every old probe and the paragraph-id lever gave no split. A static-review subagent (`h10/static-review.md`) found that 3 of the 4 passers hardcode the footnotes relationship as `rId8`, which collides with a read body picture at `rId8` (duplicate `Id`, an invalid package). c10 added that case to `read_notes_survive_writing` and an "id used once" check. The screen projected exactly 1/4. **Re-eval (5 tokens) gave Orion 1/4**: `rd7ebkfj` passed, and the other three failed only that test.
8. **Nova.** 8 Nova runs (batch-0dbb3ec7) were already active when the agent tried to launch 2 (`rollouts_still_active`), so they were started outside the agent. Result: **0/8, average sequence 943K**, failing on unescaped `&`, changed no-notes JSON, and `rId8`/`rId9` clashes.
9. **Auto Review solution 1/3 → c11 (fp `dd4e5024`).** Finding S1: the reference kept a read body hyperlink at `rId1`, which duplicates the styles relationship `rId1`. The FP panel had already noted that passer `rd7ebkfj` remaps such ids. c11 fixed the reference and added a behavioral regression test for ids `rId1`..`rId3`. The screen showed `rd7ebkfj` still at 12/12 and the other 11 runs still failing. A follow-up SQ-only tweak (c11b) made DQ fail 3 times on an identical README, so it was reverted to the exact c11a bytes.
10. **Final 12-run re-eval.** Agent-side `checks re-eval` returned a Server Error 17 times and never dispatched. The operator's web-UI trigger worked (completed about 06:01Z on 2026-10-12): Orion 1/4 (`rd7dt9et`, sequence 119,276) and Nova 0/8, exactly as `screen-c11.txt` predicted. FP PASS rated `rd7dt9et` a true positive with high confidence. The operator then submitted.

## What helped

- **Screening every downloaded patch against each new test.patch** (38 patches at cand4, 26 at c9, all 12 current-version patches at c11). It predicted c10 (1/4) and the final re-eval (Orion 1/4, Nova 0/8) exactly.
- **Re-eval instead of fresh runs** after test-only changes. The c10 re-eval cost 5 tokens, against 24 for a fresh Orion batch, and turned 4/4 into 1/4 on the same trajectories.
- **Static-review subagents comparing passer diffs with the reference.** Broad probes (21 scenarios plus a 400-case fuzz) never split the strong patches. Static review found both real discriminators: the depth-1 note reader and the hardcoded `rId8`.
- **Reading evaluator verdicts and criteria flags.** One `agentBlameUnfair` flag failed "Fair task" even at Orion 1/8. Stating that case in the README (rather than dropping it) kept the discriminator. A screen without the moved case would have given Orion 4/8.
- **Local gpt-6.1-sol as a directional gauge.** It caught the 0/4 clusters before any spend (cand5's `Target=""` cluster, c8-1's no-notes JSON cluster), and each 3-word clarification moved it to 2/4. It runs harsher than platform Orion (c6: local 4/8, platform 4/4), so a local 2/4 can mean a platform 4/4.
- **Understanding the DQ requirementCount gate.** A 74-word probe (c5-5) passed, which showed the gate is driven by the description. The XML example kept the AI score low. The gate is deterministic for a given package input, including the solution patch: c11b failed 3 times on an unchanged README, and reverting to c11a passed.
- **Scope Gate** passed every time (medium or high confidence, 0.94–1.29% overlap) at 2 tokens. It was never a bottleneck.
- **The Auto Review fix pattern.** The panel noted the passing run was stronger than the reference. Fixing the reference plus a behavior-only test (link resolves; no exact rId values) took solution from 1 to 3 without changing any verdict.

## What wasted time or tokens

- **17 agent-side re-eval attempts** on the 12-run multi-batch cohort (attempts 30–44, about 22:00Z to 05:51Z). Every one returned a Server Error and dispatched nothing. A `--use-general-tokens` variant failed too. Only the operator's web-UI trigger worked.
- **The 3-attempt DQ cap.** It stalled the task from 2026-10-10 19:56Z to 2026-10-11 05:40Z (attempts 15–23 were idle `wait_retry` runs).
- **Calibrating to the old rules** (2026-10-03/04). Four cohorts, 32 runs of codex gpt-5.6-luna "Nova", went stale when the codenames changed on 2026-10-09.
- **The Claude/Codex emulators.** 20 Claude Opus runs plus 4 codex gpt-5.6-sol runs (gpt-6.1-sol returned HTTP 400 via ChatGPT-account Codex). The emulator was harsher than platform Vega (2/4 vs 3/3), and platform Vega then went 0/4 (32 tokens) without leading to the Orion calibration.
- **Broad hardening probes** (rounds 1–2, tracked moves, `w:type="normal"`, fuzz) found no split. The fuzz harness went stale and failed the reference too.
- **Over-trimming for requirementCount.** It removed the README cues that the dominant strong-model miss depended on, giving an 8/8 identical cluster that had to be eased again.
- **Re-running checks that weren't stale** (c11b DQ, verifyTests, verifyFairness), which flipped DQ.
- **The brittle typed walker** (`_ => {}`), which cost a candidate round (cand3 → cand4).

## Tips for the next agent

- Gauge the old version first: download live passes and failures and read the evaluator verdicts before designing hardening.
- Screen every downloaded patch (platform and local) against each candidate test.patch before syncing; aim for a projected 1/4 on the current Orion batch.
- After a test-only change, re-eval the existing batch instead of starting fresh runs; it is cheaper and keeps the trajectories fixed.
- When probes find no split, run a static-review subagent over the passers' diffs versus the reference, then confirm each finding at runtime.
- Look for discriminators in pre-existing machinery interplay (fixed or hardcoded relationship ids, reader depth), not in new stated features; strong models implement every explicit new feature.
- Keep the README to about 5 headline asks plus an XML/code example; ground new cases by changing the example rather than adding sentences.
- On a 0/4 cluster, rank failures by sole-failure count and add a 3–10 word clarification or base-symptom sentence for the dominant miss.
- Treat local gpt-6.1-sol 2/4 as roughly platform 3–4/4; aim for local 1/4 before paying for Orion.
- Assert representation-free behavior (text survives in the saved part, the link resolves to its URL, the id is used once), never exact ids or internal structure.
- Make hidden-test walkers fall back to the element's public XML for unknown enum variants.
- If an evaluator flags `agentBlameUnfair`, state that case in the README or drop it; don't argue.
- Before Auto Review, probe whether any passer beats the reference on an edge case the FP panel mentions; fix the reference and add a behavior-only test.
- Refresh only stale checks, one at a time; save every check and run JSON before the next sync.

## Pitfalls

- Multi-batch re-eval from the agent CLI returns a Server Error and does not dispatch; ask the operator to trigger it from the web UI. A single-batch re-eval can return a Server Error with `sent=true` and still dispatch (c10), so check `runs view` after 1–2 minutes before retrying.
- Starting any fresh batch dismisses an eligible re-eval offer.
- A README change makes every run stale and requires fresh runs, plus DQ, prechecks, quality checks and Scope Gate.
- The DQ AI scoring regime shifted mid-session (2026-10-11 ~07:00Z: the same text went from 0.167 to 0.516). requirementCount only runs once AI < 0.5, and it reacts to solution-patch changes even when the README is unchanged.
- verifyFairness flips on identical tests across fingerprints (content-control note text: PASS at c7d, FAIL at f80b0396).
- verifyFairness rejects assertions that impose a structure policy the repo doesn't model (a table in a note, flattened control text).
- The plagiarism `llm_dedup` verdict changes between similar_idea, distinct and derivative on near-identical READMEs; it is advisory.
- The criteria panel needs "Fair task" and "noEnvBlocker" from every graded run, not just the pass rate.
- The rollout guard blocks with `rollouts_still_active` even when the dry run is clean.
- Re-eval runs are tagged `reeval-…`; wait loops filtering on the batch tag return immediately.
- `rustfmt mod.rs` reformats every child module (51 unrelated files); format only the touched files.
- Full `json()` snapshots of `read_xml` pin base bugs that solvers legitimately fix (25 of 26 patches failed); snapshot only the stable subtree.

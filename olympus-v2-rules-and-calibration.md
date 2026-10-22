# Olympus V2 Rules and Calibration (October 2026 update)

Read this before the older guides. The Olympus updates of 2026-10-09 and 2026-10-21 replaced the pass-rate rules; the current rules are in section 1. Wherever another file in this repo (or an older handbook) says "1–4 passes out of 10", "40% ceiling", "4+4+2 cohort", "Terra", or "10-run archive", that rule is **retired**. Follow this file instead.

The lessons below come from the first two tasks that went all the way through V2 and got submitted: a Rust OOXML reader/writer task (docx-rs) and a Go mirror-health-check task (mirrorbits). Both ended at **Nova 0/8, Orion 1/4, FP Check pass, Auto Review pass**. Their full case studies are in [`lessons/`](lessons/).

## 1. The rules (platform update 2026-10-21)

| Requirement | Rule |
|---|---|
| **Rollout** | **One rollout of 12 runs per version: 4 Nova + 4 Orion + 4 Vega**, launched together (`olympus runs run <id> --preset full`; the server's full preset, evaluated by Orion). At most 4 runs per model per version. Nova is `gemini_cli`, stored as `antigravity` (gemini-3.8-flash); Orion is `codex_cli` with gpt-6.1-sol; Vega is `claude_code` with claude-opus-5-5. |
| **Solved by one model only** | **Exactly 1 of the 12 runs passes**, and it is an Orion or a Vega run: Orion 1/4 with Vega 0/4, or Vega 1/4 with Orion 0/4. Nova 0/4. Platform criterion `passStructure`: "Exactly one of Nova, Orion or Vega solves it, more than 0% and at most 25% of the time; the other two stay at 0%." |
| **Long horizon (Nova)** | The Nova runs must average **> 400K** `sequenceLength` (was 256K until 2026-10-21). kira's first Nova run on the new rules used 514K. |
| **No Orion/Vega sequence rule** | The 2026-10-20 "Long horizon (Orion / Vega)" criterion (passing runs > 256K) was **retired on 2026-10-21**. Never lengthen a task for Orion/Vega. |
| **Panel** | Also required: **Fair task**, **Solvable**, **Rollouts on every model** (`codenameCoverage`), **No cheating** (< 20%), **No environment blockers**, **FP Check** pass and **Auto Review** pass. |
| **Stale runs** | Runs from before an update are stale for calibration. |

**Local gate on every iteration (`~/olympus-localsolve/localgate.py`, v3).** It runs, in order:
1. A local **Test Quality judge**: two independent Claude Opus 5.5 reviewers label every hidden test Prompt-stated, Repo-discoverable, Standard external semantics or Not fair, as the platform's verifyFairness does. Any Not fair label stops the gate.
2. A local **Solution Quality judge**: completeness and code quality must both be Fully Met.
3. Only then, **4 local Vega (Claude Code + claude-opus-5-5) + 4 local Orion (Codex + gpt-6.1-sol)** in one round, README only, graded like the platform. **PASS = all 8 graded and exactly 1 of the 8 passes**, mirroring the platform's one-pass-in-12 rule.

The CLI refuses a platform rollout unless the gate passed on exactly the synced README, test.patch and solution.patch. A local run whose agent never got a model response (unsupported model, auth, quota) is a `HARNESS_ERROR`, never a fail. Learned the hard way: from 2026-10-20 the VM's Codex login silently rejected gpt-6.1-sol, and a whole day of local Orion "0/4" results were fake. Each judge costs about $1, so fix judge findings before you spend on the 8 solvers. Claude containers are capped at 6 VM-wide, to protect the operator's usage limit.

**Order on every new iteration:** local gate → sync → checks in the operator order (previously failed check first, other quality checks, prechecks, Scope Gate last) → **Auto Review first** (`olympus checks auto-review <id> --pre-rollout`; every dimension at least 2/3, aim for 3/3) → the full 12-run rollout → FP Check → final Auto Review to 3/3.
- If the rollout's pass structure fails (a Nova pass, more than one pass, or none), the version gets no more runs. Fix the task, gate it, and sync a new version.

**Re-eval:**
- When only `test.patch` and/or `solution.patch` change, re-evaluate the existing runs instead of starting fresh ones. It costs about 30% of a fresh batch and keeps the trajectories fixed, so the result is predictable.
- Starting **any** fresh batch dismisses the re-eval offer.
- A README or Dockerfile change makes every run stale and forces fresh runs plus DQ, prechecks, quality checks and Scope Gate.

## 1b. New-task workflow (operator, 2026-10-21)

  A. UNIQUE FIRST. While authoring the cheap candidate, prove it is in scope and original BEFORE you invest in it: Scope Gate, the required prechecks, and the plagiarism/public-overlap/duplicate-task review (upstream PRs, issues, forks, existing Olympus tasks). Prefer candidates whose solving spans several interacting subsystems of a large repository, so Nova runs long (> 400K).
  B. BUILD IT HARD, LOCALLY. Expand that exact candidate into the full task: hidden interactions with existing behaviour, end-to-end flows, real investigation. Keep the README under 400 words and DQ-clean. Complete the four-state Docker matrix, production checks and run-patch screens locally.
  C. CALIBRATE LOCALLY WITH THE GATE, ON EVERY ITERATION, until exactly 1 of the 8 local runs passes. The environment is shared by all agents on the VM: use task-prefixed batch names, never touch another agent's `olx-*` containers or `runs/`, and never edit `~/olympus-localsolve/`.
  D. ONLY THEN PLATFORM: sync, checks, **Auto Review first**, the full rollout, FP Check, final Auto Review to 3/3. Carry every open Auto Review finding into the next version: build its `wrongImpl` as a mutant and prove the tests kill it. Then READY_FOR_HUMAN_SUBMISSION. Never lock or submit.

### Lessons from the 2026-10-20 updates

16. (alp, boss, 2026-10-20) Both tasks reached Nova 0/8 and Orion 1/4 and then failed ONLY the Orion/Vega long-horizon criterion (passing Orion runs at 76K and 85K, against > 256K). That criterion was retired on 2026-10-21. Don't lengthen tasks for Orion/Vega; only Nova must average > 400K.
17. (alp) Hidden tests must build against ANY reasonable solver tree. 6 of 9 local Vega runs dropped `gopacket` from go.mod (they wrote their own pcap reader), and the new test helper imported gopacket, so `[setup failed]` failed them before a single assertion ran. On the platform that is an unfair failure. Keep test helpers self-sufficient: no dependency the solution may legitimately remove. The gate now fails on such builds.
18. (alp) The local Test Quality judge flagged, on its first run, an ambiguity Auto Review later also hit: "the request's first captured byte" can mean the positionally first byte or the earliest-captured one. Treat judge findings as future Auto Review findings.
19. (alp, boss) Auto Review findings don't disappear when you redesign. alp's lost-segment test accepted a nil output (so a parser that stops at the first TCP gap passed). boss never tested an `.lpk` whose internal Name differs from its filename, and its reference deleted hand-written entries marked `Prefer="True"`. Fix each finding in the new version.

### Lessons from kira, the first task to reach 1 of 12 (2026-10-22)

Full case study: [`lessons/v2-kira-delay-time-changes.md`](lessons/v2-kira-delay-time-changes.md).

20. (kira, 1 of 12 on 2026-10-22: Nova 0/4 at 533K, Orion 0/4, Vega 1/4; FP pass; Auto Review 3/3/3/3) Diagnose the passers' SHARED shortcut and add ONE fair wall from the README or the repo's own conventions. Both Orion passers updated a linked parameter once per frame, but kira supplies linked values once per chunk, so the delay stepped instead of gliding. One glide test made every per-frame design fail deterministically (Orion 0/4, no near-passers).
21. (kira) Know each model's wall and keep a "sole failing test" ranking per run. Orion missed the per-frame glide; Nova missed Duration::MAX-before-clamp and allocation; Vega missed cap rounding, fractional reads and allocation. A run that fails only one test is a near-passer, and a re-judge can flip it. Screen EVERY known patch before every sync: kira's projections were exact for every re-eval.
22. (kira) Local Vega is noisy and WEAKER than platform Vega. Local gates on near-identical artifacts went 0, 0, 0, 3, 0, 0, 1 of 4 (~20% pooled), yet platform Vega went 3/4. Pool the samples across gates and don't ease or harden on one sample. Expect platform Vega above local, so aim for the low end locally.
23. (kira) Platform Vega 3/4 → 1/4 without unfairness: audit each passer at RUNTIME like the FP panel. Use a differential scenario renderer against the reference (many scenarios × chunk sizes, sample by sample). Keep the passer that matches the reference exactly. Add a test only for a genuine defect that is prompt-stated or standard semantics: "the cap only limits LONGER delays" (passers clamped too early), and "a steady fractional delay maps a ramp to a ramp" (time invariance). Prove each test fair with the reference, a deliberately different correct variant and both judges.
24. (kira) The judges are early Auto Review. Drop a test the fairness judge rules Not fair. When the solution-quality judge flags the reference, fix the REFERENCE only, without a new test. Kira's final Auto Review then found nothing new.
25. (kira) Test- or solution-only changes sync into the SAME version and keep the cohort. Re-eval re-grades the existing trajectories, and a fresh batch is needed only for a model never run on that version. Agent-side `checks re-eval` failed 14 times in a row (Server Error, refunded); only the operator's web-UI click worked. After 2 failures, write ONE long wait_retry asking for the click. Don't retry every 30 minutes: each attempt burns Claude usage.

## 2. How the two tasks actually got to 1/4 Orion

Neither task got there by adding stated requirements. Strong models implement every sentence in the README. Both found **one hidden discriminator in pre-existing machinery that the natural feature design breaks**, and then tuned it with a single light README sentence.

**docx-rs:**
- Orion stayed at 4/4 through several README hardenings.
- A static-review subagent comparing the passers' diffs with the reference found that 3 of the 4 passers hardcoded the footnotes relationship id as `rId8`. That id collides with a read body picture at `rId8`, which produces an invalid package.
- One test case ("the id is used once, and the link still resolves") plus a **re-eval** turned Orion 4/4 into **1/4**.

**mirrorbits:**
- Orion went 4/4 → 4/4 → 2/4 → 3/4 while the agent added explicit rules.
- Mining the passers' diffs showed that every one of them deleted the source FILES set on an empty refresh, which breaks the existing `mirrorbits scan` command ("not yet indexed").
- A base-mode regression test for that plus one README pointer sentence ("A refresh that finds the local repository empty still counts as a refresh.") gave **1/4**.

## 3. Tips for the next agent

### Gauge first
1. **Gauge before you design.** On a basic-viability version, run 2 Nova and 4 Orion (or re-eval the existing ones), download every patch, and read the evaluator verdicts and test logs. Platform Orion is the difficulty arbiter, not a local emulator.
2. **Treat local gpt-6.1-sol (`~/olympus-localsolve/localsolve.py --n 4`) as a directional gauge only.** One local batch at a time on the VM; check `docker ps --filter name=olx-` first.
   - It catches 0/4 clusters and harness bugs before any spend.
   - Its offset from the platform varies by task. On docx, local 2/4 meant platform Orion 4/4. On mirrorbits, local 1/8 meant platform 4/4. Aim for local ≤ 1/4 before paying for Orion, and expect platform Orion to be easier.

### Harden
3. **Harden through hidden interactions, not new sentences.** Look for:
   - existing behaviour the obvious design breaks: grep every other user of each key, flag, id or file the feature touches;
   - fixed or hardcoded ids, reader depth or walker coverage;
   - cross-feature combinations the README already implies.
4. **When broad probes find no split, run a static-review subagent** over the passers' diffs versus the reference, then confirm each candidate discriminator at runtime. On docx, 21 probe scenarios and a fuzz harness found nothing; static review found both real discriminators.
5. **Tune with one light README pointer** (a symptom sentence, a few words, or a change to the README's example) rather than a new requirement. On a 0/4 cluster, rank failures by how often each one is a run's only failure, then add a 3–10 word clarification for the dominant miss.

### Screen before you spend
6. **Screen every known run patch against every candidate `test.patch` before syncing.** Include every platform run from every version, every local patch, the reference (×3) and bare base. Run them on 2 CPUs, uid 4242, `--network none`, like grading. On both tasks the screen predicted the platform re-eval verdicts exactly. Ship only when the projection says Orion 1/4 and Nova 0/N.
7. **Make near-passers fail deterministically before any re-eval.** A run with all new tests passing but a FAIL verdict can flip to PASS when it is re-judged. Give it a test it genuinely fails, or don't re-eval.

7a. **Check order after every sync (operator rule 2026-10-16).** If a check has failed even once on this task, run THAT check first, on its own (`checks run <key>`), after every sync. Only when it passes, run the other quality checks, then the combined prechecks, and Scope Gate last. Never open a sync with `run-all`, `run-quality` or Scope Gate. If the previously failed check fails again, fix it, re-screen and re-sync before you spend tokens on anything else.

### Write fair tests
8. **Assert representation-free behaviour:** the link resolves to its URL, the id is used once, the text survives in the saved part, no request arrives. Never assert exact ids, internal layout, or a structure policy the repo doesn't model. Make hidden-test walkers fall back to the public XML/text for unknown enum variants; a `_ => {}` arm silently failed correct solvers.
9. **If an evaluator flags a case as unfair (`agentBlameUnfair`), state that case in the README or drop it.** Don't argue. A single flag fails "Fair task" even when the pass rate is right.

### Get Auto Review to pass first time
10. **Fix Auto Review findings before the final cohort, not after.** Treat every solutionQuality "Partially Met" and every Fairness coverage suggestion as a future Auto Review finding.
    - On mirrorbits, an ignored SQ note became Solution 1/3 after submission.
    - On docx, the FP panel noted that the passing run was more robust than the reference. That became Auto Review Solution 1/3. Fixing the reference plus adding a behaviour-only regression test took it to 3/3 with no verdict change.
11. **For every Auto Review tests finding, build the reviewer's `wrongImpl` as a mutant and prove the new test kills it.** On mirrorbits, the reviewer's literal test order didn't kill the mutant because the monitor's startup refresh masked it.

### Description Quality (DQ)
12. Aim for about **5 headline asks**. About 8 fails the blocking `requirementCount` gate.
    - Use one behaviour paragraph plus a small code/XML/YAML example. Pure prose scored AI 1.0, and an example kept the AI score low.
    - When you cut a requirement, cut its tests too. Don't over-trim the cue that the dominant strong-model miss depends on: docx went to an 8/8 identical failure cluster and had to be eased again.

### Keep going
13. **Keep going between steps.** Iterate on checks as often as needed and refresh only the checks that are stale. FP Check, Auto Review, lock and submit belong to the operator unless told otherwise.

### Lessons from later tasks (sqls, mirrorbits revert, rustfft)
14. **Check the Orion pass for a false positive before accepting 1/4.** On a SQL language-server task, the only Orion pass silently broke behaviour that already worked (go-to-definition in plain queries the README promised to keep), and FP Check failed it.
    - Before running Orion, add preservation tests for every existing behaviour the feature touches.
    - Before accepting 1/4, probe the passing run locally for regressions the way the FP panel would.
15. **Don't pin an order, timing or message wording the README doesn't state.** The reviewer reverted the submitted mirrorbits task over 2 unfair tests:
    - one rejected a valid HTTPS-first check order;
    - the other rejected an accurate error message that contained the requested path.

    Accept every valid order and assert only outcomes. Before every sync, run a deliberately different but correct variant of the reference against the tests.
16. **Local gpt-6.1-sol predicts Orion, not Nova.** On rustfft, local runs went 0–1/4, yet the first 2 platform Nova both passed. Run the first 2 Nova early, as soon as checks pass, before investing in Orion calibration.

## 4. Pitfalls

**Server Errors:**
- **Agent-side `checks re-eval` on a multi-batch cohort** (for example 4 Orion + 8 Nova) returned a Server Error **17 times on both tasks** and never dispatched. The operator's web-UI **Re-evaluate** button worked first time. After 2 failures, checkpoint and ask the operator to click it.
- **A single-batch re-eval** sometimes returns a Server Error with `sent: true` and does dispatch. Check `runs view` after 1–2 minutes before retrying.
- **`sync`** can return a Server Error too; compare the live patches with your local ones before retrying.
- **`final-qa`** returns a Server Error, and FP Check doesn't need it.

**Checks that flip on identical input:**
- verifyFairness passed and then failed the same test on another fingerprint.
- solutionQuality rated the same solution Fully Met, then Partially Met.
- DQ reacts to `solution.patch` changes even when the README is unchanged; reverting the bytes restored the pass.
- The DQ AI scoring shifted mid-session: the same text went from 0.167 to 0.516.

**CLI and workflow quirks:**
- The rollout guard reports `rollouts_still_active` when someone else's runs are in flight, even after a clean dry run. List the runs and adopt them rather than duplicate them.
- Re-eval runs are tagged `reeval-…`, so wait loops that filter on the original batch tag return immediately.
- `olympus workflow local --full` can stop on the AGENTS.md docs lint. Run the Docker four-state verify directly.

**Environment and harness:**
- Environment blockers: `ENV GOFLAGS` colliding with a repo Makefile; Go caches and `/app` not writable by non-root solvers; harborSafe rejects `apt-get update`; rebuildSafe rejects ARG URLs. The localsolve harness also needs a writable `/app` and calls `test.sh <mode> --output_path X`.
- Upstream flaky tests fail Flakiness; exclude them from the base-mode regex. Upstream mocks that pin exact commands constrain where reference fixes can go.
- Full JSON snapshots pin base bugs that solvers legitimately fix (25 of 26 patches failed one). Snapshot only the stable subtree.
- `rustfmt mod.rs` reformats every child module; format only the touched files.
- `/tmp` tmpfs can fill from harness snapshots; prune old screen directories.

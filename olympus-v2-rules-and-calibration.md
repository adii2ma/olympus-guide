# Olympus V2 Rules and Calibration (October 2026 update)

Read this before the older guides. The Olympus update announced around 2026-10-09 replaced the pass-rate rules. Wherever another file in this repo (or an older handbook) says "1–4 passes out of 10", "40% ceiling", "4+4+2 cohort", "Terra", or "10-run archive", that rule is **retired**. Follow this file instead.

The lessons below come from the first two tasks that went all the way through V2 and got submitted: a Rust OOXML reader/writer task (docx-rs) and a Go mirror-health-check task (mirrorbits). Both ended at **Nova 0/8, Orion 1/4, FP Check pass, Auto Review pass**. Their full case studies are in [`lessons/`](lessons/).

## 1. The rules

| Requirement | Rule |
|---|---|
| **Nova** | At least **8** runs, **0%** passing. Request it as `gemini_cli`; the platform stores it as `antigravity` (gemini-3.8-flash). |
| **Nova cap** | At most **8 Nova runs per version**, since every extra run is another chance to break 0%. |
| **Orion or Vega** | At least **4** runs in **one** strong pool, passing **more than 0% and at most 25%**, so exactly **1/4**. Orion is `codex_cli` with gpt-6.1-sol (6 tokens); Vega is `claude_code` with claude-opus-5-5 (8 tokens). Either pool satisfies the rule, and a 0/4 in the other pool doesn't block. |
| **Long horizon (Nova)** | The average Nova `sequenceLength` must be **> 256K**. Both tasks landed far above it (538K and 943K), so a genuinely multi-subsystem task clears it on its own. |
| **Long horizon (Orion / Vega)** | *Added 2026-10-20.* The **passing** runs of the qualifying strong pool must also average **> 256K** (criterion `strongSequenceLength`). The first two tasks hit by it failed only this: alp's passing Orion used 76K and boss's 85K, while their Nova averaged ~300K. A task the strong model solves in a short, focused patch no longer qualifies. It must need many files, deep investigation and many steps, and you get there through real repository complexity, not a longer README. |
| **Panel** | Also required: **Fair task** (no run flagged unfair, including any `agentBlameUnfair`), **no environment blockers**, cheat rate < 20%, **FP Check** pass and **Auto Review** pass. |
| **Evaluators** | Nova or Orion. |
| **Stale runs** | Runs from before the update (where "Nova" was codex gpt-5.6-luna) are stale for calibration. Re-gauge every task on the new pools. |

**Local gate before every sync (operator rule 2026-10-20).** On every candidate iteration, run three checks on the VM, in this order:
1. A local **Test Quality judge**: two independent Claude Opus 5.5 reviewers label every hidden test Prompt-stated, Repo-discoverable, Standard external semantics or Not fair, as the platform's verifyFairness does. Any Not fair label stops the gate.
2. A local **Solution Quality judge**: completeness and code quality must both be Fully Met.
3. Only then, **two rounds of 4 local Vega (Claude Code + claude-opus-5-5) + 4 local Orion (Codex + gpt-6.1-sol)**, README only, graded like the platform. A pool that passes 2+ of its first 4 is too easy and gets no round 2. At least one pool must reach **1–2 of 8**, with its passing runs averaging **> 256K** sequence length. (The first version ran 10 parallel Opus runs per gate; with several agents gating at once that exhausted the operator's Claude 5-hour limit within hours, so it was staged and capped at 6 Claude containers VM-wide.)

The CLI refuses fresh runs unless the gate passed on exactly the synced README, test.patch and solution.patch. On its first try, the judge flagged a real ambiguity in alp: "first captured byte" can mean the positionally first byte or the earliest-captured one. Each judge costs about $1, so fix judge findings before you spend on the 10 solvers.

**Rollout order that worked:** **2 Nova → 4 Orion → 6 Nova** (8 Nova total). Since 2026-10-20 the Orion pass must also be > 256K before the remaining Nova.
- Two Nova runs first catch a Nova pass for 2 runs' worth of tokens instead of 8.
- On **any** Nova pass, harden before running more Nova.

**Re-eval:**
- When only `test.patch` and/or `solution.patch` change, re-evaluate the existing runs instead of starting fresh ones. It costs about 30% of a fresh batch and keeps the trajectories fixed, so the result is predictable.
- Starting **any** fresh batch dismisses the re-eval offer.
- A README or Dockerfile change makes every run stale and forces fresh runs plus DQ, prechecks, quality checks and Scope Gate.

## 1b. New-task workflow (operator, 2026-10-20)

Follow it end to end:
  A. UNIQUE FIRST. While authoring the cheap candidate (PHASE 1–2), prove it is in scope and original BEFORE you invest in it. Sync the basic viability package, then make Scope Gate and the required prechecks pass, and do the plagiarism/public-overlap/duplicate-task review (upstream PRs, issues, forks, existing Olympus tasks). Prefer candidates that can become LONG-HORIZON: behaviour that spans several interacting subsystems of a large repository. Pick nothing a strong model can finish in one focused patch.
  B. BUILD IT HARD AND LONG, LOCALLY. Only once the candidate is verified unique, expand that exact candidate into the full task (PHASE 3–4), designed from the start to be difficult and long-horizon: hidden interactions with existing behaviour, end-to-end flows across files and packages, real investigation. Keep the README under 400 words and DQ-clean. Complete everything locally first: the four-state Docker matrix, production checks and run-patch screens.
  C. CALIBRATE LOCALLY WITH THE GATE, ON EVERY ITERATION. Run `~/olympus-localsolve/localgate.py --package <pkg> --batch <task>-<iteration>`. It runs the 2 Test Quality judges, then the Solution Quality judge, then 2 rounds of 4 local Vega + 4 local Orion, stopping a pool early once it passes 2+ of 4.
     - Iterate (fix judge findings, harden, ease, lengthen) until the gate PASSES: judges clean, then a Vega or Orion pool at 1–2 of 8 passes with the passing runs above 256K.
     - The optional quick gauge (`localsolve.py --n 4`, gpt-6.1-sol, platform Orion's model) helps between gates.
     - The environment is SHARED by all agents on this VM:
       - Claude containers take VM-wide slots, so batches from different agents run side by side and wait automatically when the VM is full.
       - Always use a fresh, task-prefixed batch name.
       - Never stop, remove or touch another agent's `olx-*` containers or `runs/` directories.
       - Never edit `~/olympus-localsolve/`.
  D. ONLY THEN PLATFORM. Sync exactly the gated README, test.patch and solution.patch. Run checks in the operator order (previously failed check first, then the other quality checks, then prechecks, Scope Gate last). Then the staged rollout with `--local-batch <gate batch>`: 2 Nova → 4 Orion (1 pass above 256K) → 6 Nova. Then FP Check, then Auto Review, iterated to 3/3 on every dimension. Carry every open Auto Review finding into the next version: build its `wrongImpl` as a mutant and prove the tests kill it. Then READY_FOR_HUMAN_SUBMISSION. Never lock or submit.

### Lessons from the long-horizon update

16. (alp, boss: Orion/Vega long-horizon update, 2026-10-20) Both tasks reached Nova 0/8 and Orion 1/4 and then failed ONLY "Long horizon (Orion / Vega)": the passing Orion run used 76K and 85K, against the 256K the passes must average. Local Opus 5.5 is fast too: its runs took 10–26 min at 120K–235K, so a 1,300-line feature can still be short-horizon. Design for depth (investigation, cross-subsystem interactions, debugging), not just size.
17. (alp) Hidden tests must build against ANY reasonable solver tree. 6 of 9 local Vega runs dropped `gopacket` from go.mod (they wrote their own pcap reader), and the new test helper imported gopacket, so `[setup failed]` failed them before a single assertion ran. On the platform that is an unfair failure. Keep test helpers self-sufficient: no dependency the solution may legitimately remove. The gate now fails on such builds.
18. (alp) The local Test Quality judge flagged, on its first run, an ambiguity Auto Review later also hit: "the request's first captured byte" can mean the positionally first byte or the earliest-captured one. Treat judge findings as future Auto Review findings.
19. (alp, boss) Auto Review findings don't disappear when you redesign. alp's lost-segment test accepted a nil output (so a parser that stops at the first TCP gap passed). boss never tested an `.lpk` whose internal Name differs from its filename, and its reference deleted hand-written entries marked `Prefer="True"`. Fix each finding in the new version.

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

# kira: "Let a delay's delay time change after it's created" (1 of 12 under the 2026-10-21 rules)

**Task:** tesselode/kira (Rust audio engine), challenge `kh70rf3yb83znbd85s5t2ahkn18g9n0v`, version `kd7290re`. It was the first task to reach the 2026-10-21 rules.

**Final cohort:**

| Pool | Result |
|---|---|
| Nova | 0/4, average sequence 533K |
| Orion | 0/4 |
| Vega | 1/4 (#4) |

- **Criteria panel:** all green.
- **FP Check:** passed with caveats. The only nit was a linked delay gliding from kira's 500 ms default in its first chunk.
- **Auto Review:** agents 3, description 3, tests 3, solution 3. The pre-rollout Auto Review on v5 had already given description 3, tests 3, solution 3.

## Timeline

1. **Version 0, full rollout:** Orion 2/4, Nova 0/4 (533K), Vega never dispatched. `passStructure` failed as "too easy".
2. **Why Orion passed:** both Orion passers (and 14 of 16 local gpt-6.1-sol runs) updated the delay-time Parameter **once per frame**. kira supplies a linked value (modulator, listener distance) once per **chunk**, so their delay *stepped* every 128 frames instead of gliding. kira already interpolates linked params within a chunk in every other effect.
   - Fix: one fair hidden test, `linked_delay_time_glides_between_chunks`. It renders a 5 Hz tone with a modulator sweeping 100→300 ms, and the same with a spatial track moving away, then asserts the largest frame-to-frame step stays below 6% of peak. The reference scores 0.031; the Orion passers scored 0.22 and 0.67.
   - Every per-frame design now failed deterministically: **Orion 0/4 after re-eval**, with no near-passers.
3. **Local gates, g1 to g9 (each is 2 fairness judges + 1 solution-quality judge + 4 Vega + 4 Orion):**
   - The **fairness judge** ruled one new test Not fair: "a linked delay starts at its first value", whereas kira's other linked params glide from their default. The agent dropped the test and reverted the matching reference change.
   - The **solution-quality judge** later rated the reference Partially Met on that same first-chunk behaviour. The agent fixed the **reference only**, without a test, plus a changelog entry. That gave Fully Met/Fully Met.
   - **Local Vega is noisy:** 0, 0, 0, 3, 0, 0 and 1 of 4 on near-identical artifacts, about 20% pooled. The agent computed P(exactly 1 of 4) ≈ 0.4 for any true rate between 12% and 35%, so it **re-sampled the gate instead of easing or hardening on one sample**.
   - It eased exactly one thing. The main Claude miss, a tween queued before the very first processed chunk, was an edge case, so the test now renders one silent chunk first. The main per-frame glide rule stayed tested.
   - It added one more wall, which only Nova-1 needed: an instant mix change after a delay-time change must still fade over one chunk, which is existing behaviour.
4. **v5:** synced as a test- and solution-only change into the same version. Then:
   - checks in the operator order, then prechecks, then Scope Gate;
   - the **pre-rollout Auto Review (3/3/3)**;
   - operator re-eval of the 8 v0 runs: Orion 0/4 and Nova 0/4, exactly as the 61-patch screen predicted;
   - a fresh 4-Vega batch. **Vega 3/4.** Platform Vega is clearly stronger than local Opus (local ~20%, platform 75%).
5. **Vega 3/4 → 1/4 without unfairness.** The agent audited the 3 passers like the FP panel would:
   - It built a differential scenario renderer (45 renders: tweens with easing, feedback and filters in the loop, the cap crossed up and down, modulator-driven, sample-rate changes, chunk sizes 1/7/37/64/128/300/512) and compared each passer with the reference sample by sample.
   - **Vega #4** matched the reference exactly (max diff 0.000) and was kept as the one passer.
   - **Vega #3 and #2** clamped the chunk's start and end delays before interpolating. While a delay time was still **below** the cap, their output already differed from an uncapped delay (diff 0.07 on a 0.05 echo). New test: `the_maximum_only_limits_longer_delay_times`. It is prompt-stated ("longer delay times get clamped") and follows a repo convention (mix is clamped after interpolating).
   - **Vega #2** also broke time invariance: a steady 10.5-frame delay put a bump on every 10th frame, because its cubic read hit an unwritten frame. New test: `steady_fractional_delays_keep_a_ramp_straight`. That is standard DSP: an LTI fractional delay maps a ramp to a ramp.
   - Before syncing: a ×4 screen of all known patches, both judges clean, and a deliberately different but correct variant passing.
   - Then the operator re-evaluated all 12 runs: **exactly 1 of 12.**

## What made it work

1. **Diagnose the passers' shared shortcut, then add ONE fair wall that follows from the README or the repo's conventions.** One test removed all of Orion.
2. **Know each model's wall.**
   - Orion: per-frame glide.
   - Nova: `Duration::MAX` before clamping, and allocation in a linked path.
   - Vega: cap rounding, fractional reads and allocation.

   Keep a "sole failing test" ranking per run. A run that fails only one test is a near-passer, and a re-judge can flip it.
3. **Screen every known patch (61 → 77) before every sync.** On kira the projections were exact for every re-eval.
4. **Treat local Vega as a lower bound.** Pool the samples across gates; don't react to one 0/4 or 3/4. Expect platform Vega to pass more than local.
5. **Judges are early Auto Review.** Drop a test the fairness judge rules unfair. Fix the reference when the solution-quality judge flags it, without adding a test. Kira's final Auto Review found nothing new.
6. **Reduce a too-easy strong pool by auditing passers at runtime** with differential renders against the reference, not by guessing at new requirements. Keep the passer that is sample-identical to the reference.
7. **Agent-side `checks re-eval` failed 14 times in a row** (Server Error, refunded). Only the operator's web-UI click worked. After 2 failures, write ONE long wait_retry for the operator; don't retry every 30 minutes, because each attempt costs Claude usage.
8. A large multi-subsystem engine (kira's mixer, clock, modulators, spatial tracks, effects) kept Nova long without any README padding: 533K average with a README under 400 words.

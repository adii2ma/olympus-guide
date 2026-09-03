# How to Prevent False Positives and Raise Test Quality

This internal guide builds on the false-positive and mutation-review sections of [agent-2.md](agent-2.md). It explains why the transactional writable CSV suite repeatedly received a 1/3 Tests score despite broad coverage, and how to avoid that cycle in a new task.

## 1. Understand the failure mode

A false positive is a solution that passes the verifier while violating the problem description or existing behavior. It is more damaging than an ordinary failed agent because it proves the test suite certifies incomplete code.

A large suite can still score 1/3. Test quality depends on whether realistic wrong implementations survive, not on test count, patch size, or assertion count.

The CSV suite covered most visible behavior, but passing-agent review found gaps one at a time. Each gap admitted a plausible implementation that looked complete:

- compatibility keyed on SQL affinity instead of only CSV layout;
- eager parsing that regressed the old streaming read path;
- per-file publication that could break multi-file atomicity;
- quote-unaware CRLF detection;
- alias coordination by pathname instead of physical identity;
- missing writable schema inference;
- no successful multi-file transaction test;
- no proof that successful commits replaced the file object;
- conflict detection against an old file descriptor instead of the current pathname;
- rowid allocation based on current live rows rather than transaction high-water state.

The lesson is to design the false-positive audit before agent runs, not after the platform finds a passing wrong patch.

## 2. Create a requirement-to-test ledger

Break the description into atomic promises. For each promise, record:

| Requirement | Fairness source | Positive test | Negative or failure test | Wrong implementation killed |
| --- | --- | --- | --- | --- |
| writable schema inference | existing public registration path | create without `schema`, mutate, reopen | malformed inferred input | writable only with explicit schema |
| physical writer ownership | prompt | same owner through alias | competing hard-link and symlink writers | canonical-path lock |
| no-op leaves file untouched | prompt | `SET x=x` | affinity-equal expression on `01` and `1.0` | raw-string comparison after formatting |
| current-path conflict | prompt | normal commit | rename replacement over pathname | reread retained descriptor |
| high-water rowid | prompt | sequential inserts | delete highest allocation, insert again | `max(live)+1` |
| multi-file atomicity | prompt | successful two-file commit | conflict and late rollback | publish only first file |

Every requirement needs at least one test that would fail a believable shortcut. A positive happy path alone is rarely enough for stateful work.

## 3. Review passing patches as adversarial evidence

After all runs, inspect every passing agent's actual production diff, not just the test summary. For each pass:

1. Map its design to every prompt requirement.
2. Trace existing behavior outside the new test paths.
3. Inspect error and rollback branches.
4. Inspect cleanup after partial progress.
5. Inspect identity, ownership, and synchronization transitions.
6. Run production checks.
7. Write small public-behavior probes for suspicious shortcuts.
8. Compare with the reference only after judging the prompt independently.

For the CSV task, this process found three early false positives:

- One agent treated affinities as part of layout compatibility, so a TEXT view could not share a stage with an INTEGER view.
- One agent eagerly loaded all rows even in default read-only mode and published files in `Sync` independently, breaking lazy `LIMIT 1` behavior and multi-file rollback.
- One agent detected CRLF using the first raw newline and rejected same-owner hard-link aliases because it compared path strings after finding the inode.

Those defects were prompt-grounded and reproduced against public APIs. The tests were then strengthened.

## 4. Use mutation-oriented review before runs

Create deliberate wrong variants or reason through them one at a time. The verifier should reject each mutation for the intended assertion.

High-value CSV mutations included:

- add affinities to the compatible-view key;
- use only canonical path as owner key;
- treat every same pathname as the same owner after inode replacement;
- update published identity without the owner mutex;
- parse every read-only scan eagerly;
- calculate inserted rowid from current live maximum;
- fail `RollbackTo` when the savepoint predates first table write;
- compare conflicts using size and mtime;
- compare conflicts through the initially opened descriptor;
- truncate and rewrite the destination in place;
- preserve a symlink by writing its target in place;
- always replace a touched file even if semantics are unchanged;
- compare semantic no-op using serialized raw strings;
- serialize a one-field empty record as a blank line;
- normalize CRLF inside unchanged quoted values;
- restore original bytes unconditionally after a late veto;
- publish prepared files sequentially without transaction-wide rollback.

Do not wait for an agent to invent every mutation. If a mutation survives locally, strengthen a fair public test before calibration.

## 5. Test state transitions, not only final values

Many false positives produce the correct final bytes while violating identity, isolation, or recovery.

For filesystem transactions, inspect:

- bytes before commit;
- bytes visible to the owning connection;
- bytes visible to another connection;
- bytes visible to a direct reader;
- file identity before and after a real change;
- file identity before and after a semantic no-op;
- state inside a public commit hook;
- state after commit veto or rollback;
- directory entries and temporary debris;
- whether a later transaction can recover.

The original CSV tests checked final bytes but did not prove atomic replacement. An in-place truncate-and-write implementation could pass. Capturing `os.FileInfo` before and after a real commit and requiring a different file object closed that gap. The symlink path needed the same check on the target object, not only `Lstat` on the link.

## 6. Exercise both directions of every state rule

Common omissions came from testing only one side:

- conflict path but not successful multi-file commit;
- savepoint rollback but not outer rollback allocation reuse;
- deleting a low rowid but not the current high-water rowid;
- in-place external overwrite but not atomic pathname replacement;
- explicit schema but not inferred schema;
- same schema affinities but not different affinities with identical CSV layout;
- read/write behavior through a direct path but not alias direction changes;
- file bytes unchanged after no-op but not file identity and modification time;
- absent final terminator but not present final terminator;
- too few fields but not too many fields.

For each rule, ask what its complement is. Add the complement when a realistic wrong implementation treats the two cases differently.

## 7. Preserve old behavior explicitly

New features often regress paths that do not use the feature. The CSV task made writable mode opt-in, so default read-only streaming was part of compatibility even though the new code focused on writes.

The regression probe used a file with a valid first row and malformed later row, then queried `LIMIT 1`. The old streaming cursor returned the first row without parsing the unreachable malformed row. Eager whole-file loading failed.

Be careful with fail-to-pass classification. A standalone compatibility test may pass on the base repository and be rejected as a new F2P test. Put unchanged upstream regression coverage in base mode, or include it in a feature-dependent top-level new test that still fails without the feature.

## 8. Do not fix false positives with unfair tests

This was the most important late lesson.

Auto review found that the suite did not force a non-conflict failure after one file in a multi-file publication had already been replaced. An attempted fix staged 128 files, started commit concurrently, polled until file 0 changed, then obstructed file 127.

That test was unfair because it assumed:

- ascending publication order;
- file 0 precedes file 127;
- publication is slow enough to intercept;
- polling observes the intermediate state;
- an unbounded loop eventually terminates.

A descending, unordered, batched, or faster correct implementation could fail the setup. The test coupled to timing and private order rather than the public all-or-nothing outcome.

The replacement used SQLite's public commit hook synchronously:

1. Stage changes to two CSV files.
2. Commit through one SQLite connection.
3. In the commit hook, observe complete prepared publications for both files.
4. Veto the commit.
5. Require both original contents to be restored.
6. Require the staged file objects to be gone.
7. Require no temporary entries.

This scenario has no polling, sleep, ordering assumption, or interception race. It rejected the previously passing agent because that agent delayed publication into an error-ignored commit phase, while the reference used the repository's documented two-phase transaction protocol.

The general rule is: when closing a coverage gap, use a deterministic public synchronization point. If no such point exists, do not invent an order-sensitive race. Test the strongest observable invariant that can be induced fairly.

## 9. Separate fairness from coverage

Coverage and fairness pull in different directions. More assertions do not automatically improve the suite.

Before adding an assertion, answer:

- Is the exact behavior stated?
- Is it clearly preserved by existing public behavior?
- Is it required by a stable standard?
- Could two competent implementations differ while satisfying the public contract?
- Does the test require a specific callback phase or operation order?
- Does it assert semantics, or merely the reference's chosen representation?

CSV fairness failures illustrate the boundary:

- Validation of malformed rows could happen at table creation or first write. Tests had to accept either.
- A symlink entry did not inherently have to survive atomic replacement until the prompt said writes through a symlink update its target.
- Exact LF versus CRLF bytes inside unchanged quoted fields were not implied by valid CSV semantics. The prompt had to promise lexical preservation before exact-byte tests were fair.
- A commit-hook backup did not have to contain a specific phase's bytes when repository guidance allowed fallible work in `Sync`.

Never justify an assertion by saying the reference passes it.

## 10. Why auto review stayed at 1/3

The CSV test suite was broad, deterministic in most areas, and had twelve large top-level tests. It still received 1/3 several times because each review found a High false-negative risk:

1. No atomic file-identity assertion.
2. No symlink-target identity assertion.
3. No atomic pathname-replacement conflict.
4. No delete-high-water rowid case.
5. No outer rollback allocation reuse.
6. No physical no-op identity check.
7. No inferred-schema writable case.
8. No successful multi-file commit.
9. No late multi-file rollback/publication-failure discriminator.

Each gap admitted a realistic passing wrong implementation. Under the rubric, one High gap was enough to keep Tests at 1/3.

The suite improved only when it moved from broad scenario coverage to mutation-complete coverage: each major promise had a positive case, a failure case, and a named wrong implementation it killed.

## 11. Audit solution quality at the same time

False-positive prevention applies to the reference too. Repeated solution reviews found defects not initially exercised:

- blank-line loss for one-field empty records;
- fallible publication in an error-ignored callback;
- transient partial multi-file publication;
- Windows directory sync failure;
- ownership races during replacement identity handoff;
- pre-write savepoint rollback failure;
- affinity-equal no-op rewrites;
- rollback clobbering an external replacement;
- embedded CRLF normalization.

Whenever a solution review finds a prompt-required bug, fix the reference and decide whether a fair regression test is missing. Do not automatically add a test if it would pin implementation shape. Add it only through public behavior.

## 12. A 3/3 test-quality workflow

### Before implementation

- [ ] Split the prompt into atomic requirements.
- [ ] Give every requirement a fairness source.
- [ ] Create a state matrix and a wrong-implementation matrix.
- [ ] Identify compatibility paths not using the new feature.
- [ ] Plan positive, negative, rollback, and cleanup cases.

### While writing tests

- [ ] Use public APIs and external outcomes.
- [ ] Use deterministic signals or callbacks.
- [ ] Avoid sleeps, polling windows, operation order, and host ordering.
- [ ] Check identities and metadata when the contract includes physical replacement.
- [ ] Check recovery after failure, not only immediate failure.
- [ ] Check both sides of state transitions.
- [ ] Keep error matching flexible.

### Before agent runs

- [ ] Run mutation probes for every major requirement.
- [ ] Verify all new top-level tests fail on base.
- [ ] Verify reference base and new modes pass repeatedly.
- [ ] Run race, vet/lint, format, build, and offline Docker checks.
- [ ] Inspect JUnit and raw logs.

### After agent runs

- [ ] Read every passing production diff.
- [ ] Reproduce suspicious behavior with public probes.
- [ ] Classify every failure as skill, fairness, environment, verifier, or missed requirement.
- [ ] Build a cross-run failure matrix.
- [ ] Strengthen fair gaps and rerun affected checks.
- [ ] Run the final False Positive Check only after all agents finish.

### Before submission

- [ ] No High or Medium realistic wrong implementation remains untested.
- [ ] No valid alternate implementation is rejected.
- [ ] No test depends on timing or private order.
- [ ] Every requirement is tested at the public boundary.
- [ ] Test patch contains only executed tests and the runner.
- [ ] Exact submitted artifacts pass the four-state matrix.

No checklist can guarantee a platform score, but this process addresses what a 3/3 Tests review measures: fair public assertions, deterministic execution, meaningful regression coverage, and resistance to plausible incomplete implementations.

## 13. Additional opinions from concurrent lifecycle APIs

For asynchronous work, a requirement ledger should track transitions as well as features. A useful generic lifecycle is: requested, queued, admitted, running, message pending, message processing, function returned, result committed, and settled. Cancellation, replacement, filtering, shutdown, and composition may have different effects at each boundary. A suite that checks only eventual output can miss implementations that acknowledge the wrong caller, start discarded work, release capacity early, or allow a stale generation to send into a replacement.

List every place an operation can wait. For a synchronous send API, those places may include a serialization gate, scheduler admission, channel delivery, filtering, update processing, and acknowledgement. Then cross each wait with all documented withdrawal signals: caller cancellation, owner cancellation, replacement, return, and program shutdown. This method found a realistic defect where cancellation was checked before a gate but not while waiting for it. The public assertion should be that the call returns at the promised lifecycle boundary, not that the implementation uses a particular lock or channel.

Distinguish three facts that are easy to conflate:

- whether a message was admitted;
- whether its visible effect began;
- what boolean or completion result its caller receives.

Shutdown may allow an already-processing update to finish while still requiring its waiting sender to receive false. Conversely, ordinary task cancellation after processing begins may leave the message committed. Tests should state and observe these distinctions explicitly when the prompt does.

Composition deserves its own interaction matrix. Test a managed operation directly, inside each existing composition primitive, and as a value delivered by the managed operation. These paths may have different completion rules. Also test controls before and after filtering: interpreting a control before the filter is a plausible shortcut that direct tests will not detect.

Every proposed regression should name the mutant it distinguishes. If the previously broken implementation also passes the new test, that test may improve broad coverage but it does not close the reported false-positive hole. Keep it only if it protects a separate prompt requirement; otherwise design a stronger public observation or document that the particular race cannot be tested deterministically without private hooks.

Concurrency tests should create happens-before relationships with barriers. Hold a public filter or update, wait until the operation has definitely entered it, perform the lifecycle change, and then release it. Avoid simultaneous `select` races whose outcome is intentionally nondeterministic. Repeating a probabilistic test a thousand times lowers the miss probability but does not make the test fair or deterministic.

Failure cleanup is part of test quality. Use idempotent release functions, buffered result channels, and cleanup paths that unblock the program even after `Fatal`. A correct assertion wrapped in a test that hangs on failure damages JUnit completeness and makes calibration evidence unreliable.

Audit the reference with the same mutation matrix used for candidates. Reviewers found real Bubble Tea bugs after all tests passed by inspecting admission, processing, and acknowledgement separately. When such a defect is public and deterministic, add a regression. When only a private interleaving could expose it, fix the reference and seek the strongest fair public invariant rather than forcing internal instrumentation.

Finally, keep functional completeness and production quality separate but mandatory. A candidate can satisfy every behavior and still be an invalid pass because the repository formatter, linter, race detector, or compatibility suite fails. False-positive review should therefore end with both a behavioral verdict and a production-check verdict.

## 14. Redscript forwarding-diagnostic false-positive case

The redscript task shows why a path-sensitive compiler warning needs both mutation coverage and adversarial review beyond the visible suite. The public observation was simple: emit one warning when a resolved wrapper has any reachable normal exit that did not evaluate `wrappedMethod`. The realistic wrong implementations were not simple, because lowering and backend semantics could erase or invent paths before the diagnostic pass saw them.

The review history exposed the following mutants one at a time:

| Mutant | Observable defect |
| --- | --- |
| Traverse every call argument eagerly | A forwarding call in a resolved short-circuit argument incorrectly covers its bypass path. |
| Analyze each switch body as an immediate switch exit | A non-empty case that falls through into a forwarding later body receives a false warning. |
| Send a final empty case directly to the switch exit | An empty label that enters a forwarding default receives a false warning. |
| Infer logical syntax from function names | An eager ordinary call named `OperatorLogicAnd` is treated as lazy and receives a false warning. |
| Read an unsubstituted generic declaration return type | A call instantiated to `Nothing` is treated as normally returning. |
| Preserve result types only for ordinary source calls | Generic operator, cast, or lowering-generated calls instantiated to `Nothing` are missed. |
| Treat `true` and `false` cases as exhaustive without a default | An unknown no-default switch loses the required unmatched path. |
| Promote inferred Boolean outcomes to source literals | A computed conditional scrutinee incorrectly eliminates a no-default unmatched path. |
| Drop literal provenance when any `case let` exists | A literal match in a mixed pattern switch fails to prune an unreachable later return. |
| Hoist every value-binding condition prefix before the branch chain | An earlier branch is incorrectly considered to have evaluated a later condition. |
| Retain branch-local prefixes in IR but assemble all of them first | Diagnostics pass, but compiled `else if` behavior changes. |

Each row is a public semantic distinction, not a demand for a particular analyzer class, metadata field, or control-flow graph representation. Tests should compile source through the documented compiler API and assert diagnostic code, level, owner span, and successful or failed compilation where relevant. Existing frontend error snapshots and backend bytecode fixtures should protect unaffected behavior.

One false-positive review found a defect that the suite had not distinguished. A candidate propagated the inferred outcomes of `Decide() ? true : false` into switch exhaustiveness. It then removed the unmatched path from a no-default switch, even though the contract reserved that refinement for syntactic literal scrutinees and matchers. A small public probe made the candidate and reference diverge and justified adding coverage. The correction was not to require a private “literal provenance” field; it was to state and test the observable difference between a source literal and a computed Boolean.

Top-level test names can conceal different root causes. In one cohort, every agent failed `nonreturning_call_paths`, but some mishandled logical operators while another ignored calls generated by `for` lowering. When a test contains several behavioral programs, inspect the first failing assertion, evaluator evidence, and candidate transfer logic before deciding that all agents hit one tripwire. Prefer separate testcases when fail-to-pass classification permits it; otherwise give subcases clear assertion context so JUnit identifies the program that failed.

The solution-quality audit must follow IR changes through code generation. Preserving a source-local condition prologue in IR fixed analysis of `else if let`, but the first assembler update emitted all prologues eagerly and changed runtime semantics. This is exactly the kind of false positive that frontend-only tests admit. The backend bytecode regression suite belongs in the base gate whenever call representation, branch representation, lowering order, or visitor exhaustiveness changes.

A useful mutation checklist for compiler flow diagnostics is:

- direct call versus operator syntax with the same resolved name;
- direct `Nothing` return versus generic instantiation to `Nothing`;
- explicit source calls versus calls introduced by desugaring;
- source expression order versus lowered temporary order;
- unknown Boolean values versus syntactic Boolean literals;
- all-expression switches versus mixed expression and pattern switches;
- matcher evaluation versus body fallthrough;
- loop zero-iteration paths versus non-returning loop setup or condition paths;
- closure construction versus deferred closure-body execution;
- diagnostic correctness versus unchanged emitted bytecode.

Passing the feature suite is only the start. Audit every legitimate pass against this matrix, run a focused public probe for any shortcut, and require formatter, Clippy, frontend regressions, and backend bytecode regressions. The strongest suite rejects incomplete analyses while allowing different internal representations, and the strongest reference remains correct even in behavior the diagnostic tests do not directly observe.

## 15. rust-minidump repeated-stream recovery false-positive case

The repeated-stream suite improved most when each added test named a concrete incomplete implementation that it rejected. Distinct stream types did not prove that `all_streams()` retained duplicates; the load-bearing sequence was `WORD, OTHER, WORD`, with an assertion that all three public directory records appeared in exact order. Likewise, testing repeated threads and modules did not constrain independently written `--dump` branches. Each manually dispatched group needed duplicate coverage or equivalent public instrumentation.

Several mutants were especially plausible:

| Mutant | Observable defect |
| --- | --- |
| Keep only the newest directory entry per type in `all_streams` | A repeated type disappears even if occurrence APIs use a separate ordered vector. |
| Print only the stream families represented by early CLI tests | Unloaded modules, handles, crashpad, macOS, stability, or another independent group remains singular. |
| Select Memory64 by directory presence | Malformed Memory64 entries suppress a valid MemoryList fallback. |
| Use the singular getter after selecting Memory64 | Only the latest valid Memory64 occurrence is printed. |
| Collapse every `UnusedStream` by type | A non-empty unused record is hidden along with empty placeholders. |
| Pass no context to every decoder registered for SystemInfoStream | Existing third-party decoders lose the documented recovered system-info argument. |
| Warn that the newest duplicate is used before decoding | Diagnostics contradict latest-valid fallback when the newest occurrence is corrupt. |

The strongest tests used distinguishable public payloads, not only heading counts. Two valid Memory64 streams used different addresses and had to appear in order while MemoryList output remained absent. Repeated structured and extension streams carried different values where their printers exposed them. Mixed unused entries placed a non-empty record before empty placeholders and checked both API retention and presentation, proving that only zero-sized placeholders were collapsed.

Recovery needs both positive and negative matrices. Test a valid older occurrence followed by malformed data, malformed data followed by a valid occurrence, several valid occurrences, and a type for which every preferred-format occurrence is malformed. For system information, combine a corrupt latest entry with a valid earlier one, then decode both built-in dependent streams and a custom public decoder that shares the SystemInfoStream numeric type. This catches global stream-type special cases without prescribing the reader's internal storage.

Finally, inspect every passing agent rather than treating a green suite as proof. The review history repeatedly found solutions that implemented nearly all core behavior but exploited uncovered manual branches or compatibility seams unintentionally. Those patches are the best mutation corpus: turn each realistic shortcut into a public, deterministic assertion, rerun the full fail-to-pass matrix, and keep the reference solution under the same regression and production-quality gates.

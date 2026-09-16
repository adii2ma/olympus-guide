# How to Calibrate a Hard but Solvable Task

This guide extends the interaction-driven difficulty method in [agent-2.md](agent-2.md). It uses the transactional writable CSV task as a case study. Keep it outside the challenge package.

The goal is not to make a named model fail through traps. The goal is to create a real upstream task whose public invariants require sustained repository reasoning. A healthy calibration lets a strong Terra Max run solve the task with substantial effort, while Terra High usually delivers a serious but incomplete implementation. Failure must come from missed behavior, not ambiguity, hidden implementation rules, broken tooling, or luck.

## 1. Define the midpoint before writing tests

Use current platform criteria rather than an informal notion of hard. At the time of this guide, the pass-rate ceiling is 40 percent, and legitimate passing agents must meet the active file, message, and effective implementation-line medians.

A useful hard-task target is:

- at least one legitimate solve in a calibration pool;
- no more than four legitimate solves in ten runs;
- Terra Max can finish, but must inspect several subsystems and revise its design;
- Terra High commonly reaches the core feature but misses one or more cross-layer invariants;
- failures are spread across meaningful interactions rather than one obscure tripwire;
- reference and alternate correct designs remain possible.

One CSV calibration batch produced one legitimate pass in ten runs. A later pool was one in twelve. That demonstrated high difficulty, but the run review also showed a warning: many agents failed on the same lone-empty-field edge after completing almost all transaction machinery. Pass rate alone therefore overstated the breadth of difficulty.

## 2. Choose interaction-rich scope

Hard tasks should combine several existing public concerns that must agree. The CSV task combined:

- SQLite virtual-table update callbacks;
- transaction visibility and rollback;
- nested savepoints;
- rowid allocation state;
- self-referential statement snapshots;
- physical file identity across hard links and symlinks;
- writer exclusion across connections;
- compatible views on one connection;
- byte-level conflict detection;
- multi-file all-or-nothing publication;
- atomic file replacement and cleanup;
- CSV quoting, BOM, headers, terminators, permissions, and affinities;
- preservation of the old streaming read-only path.

No single item is deliberately obscure. The difficulty comes from preserving all of them at once.

Avoid tasks that are only:

- a copy of a nearby implementation;
- a mechanical API rename;
- many unrelated edge cases under one vague prompt;
- a large description with a small implementation;
- one parser corner case that decides the whole pass rate;
- a private algorithm hidden behind ordinary public behavior.

## 3. Build a state matrix

Before writing code, list the dimensions that can interact. For writable CSV, a compact matrix looked like this:

| Dimension | Important states |
| --- | --- |
| source | filename, inline data, `RegisterFS` |
| mode | default read-only, explicit off, writable on |
| transaction | none, active, committed, rolled back |
| savepoint | before first write, after write, nested, released, rolled back |
| operation | insert, update, delete, self-insert |
| rowid | automatic, explicit NULL, explicit value, changed value, deleted high-water row |
| observer | owner, second connection, direct file reader, reopened table |
| path | direct, hard-link alias, symlink alias, same pathname after replacement |
| layout | header on/off, inferred/declared schema, delimiter, LF/CRLF, BOM, final terminator |
| commit | real change, semantic no-op, byte conflict, pathname replacement, publication failure, hook veto |
| value | NULL, empty text, integer, real, embedded delimiter, quote, LF, CRLF |

Do not test the full Cartesian product. Select combinations that invalidate simplistic designs.

## 4. Name the wrong implementation each test kills

Every difficult test should have a concrete opponent. Examples from the CSV task:

- Savepoint snapshots rows but not `nextID`: roll back an allocation and require reuse.
- Rowids use `max(live)+1`: delete the current high-water row, then insert and require the next higher value.
- Self-insert scans the growing stage: use an input predicate that can match new rows and require a statement snapshot.
- Ownership uses canonical pathname: contend through a hard link and through a symlink.
- Same-path objects are treated as identical: replace the pathname with a new inode and let another connection own the replacement.
- Compatible views include SQL affinities in the key: use the same CSV layout with TEXT and INTEGER schemas and require shared staged rows.
- Read-only cursors became eager: query `LIMIT 1` over a valid first row and malformed later row.
- Conflict checks use size and mtime: replace bytes with the same length and restored timestamps.
- Conflict checks use the original descriptor: rename a different file over the pathname.
- No-op detection compares serialized strings only: start with numeric `01` or `1.0`, perform a semantically equal expression update, and require the same file object.
- Publication writes in place: compare file identity before and after a real commit.
- Symlink writes preserve the link but overwrite its target in place: compare target identity as well as alias type.
- All-empty single-field records use a blank line: insert NULL and empty text into a one-column file and reopen it.
- CRLF detection scans raw newlines without quote awareness: place an embedded newline before the first record boundary.
- Rollback always restores original bytes: externally replace a file after prepared publication, veto commit, and require the external file to win.

This mutation-oriented approach produces real difficulty without private coupling.

## 5. Calibrate Terra High and Terra Max in stages

Use separate roles for the two run levels.

### Terra High pilot

Terra High should be able to understand the task, compile a substantial solution, and pass much of the suite. Its failures should reveal which cross-layer behavior is easy to miss.

Healthy Terra High evidence:

- it modifies the correct production subsystem;
- baseline tests still pass;
- core insert, update, delete, transaction, and serialization paths work;
- remaining failures concern ownership, rollback, no-op identity, or layout interactions;
- it does not fail because the prompt hid a requirement.

If Terra High passes quickly by copying a local pattern, the task is too easy. Add one or two public interaction requirements, not a pile of isolated edge cases.

### Terra Max solvability pilot

Terra Max should have a credible path to a full solve. Run more than one attempt because one trajectory is not a distribution.

Healthy Terra Max evidence:

- at least one run passes legitimately;
- the run needs broad repository inspection and iterative debugging;
- the implementation is substantive and upstream-shaped;
- failed Max runs miss stated behavior rather than encountering harness faults;
- different Max runs fail at different meaningful seams.

If all Max runs stop at the same assertion, investigate before declaring success. The assertion may be unfair, the prompt may be unclear, or one peripheral tripwire may dominate the score.

## 6. How the CSV task reached roughly one pass in ten

The task was hard because agents tended to solve the obvious center first: parse the file, stage rows, implement `Update`, and rename a temporary file. The hidden difficulty remained at subsystem boundaries.

Repeated agent misses included:

- sharing staged state across views but incorrectly treating affinity as layout;
- replacing the streaming read-only cursor with eager whole-file parsing;
- committing one CSV before discovering a conflict in another;
- detecting CRLF from the first raw newline instead of the first record boundary;
- keying ownership by path rather than inode identity;
- forgetting schema inference when writable mode omitted `schema`;
- losing a one-column empty record because `encoding/csv.Writer` emits a blank line;
- rewriting numeric lexical forms during semantic no-ops;
- accepting an empty header-enabled file and inventing a header;
- mishandling savepoints created before the virtual table's first write;
- leaving stale state after failed replacement;
- restoring original bytes over a newer external file after commit veto.

Most agents built hundreds of lines of real transaction code. Terra High-like runs commonly reached 9 to 11 of 12 broad tests. A Terra Max-like run could finish after tracing callback semantics, aliases, and byte layout together. That is the intended struggle profile.

The weak part of the early calibration was concentration: ten agents failed the same all-empty one-column record case. Two had otherwise passed 11 of 12 tests. That edge was fair, but it was too influential as a difficulty signal. The lesson is to retain it as data-integrity coverage while ensuring other core mutations independently discriminate incomplete implementations.

## 7. Add hints only when they improve comprehension

Do not add a hint merely because an agent failed. First classify the failure:

- **skill failure:** keep the task unchanged;
- **prompt ambiguity:** clarify observable behavior;
- **unfair assertion:** remove or weaken the test, or state the contract;
- **environment failure:** fix Docker or the harness;
- **shared narrow tripwire:** add a restrained behavioral sentence if the behavior is genuinely required;
- **missing coverage:** strengthen tests, not the prompt.

A good vague hint names the public invariant, not the implementation. Examples:

- Good: "An all-empty row must remain a row after reopening."
- Bad: "Special-case `csv.Writer.Write([]string{""})` by emitting two quotes."
- Good: "A savepoint may exist before this table is first modified."
- Bad: "Store a baseline snapshot under missing savepoint IDs."
- Good: "Ownership follows the underlying file even when a pathname is rebound."
- Bad: "Use `os.SameFile` in `Begin` and `visibleTxn`."

If Terra Max repeatedly fails one required interaction, add one sentence like the good examples, rerun, and verify Terra High still does not trivially solve the whole task. Do not turn the description into a test transcript.

## 8. Keep difficulty distributed across core behavior

Use a balance such as:

- 40 percent core feature behavior;
- 25 percent transaction, concurrency, or lifecycle interactions;
- 20 percent compatibility and regression behavior;
- 15 percent boundary serialization and failure cleanup.

This is a design heuristic, not a scoring formula. The important point is that a solver cannot pass by implementing only happy-path mechanics, but also does not fail solely because of one esoteric byte sequence.

For the CSV task, the core discriminators were transaction staging, savepoints, alias ownership, conflict handling, and atomic publication. Empty records, CRLF spelling, and numeric lexical preservation protected data integrity but should not be the only reasons agents fail.

## 9. Detect when the task is too hard or impossible

Stop and revise if:

- no reference implementation can pass clean production checks;
- all capable agents fail before exercising the feature;
- different correct designs are rejected by exact internal assumptions;
- the task needs unstated callback order, timing, filesystem order, or platform quirks;
- zero agents solve after prompt clarity and harness correctness are confirmed;
- most failures are compile, environment, dependency, or timeout failures;
- the only passing implementation uses reference-specific structure;
- the task requires behavior upstream would not accept.

The 128-file mid-publication polling test was an example of artificial hardness. It required file 0 to publish before file 127 and required the commit to be slow enough for polling to intervene. A reverse-order or faster correct implementation could fail. Removing it made the task fairer without making the transaction requirement easy.

## 10. Detect when the task is too easy

Strengthen the task if:

- more than the active pass-rate ceiling solve legitimately;
- agents finish with a small obvious patch copied from adjacent code;
- passing implementations omit cleanup, compatibility, or negative cases;
- Max and High both solve in one pass without architectural revision;
- successful diffs fall below effective scope requirements;
- tests assert only happy paths or counts rather than meaningful state.

Strengthen with interactions already implied by the feature. For writable CSV, adding same-connection view sharing, alias ownership, rollback of allocation state, direct-reader atomicity, and multi-file all-or-nothing behavior increased real scope. Adding an arbitrary private temp-file naming rule would not.

## 11. Use a calibration ledger

For every run, record:

| Field | Record |
| --- | --- |
| model and effort | Terra High or Terra Max |
| baseline | pass, fail, or not reached |
| new tests | exact passed and failed tests |
| production checks | format, lint/vet, build, race |
| implementation size | relevant files and effective LOC |
| root cause | skill, missed requirement, unfair test, environment, verifier, resource |
| valid alternative | yes or no, with evidence |
| shared failure cluster | tests and number of runs |
| prompt action | none, clarify, remove assertion |
| test action | none, strengthen, make deterministic |

Review the matrix across runs. Do not infer difficulty from one summary or one pass rate.

## 12. Final difficulty gate

- [ ] At least one legitimate solution exists.
- [ ] Pass rate is within current limits.
- [ ] Passing-agent file, message, and effective LOC medians meet current limits.
- [ ] Terra Max struggles but has a fair path to completion.
- [ ] Terra High failures are substantive skill misses, not blocked setup.
- [ ] No single peripheral assertion dominates most failures.
- [ ] Every hard assertion is public, discoverable, or standard.
- [ ] Concurrency tests use signals rather than timing.
- [ ] Alternate correct architectures can pass.
- [ ] The False Positive Check confirms that passes are real.

Difficulty is successful only when it measures sustained engineering judgment. A low pass rate produced by fragile tests, hidden policies, or one serialization trick is not a hard task; it is a noisy verifier.

## 13. Additional opinions from run-bundle calibration

Calibration quality is limited by artifact quality. A run archive without its patch, JUnit, log, and trajectory is an infrastructure failure, not evidence that the solver failed. Record download errors separately and never count those runs in the difficulty denominator. Likewise, a suite timeout after one deadlocked test does not mean every later test failed; report the tests reached, the blocking root cause, and the tests that never ran.

Design the suite to reveal a solution's shape even when it is incomplete. Put representative core behavior before deep interactions, bound every blocking test, and release resources on failure so later independent tests can run. Test ordering must improve diagnostics rather than manufacture difficulty. A solver that passes nine tests and hangs on the tenth is more informative than a report of fifty undifferentiated missing cases.

Use a solvability staircase:

1. public API and basic success;
2. ownership and ordinary cancellation;
3. ordering, capacity, and replacement;
4. filtering and composition;
5. shutdown and adversarial lifecycle intersections;
6. compatibility, race, and production checks.

A healthy serious failure should climb most of this staircase and stop at one or more documented interactions. If most runs fail to compile the public surface, the scope or prompt may be inaccessible. If they all reach the same late seam, inspect whether that seam is a core invariant, an unclear sentence, or an over-weighted peripheral tripwire.

Hints should repair comprehension, not lower the contract. The best hint names the duration of a public state: for example, a caller waiting behind earlier work is still pending and remains subject to cancellation. It should not mention mutexes, channels, scheduler fields, or the reference architecture. After adding one sentence, rerun a fresh cohort; historical trajectories cannot show the effect of new wording.

Do not tune toward a model name or guarantee that exactly one to three of ten runs will pass. The meaningful target is a confidence interval supported by fresh independent attempts. A small cohort can swing sharply through normal run variation. Preserve separate ledgers for every prompt or feature-test version, while retaining old legitimate passes only as historical solvability evidence.

Failure diversity is a stronger signal than raw failure count. Several agents missing different interactions suggests distributed engineering depth. Ten agents failing the same narrow case suggests ambiguity, a dominant tripwire, or an unusually hard but localized concept. Keep the case if it protects important data or lifecycle integrity, but do not let it be the only discriminator.

When a task is too hard, soften it in this order:

- repair harness and environment defects;
- make failures local and fully reported;
- clarify ambiguous observable behavior;
- remove unfair implementation coupling;
- add one restrained invariant-level hint;
- only then reconsider authentic scope.

When a task is too easy, first audit passing patches for false positives. If they are legitimate, add one natural interaction between existing requirements or a clearly announced scale constraint. Do not add unrelated edge cases, unpredictable filenames as semantic traps, or private ordering rules.

Reference quality also affects perceived solvability. A reference that passes feature tests but fails lint, races, or a prompt-permitted external-context path proves only that the verifier is incomplete. Freeze calibration only after the reference passes the exact repository production gates and an adversarial review beyond the visible suite.

The final calibration narrative should therefore include cohort version, usable run count, legitimate passes, root defects among failures, missing artifacts, median implementation scope for passes, and any shared failure cluster. This makes “hard but solvable” an auditable conclusion rather than a pass-rate slogan.

## 14. Redscript forwarding-diagnostic calibration case

The control-flow-aware `@wrapMethod` diagnostic was a useful example of difficulty changing sharply because of one sentence. The production work was substantial: a correct solution had to connect resolution, lowering, IR metadata, flow analysis, backend assembly, and parameter metadata. Most failed runs still implemented hundreds of relevant lines and passed nearly the entire behavioral suite.

One cohort had no legitimate pass among four usable runs. Two near-solves repeatedly missed the same pair of interactions: a value-binding condition lowered before its actual branch, and a literal expression label lowered through a mixed pattern switch. Both failures came from losing source evaluation provenance during lowering. The description already said that generated temporaries must not change optional evaluation, but it did not make the interaction with mixed control-flow forms salient enough for the cohort.

An attempted clarification named both combinations directly. The next cohort went from zero of four to three of three legitimate passes. That was evidence that the wording had become a map of the difficult cases rather than a statement of the invariant. Replacing it with “Combining control-flow forms does not relax source-order or reachability rules” preserved fairness while making the interaction discoverable without naming the fixtures or the required IR shape.

The following cohort again had four failures, and all four reported the same top-level `nonreturning_call_paths` test. Reading only the test names would have suggested one impossible tripwire. The diffs and evaluator evidence showed two different causes instead:

- some implementations returned early from their logical-operator transfer and never applied the resolved `Nothing` return type;
- the strongest near-solve analyzed source `for` syntax and never saw a generic `Nothing` call introduced by loop lowering.

The shared public invariant was narrower than either implementation detail: non-returning behavior belongs to every resolved call, including calls implicit in source syntax. Adding only that sentence was a reasonable feasibility adjustment. It could help the 25-of-26 near-solve without handing the other agents solutions to their unrelated switch, short-circuit-temporary, or conditional-prefix defects.

This case suggests a calibration procedure:

1. Compare only runs produced from the same description and test version.
2. Read JUnit, evaluator reasoning, and production diffs before treating repeated test names as repeated root causes.
3. If one invariant is genuinely unclear, replace a weak sentence with one invariant-level clarification; do not append examples from the hidden suite.
4. If a clarification causes a pass-rate cliff, generalize it instead of adding new obscure tests to compensate.
5. Preserve hard interactions in tests, and calibrate comprehension through the smallest behavioral wording change.
6. Re-run a fresh cohort. Historical runs explain the edit but cannot establish the new pass rate.

Difficulty was distributed across branch correlation, lazy arguments, switch dispatch and fallthrough, loop exits, deferred closures, annotation scope, and non-returning calls. The repeated late failures therefore represented genuine cross-layer reasoning, while the exact-case hint represented over-guidance. That distinction is the middle point to seek in future compiler tasks.

## 15. rust-minidump repeated-stream recovery calibration case

This task had real cross-component scope: the reader had to retain ordered duplicate directory entries, public accessors had to expose occurrences and recover from malformed replacements, system information had to provide architecture context without breaking third-party decoders, classification had to retain indexes, and two consumers had to update their presentation behavior. Legitimate solutions changed three production files and required roughly the target implementation scale, so it was not an artificial edge-case exercise.

The raw pass rate still needed careful interpretation. In one cohort, five of ten agents passed while the other five passed all baseline tests and nine of ten feature tests, converging on the same third-party SystemInfoStream compatibility rule. After the suite was strengthened, two of ten passed and most failures completed eleven to thirteen of fourteen feature tests, again clustering on system-info context or non-empty `UnusedStream` presentation. Those numbers established solvability and substantial work, but they also showed that the headline failure rate was concentrated in narrow compatibility seams.

The useful calibration lesson is to measure both breadth and concentration. Storage, iteration, fallback, classification, and CLI changes created distributed engineering work, but a single late assertion should not be credited as the whole source of difficulty. Read each failed patch and JUnit body, record how much of the behavior it implemented, and identify whether repeated failures share one root cause. Keep a subtle case when it protects an explicit public compatibility promise, but make sure the rest of the suite independently rejects plausible partial designs.

Difficulty also changed when coverage was added. Testing only representative output branches made the task easier than the description because manual dispatch branches could remain singular. Adding duplicate `all_streams` coverage, independent CLI groups, multiple valid Memory64 streams, malformed-only Memory64 fallback, and mixed non-empty/empty unused records aligned measured difficulty with the actual contract rather than adding unrelated traps. This is the preferred way to harden an under-discriminating task: close natural invocation-path gaps instead of inventing obscure behavior.

## 16. Resolve and Externalize DICOM JSON Bulk Data calibration experience

This task looked difficult for the right reason at first: a solver had to extend nine public entry points, preserve generic dictionaries, support local non-`Send` futures, traverse nested data sets deterministically, validate references before provider calls, preserve pixel-fragment structure, and keep every legacy convenience function unchanged. That is broad production work with several interacting contracts.

The raw agent pass rate still overstated the effective difficulty. Many unsuccessful solutions completed almost the entire async resolver and writer design and failed only one typed-empty binary case. They correctly avoided calling the writer for an empty `Vec<u8>`, but delegated the value to the legacy serializer and emitted an empty `InlineBinary` field instead of the required VR-only object. A later candidate had the analogous defect for an empty ordinary PN or US value, rebuilding it as `"Value": []` instead of preserving legacy spelling. These failures were fair because the description explicitly covered empty binary output and exact no-bulk compatibility, but they were narrow tripwires rather than evidence that the agents could not build the core feature.

The useful calibration signal was therefore the failure distribution, not only the pass count. Record how many feature tests each run passed, whether it reached runtime behavior, and whether failures share one root cause. If six agents pass twenty-one of twenty-two cases and all miss the same boundary value, describe the task as broad with a subtle compatibility edge. Do not credit that one assertion with all of the apparent difficulty.

Strengthening coverage can also change calibration even when only `test.patch` changes. New compile-time public-surface checks or extra wrapper matrices may cause previously near-complete implementations to stop before behavioral tests. Before adopting reviewer suggestions, replay every legitimate passing patch and representative failing patches against the proposed test revision. Keep additions that close material false-positive paths, but avoid turning several medium-strength observations into a new wall of peripheral compile failures.

For this task, the healthiest difficulty came from interactions that belong naturally to DICOM JSON bulk handling:

- recursively mixing ordinary, sequence, primitive bulk, and encapsulated pixel values;
- preserving the declared VR independently in provider requests and emitted JSON;
- stopping on the first provider failure while maintaining tag and item order;
- retaining exact ordinary serialization on calls that make no provider requests;
- carrying a complete `AttributeSelector` through nested conversion failures;
- accepting both OB and OW encapsulated Pixel Data without flattening fragments.

These cases exercise design judgment across the feature. By contrast, an unstated interpretation of whitespace, an incidental trait requirement introduced by `assert_eq!`, or an output-validation policy inherited only from the reference makes the verifier stricter without making the engineering task better.

The final calibration lesson is to keep a versioned evidence table. For each description and test-patch fingerprint, record legitimate passes, near-solves, compile failures, exact failing assertions, and the production scope of passing patches. Test-only revisions may preserve useful historical evidence, but they do not prove the new pass rate until the old patches are replayed or a fresh cohort is run.

## 17. Tokio Modbus Unix-socket transport calibration experience

The Unix-socket transport task had genuine cross-layer work: feature-gated public entry points, existing protocol framing, asynchronous and synchronous clients, timeout behavior, and server admission and shutdown. Yet the first complete platform cohort produced eight legitimate passes in ten runs. The two failures were near-solves with the same observable defect: synchronous attachment of an already-connected stream assumed an ambient asynchronous runtime, even though an ordinary synchronous caller need not provide one. This was a fair failure of the stated API, but two shared misses did not make an 80-percent-pass task hard enough.

The author strengthened tests only where a public promise was under-observed. A plain synchronous caller exposed the attachment defect. Independent raw peers checked interoperability instead of letting a solver's own client and server agree on an incorrect format. Later checks covered timeout variants and error handling through public observations. The reference implementation and existing passing patches were replayed against those revisions; executable wrong-behavior variants were also checked. After the guarded evaluation-only refresh, the platform result remained eight passes and two attachment failures. Stronger coverage improved confidence in the verdict; it did not retroactively lower the pass rate.

The temptation at that point was to add a private transaction-number policy, implementation layout rule, or timing-sensitive assertion simply to reject more solutions. None followed from the public contract. The better disposition was to report the task as solvable but insufficiently discriminating, keep the genuine passes, and stop where the staged-rollout and pass-rate gates required reconciliation. Historical dispatch order also matters: a ten-run denominator is not a substitute for the required staged sequence.

The reusable rule is to separate *fairness hardening* from *difficulty tuning*. Add tests for behavior users can observe and the prompt authorizes; replay preserved solutions when only tests change; inspect actual patches and logs before reclassifying a run. If a legitimate cohort still passes too often, do not manufacture difficulty or erase valid results. Record the unresolved calibration outcome honestly.

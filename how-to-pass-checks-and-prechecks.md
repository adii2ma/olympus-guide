# How to Pass Checks and Prechecks

This is an internal task-authoring guide. Keep it outside every challenge package. It complements [agent-2.md](agent-2.md), which explains the full creation and review workflow. This guide focuses on the mechanical and quality gates that repeatedly failed while building the transactional writable CSV task.

## 1. Start with the exact package shape

An Olympus package should contain only:

```text
challenges/<task>/
  README.md
  Dockerfile
  test.patch
  solution.patch
  upstream/
```

Keep internal notes, agent archives, XML, logs, extracted patches, and these guides outside that directory. Before every upload, list the package root and remove accidental files.

The upstream checkout should be pinned and clean. Record the commit hash before writing the task. Recheck public issues, pull requests, commits, branches, forks, and releases immediately before agent runs and submission. A public implementation of the core task is a terminal problem, not something tests can repair.

## 2. Pass the repository and scope gates first

Do not invest in a task until all of these are true:

- the repository is public, active, production-grade, and permissively licensed;
- the repository meets the current language, star, and recent-activity requirements;
- the feature fits the project's philosophy and architecture;
- no current public change already implements the same behavior;
- the work requires substantive production code rather than an obvious local copy;
- a maintainer could plausibly accept the approach;
- the reference and legitimate agent solutions can meet the current effective file, message, and implementation-line requirements without padding.

Read the upstream README, contribution guide, touched subsystem, adjacent tests, CI workflow, release notes, and extension points. The CSV task needed more than reading `ext/csv`: transaction callback behavior in `vtab.go`, affinity conversion, filesystem aliases, and the existing streaming cursor all affected the contract.

## 3. Make the description pass quality checks

Write the description as a natural maintainer request, not a test inventory. State observable behavior, state transitions, errors, compatibility, and important boundaries. Do not name private helpers, files to create, data structures, algorithms, or callback sequencing unless that sequencing is itself a public contract.

For every material assertion, record one fairness source:

1. Prompt-stated behavior.
2. Repository-discoverable behavior.
3. Stable external semantics.

If none applies, remove the assertion or state the behavior in the description. Do not use the reference solution as evidence that a solver could discover a rule.

Run these description checks before submission:

- ASCII-only scan if the platform requires ASCII. The CSV task failed because one em dash remained.
- No setup sentence that the repository already makes obvious.
- No repeated reader behavior unless the new feature changes or must preserve it.
- No exact internal policy that the tests do not exercise.
- No broad list of variants when tests cover only one variant.
- No implementation explanation disguised as an acceptance requirement.
- No platform, grading, hidden-test, solver, or reviewer language.
- No test names, fixture values, or reference-only terminology.
- Every edge that hidden tests distinguish is stated or clearly discoverable.

The CSV description initially over-specified symlink aliases and exact layout mismatch dimensions beyond the tests. It also repeated existing affinity and reader normalization behavior. The fix was to keep only load-bearing public behavior, then add explicit wording only when an exact physical property was intentionally tested.

## 4. Build a clean test patch

`test.patch` must be a valid unified diff containing only tests, fixtures required by those tests, and an executable root-level `test.sh`. It must not contain production code or environment setup.

Use unique helper and test identifiers when hidden tests share a Go package namespace. The first CSV verifier collided with a common helper added by a legitimate agent test. A short neutral suffix such as `C7A91` prevents namespace collisions without introducing benchmark language.

Before accepting a test, ask:

- Does it use public behavior rather than private state?
- Can a different correct implementation pass?
- Does it avoid exact error text unless the text is public API?
- Is the setup deterministic?
- Does it avoid sleeps, polling windows, map order, directory order, and current time?
- Does it clean up all files and callbacks?
- Does it work offline and in a fresh temporary directory?
- Does the assertion detect a named realistic wrong implementation?
- Is the behavior stated, discoverable, or standard?

The CSV task exposed several fairness traps:

- Requiring a symlink object to survive replacement was unfair until symlink-target behavior became an explicit contract.
- Requiring malformed input validation only on the first write was unfair because eager validation at table creation was also valid. The corrected helper accepted failure at either point.
- Requiring a backup to contain original bytes at a specific commit-hook phase pinned an internal callback split. That assertion was removed.
- Requiring exact embedded newline spelling was unfair until the description explicitly promised lexical preservation.
- A 128-file polling test assumed ascending publication order and a visible timing window. It was removed and replaced with a synchronous public commit-hook scenario.

## 5. Make `test.sh` satisfy the exact contract

The runner must support exactly:

```bash
./test.sh --output_path results.xml base
./test.sh --output_path results.xml new
```

Both modes must always write useful JUnit XML for a valid invocation. Base runs a meaningful adjacent upstream regression slice. New runs only the newly added tests. The modes must not silently overlap or default.

For Go, prefer an explicit `-run` allowlist for base tests and a unique prefix for new tests. Do not rely on `go test -skip`; the platform precheck rejected that runner. The CSV runner used:

```bash
if [[ "$mode" == base ]]; then
    pattern='^(Test_uintArg|Test_boolArg|Test_runeArg|Test_getSchema|TestRegister|TestAffinity|TestRegister_errors|Example)$'
else
    pattern='^TestCSVWritableC7A91'
fi
```

Use `set -uo pipefail`, not a design that can exit before XML is emitted. Capture the `go test` status, run the offline JUnit converter, print the raw log, and return the test failure before the reporter status when tests fail.

The runner must also:

- reject missing, unknown, or multiple modes;
- reject a repeated or valueless `--output_path`;
- avoid installing dependencies;
- avoid network access;
- avoid fail-fast selection that hides later failures;
- preserve compiler, panic, assertion, stdout, and stderr details;
- run from `/app`;
- select at least one test in each mode.

Inspect the generated XML. A zero exit code or green console line is not enough.

## 6. Respect fail-to-pass classification

Every top-level test selected by `new` must fail, error, or skip on the base repository. The CSV task once added a standalone read-only streaming regression that correctly passed on base, so Verify Solution rejected the package.

There are two valid fixes:

- put an unchanged-behavior regression in the base slice; or
- include the regression inside a feature-dependent new test whose overall top-level test fails without the feature.

Do not weaken a real regression merely to satisfy classification. Place it in the correct mode.

## 7. Build a reproducible Dockerfile

The Dockerfile should:

- use the approved language base image;
- use `WORKDIR /app`;
- copy dependency manifests before sources;
- download every dependency and install pinned test tools during build;
- copy the repository without cloning it;
- build the relevant repository scope, preferably `go build -buildvcs=false ./...` for Go;
- avoid running tests or applying patches during build;
- leave an interactive `/bin/bash` command;
- support all runtime checks with `--network none`.

The CSV Dockerfile originally built only `./ext/csv`, which caused a cache-coverage warning. Building `./...` resolved it. A late `ENV GOPROXY=off` also drew a warning because it did not affect earlier commands and could confuse offline behavior. Do not set it unless the build has a deliberate, verified reason.

Remember that login shells can reset `PATH`. Validate using the same command form the platform uses. The installed `go-junit-report` was visible in a normal container shell but disappeared under an unnecessary `bash -lc` in one local reproduction. That was a reproduction-command issue, not a package defect.

## 8. Validate the four artifact states

Always test the exact submitted artifacts from a clean pinned checkout:

| State | Base mode | New mode |
| --- | --- | --- |
| `test.patch` only | pass | fail for missing feature |
| `test.patch` plus `solution.patch` | pass | pass |

For the final CSV package, the exact Docker matrix was:

- base repository, base mode: 38 passed;
- base repository, new mode: all 12 top-level tests failed;
- reference solution, base mode: 38 passed;
- reference solution, new mode: all 12 passed.

Also run:

```bash
git apply --check test.patch
git apply --check solution.patch
go test ./ext/csv
go test -race ./ext/csv -run '^TestCSVWritableC7A91'
go vet ./ext/csv
```

Adapt production checks to the repository. Inspect raw logs and JUnit after each run.

## 9. Audit the reference beyond the visible tests

The reference must satisfy the full description, not only the current suite. The CSV reference repeatedly passed tests while external review found real defects:

- one-column NULL or empty records disappeared as blank lines;
- multi-file publication could expose partial state;
- directory syncing was not Windows-safe;
- same-path replacement was keyed by pathname rather than identity;
- publication identity updates had a race;
- a savepoint created before the first CSV write could not roll back;
- numeric-affinity no-ops rewrote `01` as `1`;
- rollback could overwrite a newer external replacement;
- CRLF inside unchanged quoted fields could be normalized.

Perform an adversarial code review after the reference passes. Trace failure cleanup, callback errors, alias identity changes, outer rollback, nested savepoints, no-op behavior, cross-platform filesystem calls, and existing read-only paths.

## 10. Reconcile every review revision

Keep a ledger with these states:

- resolved;
- partially resolved;
- unresolved;
- regressed;
- superseded.

Before editing, copy every issue from the previous review. After editing, prove the current artifact resolves it. Separate new findings from old ones. This prevents stale feedback and accidental regressions while fixing a later issue.

## 11. Final precheck checklist

### Package

- [ ] Only `README.md`, `Dockerfile`, `test.patch`, `solution.patch`, and `upstream/` are present.
- [ ] Upstream is pinned and clean.
- [ ] Both patches apply independently and in the documented order.
- [ ] No logs, XML, archives, caches, or internal notes are included.

### Description

- [ ] Natural maintainer prose.
- [ ] ASCII-only when required.
- [ ] Every non-discoverable assertion is stated.
- [ ] No implementation plan, solution leak, or platform language.
- [ ] No redundant repository-obvious setup.

### Tests and runner

- [ ] Test patch contains no solution code.
- [ ] `test.sh` is executable.
- [ ] Base and new modes are strict and disjoint.
- [ ] Base is a meaningful regression slice.
- [ ] Every new top-level test fails on base.
- [ ] JUnit contains real failure details.
- [ ] No timing, ordering, network, or machine assumptions.
- [ ] Every assertion has a fairness source.

### Docker and solution

- [ ] All runtime dependencies are installed during build.
- [ ] Runtime works with `--network none`.
- [ ] Repository-wide build or the justified target set succeeds.
- [ ] Reference passes format, vet/lint, build, race, base, and new checks.
- [ ] Reference changes only required production files.

### Final evidence

- [ ] Four-state matrix verified from exact artifacts.
- [ ] Actual logs and XML inspected.
- [ ] Agent failures classified by cause.
- [ ] Every passing agent manually audited.
- [ ] False Positive Check passes.
- [ ] Public-overlap search repeated.

Passing checks is not a clerical final step. The checks measure whether the task is fair, reproducible, and capable of distinguishing a complete implementation from a convincing partial one.

## 12. Additional opinions from lifecycle-heavy Go tasks

The safest release process treats the files that will be pasted or uploaded as the source of truth. A clean working tree is useful, but it does not prove that a saved patch is current, valid, or free of duplicated hunks. After every revision, apply the saved `test.patch` and `solution.patch` to a fresh checkout and validate that reconstructed tree. This catches stale patch files, accidental splice corruption, missing file modes, and changes that existed only in the author's working directory.

Run cheap static gates before spending time on agent calibration. In particular:

- parse both patches with Git rather than judging their appearance;
- list every changed path and confirm that test and production changes are separated;
- enumerate the tests selected by each runner mode instead of trusting a regular expression;
- run the repository's pinned formatter and linter, not only the compiler or `go vet`;
- scan added comments and identifiers for platform-oriented wording;
- inspect the actual generated JUnit for a compile failure, assertion failure, panic, skip, and success.

Language-specific CI matters. A Go implementation can pass `go test` and `go vet` while failing `golangci-lint` through `ineffassign`, `revive`, formatting, or import checks. Read the repository workflow and run the same major version and configuration. Test-only helper packages are part of that lint surface when the final tree includes them.

Treat category selection and repository metadata as semantic checks. Adding new exported types, functions, options, and lifecycle behavior is a feature request even when it improves an existing subsystem. Do not choose enhancement merely because the change feels incremental. Likewise, record the exact repository URL, issue URL when genuine, and pinned commit from the clean checkout rather than reconstructing them later.

Make build-failure classification explicit. When new tests cannot compile on the base because the requested API is absent, the wrapper should turn that package build failure into the complete declared fail-to-pass set. An unclassified `package.[build failed]` record can fail verification even when every named feature test is otherwise represented correctly. Keep a manifest of expected top-level tests and subtests, and update it whenever coverage changes.

One defective test should not erase the rest of the report. Every blocking concurrency test needs bounded waits and cleanup that releases held filters, updates, gates, or task functions on failure. A suite-wide timeout that reports forty missing tests after one deadlock gives poor authoring evidence and can be mistaken for broad task impossibility.

Precheck feedback must be reconciled against the current artifact. If a reviewer describes code that is no longer present, compare its cited lines with the saved patch and the platform version before editing. Re-uploading a current patch may be the correct fix. If an equivalent refactor makes the intended behavior clearer to static review without harming quality, it can still be worthwhile, but do not introduce semantic churn to answer stale evidence.

A useful final order is: static validator, clean patch application, test-only base/new matrix, solution base/new matrix, full repository checks, race checks, pinned lint, offline Docker, JUnit inspection, package inventory, and one last upstream-overlap search. Calibration should start only after this sequence is green.

## 13. Redscript forwarding-diagnostic precheck case

The redscript forwarding task accumulated several preventable failures even after its behavioral tests were strong. The main lesson is that a compiler change must be checked across every layer its patch touches, not only where the new diagnostic is asserted.

The reference changed frontend lowering and IR, backend assembly, input loading, and monomorphization. A base runner that executed only frontend diagnostic tests therefore left ordinary bytecode generation unprotected. The corrected base mode ran both the existing frontend error suite and the backend bytecode regression target. This was load-bearing: a later attempt correctly retained branch-local condition prefixes for analysis but assembled every prefix before the first branch, changing program execution while all diagnostic assertions could still pass.

Several mechanical gates also produced actionable rules:

- Build paths are relative to the Docker build context. If the context is the pinned checkout copied to `/app`, use its root `Cargo.toml`; `upstream/Cargo.toml` does not exist inside that image.
- Avoid pinning a Debian package to one repository revision unless the base image and apt snapshot guarantee it. Install the capability or use the language image that already supplies it.
- Give added integration-test files a short neutral suffix when an implementer is likely to choose the obvious feature name. The suffix prevents path collisions without changing test semantics.
- Do not put an unchanged-behavior assertion in `new` as an independent top-level test. One excluded-annotation preservation test passed on the base checkout and failed Verify Solution even though it was valuable regression coverage. Move such coverage to `base`, or combine it with a feature-dependent top-level test whose overall result still fails before the solution.
- Do not repeat the supplied title as a Markdown heading in the description. Also scan for four-space indentation: it renders natural prose as a code block and can fail description formatting despite correct words.
- When production code changes backend call or control-flow assembly, include the existing backend regression suite in `base`; an empty bundle-emission smoke test is not equivalent.

For this Rust package, the final validation sequence was:

```bash
git apply --check test.patch
git apply --check solution.patch
./test.sh --output_path /tmp/new.xml new
./test.sh --output_path /tmp/base.xml base
cargo fmt --all -- --check
cargo clippy --locked --offline \
  -p redscript-compiler-frontend \
  -p redscript-compiler-backend \
  -p redscript-compiler-api \
  --all-features -- -D warnings
cargo clippy --locked --offline \
  -p redscript-compiler-api \
  --test wrapped_method_forwarding_2b3de8 -- -D warnings
```

Run `new` once with only `test.patch` and inspect the XML, not just the exit code. Every declared fail-to-pass testcase must fail, error, or skip. Then apply the solution and require every new testcase and both base regression targets to pass. Finally build the exact Docker context, recreate the upload archive from the four package artifacts, compare each archived entry byte-for-byte with its source, and record hashes. A green working tree does not prove that the uploaded archive contains the same revision.

Description-only calibration edits do not justify skipping these checks. They leave the production patch unchanged, but they make prior platform results stale and can introduce duplicate headings, code-block indentation, non-ASCII punctuation, or a stale archive. Cheap static and archive checks should always be repeated; reference tests and production gates should be repeated before declaring the revised package ready.

## 14. rust-minidump repeated-stream recovery precheck case

This task revealed two verifier failures that an ordinary green test run does not catch. First, fail-to-pass classification applies to every declared new top-level test. A malformed-Memory64 fallback test initially passed on the base because the isolated outcome was already supported. The assertion was valuable, but it had to be placed in a feature-dependent testcase or otherwise structured so the whole declared testcase required the new implementation. Always run the exact `new` manifest against a clean test-only checkout and reject any unexpected pass before uploading.

Second, a multi-command JUnit adapter must track command outcomes independently. The first runner aggregated parsed Rust test rows and added a synthetic error only when the aggregate was empty. If the library command passed and emitted rows while the stackwalk integration target failed to compile, the XML could contain only the passing library cases even though the script exited nonzero. The corrected runner captures each command's log and adds an explicit JUnit error, with the compiler or collection diagnostic, whenever that command fails without producing its expected rows.

The most reliable validation sequence was:

1. Start from the pinned clean checkout and apply only `test.patch`.
2. Run base and require the adjacent upstream slice to pass.
3. Run new and verify every declared testcase is failed, errored, or skipped in JUnit.
4. Apply `solution.patch` after `test.patch` and rerun both modes.
5. Reconstruct the same states in the exact Docker image with `--network none`.
6. Inject one temporary target-specific compile error and confirm its full diagnostic appears as a JUnit error even when another command passes; then remove the probe completely.
7. Run formatter, warning-denied Clippy, and warning-denied rustdoc checks for all touched crates.
8. List and hash the final artifacts, confirm patches apply cleanly, and verify the package contains no temporary XML, logs, build output, or unrelated files.

Review feedback must be reconciled against the current patch before making changes. In this case, preserving the already clean production patch while editing only the test patch avoided reintroducing earlier memory-selection and third-party-decoder regressions. A narrow artifact edit is valuable only after proving that description, tests, and solution remain aligned across every invocation path.

# Project Olympus Agent Instructions

These instructions apply to the entire `/Volumes/WD_NVMe/olympus` workspace.

Project Olympus submissions are challenge packages. Each package must contain a pinned upstream repository, a clear problem description, a hidden-test patch, a reference solution patch, and a Dockerfile that makes the environment reproducible. Treat the Olympus rules as mandatory, not advisory.

## Core Rule

Every challenge must be fair, deterministic, difficult, reproducible, and aligned with the upstream project. Do not submit or finalize work unless it satisfies all requirements in this file.

## Reviewer Assessment Mode

When the user asks for Olympus or Shipd reviewer-assessment work, switch from challenge-creation mode to reviewer mode. The deliverable is a platform-ready review verdict, not a challenge package, unless the user explicitly asks to create or edit a submission. Mars is a retired historical track and must not be used for current criteria or downgrade decisions.

### Non-Negotiable Review Discipline

Do not approve from a summary, a green platform check, a manager/user claim, or one successful run. A clean review means you personally inspected every material artifact: prompt, criteria panel, Dockerfile, `test.sh`, full `test.patch`, full `solution.patch`, QA report, relevant agent runs, solved and failed trajectories, JUnit XML, test logs, solution diffs, related submissions, upstream PRs/issues/commits/releases, and production-check evidence. If you have not gone through the decisive artifacts closely enough to defend the verdict line by line, do not approve.

Use a double-read standard for approvals. First read for field scoring against every active guideline (`P1`-`P6`, `T1`-`T6`, `S1`-`S4`). Then read again looking only for reasons a manager would revert: missed upstream overlap, repo-fit doubts, hidden/private implementation coupling, weak edge coverage, unrun tests, stale generated artifacts, production-check failures, low effective LOC, too-high solve rate, padding, comments/meta wording, public-extension compatibility breaks, and solution bugs outside the visible tests. If time is too short to do both passes, request changes or defer the review.

Use a single-review completeness standard. Maintain an issue ledger while reading and, before submitting the verdict, consolidate every independently discoverable problem into that review: prompt ambiguity or prescriptiveness, fairness gaps, missing or weak tests, harness and JUnit defects, Docker/environment problems, solution bugs, regressions, production-check failures, upstream overlap, repo-fit concerns, scope/LOC issues, comments/meta artifacts, and smaller nits. Finding the first decisive blocker does not end the audit; continue the remaining low-cost static and artifact checks and perform the adversarial second pass so the author receives all actionable findings at once. Do not knowingly save an issue for a later revision or reveal issues one by one across resubmissions. This prevents avoidable back-and-forth, repeated checks, wasted time and tokens, and frustration for both reviewer and author. A clearly invalid duplicate or out-of-scope task still does not require expensive reproduction, but the rejection must include all issues already visible from the prompt, patches, checks, and upstream evidence inspected.

For every resubmission, perform a mandatory version-reconciliation pass before scoring it. Copy every finding from the immediately previous review into a ledger and classify it as `resolved`, `partially resolved`, `unresolved`, `regressed`, or `superseded`, with current-version evidence for the classification. Never repeat a resolved finding as though it still exists, reuse stale wording from an older patch, or penalize the author twice for the same fixed defect. If only part of a finding remains, state exactly what was fixed and narrow the new feedback to the remaining behavior. Keep genuinely new findings in a separate section and explain why they are new, such as a regression introduced by the revision, a new interaction created by changed code, or evidence that was unavailable before. Re-open a resolved finding only with concrete proof that the current version still fails it. Before sending the verdict, compare every sentence against the current files and publish a short resolved/unresolved/new summary when there was a prior review. This gate is mandatory: stale or repeated feedback wastes author work, reruns, time, and tokens, creates avoidable frustration, and damages trust in the review process.

Upstream context must be checked deeply, not superficially. Read the relevant upstream README/docs/contributing style, touched files, adjacent existing tests, public issues, open PRs, recently merged PRs, commits, and release notes. Search multiple phrasings of the feature/bug, not just the challenge title. If an open PR implements the behavior, an issue says the feature is out of scope, or the solution architecture would not plausibly pass maintainer review, do not approve.

Comments are a hard gate. Inspect every added comment in the prompt-adjacent artifacts, tests, `test.sh`, Dockerfile, fixtures, and solution. Request changes for any comment, build tag, filename, string, or doc block that mentions hidden tests, challenge, prompt, submission, solver, Shipd, Olympus, Mars, reviewers, grading, anti-cheating, or why a benchmark/test exists. A short hash or unique suffix in a test filename is not a blocker by itself when it is used to avoid clashing with upstream files and the rest of the name looks repo-native. Also request changes for synthetic or AI-style explanatory comments, tutorial banners, redundant docstrings, TODOs, and comments that restate obvious code. Only short repo-native comments explaining genuinely non-obvious logic are acceptable.

Before reviewing a submission:

- Read `olympus-mars-reviews/REVIEWS.md`.
- Read `olympus-mars-reviews/manager-reversions-and-feedback.md` for the current manager-reversion calibration. Treat those reversions as hard gates, not optional style preferences.
- Go through everything in detail before giving a verdict. Do not approve from summaries, headlines, aggregate scores, or a single passing run. Read the prompt, criteria panel, Dockerfile, `test.sh`, full `test.patch`, full `solution.patch`, QA report, agent run artifacts, solved and failed evals, test logs, solution diffs, related submissions, and upstream public context closely enough to defend every field score and reason.
- Confirm the submission type first. Olympus is the only current track in this workspace; Cipher and other programs can still have separate criteria. Treat any Mars label as historical metadata.
- Approve only when the submission is genuinely clean. If you notice any unresolved issue, tiny harness wrinkle, stale check, stray challenge/meta comment, unrun edited test, weak edge coverage, questionable upstream fit, or "probably fine" concern, request changes instead of approving with a note.
- Treat the active "Submission criteria" panel on the platform as authoritative. Effective July 11, 2026, Olympus allows at most a 40% agent pass rate and requires successful-agent medians of at least 2 changed files, 40 messages, and 250 effective implementation LOC. The payout range is $150-$350, with final payout based on scope, difficulty, fairness, and overall quality.
- Count only effective implementation work toward LOC/file bars. Do not count tests, docs, generated churn, comments, formatting, or unrelated padding.
- When estimating effective implementation LOC, count ordinary source code. Import/`use` lines, braces, punctuation-only lines, attributes, and short control lines are normal code and they count; do not strip them or count only "core logic." Exclude blank lines, comment-only lines, generated files, test files, and artificial inflation such as dead helpers, untested scope creep, duplicated wiring, or boilerplate added only to clear the bar. The decisive scope metric is the successful-agent median, which must be at least 250 effective LOC under the current criteria. Compare the reference against those passing-agent diffs; extra unneeded reference code is padding, not scope.
- Never approve a submission that misses the active effective-LOC or file-count floor, even by a small amount and even when the implementation is otherwise solid. If it is just under the bar, request substantive scope, not padding.
- Use the current platform scoring strictly. The old 21-item Yes/No checklist is retired. Reviews now score exactly three fields: Problem Description, Tests, and Solution & Code.
- Each field is rated 0-3 in the UI order `0 Failing`, `1 Weak`, `2 Minor`, `3 Clean`. Hover over each rating on the platform and apply the active tooltip text before selecting it.
- When giving a review verdict, always include the challenge title/name next to the verdict.
- When giving a review verdict, always include the three field scores, the confidence for each field, and a clear reason for any field scored below 3.
- Match the current platform form. Do not write one large generic author-feedback block unless the UI explicitly asks for it. Put short author-facing reasons in the affected field's required reason box, leave Clean fields without a reason, and use Other Notes only for author-visible issues outside the three scored fields.
- Field reason boxes should be brief, plain, and actionable: say what kept that field below 3 and what needs changing. Do not include private reviewer logistics, raw trace details, long internal evidence, or broad review narration in those boxes.
- Reference the active guideline IDs in reviewer reasoning: Problem Description uses `P1`-`P6`, Tests uses `T1`-`T6`, and Solution & Code uses `S1`-`S4`. If an issue is not covered by the current guidelines, note it separately and tell the user it should be raised for docs coverage.
- Keep Other Notes only for issues outside the three scored fields, such as LOC, difficulty/scope, repo fit, AI slop, duplicate/overlap risk, already-solved public PR/issue risk, or track-fit concerns. Other Notes are author-visible; tags are internal only.
- Do not use a Downgrade to Mars control or recommend Mars. Mars has been merged into Olympus and is no longer an active fallback.
- The False Positive Check is mandatory. Run it only after all agent runs are complete, inspect every passing agent it evaluates, and confirm each one genuinely satisfies the full prompt rather than merely passing the tests. Any flagged false positive is evidence of a test gap and blocks submission or approval until the tests are strengthened, runs are repeated as needed, and the final FP Check passes.
- If the submission should be rejected, do not spend the full review doing unnecessary reproduction. Write clear author-facing feedback, internal reasoning, and move on.
- Keep author-facing field reasons and Other Notes separate from internal reasoning. Internal reasoning is for our own reference and should include guideline IDs, evidence, agent-run findings, local reproduction notes, and confidence justification.
- Use agent and check summaries only as leads. Read the decisive agent run artifacts and verify failures against the actual trajectory, judge trace, eval result, test log, and solution diff before attributing pass/fail causes.
- Always inspect actual agent-run JUnit XML and test logs before turning a local harness suspicion into author feedback. If a local base image is missing a helper such as `go-junit-report` but the platform's agent-run artifacts show rich per-test JUnit with real failure bodies from the wrapper, treat the missing helper as an internal reproduction note rather than an author blocker. Request changes only when the actual submitted/platform path produces generic or missing failure details.
- Check whether an open PR, merged PR, issue, commit, or release already solves the exact task or states the task is out of scope. That is a major rejection signal.
- Check whether the approach would plausibly be accepted upstream, not merely whether it can pass tests. If maintainers would require a different architecture or data model for correctness, such as retaining per-accumulator revision/removal state rather than layering on existing query invalidation, do not approve the solution as mergeable.
- Do not approve tests or prompts that force private implementation-shape rules when the public/upstream behavior does not require them. Examples include exact tree fanout thresholds, rebuild/collapse algorithms, forced internal layout normalization, or dropping unrelated internal entries unless those details are part of a documented upstream contract.
- Do not approve prompts or tests that force a private helper file, helper name, or function signature when the real requirement is public behavior. For exporter/importer tasks, put the main checks through the documented converter/API surface, such as `GPX.back`, not a newly invented local helper like `geojsonToGPX`, unless that helper is itself meant to be public upstream API.
- Do not approve prompts or tests that require private plumbing or root-level helpers when the feature can be specified through public outcomes. Examples include private vault stores, target-class serialization internals, private scale maps, root-only utility signatures, or mandatory module paths that are not already public API.
- For public API additions, hidden tests must exercise the exported API directly. Local numeric aliases, copied constants, or a compile-time assertion that exists only in the reference solution do not prove that solver implementations expose the promised names or values.
- For existing public extension points, perform a compatibility-bound audit. Traits, interfaces, abstract base classes, decorators/macros, plugin hooks, builder factories, and callback APIs must not gain new required bounds, derives, methods, constructor arguments, generic constraints, marker interfaces, or protocol requirements unless the prompt explicitly makes that a public breaking change and tests cover it. If a new feature only needs comparability, hashing, serialization, ordering, async support, or other capability on the path that uses the new feature, keep that requirement local to that path rather than making all existing implementers opt in. Tests should include at least one old-style public implementer that deliberately lacks the new capability and still works for unaffected behavior.
- Be strict about implementation-coupled snapshot/rollback tasks. If the public API is search, tag search, store/get, rollback, diff, list/metadata, or transaction behavior, tests should use those surfaces instead of reading private containers such as token maps, register maps, document indexes, tag maps, stores, or `.db` fields.
- Check source/runtime artifact consistency. If a repo ships compiled `lib/*.js`, `.d.ts`, or `.js.map` files and runtime entrypoints load those files, a TypeScript/source-only solution is incomplete unless the matching compiled artifacts are included or the repo's accepted build flow regenerates them for submission.
- Check generated-output and snapshot-manifest consistency. If the repo audits build output lists, generated files, type snapshots, or manifest files, a solution that adds source files without updating the tracked generated outputs or manifests is incomplete.
- For wrapper/binding/admission-control features, inspect ordering and precedence carefully: binding before/after registration, wrapper staying on top, explicit bindings overriding option defaults, same-tick races, no result leakage while held, required error/sub metadata, and close/drain semantics for already-admitted work.
- For change-listener/observable features, check all public roots and containers: primitive root values, parent/root listeners, key-level listeners for Map/Set entries, allow/veto/dispose behavior, callback ordering, and that unrelated changes are not delayed or batched just because a before-change listener exists elsewhere.
- For cache/retention/TTL features, test rewriting a key after its old entry expires and verify expired-entry removal updates all policy metadata and expiration counters before new FIFO/LFU/LRU state is created.
- For filesystem/symlink/provenance features, test symlink-only access paths, directory aliases, physical path provenance, missing roots, broken symlinks, and symlink cycles back to ancestors. Missing and broken symlink diagnostics should not be collapsed into generic resolver errors when the public contract distinguishes them.
- For option-parity work across legacy and new writers/backends, compare the existing public semantics directly. Tests should cover formula-backed titles, transparency/alpha, option forwarding, non-default methods, decoy parent data, and observable coordinates or output, not only element counts or successful rendering.
- For math, geometry, physics, structural, numerical, or vector/component APIs, check cross-output invariants before approving. Scalar results, vector components, component bases/origins, directional projections, and "no perpendicular component" promises must be mutually consistent, especially at endpoints, squash loads, zero-capacity states, and oblique directions. Do not approve a prompt, tests, or solution that separately checks zero scalar capacity and nonzero components but never asserts that `get_moment()` or the equivalent public projection resolves from those components in the requested direction and stated basis.
- Do not approve tasks that are mostly guided refactors or obvious copies of an existing local pattern when agent pass rate is high. Pattern-matching tasks need a real added interaction, cross-feature edge case, dialect/backend-specific behavior, or nontrivial constraint that raises difficulty.
- Do not approve descriptions that read like API tables or numbered rows of function signatures, exception strings, and matching algorithms. Rewrite them as natural behavior grouped by capability, state parity rules once, and leave exact symbol lists only where public API names are genuinely required.
- For public API additions, every tested call shape must be stated in the prompt. Do not approve hidden tests that require data-last/curried overloads, builder/thunk return shapes, generic call syntax, exact thrown/failure types, or new helper API names when the prompt only states the behavior. Existing repo style can support expectations, but it cannot replace the spec when tests fail a behaviorally correct implementation.
- For concurrency/context/lifecycle features, require coverage of default and explicit contexts, parent-state changes after task creation/admission, related state fields, and negative cases such as thread/process behavior when the prompt distinguishes them.
- Treat production quality checks as part of the Solution & Code score. If the repo's normal checks such as ruff, pyright, mypy, eslint, formatting, import sorting, or unused-symbol checks fail on the submitted solution, score Solution & Code below Clean and request changes when the failure is caused by the solution, even when `test.sh` base/new passes.
- Treat unrelated patch surface as a solution blocker. Extra notebooks, demo/validation helper modules, unrequested public helpers, accidental executable file-mode changes, docs churn, and other nonessential files should be removed unless the prompt or upstream conventions clearly require them. Do not approve by saying they are harmless or by counting them toward scope.
- Treat comment-heavy, synthetic, or AI-ish solution code as solution slop. Ask for removal unless a short comment clearly matches upstream style and explains genuinely non-obvious logic.
- Pre-approval gate: before approving, explicitly ask whether the tests verify public/user-visible behavior or force the author's implementation shape. If they force private internals, private helpers, exact file/module layout, or synthetic comment-heavy code, the decision must be Request Changes even if platform checks pass and agents solve it.

Reviewer work may create notes under `olympus-mars-reviews/`. Do not add those notes inside a challenge package.

## Workspace Organization

Keep one directory per challenge, limited to submission artifacts and the pinned upstream checkout:

```text
challenges/
  <repo-or-topic-slug>/
    README.md
    Dockerfile
    test.patch
    solution.patch
    upstream/
```

Use these files consistently:

- `README.md`: final problem description only.
- `Dockerfile`: the exact Dockerfile to paste into Olympus.
- `test.patch`: unified diff containing `test.sh` and all new or modified tests.
- `solution.patch`: unified diff containing only the reference implementation.
- `upstream/`: local clone of the target repo checked out at the pinned commit.

Do not add extra `.md` files, worklogs, scratch documents, metadata files, or status markers unless the user explicitly asks for them. Keep verification notes in the conversation or in temporary files outside the package. Do not mix multiple challenge attempts in one directory. Do not leave generated logs, build artifacts, virtual environments, dependency folders, zip files, or temporary XML files in the challenge package unless they are intentionally part of the submission.

## Repository Selection

Only use upstream repositories that meet all of these requirements:

- Public GitHub repository.
- At least one commit in the last 12 months.
- At least 500 stars.
- Production-level codebase.
- Primary language is TypeScript, JavaScript, Python, Go, Rust, C++, or Java.
- Permissive open-source license accepted by Olympus.
- Optional but useful: a GitHub issue URL that describes the problem.

Before investing in a task idea:

- Verify the exact commit SHA and keep it available for the submission form.
- Confirm the repository is active and not abandoned.
- Confirm the license is permissive and accepted.
- Search open PRs, merged PRs, issues, commits, and release notes for the exact behavior. Reject the idea if an existing PR or commit already solves it.
- Confirm the task fits the project philosophy. Do not invent features that feel out of scope for the project.

## Problem Description

The problem description must be self-contained. An agent should be able to solve the task using only the upstream repo and the description.

The description must be:

- Clear, concise, and unambiguous.
- Verifiable through tests.
- Aligned with the upstream project philosophy.
- Specific about expected behavior and edge cases.
- Non-prescriptive: describe what must work, not how to implement it.
- Behavioral rather than over-described: specify externally visible API, inputs, outputs, errors, state transitions, and edge cases, but do not turn the prompt into an implementation plan, file-by-file checklist, hidden-test transcript, or exhaustive mechanical spec.
- Written like a maintainer issue in natural prose. Avoid headings, bulleted requirement inventories, and code snippets unless exact public API shape must be pinned for fairness.
- Free of solution leaks, file-by-file instructions, or hints that make the task easier than intended.

The description must not:

- Require undiscoverable behavior.
- Depend on external services or network access.
- Ask for behavior that conflicts with existing project patterns.
- Use Olympus, Mars, Shipd, or other challenge-platform wording as the upstream feature/API/domain unless the real upstream project already uses those terms. Challenge branding in problem statements, tests, fixtures, generated output, or implementation makes the task unrealistic and defeats the purpose of testing a plausible upstream change.
- Include implementation details that are only present in the reference solution.
- Leave important semantics for the agent to guess.
- Read like a numbered API inventory. Avoid listing every function signature, exception string, registry row, or algorithm branch as separate rows when flowing behavior prose would make the same requirement clear.

## Test Patch Requirements

The tests define success. They must fail against the base commit and pass after applying `solution.patch`.

Tests must be:

- Deterministic across machines and repeated runs.
- Strong enough that incomplete or inaccurate solutions fail.
- Extensive enough to cover the requested behavior and obvious edge cases.
- Fair: do not test unspecified or undiscoverable behavior.
- Realistic for the upstream repo: do not introduce Olympus, Mars, Shipd, or platform-branded names, fixtures, strings, or expected outputs unless those terms already belong to the upstream project.
- Normal-upstream in shape: do not add comments, filenames, build tags, package names, or strings that mention hidden tests, challenge, prompt, submission, solver, Shipd, Olympus, Mars, or reviewer workflow. A short hash or unique suffix in a test filename is acceptable when it prevents an upstream filename collision and the name otherwise follows repo test naming; do not treat that suffix alone as generated slop. In particular, never add `//go:build hidden`, `hidden` build-tag comments, or challenge-oriented comments to `test.patch`, `test.sh`, Dockerfile, fixtures, or tests.
- Offline: no network access is allowed at runtime.
- Focused on the task while still preserving regression coverage.
- Directly tied to changed behavior: do not edit tests that are not run by `test.sh`, and do not leave new source/build-output snapshot files outside the executed test targets.
- Behavioral: assert observable public behavior and interoperability, not private storage shape or rebuild mechanics. For example, Git notes tests should check that notes are readable/writable and interoperate with Git-compatible object-name paths, not require a private 16/17 fanout threshold, collapse rule, full-tree rebuild rule, or deletion of non-note tree entries unless upstream docs require it.
- Public API proof: when the prompt promises exported constants, methods, overloads, or public names, tests must use those exported names directly. Local numeric constants, shadow copies of symbols, or reference-solution-only compile checks do not verify the public contract.
- Backward-compatibility proof: when the solution touches an existing public extension contract, tests must include a negative-capability compatibility case. For example, if the new comparison path needs `PartialEq`, `Eq`, `Hash`, `Serialize`, ordering, async, or a new callback method, include an existing-style accumulator/plugin/subclass/implementation that lacks that capability and still compiles or works for unaffected operations. Do not silently add the new derive/bound to every fixture and call the API compatible.
- Strong on outputs, not just presence: tests that only count rendered nodes, groups, calls, or files are usually too weak. Assert the meaningful values: coordinates, non-overlap, payloads, forwarded options, alpha/transparency, formulas, stale-state cleanup, diagnostic kind, parent/root propagation, and callback ordering where those are the behavior.
- For numerical geometry, mechanics, or capacity calculations, assert the algebraic/geometric invariants linking public outputs, not only each output in isolation. Test scalar-vs-component projection, perpendicular component removal, the stated centroid/origin/basis, endpoint/squash/zero-capacity cases, and at least one oblique direction where shifted-origin or yield-weighted-origin implementations would fail.
- For snapshot or rollback features, assert restored behavior through public APIs and documented metadata. Do not require internals like token-map size, register size, per-field index objects, tag-map objects, store objects, or private backend fields unless those are already documented public contracts.
- For conversion/export tasks, assert exported files and round-trip behavior through the existing public converter entry point. Do not make most advanced tests import an unrequested private module or require an exact helper signature unless the problem is explicitly to add that helper as public API.
- For action-wrapper or escrow-like features, tests must cover public race/ordering behavior, not just the happy path: held calls must not leak primary results, abort/commit must validate key and fence, errors and subscriptions must carry required metadata, and close must wait for pending admitted work while rejecting only new admissions.
- For async/concurrency/context inheritance features, test the default context path as well as explicit context managers, mutate the parent after task creation to prove creation-time capture, include related settings that share the same rule, and test the non-inheritance path when threads or processes are supposed to differ.

Do not use timing assumptions, randomness, host ordering, live services, current dates, machine-specific paths, or anything that can vary outside the repository.

### `test.sh` Contract

Every `test.patch` must add a root-level executable `test.sh` with two modes:

```bash
./test.sh --output_path results.xml base
./test.sh --output_path results.xml new
```

Mode requirements:

- `base`: runs existing upstream tests in the task's blast radius as regression checks. It must pass before and after the solution.
- `new`: runs the new challenge tests. It must fail without the solution and pass with the solution.
- Both modes must write JUnit XML to the requested output path.
- Argument parsing should be strict when practical: prefer exactly one valid mode, reject unknown arguments and multiple modes, and avoid silently defaulting to `new` when the mode is missing. For reviewer verdicts, treat loose parsing as a nit or Other Notes when the documented platform invocations work and JUnit XML is written correctly. Escalate it to a Tests blocker only when it can hide missing output, run the wrong suite, drop XML/failure details, or when platform artifacts show validation friction.
- If `test.sh` hand-builds JUnit XML, it must preserve real failure details, including assertion messages, panic text, compiler diagnostics, or test stderr/stdout where relevant. Do not replace failures with generic messages such as `test failed`, hide failures, or fabricate passing results.
- The harness must work from `/app` inside the Docker container.
- Base mode must be a real adjacent regression slice, not a token smoke test. Include existing coverage directly around the touched APIs, storage/backends, object plumbing, serializers/writers, and fixtures. Do not select only a few unrelated root-package tests when the solution touches repository-level behavior.
- Exclusions are allowed only for existing flaky, networked, resource-heavy, or unrelated failing tests. State every exclusion and reason in the final review notes to the user, and keep the exclusion narrow enough that adjacent behavior still runs.
- Do not set global environment flags that make ordinary test commands run zero tests or silently skip broad coverage, such as a Dockerfile-level `GOFLAGS` containing `-run=^$`. Test selection belongs in `test.sh`, where it is visible and mode-specific.
- Avoid fail-fast behavior in test mode. The platform needs the complete failure set, not only the first failing assertion or package.

Do not exclude valid existing tests because the reference solution breaks them.

### Harness Hardening

When creating or modifying a verifier harness, test runner, upload bundle, or zip package, use the local `cipher-task-creator` skill even when the task is not cybersecurity-specific:

```text
/Users/ishaan/.codex/skills/cipher-task-creator/SKILL.md
```

For Olympus `test.sh`, apply the relevant hardening principles from that skill:

- Runtime checks must work from `/app`.
- The Docker image must install all dependencies at build time and run tests with `--network none`.
- The harness must not rely on network access, host paths, mutable expected-value files, or solver-writable ground truth.
- Prefer simple, deterministic runners over fragile shell pipelines.
- Always validate from the artifact that will be submitted. For Olympus, this means testing the exact `Dockerfile`, `test.patch`, and `solution.patch`; for Harbor/Cipher upload flows, validate the actual zip files.

Do not force Harbor-only files such as `instruction.md`, `task.toml`, `environment/`, `tests/`, `solution/`, `reward.txt`, or `solve.sh` into an Olympus package unless the user explicitly asks for a Harbor/Cipher task.

Generate `test.patch` from a clean checkout after making only test-related changes:

```bash
git diff > test.patch
git stash
git apply test.patch
git stash pop
```

## Solution Patch Requirements

The reference solution proves the task is solvable. It must satisfy the full problem description, not just the tests.

The solution must:

- Pass `test.sh base` and `test.sh new`.
- Preserve existing behavior.
- Follow upstream code patterns, style, and abstractions.
- Keep changes scoped to the task.
- Avoid challenge-platform branding in implementation, public APIs, errors, comments, fixtures, or tests unless the upstream project itself already uses that vocabulary.
- Do not tighten existing public API contracts unless the prompt explicitly requires a breaking change. Adding a trait/interface/protocol bound, required derive, abstract method, constructor parameter, generic constraint, marker type, decorator requirement, or plugin-hook obligation is a public regression when existing user implementations that do not use the new behavior would stop compiling or running.
- Avoid unrelated formatting, refactors, dependency changes, generated files, and metadata churn.
- Avoid extra public API surface. Do not add public constants, fields, methods, operations, helper modules, error types, or exported root functions that the prompt does not require and tests do not cover. Remove dead fields and unused structs even if they look like future support.
- Pass the upstream repo's normal production checks when they are part of the project workflow or platform checks, including lint, type checking, formatting, import sorting, and unused-symbol checks. A solution that passes hidden tests but fails pyright, ruff, mypy, eslint, or equivalent repo checks is not ready.
- Avoid AI-style comments, unexplained defensive code, and new patterns that do not fit the project.
- Do not add explanatory comments, TODOs, notes, or other comment filler just to make the patch look larger. Remove comments that read synthetic, redundant, or AI-generated. A new comment is acceptable only when the upstream code style would normally require it to explain genuinely non-obvious logic.
- Remove or justify every extra/unrequested artifact. Example notebooks, demo scripts, validation helper modules, extra public helper methods, accidental chmod/file-mode changes, and broad docs/example additions are not acceptable just because they fit the repo in general; they must be required by the task or clearly normal for the upstream change.
- Include all runtime files required by the upstream package layout. If source changes are mirrored into tracked build outputs, type declarations, or source maps in that repo, missing those artifacts is a regression, not harmless cleanup.
- Include generated-output manifests and snapshot lists when the repo's normal checks audit them. If `package.json`, build-output tests, or scripts require snapshot files for new source artifacts, update those snapshots as part of the solution.
- Preserve parity with existing public backends and writers. When extending a parallel writer/exporter/serializer path, copy the established user-visible semantics such as formula titles, alpha/transparency, option defaults and overrides, error classes, and metadata unless the prompt explicitly narrows the behavior.
- Satisfy cross-output invariants, not just visible scalar assertions. If a public scalar is defined from component vectors, directional resolution, or a stated coordinate basis, the scalar, components, basis/origin, and perpendicular-component semantics must agree at normal points and endpoint/singular states.
- Avoid broad fallback code that swallows real failures. Socket/env fallbacks, `except Exception: pass`, generic resolver errors, and unexplained defensive branches are solution-quality issues unless the repo already uses that pattern and tests cover it.
- Avoid duplicated core wiring across source and generated/runtime files. One option path and one setup/modifier registration should exist unless the upstream architecture explicitly requires more.
- If the passing-agent medians fall below the current Olympus scope metrics after removing comments, tests, generated noise, and dead helpers, broaden the real feature or request revision. Do not count boilerplate or invented helper surface to clear the bar.

Generate `solution.patch` from a clean checkout after applying `test.patch` and then making only solution-related changes:

```bash
git diff > solution.patch
```

Review the patch manually before considering it final.

## Dockerfile Requirements

The Dockerfile must build a complete offline runtime environment. Olympus runs containers with `--network none`, so all dependencies must be installed during image build.

Dockerfile rules:

- Start from the Olympus base image matching the repository language, or from `public.ecr.aws/x8v8d7g8/mars-base:latest` when the image still installs all needed dependencies and passes the offline build/test checks. Do not reject a submission solely for using `mars-base`.
- Use `WORKDIR /app`.
- `COPY . .`
- Install all required dependencies during the build.
- Use `/bin/bash` as the command or entrypoint.
- Work without `test.patch` or `solution.patch` applied, because Olympus applies patches after building.
- Do not run the test suite during Docker build. The Dockerfile sets up the offline environment; `test.sh` is the only runtime test entry point.
- Do not bake test selection or skip behavior into global environment variables. Build-time env such as `GOFLAGS`, `PYTEST_ADDOPTS`, or equivalent must not make ordinary repo test commands run zero tests or omit coverage unless the project already requires it and the reason is documented.
- Keep Dockerfile comments minimal and repo-neutral. Do not mention the prompt, challenge, submission, hidden tests, or reviewer workflow.

Base images:

```Dockerfile
FROM public.ecr.aws/d3j8x8q7/olympus-base-python:latest
FROM public.ecr.aws/d3j8x8q7/olympus-base-typescript:latest
FROM public.ecr.aws/d3j8x8q7/olympus-base-go:latest
FROM public.ecr.aws/d3j8x8q7/olympus-base-rust:latest
FROM public.ecr.aws/d3j8x8q7/olympus-base-jvm:latest
FROM public.ecr.aws/d3j8x8q7/olympus-base-cpp:latest
FROM public.ecr.aws/x8v8d7g8/mars-base:latest
```

Template:

```Dockerfile
FROM public.ecr.aws/d3j8x8q7/olympus-base-<language>:latest

WORKDIR /app
COPY . .

# Install all dependencies needed by base and new tests here.

CMD ["/bin/bash"]
```

## Local Review Workflow

Run the full reviewer flow locally before using platform checks:

1. Clone the upstream repo into the challenge directory.
2. Check out the exact commit SHA selected for the submission.
3. Apply `test.patch`.
4. Build the Docker image.
5. Run the container with `--network none`.
6. Run `./test.sh --output_path /tmp/base.xml base`; it must pass.
7. Run `./test.sh --output_path /tmp/new.xml new`; it must fail without the solution. Partial passing is acceptable only for backwards-compatibility checks inside the new suite.
8. Apply `solution.patch`.
9. Rebuild if the solution changes build inputs.
10. Run both modes again; both must pass.
11. Complete all agent runs, then run the mandatory False Positive Check. Inspect every passing agent and resolve every false-positive finding before treating the task as ready.
12. Re-check that no open or merged PR already solves the exact issue.
13. After the review is complete, remove temporary review artifacts created locally, including `/tmp` extraction folders, temporary upstream checkouts, one-off Docker images, and stopped containers. Keep downloaded submission archives, saved run zips, review notes, and other evidence under `olympus-mars-reviews/`.

## Submission Quality Bar

Before finalizing, confirm:

- The challenge is genuinely difficult for current strong agents.
- The active assignment criteria from the Olympus submission panel are met: pass rate at most 40%, with successful-agent medians of at least 2 files, 40 messages, and 250 effective implementation LOC. Do not pad with tests, docs, formatting churn, or unrelated edits.
- The tests are neither permissive nor unfair.
- The problem description includes all behavior needed to pass the tests.
- The reference solution meets the full description.
- Existing tests still pass unless an exclusion is documented and justified.
- The Dockerfile installs everything required before runtime.
- The package is clean, organized, and reproducible from the pinned commit.

Consult the current Olympus "Submission criteria" panel for assignment-specific bars. Those criteria can change, but when the user reports the active bar, enforce it locally and reject smaller candidates instead of trying to polish them.

## Token and Check Discipline

Be deliberate with platform checks:

- Plan and self-review before running agent attempts.
- Start with a smaller batch when checking task difficulty.
- Read check outputs and agent logs carefully.
- Fix all obvious issues in one pass before rerunning checks.
- Remember that editing submission content after checks makes those results stale and requires reruns.
- Run the mandatory False Positive Check last, after all agent runs. If it flags any passing run, strengthen the tests and rerun the affected checks and agents before submission.

If no agent can solve the task, investigate whether the description is ambiguous, the tests are unfair, or the task is underspecified before assuming the task is acceptable.

## Final Agent Checklist

Do not mark a challenge ready unless every item below is true:

- `README.md` contains a self-contained, non-prescriptive problem description.
- `test.patch` applies cleanly to the pinned commit.
- `test.patch` adds a compliant executable `test.sh`.
- `test.patch`, `test.sh`, Dockerfile, fixtures, and tests contain no hidden/challenge/submission/prompt/platform meta comments or build tags.
- A dedicated comment sweep found no benchmark/meta/AI-style comments, tutorial banners, redundant docstrings, or obvious-code explanatory comments in any submitted artifact.
- `test.sh` runs the documented `base` and `new` commands and writes requested JUnit XML. Prefer rejecting missing, duplicate, and unknown modes/arguments, but treat loose malformed-argument handling as a nit unless it can hide failures, select the wrong suite, or break XML collection.
- `test.sh base` passes before the solution.
- `test.sh base` runs a meaningful adjacent regression slice and does not rely on global flags that skip tests.
- `test.sh new` fails before the solution.
- Every edited test or snapshot target is actually run by either base or new mode.
- `solution.patch` applies cleanly after `test.patch`.
- Agent outcomes satisfy the current Olympus bars: pass rate at most 40%, and successful-agent medians of at least 2 files, 40 messages, and 250 effective implementation LOC.
- `test.sh base` passes after the solution.
- `test.sh new` passes after the solution.
- Repo production checks relevant to the touched files pass, or any unavailable checks are explicitly noted and not replaced by guesswork.
- `solution.patch` contains no unrelated notebooks, demo helpers, extra public API surface, docs churn, accidental executable mode changes, dead fields, synthetic comments, broad swallowed-error fallbacks, or padding.
- `solution.patch` does not tighten existing public extension contracts with new required bounds, derives, abstract methods, constructor args, generic constraints, or marker protocols unless explicitly required and covered by compatibility tests.
- Runtime/generated artifacts and snapshot manifests are updated when the repo requires them.
- New tests exercise public/user-visible behavior directly, including exported public symbols and option parity, rather than private helpers or local copies of constants.
- Docker builds successfully and runtime tests work with `--network none`.
- No existing PR, merged PR, commit, or release already solves the exact task.
- The feature and implementation architecture are plausible for upstream maintainers, not merely a test-passing private subsystem.
- All agent runs are complete and the mandatory final False Positive Check passes; every passing agent genuinely satisfies the full task.
- All files are organized under the correct challenge directory.

## Task observations: rust-minidump repeated-stream recovery

The repeated-stream recovery task showed that a challenge can be locally correct and still need several review revisions because its contract crosses storage, public APIs, compatibility, classification, and CLI presentation. The strongest workflow was to keep a requirement-to-invocation-path ledger and update it after every review. In this case, the ledger had to include ordered directory retention, exact-occurrence access, latest-valid legacy fallback, raw and typed decoding, recovered system information, third-party decoder compatibility, classified iterators, empty `UnusedStream` presentation, every independently dispatched `--dump` group, and the Memory64-versus-MemoryList policy.

Two compatibility rules were especially easy to miss. A concrete system-info occurrence must decode its own bytes, but a generic third-party `MinidumpStream` implementation using the same numeric stream type must still receive the documented recovered system-info context. Also, repeated-stream output must not change ordinary no-duplicate behavior: when both memory-list formats exist, valid Memory64 streams retain precedence rather than causing both formats to print.

The verifier needs the same path-by-path discipline as the solution. Every declared new top-level test must fail, error, or skip on the pinned base. Each cargo invocation must contribute an explicit JUnit error when it fails before producing test rows; aggregating successful rows from another invocation must not hide a compile or collection failure. Validation is complete only after reconstructing a clean checkout from the exact Dockerfile and patches, running base and new modes with and without the solution, exercising the image with `--network none`, inspecting the XML, and running formatter, Clippy, and rustdoc warning gates.

Finally, do not infer test completeness from representative branches when production dispatch is manual. For this task, distinct-stream tests did not prove duplicate retention in `all_streams`, and thread/module CLI tests did not prove unloaded modules, handles, crashpad, macOS, stability, or other independent branches. A mutation-oriented audit of every invocation path was what turned a strong suite into a complete one.

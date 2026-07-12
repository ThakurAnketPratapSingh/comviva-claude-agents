---
name: autotest
description: >
  Generate a JUnit 5 + Mockito + AssertJ test suite for a Java class, package,
  or acceptance criteria. Use when the user asks to generate/scaffold/write
  tests, e.g. "/autotest KafkaPush", "/autotest src/main/java/.../service/",
  "/autotest --diff", or "/autotest from AC: <criteria>".
---

# AutoTest — generate tests for Java code

Parse the arguments to determine the mode, then delegate generation to the
`autotest-creator` agent (one agent per target class; launch them in parallel
when there are multiple classes).

## Argument parsing

- `<ClassName>` or a path to a `.java` file → single-class mode. If given a
  bare class name, resolve it with Glob (`**/<ClassName>.java`); if ambiguous,
  list matches and ask.
- A directory path → package mode: every `.java` file in it (non-recursive
  unless the path ends with `/**`), minus files matched by `exclude_paths` in
  `.rovo-test.yml`. Classes that already have a non-empty, non-commented test
  class are NOT dropped — they become **review targets** (see Execution).
- `--diff` → run `git diff --name-only --diff-filter=d HEAD` plus
  `git diff --name-only --diff-filter=d <default-branch>...HEAD`; take the
  union of changed `src/main/java/**/*.java` files as targets (`--diff-filter=d`
  keeps deleted files from becoming targets).
- `from AC:` or a Jira key / pasted acceptance criteria → acceptance-criteria
  (test-first) mode; pass the criteria verbatim to the agent.
- `--target-coverage <n>` → override the coverage target from `.rovo-test.yml`
  for this run.

## Execution

1. Read `.rovo-test.yml` in the repo root for config (use the agent's built-in
   defaults if absent); pass relevant settings into each agent prompt along
   with the resolved absolute file path(s).
2. Split targets into two groups by checking `src/test/java` for a mirrored
   test class that is non-empty and non-commented — match `<ClassName>Test.java`,
   `<ClassName>Tests.java`, or `<ClassName>IT.java`:
   - **No existing test** → launch an `autotest-creator` agent in GENERATE
     mode with: the file path, the config values, and the instruction to
     compile and run the tests before reporting.
   - **Existing test** → launch an `autotest-creator` agent in REVIEW mode
     with: the production file path, the existing test file path, the config
     values, and the instruction to run the existing tests with coverage and
     report the measured coverage percentage vs the target — without
     rewriting the existing tests.
3. Execution strategy. Concurrent Maven/Gradle builds in the same module
   collide on the shared `target/`/`build/` directory, but WRITING test files
   is safe to parallelize — so parallelize generation and batch the builds:
   - **Single target** → one agent, which verifies itself (compile + run).
   - **Multiple generate-mode targets** → three phases:
     a. *Generate (parallel)*: launch generate-mode agents in parallel (cap
        4), each told verification is BATCHED — they write their test file
        and stop without building.
     b. *Verify (one build per module)*: run all new tests together:
        `mvn -q test -Dtest=Test1,Test2,... -DfailIfNoTests=false` with the
        agent's build-speed flags (`-pl <module> -am` in a multi-module
        reactor; different modules' builds may run in parallel).
     c. *Fix (parallel)*: for each failing test file, launch a fix-mode agent
        in parallel with that file's compile/surefire output — edit-only, no
        builds. Re-run the batch build. At most 3 fix rounds; report anything
        still failing as FAILED, never as passed.
   - **Review-mode targets** run their own builds: run them after the
     generate batch finishes, sequentially within a module (parallel across
     modules).

## Final report to the user

Aggregate the agent reports into one table: class → mode
(generated/reviewed) → test file → tests written or existing test count →
compile/run status → measured coverage % vs target. For reviewed classes,
also list the coverage gaps the agent identified (untested public methods or
branches). List any suspected production bugs the agents flagged. Do not
commit anything — leave new/modified files in the working tree for the user
to review, and remind them to review before merging.

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
  `.rovo-test.yml` and minus classes that already have a non-empty,
  non-commented test class.
- `--diff` → run `git diff --name-only HEAD` plus `git diff --name-only
  <default-branch>...HEAD`; take the union of changed `src/main/java/**/*.java`
  files as targets.
- `from AC:` or a Jira key / pasted acceptance criteria → acceptance-criteria
  (test-first) mode; pass the criteria verbatim to the agent.
- `--target-coverage <n>` → override the coverage target from `.rovo-test.yml`
  for this run.

## Execution

1. Read `.rovo-test.yml` in the repo root for config (use the agent's built-in
   defaults if absent); pass relevant settings into each agent prompt along
   with the resolved absolute file path(s).
2. For each target class, launch an `autotest-creator` agent with: the file
   path, the config values, and the instruction to compile and run the tests
   before reporting.
3. Cap parallelism at 4 agents; queue the rest.

## Final report to the user

Aggregate the agent reports into one table: class → test file → tests written
→ compile/run status → coverage vs target. List any suspected production bugs
the agents flagged. Do not commit anything — leave the new test files in the
working tree for the user to review, and remind them to review before merging.

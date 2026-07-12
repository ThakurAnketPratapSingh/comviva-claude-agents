# AutoTest Creation — Claude Code Agent

> "No more 'I'll write tests later.' Tests arrive with the code."

Generates complete **JUnit 5 + Mockito + AssertJ** test suites for Java
classes, packages, branch diffs, or acceptance criteria — then **compiles and
runs** what it generates before reporting. Inspired by the MobiLytix Rovo
AutoTest Creation agent, rebuilt for Claude Code so it works in any repo,
any IDE, any terminal.

## What's inside

| Piece | File | Role |
|---|---|---|
| Agent | `agents/autotest-creator.md` | The test engineer: reads code, writes tests, compiles, runs, reports |
| Command | `skills/autotest/SKILL.md` | The `/autotest` command: parses arguments, fans out one agent per class |

## Install

```
claude plugin marketplace add ThakurAnketPratapSingh/comviva-claude-agents
claude plugin install autotest@comviva-agents
```

## Usage

Run inside a Claude Code session in your Java project:

```
/autotest KafkaPush                          # one class (bare name is resolved)
/autotest src/main/java/.../service/         # every class in a package
/autotest --diff                             # only classes changed on your branch
/autotest from AC: Given X, when Y, then Z   # test-first: tests from a story's AC
/autotest KafkaPush --target-coverage 85     # per-run coverage override
```

Plain English works too: *"generate tests for ProcessDataRecords"*.

## What the generated tests look like

```java
@DisplayName("KafkaPush unit tests")
@ExtendWith(MockitoExtension.class)
class KafkaPushTest {

    @Mock  private KafkaTemplate<String, String> kafkaTemplate;
    @InjectMocks private KafkaPush kafkaPush;

    @Test
    @DisplayName("push() skips RESULT_CODE check when skipDefaultResponseParams is enabled")
    void push_skipDefaultResponseParamsEnabled_skipsResultCodeCheck() {
        // given
        ...
        // when
        ...
        // then
        assertThat(result).isNotNull();
        verify(kafkaTemplate).send(anyString(), anyString());
    }
}
```

Every suite follows the same rules:

- **Naming**: `methodUnderTest_scenario_expectedOutcome` method names, plus a
  plain-English `@DisplayName` on the class and every test — readable by
  non-developers in Surefire/IDE reports.
- **Structure**: `// given / when / then` blocks in every test.
- **Scope by class type**: plain Mockito unit test for services,
  `@WebMvcTest` + MockMvc for controllers, `@DataJpaTest` for repositories,
  `ApplicationContextRunner` for `@ConfigurationProperties`. `@SpringBootTest`
  only as a last resort.
- **Coverage, always**: happy path, null inputs, empty collections, boundary
  values, exception paths, and `verify(...)` on mock side effects.
- **Placement**: mirrored package under `src/test/java`, named
  `<ClassName>Test.java`.

## Guarantees

1. **It compiles and passes before it reports.** A single class is verified
   with one `mvn test -Dtest=<Class>Test` run (compile + test in one
   invocation, QA plugins skipped, `mvnd` used when available); multi-class
   runs generate all test files in parallel, then verify them together in
   one batched build per module — the agent fixes its own mistakes until
   green.
2. **It never edits production code.** If a generated test exposes a real
   bug, the test is marked `@Disabled("documents suspected bug: ...")` and
   the bug is called out in the summary. (Sole exception: if the class under
   test is missing the Comviva copyright header or `@author` javadoc, the
   agent adds them — a comment-only addition, reported in the summary.)
3. **It never commits.** Generated files sit in your working tree for review;
   you commit them with your feature branch.
4. **It skips what shouldn't be tested**: getters/setters, Lombok-generated
   code, and DTO/model packages. Classes that already have real tests are
   not regenerated — they are **reviewed** instead: the existing tests are
   run with JaCoCo, and the report shows the measured coverage percentage
   vs the target plus any coverage gaps.
5. **Every file it creates carries the Comviva copyright header** and an
   `@author` javadoc derived from your git config; it also adds these to
   touched files that are missing them.

## Configuration

Optional per-repo overrides via `.rovo-test.yml` in the repo root:

```yaml
coverage:
  target: 80          # line-coverage goal reported per class
  fail_below: 60
test_naming: method_scenario_outcome
include_edge_cases:
  - null_inputs
  - empty_collections
  - boundary_values
  - exception_paths
exclude_paths:
  - "**/model/**"
  - "**/dto/**"
  - "**/generated/**"
```

Without the file, these same values apply as defaults. The agent also detects
the project stack itself — Maven vs Gradle, Spring Boot 2.x (`javax`) vs
3.x (`jakarta`) — from the build file, so no per-repo setup is required.

## Requirements

- Claude Code installed and logged in
- Java project that builds locally (`mvn test-compile` must work — internal
  dependencies resolved from Nexus/Artifactory)
- Maven (or Gradle) on PATH

## FAQ

**Does it work test-first, before the code exists?**
Yes — `from AC:` mode writes one test per acceptance criterion as an
executable specification. It won't compile until the implementation lands;
that's the point.

**What if a generated assertion is wrong?**
Fix it in your branch like any code-review finding. If it's a recurring
pattern, refine `agents/autotest-creator.md` in this repo so the whole team
benefits.

**Which classes does `--diff` pick up?**
The union of `git diff HEAD` and `git diff <default-branch>...HEAD`,
filtered to `src/main/java/**/*.java`.

**Can it modify existing tests?**
No. A class that already has a non-empty test class goes through review mode
instead: the existing tests are run with coverage, and you get the measured
percentage, pass/fail status, and a gap list — the only change ever made to
an existing test file is adding a missing copyright header or `@author`
javadoc. Regeneration requires deleting the old test file first.

# Comviva Claude Code Agents

Internal Claude Code plugin marketplace for MobiLytix teams.

**Repo URL:** `https://github.com/ThakurAnketPratapSingh/comviva-claude-agents.git`

Current plugins:

| Plugin | What it does |
|---|---|
| `autotest` | Generates JUnit 5 + Mockito + AssertJ test suites for Java classes, packages, git diffs, or acceptance criteria - then compiles and runs them before reporting. |

---

## Team member guide - step by step

### Prerequisites (once per machine)

1. **Install Claude Code** if you don't have it:
   ```
   npm install -g @anthropic-ai/claude-code
   ```
   Then run `claude` once in any folder and log in when prompted.
2. **Check repo access** - you need read access to this GitHub repo. Verify with:
   ```
   git ls-remote https://github.com/ThakurAnketPratapSingh/comviva-claude-agents.git
   ```
   If it lists refs, you're good. If it asks for credentials, use your GitHub
   username and a personal access token - git will cache them.
3. **Maven on PATH** - the agent runs `mvn test-compile` and `mvn test`.
   Verify with `mvn -v`. Your project must build locally (internal
   dependencies like `Encryption-Utils` must resolve from Nexus/Artifactory).

### Step 1 - Add the marketplace (once per machine)

Open a terminal (any folder) and run:

```
claude plugin marketplace add ThakurAnketPratapSingh/comviva-claude-agents
```

Or, inside a running Claude Code session, type:

```
/plugin marketplace add ThakurAnketPratapSingh/comviva-claude-agents
```

(The full git URL also works in place of the `owner/repo` shorthand.)

### Step 2 - Install the autotest plugin (once per machine)

```
claude plugin install autotest@comviva-agents
```

(or `/plugin install autotest@comviva-agents` inside a session).

### Step 3 - Verify the install

1. Open a terminal **in your Java project** (e.g. the eventprocessor repo):
   ```
   cd D:\path\to\your\project
   claude
   ```
2. Type `/` - you should see `autotest` in the command list.
3. Type `/plugin` -> Manage plugins -> confirm `autotest` shows as enabled.

### Step 4 - Generate tests

From inside a Claude Code session in your project, pick the mode that fits:

| You want to... | Type |
|---|---|
| Test one class | `/autotest KafkaPush` (bare class name is fine - it finds the file) |
| Test a whole package | `/autotest src/main/java/com/comviva/mrtm/ig/eventprocessor/service/` |
| Test only what you changed on your branch | `/autotest --diff` |
| Write tests BEFORE code, from a story | `/autotest from AC: Given X, when Y, then Z` |
| Override the coverage target for one run | `/autotest KafkaPush --target-coverage 85` |

Plain English also works: just type *"generate tests for ProcessDataRecords"*.

### Step 5 - What happens during the run

1. The agent reads the target class and every collaborator it references.
2. It writes `<ClassName>Test.java` into the mirrored package under
   `src/test/java` - JUnit 5 + Mockito + AssertJ, `@DisplayName` on every
   test, `methodUnderTest_scenario_expectedOutcome` naming, given/when/then
   structure, covering happy path, nulls, empty collections, boundary values,
   and exception paths.
3. It compiles (`mvn test-compile`) and runs (`mvn test -Dtest=...`) the new
   tests, fixing its own mistakes until green. It **never edits production
   code** - if a test exposes a real bug, the test is marked
   `@Disabled("documents suspected bug: ...")` and the bug is reported to you.
4. You get a summary: files created, tests by category, compile/run status,
   coverage vs target, suspected bugs.

### Step 6 - Review and commit (you, not the agent)

Nothing is committed automatically. The generated test files sit in your
working tree:

1. **Read the tests.** Understand every assertion - don't merge blind.
2. Adjust anything domain-specific the agent couldn't know.
3. Commit them on your feature branch and raise your MR as usual.

### Getting updates

When this repo changes (new agent versions, new plugins):

```
claude plugin marketplace update comviva-agents
```

---

## Auto-install for a whole project (recommended for team leads)

To make the plugin install automatically for everyone who opens a project,
commit this to that project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "comviva-agents": {
      "source": {
        "source": "github",
        "repo": "ThakurAnketPratapSingh/comviva-claude-agents"
      }
    }
  },
  "enabledPlugins": { "autotest@comviva-agents": true }
}
```

Teammates get prompted to trust and install on their next Claude Code session
in that repo - no manual steps.

---

## Per-repo configuration (optional)

Drop a `.rovo-test.yml` in the target repo's root to override defaults:

```yaml
coverage:
  target: 80
  fail_below: 60
test_naming: method_scenario_outcome
include_edge_cases:
  - null_inputs
  - empty_collections
  - boundary_values
  - exception_paths
exclude_paths:
  - "**/model/**"
  - "**/generated/**"
```

Without the file, the agent uses those same values as defaults.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `/autotest` not in the command list | Restart the Claude Code session (plugins load at startup); check `/plugin` shows it enabled |
| Marketplace add fails with auth error | Run `git ls-remote <repo-url>` first so git caches your GitHub credentials |
| `mvn test-compile` fails on internal deps | Your project must build locally first - fix `settings.xml` / Nexus access, then re-run |
| Agent generated a wrong assertion | Fix it and mention it in your MR; refine `agents/autotest-creator.md` here if it's a recurring pattern |

---

## Adding the next agent

1. Create `plugins/<name>/` with `.claude-plugin/plugin.json`, plus `agents/`
   and/or `skills/` directories.
2. Add an entry to `.claude-plugin/marketplace.json`.
3. Commit and push - the team picks it up on marketplace update.

Planned next: review-agent, bug-completion, epic-breakdown (mirroring the
MobiLytix Rovo agent fleet).

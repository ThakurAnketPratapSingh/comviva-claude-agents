# Comviva Claude Code Agents

Internal Claude Code plugin marketplace for MobiLytix teams. Current plugins:

| Plugin | What it does |
|---|---|
| `autotest` | Generates JUnit 5 + Mockito + AssertJ test suites for Java classes, packages, git diffs, or acceptance criteria — then compiles and runs them before reporting. |

## Install (each team member, one time)

From any terminal with Claude Code installed:

```bash
claude plugin marketplace add <git-url-of-this-repo>
claude plugin install autotest@comviva-agents
```

Or inside a Claude Code session:

```
/plugin marketplace add <git-url-of-this-repo>
/plugin install autotest@comviva-agents
```

Updates: when this repo changes, `claude plugin marketplace update comviva-agents`
(or reinstall) picks up the new version.

## Auto-install for a whole repo (recommended)

To make the plugin available automatically to everyone who clones a project,
commit this to that project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "comviva-agents": {
      "source": { "source": "git", "url": "<git-url-of-this-repo>" }
    }
  },
  "enabledPlugins": { "autotest@comviva-agents": true }
}
```

Teammates get prompted to trust and install on their next session in that repo.

## Usage

```
/autotest KafkaPush                          # one class (name is resolved for you)
/autotest src/main/java/.../service/         # a package
/autotest --diff                             # only classes changed on your branch
/autotest from AC: <acceptance criteria>     # test-first mode from a story
/autotest KafkaPush --target-coverage 85     # per-run coverage override
```

You can also just ask in plain English ("generate tests for ProcessDataRecords")
— the `autotest-creator` agent triggers on that too.

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

## Adding the next agent

1. Create `plugins/<name>/` with `.claude-plugin/plugin.json`, plus `agents/`
   and/or `skills/` directories.
2. Add an entry to `.claude-plugin/marketplace.json`.
3. Commit and push — the team picks it up on marketplace update.

Planned next: review-agent, bug-completion, epic-breakdown (mirroring the
MobiLytix Rovo agent fleet).

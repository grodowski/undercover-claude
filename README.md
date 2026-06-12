# undercover

A Claude Code plugin that creates a test coverage feedback loop using the [undercover](https://github.com/grodowski/undercover) gem.

## What it does

After running tests, Claude automatically runs `undercover` to find untested code changes, then writes tests for any gaps, looping until coverage passes.

Unlike total coverage percentages, undercover checks **only the lines you changed**, so the feedback is always relevant to the work at hand.

## Installation

**From marketplace:**
```
/plugin marketplace add grodowski/undercover-claude
/plugin install undercover@undercover-ci
```

**From local clone (development):**
```
/plugin marketplace add /path/to/undercover-claude
/plugin install undercover@undercover-ci
```

## Prerequisites

Your Ruby project needs SimpleCov configured to generate `coverage/coverage.json`. You don't have to wire this up by hand — let the skill do it.

### Quick Setup

Inside Claude Code, run the setup prompt:

```
/coverage set up coverage for this project
```

Claude adds `simplecov` and `undercover` to your Gemfile, wires `SimpleCov::Formatter::Undercover` into `spec_helper.rb`/`test_helper.rb`, creates `.undercover` with a compare ref, runs `bundle install`, and runs your tests once to generate coverage data — then you're ready for the feedback loop.

## How It Works

The plugin installs a skill that instructs Claude to run `undercover` after every test run and act on the results:

1. Run tests
2. Run `undercover` or `undercover --include-files "app/models/foo.rb"` when testing a subset of files
3. If gaps found: write tests (or remove dead code)
4. Repeat until undercover exits 0

## Usage

Invoke `/coverage` inside Claude Code. Type what you want to do after
the command and the skill routes to the right guide (it defaults to the feedback
loop if you say nothing):

- `/coverage set up coverage` — SimpleCov setup instructions
- `/coverage set up CI` — CI with UndercoverCI or GitHub Actions
- `/coverage parallel tests` — coverage merging for parallel runners
- `/coverage check this PR` — read and address UndercoverCI feedback on a PR

### Feature branches and CI

By default undercover checks uncommitted changes only. For feature branches and CI, set a compare ref in `.undercover` or use the `--compare` flag:

```
-c origin/main
```

### Addressing UndercoverCI feedback on a PR

When coverage is reported on a GitHub PR by [UndercoverCI](https://undercover-ci.com),
the warnings live in the `coverage` check's annotations — which `gh pr view` and the
GitHub MCP server don't expose. The skill instructs Claude to pull them directly with
`gh api`, then fix the gaps and verify locally with `--compare`.

## Files

```
undercover-claude/
├── .claude-plugin/
│   ├── plugin.json           # Plugin manifest
│   └── marketplace.json      # Marketplace catalog (self-hosting)
├── skills/
│   └── coverage/
│       ├── SKILL.md          # Feedback loop + router (loads on every invocation)
│       └── references/       # Guides loaded on demand
│           ├── setup.md      # SimpleCov + Gemfile + .undercover + permissions
│           ├── ci.md         # GitHub Actions / CircleCI / UndercoverCI / merging
│           ├── pr-feedback.md # Reading UndercoverCI PR annotations via gh api
│           └── skipping.md   # :nocov:, add_filter, --exclude-files
└── README.md
```

## License

MIT

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

Your Ruby project needs SimpleCov configured to generate `coverage/coverage.json`. Use the `/undercover:coverage` skill inside Claude Code for setup instructions.

### Quick Setup

1. Add to your Gemfile:

```ruby
group :test do
  gem 'simplecov'
  gem 'undercover'
end
```

2. Add to the top of `spec_helper.rb` or `test_helper.rb`:

```ruby
require 'simplecov'
require 'undercover/simplecov_formatter'

SimpleCov.formatter = SimpleCov::Formatter::Undercover

SimpleCov.start do
  add_filter(/^\/spec\//)  # For RSpec
  add_filter(/^\/test\//)  # For Minitest
  enable_coverage(:branch)
end
```

3. Run `bundle install` and run your tests once to generate coverage data.

## How It Works

The plugin installs a skill that instructs Claude to run `undercover` after every test run and act on the results:

1. Run tests
2. Run `undercover` or `undercover --include-files "app/models/foo.rb"` when testing a subset of files
3. If gaps found: write tests (or remove dead code)
4. Repeat until undercover exits 0

## Usage

Invoke `/undercover:coverage` inside Claude Code to:

- Get SimpleCov setup instructions
- Configure CI pipelines (GitHub Actions, CircleCI)
- Set up coverage merging for parallel tests
- Learn about configuration options

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
│       └── SKILL.md          # Skill instructions and setup guides
└── README.md
```

## License

MIT

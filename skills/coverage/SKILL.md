---
name: coverage
description: |
  Test coverage feedback loop using undercover. Guides Claude to run undercover
  after tests to identify untested code changes, handle partial test runs with
  --include-files, and set up SimpleCov, CI pipelines, and coverage merging.
---

# Undercover Coverage Feedback

## New project setup

When asked to set up coverage for a Ruby project, follow these steps.

If a Gemfile already exists, add to it. Never overwrite an existing Gemfile.
If a `.undercover` file already exists, never overwrite it — it contains the compare ref for this project.

### .gitignore
Add `coverage/` to `.gitignore` — SimpleCov output should not be committed:
```
coverage/
```

### Gemfile additions
```ruby
group :test do
  gem 'simplecov'
  gem 'undercover'
end
```

### spec_helper.rb / test_helper.rb (at the very top, before any other requires)

**Critical:** undercover requires `SimpleCov::Formatter::Undercover` — not `SimpleCov::Formatter::JSONFormatter` or any other formatter. Using the wrong formatter means undercover gets no coverage data and silently passes everything.

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

Run `bundle install` after adding the gems. Run the test suite once to generate `coverage/coverage.json` before the first undercover check.

## Coverage Feedback Loop

After every test run, run undercover to check for coverage gaps.

Only use `--include-files` when you ran a partial test suite. Full suite → no flag.

**Full test suite:**
```
bundle exec undercover --format json
```

**Partial test run** (single file or subset — undercover must match scope):
```
bundle exec undercover --format json --include-files "app/models/foo.rb"
bundle exec undercover --format json --include-files "app/models/*.rb,app/services/foo*.rb"
```
`--include-files` accepts comma-separated glob patterns. When running a subset of
tests, SimpleCov only has coverage data for loaded files. Without `--include-files`,
undercover will falsely flag all other changed files as uncovered.

Workflow:
1. Run tests
2. Run `bundle exec undercover --format json` (scoped with `--include-files` if partial)
3. If gaps found: **write tests** for uncovered code. Prefer real tests over `:nocov:` — only use `:nocov:` when the code genuinely cannot or should not be tested. Only delete code if you are certain it is dead/unreachable — understand why it's uncovered before removing it
4. Repeat until undercover exits 0

### JSON output format

```json
{
  "warnings": [
    {
      "node": "Foo#bar",
      "type": "instance method",
      "file": "lib/foo.rb",
      "first_line": 10,
      "last_line": 15,
      "coverage": 0.5,
      "uncovered_lines": [11],
      "uncovered_branches": [
        {"line": 13, "block": 0, "branch": 0},
        {"line": 13, "block": 0, "branch": 1, "description": "else"}
      ]
    }
  ],
  "summary": {"total_warnings": 1, "files_affected": 1},
  "validation": "stale_coverage"
}
```

- `uncovered_lines` — lines with zero hits
- `uncovered_branches` — branches with zero hits; `description` present for ternary/case branches
- `validation` — only present on errors (e.g. `stale_coverage`); exit code is 0 in this case
- `warnings` empty + no `validation` = all changed code covered

## What Undercover Reports

Undercover checks **only the lines changed in your current diff** (compared
against a git ref). It does NOT measure total coverage.

- "✅ No coverage is missing in latest changes" means the changed lines are
  covered — nothing more. The rest of the codebase could have 0% coverage.
- SimpleCov's percentage (e.g. "97.6% covered") and undercover's result are
  independent. Do not conflate them.
- If no lines changed relative to the compare ref, undercover always passes.
  This is expected, not a confirmation of coverage quality.

Use undercover to ensure **new and modified code** is tested. Use SimpleCov's
total percentage to track overall coverage trends separately.

### The compare ref

**Undercover only sees files that are staged or have unstaged changes to tracked files.
Completely untracked files (never added to git) are invisible — undercover will not check them.**

By default (no `--compare`), undercover checks **uncommitted changes** — staged
and unstaged vs HEAD. This only catches work in progress.

If you commit as you go (the normal workflow), the default diff is empty after each
commit and undercover always passes — a false green. **`--compare` is required.**

Use `--compare` in two situations:

**On a feature branch** — to check all commits since branching off main:
```
-c origin/main
```

**No remote / local-only repo** — compare against the initial commit SHA:
```
-c <initial-commit-sha>
```

Set it in `.undercover` so it applies consistently. If `.undercover` already exists
in the project, use the ref it specifies — do not overwrite it.

```
-c origin/main
```

---

# SETUP GUIDES

When asked to set up undercover, SimpleCov, or CI coverage, use these guides.

## 1. SimpleCov Setup (Required)

### Gemfile
```ruby
group :test do
  gem 'simplecov'
  gem 'undercover'
end
```

### spec_helper.rb or test_helper.rb (at the very top!)

**Must use `SimpleCov::Formatter::Undercover` — not JSONFormatter or any other.**
```ruby
require 'simplecov'
require 'undercover/simplecov_formatter'

SimpleCov.formatter = SimpleCov::Formatter::Undercover

SimpleCov.start do
  add_filter(/^\/spec\//)  # For RSpec
  add_filter(/^\/test\//)  # For Minitest
  enable_coverage(:branch) # Enable branch-level warnings
end
```

Then run `bundle install` and run tests once to generate `coverage/coverage.json`.

## 2. CI Setup

### GitHub Actions (simple)

Fails the build on coverage gaps, no external service needed:

```yaml
name: Tests
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0  # required for --compare to access origin/main
    - uses: ruby/setup-ruby@v1
      with:
        ruby-version: '3.3'
        bundler-cache: true
    - name: Run tests
      run: bundle exec rspec
    - name: Check coverage
      run: bundle exec undercover --compare origin/main
```

`fetch-depth: 0` is required — the default shallow clone won't have `origin/main` to diff against.

### UndercoverCI (posts checks to GitHub PRs)

A hosted service that posts coverage results directly to pull requests.

1. Sign up at https://undercover-ci.com
2. Install the GitHub App on your repository
3. Add to your workflow:

```yaml
    - name: Upload coverage to UndercoverCI
      run: |
        ruby -e "$(curl -s https://undercover-ci.com/uploader.rb)" -- \
          --repo ${{ github.repository }} \
          --commit ${{ github.event.pull_request.head.sha || github.sha }} \
          --simplecov coverage/coverage.json
```

### CircleCI

```yaml
version: 2.1
jobs:
  build:
    docker:
      - image: cimg/ruby:3.3
    steps:
      - checkout
      - run: bundle install
      - run: bundle exec rspec
      - run:
          name: Upload coverage to UndercoverCI
          command: |
            ruby -e "$(curl -s https://undercover-ci.com/uploader.rb)" -- \
              --repo $CIRCLE_PROJECT_USERNAME/$CIRCLE_PROJECT_REPONAME \
              --commit $CIRCLE_SHA1 \
              --simplecov coverage/coverage.json
```

### Uploader Options

| Option | Purpose |
|--------|---------|
| `--repo` | Repository in `org/repo` format |
| `--commit` | Current build commit SHA |
| `--simplecov` | Path to SimpleCov JSON (default: `coverage/coverage.json`) |
| `--cancel` | Skip coverage check for this commit |

## 3. Parallel Tests with Coverage Merging

For parallel test runners (CircleCI parallelism, etc.), each container generates its own coverage data that must be merged before uploading.

### CircleCI Parallel Example

```yaml
version: 2.1
jobs:
  test:
    parallelism: 4
    docker:
      - image: cimg/ruby:3.3
    steps:
      - checkout
      - run: bundle install
      - run:
          name: Run RSpec in parallel
          command: |
            circleci tests glob "spec/**/*_spec.rb" | circleci tests split | xargs bundle exec rspec
            cp coverage/.resultset.json coverage/.resultset-$CIRCLE_NODE_INDEX.json
      - persist_to_workspace:
          root: coverage
          paths: [".resultset-$CIRCLE_NODE_INDEX.json"]

  coverage:
    docker:
      - image: cimg/ruby:3.3
    steps:
      - checkout
      - attach_workspace:
          at: /tmp/coverage
      - run:
          name: Merge and upload coverage
          command: |
            bundle exec ruby scripts/merge_coverage.rb
            ruby -e "$(curl -s https://undercover-ci.com/uploader.rb)" -- \
              --repo $CIRCLE_PROJECT_USERNAME/$CIRCLE_PROJECT_REPONAME \
              --commit $CIRCLE_SHA1 \
              --simplecov coverage/coverage.json

workflows:
  test_and_coverage:
    jobs:
      - test
      - coverage:
          requires: [test]
```

### Merge Script (`scripts/merge_coverage.rb`)

Create the `scripts/` directory and add this file:

```ruby
require 'simplecov'
require 'undercover/simplecov_formatter'

puts 'Merging coverage results from parallel containers...'

SimpleCov.formatter = SimpleCov::Formatter::Undercover
SimpleCov.collate(Dir['/tmp/coverage/.resultset-*.json']) do
  enable_coverage(:branch)
end

puts 'Done! Merged coverage saved.'
```

## 4. Claude Permissions (Recommended)

For the coverage feedback loop to work without interruption, suggest adding
permissions to `.claude/settings.json` in the project root.

**First detect the test framework** (check Gemfile, presence of spec/ vs test/,
or rakefile), then suggest only the relevant commands:

- RSpec: `"Bash(bundle exec rspec*)"`
- Minitest/rake: `"Bash(bundle exec rake test*)"` or `"Bash(bundle exec rake spec*)"`
- Rails test: `"Bash(rails test*)"`
- Always include: `"Bash(bundle exec undercover*)"`

Example for an RSpec project:
```json
{
  "permissions": {
    "allow": [
      "Bash(bundle exec rspec*)",
      "Bash(bundle exec undercover*)"
    ]
  }
}
```

Without this, Claude will ask for permission before each test run and undercover
check, breaking the automated feedback loop.

## 5. Configuration File

Create `.undercover` in project root for default options:
```
-c origin/main
```

CLI options override file settings.

## 6. Skipping Coverage

There are two different problems, requiring different solutions:

**Problem A: File is loaded in tests but you don't want coverage warnings**
`:nocov:` is appropriate for code that genuinely cannot or should not be tested — legacy code explicitly acknowledged as untested, platform-specific branches, glue code with no meaningful test surface. Prefer a real test, but don't contort your code to satisfy a tool.

Use `:nocov:` comments on their own lines — **not** as end-of-line comments (those are ignored by SimpleCov):
```ruby
# :nocov:
def legacy_untested_method
  # acknowledged as untested
end
# :nocov:
```

Wrong — SimpleCov ignores end-of-line `:nocov:`:
```ruby
def legacy_untested_method # :nocov:  ← does NOT work
end
```

**Problem B: File is never required in tests (rake tasks, scripts, initializers, etc.)**
`:nocov:` won't work here because SimpleCov never sees the file. Use filters instead.

Note: undercover scans `*.rb`, `*.rake`, `*.ru`, and `Rakefile` by default. Any
file not loaded by the test suite won't appear in SimpleCov's coverage data, so
undercover will flag it — rake tasks, scripts, initializers, CLI tools, etc.

### SimpleCov `add_filter` (excludes from coverage report entirely)

Only add filters for files that are genuinely never loaded in tests. A good
signal: undercover keeps warning about a file even though there's no way to
test it (e.g. a Rakefile, a CLI script, a code generator).

Add to `spec_helper.rb` or `test_helper.rb`:
```ruby
SimpleCov.start do
  add_filter(/^\/spec\//)
  add_filter(/^\/test\//)
  add_filter('lib/tasks/')  # only if rake tasks are never required in tests
end
```

### Undercover `--exclude-files` (excludes from undercover warnings only)

Use in `.undercover` when SimpleCov tracks the file but undercover shouldn't
warn about it — e.g. a path you can't meaningfully test:
```
--exclude-files "lib/tasks/*"
```

**When to use which:**
- `add_filter`: file is never loaded in the test process at all
- `--exclude-files`: file is loaded but you've made a deliberate decision not to test it

---

## CLI Reference

```
undercover [options]
  -s, --simplecov PATH    SimpleCov JSON report file
  -l, --lcov PATH         LCOV report file (deprecated)
  -p, --path PATH         Project directory
  -g, --git-dir PATH      Override .git directory location
  -c, --compare REF       Compare against git ref (branch/commit/tag)
  -r, --ruby-syntax VER   Target Ruby syntax version
  -w, --max-warnings N    Stop after N warnings
  -f, --include-files GLOBS  Glob patterns to include (default: *.rb,*.rake,*.ru,Rakefile)
  -x, --exclude-files GLOBS  Glob patterns to exclude (default: test/*,spec/*,db/*,config/*,*_test.rb,*_spec.rb)
      --format FORMAT     Output format: text, json (default: text)
  -h, --help              Show help
```

## Further Reading

- Docs: https://undercover-ci.com/docs
- GitHub: https://github.com/grodowski/undercover

Consult these when the skill doesn't cover your situation.

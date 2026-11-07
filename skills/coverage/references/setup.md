# Undercover / SimpleCov Setup

When asked to set up coverage for a Ruby project, follow these steps.

If a Gemfile already exists, add to it. Never overwrite an existing Gemfile.
If a `.undercover` file already exists, never overwrite it — it contains the compare ref for this project.

## .gitignore
Add `coverage/` to `.gitignore` — SimpleCov output should not be committed:
```
coverage/
```

## Gemfile additions
```ruby
group :test do
  gem 'simplecov'
  gem 'undercover'
end
```

undercover 0.9 requires SimpleCov 1.0+ and Ruby 3.2+. This skill assumes those versions.

## spec_helper.rb / test_helper.rb (at the very top, before any other requires)

**Critical:** undercover requires `SimpleCov::Formatter::Undercover`, not `SimpleCov::Formatter::JSONFormatter` or SimpleCov's default formatter. Their `coverage.json` lacks the data undercover uses to honour SimpleCov's `skip` filters, so filtered files get flagged as uncovered.

```ruby
require 'simplecov'
require 'undercover/simplecov_formatter'

SimpleCov.formatter = SimpleCov::Formatter::Undercover

SimpleCov.start do
  enable_coverage(:branch)
end
```

SimpleCov's HTML formatter also writes `coverage/coverage.json`. To keep the HTML report, list the undercover formatter last so its file wins:
```ruby
SimpleCov.formatters = [SimpleCov::Formatter::HTMLFormatter, SimpleCov::Formatter::Undercover]
```

Run `bundle install` after adding the gems. Run the test suite once to generate `coverage/coverage.json` before the first undercover check.

## Claude Permissions (Recommended)

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

## Configuration File

Create `.undercover` in project root for default options:
```
-c origin/main
```

CLI options override file settings.

## CLI Reference

```
undercover [options]
  -s, --simplecov PATH    SimpleCov JSON report file
  -l, --lcov PATH         LCOV report file (deprecated)
  -g, --git-dir PATH      Override .git directory location
  -c, --compare REF       Compare against git ref (branch/commit/tag)
  -w, --max-warnings N    Stop after N warnings
  -f, --include-files GLOBS  Glob patterns to include (default: *.rb,*.rake,*.ru,Rakefile)
  -x, --exclude-files GLOBS  Glob patterns to exclude (default: test/*,spec/*,db/*,config/*,*_test.rb,*_spec.rb,vendor/*)
      --format FORMAT     Output format: text, json (default: text)
  -h, --help              Show help
```

## Further Reading

- Docs: https://undercover-ci.com/docs
- GitHub: https://github.com/grodowski/undercover

Consult these when the skill doesn't cover your situation.

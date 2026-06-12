# Skipping Coverage

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

## SimpleCov `add_filter` (excludes from coverage report entirely)

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

## Undercover `--exclude-files` (excludes from undercover warnings only)

Use in `.undercover` when SimpleCov tracks the file but undercover shouldn't
warn about it — e.g. a path you can't meaningfully test:
```
--exclude-files "lib/tasks/*"
```

**When to use which:**
- `add_filter`: file is never loaded in the test process at all
- `--exclude-files`: file is loaded but you've made a deliberate decision not to test it

# Skipping Coverage

There are two different problems, requiring different solutions:

**Problem A: File is loaded in tests but you don't want coverage warnings**
`# simplecov:disable` is appropriate for code that genuinely cannot or should not be tested: legacy code explicitly acknowledged as untested, platform-specific branches, glue code with no meaningful test surface. Prefer a real test, but don't contort your code to satisfy a tool.

Wrap the code in a `# simplecov:disable` / `# simplecov:enable` pair, each on its own line. Text after the directive is a free-form reason:
```ruby
# simplecov:disable legacy, acknowledged as untested
def legacy_untested_method
  do_something
end
# simplecov:enable
```

A directive at the end of a code line skips that line only:
```ruby
def platform_name
  RUBY_PLATFORM.include?("darwin") ? "macOS" : "other" # simplecov:disable
end
```

To skip one criterion and keep the others, name it after `disable` and `enable`: `line`, `branch` or `method`, comma separated (`# simplecov:disable branch`).

Existing `# :nocov:` pairs still work; write new skips with `simplecov:disable`.

**Problem B: File is never required in tests (rake tasks, scripts, initializers, etc.)**
`# simplecov:disable` won't work here because SimpleCov never sees the file. Use filters instead.

Note: undercover scans `*.rb`, `*.rake`, `*.ru`, and `Rakefile` by default. Any
file not loaded by the test suite won't appear in SimpleCov's coverage data, so
undercover will flag it — rake tasks, scripts, initializers, CLI tools, etc.

## SimpleCov `skip` (excludes from coverage report entirely)

Only add filters for files that are genuinely never loaded in tests. A good
signal: undercover keeps warning about a file even though there's no way to
test it (e.g. a Rakefile, a CLI script, a code generator).

Add to `spec_helper.rb` or `test_helper.rb`:
```ruby
SimpleCov.start do
  skip 'lib/tasks/'  # only if rake tasks are never required in tests
end
```

SimpleCov already skips `spec/` and `test/` by default.

## Undercover `--exclude-files` (excludes from undercover warnings only)

Use in `.undercover` when SimpleCov tracks the file but undercover shouldn't
warn about it, e.g. a path you can't meaningfully test. The option replaces the
default exclude list, so keep the defaults in it, or undercover starts warning about
changed spec and test files:
```
--exclude-files "test/*,spec/*,db/*,config/*,*_test.rb,*_spec.rb,vendor/*,lib/tasks/*"
```

In a monorepo, globs are matched relative to the directory SimpleCov ran in, so
write `lib/tasks/*`, not `apps/dash/lib/tasks/*`.

**When to use which:**
- `skip`: file is never loaded in the test process at all
- `--exclude-files`: file is loaded but you've made a deliberate decision not to test it

---
name: coverage
description: |
  Test coverage feedback loop using undercover. Guides Claude to run undercover
  after tests to identify untested code changes, handle partial test runs with
  --include-files, and set up SimpleCov, CI pipelines, and coverage merging.
argument-hint: what you want to do — e.g. "run the loop", "set up coverage", "set up CI", "check this PR"
---

# Undercover Coverage Feedback

## Routing

The user may have stated an intent after the command (shown below as `$ARGUMENTS`).
Use it to decide what to do, then read the matching reference file before acting:

- **Set up coverage in a Ruby project** (SimpleCov, Gemfile, `.undercover`, permissions) → read `references/setup.md`
- **Set up CI** (UndercoverCI or GitHub Actions; also CircleCI and parallel test merging) → read `references/ci.md`
- **Read/address UndercoverCI feedback on a PR** → read `references/pr-feedback.md`
- **Skip coverage** for a file (`:nocov:`, `add_filter`, `--exclude-files`) → read `references/skipping.md`
- **Run the feedback loop** after tests, or anything else → use the loop below (the default)

Stated intent: $ARGUMENTS

If no intent was given, default to the feedback loop.

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
3. If gaps found: **write tests** for uncovered code. Prefer real tests over `:nocov:` — only use `:nocov:` when the code genuinely cannot or should not be tested (see `references/skipping.md`). Only delete code if you are certain it is dead/unreachable — understand why it's uncovered before removing it
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

## Reference files

Deeper guides, loaded on demand (see Routing above):

- `references/setup.md` — SimpleCov + Gemfile + `.undercover` + Claude permissions + CLI reference
- `references/ci.md` — GitHub Actions, UndercoverCI, CircleCI, parallel test coverage merging
- `references/pr-feedback.md` — reading UndercoverCI check annotations on a PR via `gh api`
- `references/skipping.md` — `:nocov:`, `add_filter`, `--exclude-files` and when to use each

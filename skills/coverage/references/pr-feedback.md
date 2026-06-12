# Reading UndercoverCI feedback on a PR

UndercoverCI posts coverage results as a GitHub **check run** named `coverage`. The
warnings live in the check's *annotations*. `gh pr view`, `statusCheckRollup`, and
GitHub MCP only expose the check's conclusion and a link to undercover-ci.com — **not
the annotations**. To read the actual feedback, use `gh api` (two calls):

1. List check runs for the PR head to get the `coverage` run id and annotation count:
```
gh api repos/OWNER/REPO/commits/BRANCH/check-runs \
  --jq '.check_runs[] | {name, conclusion, id, annotations_count: .output.annotations_count}'
```
2. Fetch that run's annotations (the coverage warnings):
```
gh api repos/OWNER/REPO/check-runs/CHECK_RUN_ID/annotations \
  --jq '.[] | {path, start_line, end_line, message, title}'
```

Each annotation gives the file, line range, and the missing line/branch coverage —
the same warnings undercover reports locally. Then open the files, add tests, and
verify locally with `bundle exec undercover --compare BASE_BRANCH` (see the compare
ref section in SKILL.md).

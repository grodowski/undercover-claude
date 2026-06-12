# CI Setup for Undercover

When asked to set up CI coverage, use these guides.

## GitHub Actions (simple)

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

## UndercoverCI (posts checks to GitHub PRs)

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

## CircleCI

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

## Uploader Options

| Option | Purpose |
|--------|---------|
| `--repo` | Repository in `org/repo` format |
| `--commit` | Current build commit SHA |
| `--simplecov` | Path to SimpleCov JSON (default: `coverage/coverage.json`) |
| `--cancel` | Skip coverage check for this commit |

## Parallel Tests with Coverage Merging

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

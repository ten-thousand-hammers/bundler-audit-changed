# bundler-audit-changed

Runs `bundler-audit` and fails only on advisories against gems **this branch
added or changed**.

A CVE published against a gem nobody touched is a real problem, but it is not
the pull request's problem. Left unscoped, one new advisory turns every open
branch red until someone bumps the gem, and the usual response is to stop
reading the check. This action keeps the signal on the branch that caused it.

Pair it with a scheduled workflow that audits the whole bundle — that is what
surfaces advisories nobody introduced. See [Companion workflow](#companion-workflow).

## Usage

```yaml
- uses: actions/checkout@v6
  with:
    fetch-depth: 0 # the merge-base diff cannot resolve in a shallow clone

- uses: ruby/setup-ruby@v1
  with:
    bundler-cache: true

- uses: ten-thousand-hammers/bundler-audit-changed@v1
```

## How it works

1. Resolves the base commit: the merge base with the pull request's base branch,
   or `main`. On a push to the base branch itself it steps back to `HEAD~1`, so a
   merge that introduces a vulnerable gem still fails.
2. Diffs `Gemfile.lock` from there to `HEAD` and collects the gems whose own
   resolved version changed. Lines indented six spaces are transitive dependency
   constraints and are ignored; only the four-space resolved entries count.
3. Runs `bundler-audit check --update --format json`.
4. Partitions the advisories. Gems this branch changed **fail** the step; the
   rest are written to the job summary and left alone.

Because the diff is three-dot from the merge base, gem bumps that landed on the
base branch and have not been merged in yet do not count as "changed by this
branch".

## Inputs

| Name | Default | Description |
| --- | --- | --- |
| `base-ref` | pull request base, else `main` | Branch to compare against. |
| `bundler-audit-command` | `bundle exec bundler-audit` | How to invoke bundler-audit. |

## Outputs

| Name | Description |
| --- | --- |
| `introduced-count` | Advisories against gems this branch changed. |
| `preexisting-count` | Advisories that already affected the base branch. |

## Requirements

`jq`, and a checkout with `fetch-depth: 0`. The step fails loudly rather than
guessing if no merge base can be resolved, or if `bundler-audit` exits with
anything other than 0 (clean) or 1 (advisories found).

## Companion workflow

```yaml
name: Security Audit
on:
  schedule:
    - cron: "15 6 * * *"
  workflow_dispatch:
jobs:
  Audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - run: bundle exec bundler-audit check --update
```

## See also

[`brakeman-changed`](https://github.com/ten-thousand-hammers/brakeman-changed)
applies the same idea to Brakeman warnings.

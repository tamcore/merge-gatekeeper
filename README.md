# Merge Gatekeeper

A GitHub Action that ensures all CI checks pass before allowing a PR to be merged. It polls check runs and commit statuses, retrying while checks are in progress.

This action is implemented using [actions/github-script](https://github.com/actions/github-script) for simplicity and maintainability.

## Features

- ✅ Polls both **check runs** and **commit statuses**
- 🔄 Retries while checks are still in progress
- 🎯 Deduplicates check runs (keeps the latest)
- 🚫 Supports ignoring specific checks via regex patterns
- 📊 Generates a summary table in the GitHub Actions UI
- ⚡ Handles workflow re-runs with immediate checks

## Required Permissions

Add these permissions to your workflow:

```yaml
permissions:
  actions: read
  checks: read
  statuses: read
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `github-token` | No | `${{ github.token }}` | GitHub token to access PR commit statuses and checks |
| `initial-delay-seconds` | No | `5` | Seconds to sleep before the first check |
| `max-retries` | No | `5` | Number of retries while checks are still in progress |
| `polling-interval-seconds` | No | `60` | Seconds to wait between retry attempts |
| `ignored-name-patterns` | No | `''` | Newline-separated list of regex patterns to exclude jobs |
| `full-details-summary` | No | `false` | Show all checks in the summary (not just failures) |

## Usage Examples

### Dedicated Gatekeeper Workflow

Create a separate workflow that runs the gatekeeper:

```yaml
name: Merge Gatekeeper

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  actions: read
  checks: read
  statuses: read

jobs:
  gatekeeper:
    runs-on: ubuntu-latest
    steps:
      - name: Merge Gatekeeper
        uses: tamcore/merge-gatekeeper@v1
        with:
          initial-delay-seconds: 10
          max-retries: 10
          polling-interval-seconds: 30
```

### Appended to Existing Workflow

Add the gatekeeper as the final job in your existing CI workflow:

```yaml
name: CI

on:
  pull_request:

permissions:
  actions: read
  checks: read
  statuses: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test

  gatekeeper:
    runs-on: ubuntu-latest
    needs: [build, test]
    if: always() && !cancelled()
    steps:
      - name: Merge Gatekeeper
        uses: tamcore/merge-gatekeeper@v1
        with:
          initial-delay-seconds: 0
          max-retries: 3
```

### Ignoring Specific Checks

Use **regex patterns** (not glob wildcards) to ignore certain checks:

```yaml
- name: Merge Gatekeeper
  uses: tamcore/merge-gatekeeper@v1
  with:
    ignored-name-patterns: |
      ^optional-.*
      ^experimental/.*
      coverage-report
      merge-gatekeeper.*
```

> **Note:** These are JavaScript regular expressions, not glob patterns.
> - Use `.*` (dot-star) for "match anything", not `*` alone
> - `merge-gatekeeper.*` matches `merge-gatekeeper-1`, `merge-gatekeeper-2`, etc.
> - `^` anchors to the start, `$` anchors to the end

## Limitations

### Job Name Matching

**Do NOT set a custom `name` for the job running this action.**

The action identifies itself using `GITHUB_JOB` (which is always the job key in your workflow). If you set a custom job `name`, the API returns that name, but the action only knows the job key — causing a mismatch.

```yaml
jobs:
  # ✅ Good - no custom name
  gatekeeper:
    runs-on: ubuntu-latest
    steps:
      - uses: tamcore/merge-gatekeeper@v1

  # ❌ Bad - custom name causes mismatch
  gatekeeper:
    name: "My Custom Gatekeeper Name"
    runs-on: ubuntu-latest
    steps:
      - uses: tamcore/merge-gatekeeper@v1
```

### Race Condition with Late-Starting Jobs

Jobs that start after the action completes won't be checked. To mitigate this:

1. Use `needs:` to make the gatekeeper job depend on other jobs
2. Increase `initial-delay` and `polling-interval`
3. Add the gatekeeper to the same workflow as other jobs

### No Event on Manual Job Retry

GitHub doesn't fire events when a single job is retried manually. If you retry a failed job, the gatekeeper won't know about it. **Mitigation:** Re-run the gatekeeper job too, or use "Re-run all jobs".

### API Lag

Check runs may not appear immediately in the GitHub API. The action warns if it can't find itself in the check runs list. This is usually resolved within the polling interval.

## License

MIT

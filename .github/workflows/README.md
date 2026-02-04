# Test Workflows for `act`

This directory contains test workflows for local testing with [act](https://github.com/nektos/act).

## Running Tests

From the repository root:

```bash
# List available workflows and jobs
act -l

# Run the self-test job
act push -j self-test

# Run with verbose output
act push -j self-test -v

# Run with a specific event file
act pull_request -e .github/workflows/test-events/pull_request.json -j self-test
```

## Notes

- The self-test job uses `uses: ./` to reference the local action
- `act` may not perfectly replicate GitHub's environment
- Some API calls may behave differently locally vs on GitHub

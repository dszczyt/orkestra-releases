# GitHub repo health

Checks a public GitHub repository (archived flag and open issue count) and posts a healthy or unhealthy report to a webhook. Uses the public GitHub API, no key needed.

## Run

```
orkestra run https://github.com/dszczyt/orkestra-releases/blob/main/examples/github-repo-health/workflow.yaml
```

Or from a local checkout: `orkestra run examples/reference/github-repo-health/workflow.yaml`.

## Inputs

None. Set `owner`, `repo`, `issue_count_threshold` and `alert_webhook` in the `globals` configmap, or override with `--set`.

## Environment

None.

## Test

```
orkestra test workflow.yaml workflow_test.yaml
```

The test mocks every HTTP call, so it needs no network.

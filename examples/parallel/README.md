# User overview

Fetches a user's profile and recent orders at the same time with a `parallel` step, then merges them.

It shows:

- `parallel` on a step: a fixed list of named branches. Each branch is one operation, with its own `input` (a jq expression over the step's input; the default is the input itself) and its own `retry`. The step itself has no `operation` and no `retry`.
- The join: the step's output is an object keyed by branch name, `{"profile": ..., "orders": ...}`. Its transitions see that object as `.input`.
- `on_failure: continue`: a failed branch does not fail the step. Its slot holds its error object (`_executor_error` and `_error`), so the next step can find failures with jq. With `on_failure: fail` (the default), the step fails with `_error.type: branch` and the failed branch's name.
- `concurrency` (not set here): by default every branch runs at once.

A branch that needs several steps uses an `orkestra/operation/workflow` operation. A branch cannot be a wait operation.

## Test it

```bash
orkestra test workflow.yaml workflow_test.yaml
```

Locally the branches run one at a time, in declaration order. On a server they run in parallel.

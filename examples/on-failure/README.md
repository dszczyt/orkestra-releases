# Invoice import with a catch-all

Imports an invoice into the ledger. When something fails and no transition handles it, the workflow's `on_failure` step alerts the operations team.

It shows:

- `on_failure` on the workflow: the step that starts after the first unhandled failure of a top-level task. Its input is `{failed_task: {id, step, status, input, output}, error}`, where `error` is the failed task's `_error`. Its transitions run as usual, so it can be a whole path.
- The run always ends `failed` after `on_failure`: it is for cleanup and notice. To recover and let the run succeed, use a `status: failed` transition on the step instead.
- A failure inside the `on_failure` path does not start it again. The run keeps its first failure (`.run.error`).
- A jq error in a transition's `on` or `input` is a task failure with `_error.type: transition`, so it reaches `on_failure` too.

## Test it

```bash
orkestra test workflow.yaml workflow_test.yaml
```

A phase whose run ends `failed` passes when it has expectations and they hold, as here with `.run.status == "failed"`.

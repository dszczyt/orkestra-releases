# Order booking saga

Books an order with three calls: reserve stock, charge the card, book the shipment. When the run fails, the work already done is undone, newest first, then the team is told.

It shows:

- `compensate` on a step: the step that undoes it. When the run starts failing (a failure no transition handles), every completed task of such a step gets one compensation task. They run one at a time, newest first: here `refund_card`, then `release_stock`.
- A compensation's input is `{input, output}` of the task it undoes, so `refund_card` reads the charge id from the charge's output. Its transitions are ignored: a compensation is one operation, with its own `retry`.
- A failed compensation stops compensating. The tasks left are listed on the run (`.run.uncompensated` in tests, `uncompensated_task_ids` in `GET /v1/runs/{id}`), so an operator can finish by hand. `on_failure` then runs with the failed compensation's error.
- `on_failure` runs after compensation, and the run ends `failed`.

A `map` step may declare `compensate`: each completed child is undone. A `parallel` step may not. A canceled run is not compensated, and a completed sub-workflow is not compensated through its own steps: give the workflow operation step a `compensate` if undoing it matters.

## Test it

```bash
orkestra test workflow.yaml workflow_test.yaml
```

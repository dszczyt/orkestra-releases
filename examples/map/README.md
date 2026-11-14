# City forecasts

Fetches a weather forecast for each city of the input with a `map` step, two cities at a time, then builds a report.

It shows:

- `map` on a step: `items` is a jq expression that yields an array, and `input` builds each child's input from `.item` (and `.index`). The step's operation runs once per item, as a child task of the same run.
- `concurrency`: at most two children run at once. The others wait for a slot.
- The join: the step's output is the array of the children's outputs, in item order. Its transitions see that array as `.input`, and `task("forecasts").output` returns it.
- `on_failure: fail` (the default): the first failed child cancels the children not started yet, and the step fails with `_error.type: branch`, the failed child's `branch` (its item index) and its `cause`. A `status: failed` transition routes it. With `on_failure: continue`, the step completes instead, with each failed child's error object in its slot.

A server caps a map step at 1000 items (`orkestra worker --max-map-items`). A step with more items fails with `_error.type: limit` before it creates any child.

## Test it

```bash
orkestra test workflow.yaml workflow_test.yaml
```

Locally the children run one at a time, in item order, so the mock's `responses:` answers city N with entry N. On a server they run in parallel.

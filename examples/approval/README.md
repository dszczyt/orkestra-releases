# Order approval

A manager approves an order in Slack. The run parks on a signal wait for up to 48 hours. An approved order ships after a 10-minute cool-down. Without an answer, the order is escalated.

It shows the three building blocks of a human approval:

- `orkestra/operation/wait` with `signal`, `key` and `timeout`: the run parks until `POST /v1/runs/{run_id}/signals/approval` arrives with the order's key. The wait's output is the signal payload. At the deadline the wait fails with `_error.type: timeout`, and a `status: failed` transition routes it.
- `orkestra/operation/wait` with `duration`: a timer.
- `orkestra/operation/pass`: a routing step with no side effect.

## Run it locally

```bash
orkestra run workflow.yaml --context run.ctx --input '{"order_id": "A-1042"}'
orkestra signal --context run.ctx --name approval --key A-1042 --payload '{"approved": true}'
orkestra run workflow.yaml --context run.ctx
```

The first run parks on the approval. The signal wakes the wait, and the second run continues to the end.

## On a server

```bash
curl -X POST "$ORKESTRA_URL/v1/runs/$RUN_ID/signals/approval" \
  -H "Authorization: Bearer $ORKESTRA_TOKEN" \
  -d '{"key": "A-1042", "payload": {"approved": true}}'
```

## Test it

```bash
orkestra test workflow.yaml workflow_test.yaml
```

The test clock is virtual: the cool-down completes at once, and the phase flag `advance_clock: 49h` moves time past the approval timeout.

# Approving an agent's tool calls

An agent whose `send_email` tool has a side effect. `approval: required` puts a person in front of every call: the model asks, the run stops, a person approves or denies, and only an approval lets the email go out.

```yaml
tools:
  send_email:
    operation: send_email
    approval: required
    approval_timeout: 24h     # default 1h; a timeout is a denial
```

Any tool can ask for approval, MCP tools included (`approval: none` or `required` is mandatory on those). It needs parallel mode, the default, because the wait lives in the dispatch workflow that handles each call.

## How a call is approved

Each tool call is a child run. In front of the tool there is a wait step on the signal `approval`, keyed by the tool call's id, with `early: ignore` (an answer sent before the call existed does not count) and `scope: runs:approve` (the sender must hold that scope). Three things can happen:

- `{"decision": "approve"}`: the tool runs with the original arguments.
- any other decision, or a payload with no decision: the model gets the error result `denied: <reason>`, and can react.
- no answer in `approval_timeout`: the model gets `denied: approval timed out`.

The step that runs the tool is marked, and the engine refuses it unless the task before it is an approved wait with a recorded sender. A graph edited by hand cannot route around the gate, and `orkestra validate` rejects one that tries.

One approval covers the call, retries included: if the tool step has a `retry:` (or a worker dies mid-call and the task runs again), the same approval is reused. Make a tool that must not be sent twice idempotent, or leave `retry:` off it.

## Locally

`orkestra run` stops at the approval and leaves the run `running`. Answer, then run the same command again:

```
orkestra signal --context run.json --name approval --key <tool call id> \
  --payload '{"decision": "approve", "reason": "looks right"}'
orkestra run workflow.yaml --context run.json
```

`orkestra_get_run` (the MCP tool) and the run's task list show what is waiting: the tool, its arguments, the child run and the key. Pass `--run <run_id>` to `orkestra signal` when two calls could share a key (a provider that repeats tool call ids), so the answer goes to the call you reviewed.

An approval is never given through MCP: no MCP tool sends a signal. Locally, though, there is no separation of scopes: `orkestra signal` holds every scope and records the sender as `local`, so anything that can run it on the same context file can approve. The recorded sender is a record, not an authentication. The separation is for a server, where the key decides.

## On a server

`GET /v1/runs/{id}` lists `pending_approvals`, each with the child `run_id`, the `key`, the `tool` and its `arguments`. Answer with:

```
POST /v1/runs/<run_id>/signals/approval
{"key": "<key>", "payload": {"decision": "approve", "reason": "looks right"}}
```

The answer is delivered only when the response's `delivered_to` is not null; a null means it was buffered, and a scoped wait does not take a buffered signal, so send it again. The key must hold the `runs:approve` scope:

```
orkestra keys create <org-id> --scope runs:approve
```

Do not give that scope to the key of an agent client. Console sessions act with the organization's full scope today, so any member can approve. The approver recorded on the wait is the authenticated caller, never a field of the body.

## Test it

```
ORKESTRA_LLM_API_KEY=test orkestra test workflow.yaml workflow_test.yaml
```

A test sends its signals to the whole run tree, in a later phase than the one that asked.

## A call that breaks the tool's schema

The engine checks the model's input against the tool's `input_schema` before anything else. A call that breaks it is refused before a person is asked and before the tool runs: the model gets an error result naming the fault (`invalid input for tool "send_email": ...`) and can call again. The run does not fail. If the id the provider gave a call is empty or repeated, the adapter gives it a unique one (`call_0`, `call_1`, ...), and that is the approval key.

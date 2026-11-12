# Agent loop

A Claude agent that answers weather questions with two tools, `geocode` and `forecast` (Open-Meteo, no key needed). There is no agent framework: the loop is three steps and a few transitions.

```
think -> geocode | forecast   when Claude asks for a tool (one per turn)
tool  -> think                with the tool result appended to the conversation
think -> end                  when Claude answers, or after max_turns calls
```

Each LLM call and each tool call is a task. Orkestra stores it, retries it (`think` retries 429, 5xx and 529), and shows it in the run trace. With `orkestra run --context <file>`, an interrupted run resumes from the last completed task, without calling the model again for the turns already done.

## How the conversation is carried

- `think` reads `.input.messages`. The first call has none, so it builds one from `.input.question`.
- `call_geocode` / `call_forecast` pass the next step `{messages, call}`. `messages` is `think`'s own input messages (read with `task("think").input`) plus Claude's reply. `call` is the `tool_use` block.
- `geocode_done` / `forecast_done` append a `tool_result` with the same `tool_use_id` and go back to `think`.
- To add a tool, add an operation, a step, one `call_<tool>` transition and one `<tool>_done` transition, and describe it in `globals.tools`.

## Turn budget

`globals.max_turns` (default 6) caps LLM calls per run:
- the last allowed call sends `tool_choice: {type: none}`, so Claude has to answer with what it has;
- `budget_exhausted` ends the run if Claude still asks for a tool after `max_turns` calls.

Tool HTTP errors do not fail the run (`disable_http_failure: true`): they go back to Claude as the tool result.

## Run

Download the workflow, then run it from the folder that holds it:

```
curl -fsSLO https://raw.githubusercontent.com/dszczyt/orkestra-releases/main/examples/agent-loop/workflow.yaml
ORKESTRA_LLM_API_KEY=sk-ant-... orkestra run workflow.yaml \
  --context /tmp/agent.json --input '{"question": "Do I need an umbrella in Lyon today?"}'
```

The answer is in the last `think` output, at `body.content[0].text`.

`ORKESTRA_LLM_API_KEY` is your Anthropic API key. Environment variables in `orkestra/secret` are expanded only when the CLI loads a LOCAL workflow file, so download the file before running it.

## Run it from another agent (MCP)

Register Orkestra once; it serves every workflow, so there is nothing to set up per workflow:

```
claude mcp add orkestra -e ORKESTRA_LLM_API_KEY=sk-ant-... -- orkestra mcp serve
```

The agent calls `orkestra_run` with `{"workflow": "/absolute/path/to/workflow.yaml", "input": {"question": "..."}}` and gets `{"status", "run_id", "step", "output"}` for the last step. Add `"context": "/absolute/path/to/agent.json"` to keep the run, then read each turn with `orkestra_get_run` and `orkestra_get_task`.

## Test

```
ORKESTRA_LLM_API_KEY=test orkestra test workflow.yaml workflow_test.yaml
```

Every HTTP call is mocked, so the test needs no network. The LLM mock uses `responses:` to answer differently on each turn. One test checks the whole loop (tool ids, conversation growth). The other checks that the budget stops a model that never stops calling tools.

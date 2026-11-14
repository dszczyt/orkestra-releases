# Agent loop

A Claude agent that answers weather questions with two tools, `geocode` and `forecast` (Open-Meteo, no key needed). The whole agent is one `orkestra/agent` resource: a model, a prompt, two tools and a budget. There is no agent framework and nothing hidden: the loader expands the agent into plain steps, transitions and operations, and `orkestra compile` prints them.

```
think -> calls          when Claude asks for tools (all the calls of a turn run in parallel, one child run each)
calls -> think          with the tool results appended to the conversation
think -> gave_up        when the budget is spent and Claude still asks for a tool
think -> end            when Claude answers
```

Each model call and each tool call is a task. Orkestra stores it, retries it (the model call retries 429, 5xx and 529), and shows it in the run trace. With `orkestra run --context <file>`, an interrupted run resumes from the last completed task, without calling the model again for the turns already done.

## See what it expands to

```
orkestra compile workflow.yaml
```

The output is the workflow with the agent replaced by `weather.think`, a `weather.calls` map step (one child run per tool call of a turn, in a dispatch workflow that holds the two tools), the `weather.*` transitions, one `orkestra/tool` per tool and one `orkestra/budget` named `weather`. It is byte-stable: the same source and the same Orkestra version give the same bytes, and a header says which source and compiler made them. `${ORKESTRA_LLM_API_KEY}` is printed unexpanded.

The server stores these compiled bytes. A run is pinned to them, so a later Orkestra version that expands agents differently never changes a run in flight.

## How the loop works

- The conversation is a run-scoped log named after the agent. A turn is written with the task result, in one transaction.
- `think` answers the tool calls of a turn by a `calls` step: one child run per call, all at once (`max_parallel_tools`, default 8; `parallel_tools: false` runs one call per turn instead), each running the operation you named. Its result goes back to the model as a `tool_result` with the same `tool_use_id`. A tool that fails does not fail the run: the error goes back to the model as the result.
- A tool name the model invents is not an error of the run either: it gets an error result and a chance to answer.
- To add a tool, add an operation and an entry under `tools:`.

## Budget

`budget: {turns: 6, cost_usd: 0.05, wall_clock: 60s, on_exhausted: gave_up}` bounds the loop. An agent needs at least one of `turns`, `tokens` or `cost_usd`. The last allowed turn may not call a tool, so Claude has to answer with what it has. A model that ignores that is refused on the next turn, and the run goes to `gave_up`.

## Run

Download the workflow, then run it from the folder that holds it:

```
curl -fsSLO https://raw.githubusercontent.com/dszczyt/orkestra-releases/main/examples/agent-loop/workflow.yaml
ORKESTRA_LLM_API_KEY=sk-ant-... orkestra run workflow.yaml \
  --context /tmp/agent.json --input '{"question": "Do I need an umbrella in Lyon today?"}'
```

The answer is in the output of `weather.think`, at `text`.

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

Every HTTP call is mocked, so the test needs no network. The LLM mock uses `responses:` to answer differently on each turn. Four tests cover the loop (tool ids and conversation growth), a model that never stops calling tools (the budget ends it), a tool name the model invents, and a tool that fails.

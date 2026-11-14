# LLM tool loop

A weather agent in one operation, one tool and one budget. The model calls `lookup` for the weather and answers. There is no agent framework: the loop is `think -> run_lookup -> think`, and the budget is what makes it safe to run.

```
think      -> run_lookup   when the model asks for the tool (the arguments are the model's)
run_lookup -> think        with the result, in the same conversation
think      -> end          when the model answers
```

## Tools

`orkestra/tool` declares a function: a name, a description the model reads, and a JSON Schema for its input. The operation lists the tools it offers (`tools: [lookup]`). The set is closed: the model is offered exactly those, in that order, and the graph decides where a call goes. A tool name that is not declared fails `orkestra validate`. The tool list is part of the conversation's frozen prefix, so every operation sharing a conversation offers the same one.

`parallel_tool_calls: false` asks for one call per turn. `tool_choice` is a jq expression that gives `"auto"`, `"none"` or `null`.

## Budget

`orkestra/budget` limits what the tasks of a set of steps may use: `turns` (model calls that ended, successful or not), `tokens`, `cost_usd`, `wall_clock`. A task is checked after it is claimed and before it runs. If a limit is reached the task fails without calling the model, with `_error: {type: "limit", name: "budget.turns", budget: "agent", step, task_id}`, and the usual failure rules apply: `on_exhausted: give_up` sends the run to that step, and without it the run fails.

- `.budget` is available in jq, in operations and in transitions: `{name, limits, used, remaining}`. This workflow uses it once: on the last allowed turn `tool_choice` is `"none"`, so the model has to answer with what it has instead of being cut off.
- `turns` is exact. `tokens` and `cost_usd` are checked between tasks, so one call can overshoot by one context window plus `max_tokens`. `wall_clock` counts active time: time spent in a wait task does not count, and a hung operation is cancelled when it runs out.
- `orkestra validate` refuses a workflow with a loop that calls a model unless a budget with `turns`, `tokens` or `cost_usd` covers it. With a `turns` limit it prints the worst case: `turns * max_tokens` output tokens.
- A `cost_usd` budget must count only models with a known price (or an operation with `pricing:`): a limit on an unpriced call would never trip.

## Run

```
ORKESTRA_LLM_API_KEY=sk-ant-... orkestra run workflow.yaml --input '{"question": "Do I need an umbrella in Lyon today?"}'
```

`weather_url` and `notify_webhook` in the `globals` ConfigMap are placeholders: point them at a real weather endpoint that answers `GET /lookup?city=...`, and at a webhook. `ORKESTRA_LLM_API_KEY` is your Anthropic API key; environment variables in `orkestra/secret` are expanded only when the CLI loads a LOCAL workflow file.

## Test

```
orkestra test workflow.yaml workflow_test.yaml
```

Every HTTP call is mocked. One test follows the loop to an answer. The other mocks a model that never stops asking for the tool, and checks that the fourth `think` task is refused with `budget.turns` and that the run ends through `give_up`.

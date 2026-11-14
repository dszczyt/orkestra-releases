# Sub-agent as a tool

A lead agent whose only tool is another agent. `workflow: ./researcher/workflow.yaml` in the tool list runs that workflow as a child run for each call: the call's arguments are the child's input, and the child's final output is the tool result (`result: .text` keeps only the text of the researcher's last answer, so the rest does not cost tokens).

The researcher is a workflow of its own, with its own agent, its own tool and its own budget. Nothing connects the two but the tool call.

## Budgets add up

The lead's budget covers the child runs its tool step starts, at any depth. Here `turns: 4` is the lead's own model calls plus the researcher's. A sub-agent cannot spend what its caller was never given, and no cap has to be passed down. The researcher's own budget still applies to it as well.

## Parallel calls

By default all the tool calls of one turn run at once, one child run each (`parallel_tools: true`, at most `max_parallel_tools`, default 8). With two `research` calls in a turn, two researchers run side by side. `parallel_tools: false` runs one call per turn.

## Test it

```
ORKESTRA_LLM_API_KEY=test orkestra test workflow.yaml workflow_test.yaml
```

Both agents call the one mocked model endpoint, in the order the calls happen. The tasks of the child runs are in `.tasks`, not in `.history`.

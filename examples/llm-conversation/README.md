# LLM conversation

Two model calls that share a conversation. `ask` answers a question; `translate` asks the model, in the same conversation, to say it again in French. Nothing passes the first answer along: the second call sees it because both operations name `conversation: chat`.

## How it works

- A conversation is an append-only log owned by one run. A call that names one sends the earlier turns, then its own user turn.
- The turn (the user message and the model's reply, provider blocks unchanged) is written in the same transaction as the task result. A call that fails or is refused leaves the conversation as it was.
- Operations that share a conversation must share `provider` and `system`. They are literals, so `orkestra validate` checks it. The `model` may differ.
- The task output holds `text`, `tool_calls` and `usage`. Its `content` is empty: the blocks are stored once, in the conversation.
- In jq, `conversation("chat")` returns the messages as `[{role, text, tool_calls, tool_results}]`. In `orkestra test`, `.conversations.chat` is the same list. With `--context`, the context file keeps them under `conversations`; with Postgres they are in the `conversations` and `conversation_entries` tables.

`tool_results` (a jq array of `{id, content, is_error}`) answers the tool calls of the previous turn. Tools themselves come in a later slice.

## Run

```
ORKESTRA_LLM_API_KEY=sk-ant-... orkestra run workflow.yaml --input '{"question": "What is the capital of France?"}'
```

`ORKESTRA_LLM_API_KEY` is your Anthropic API key. Environment variables in `orkestra/secret` are expanded only when the CLI loads a LOCAL workflow file.

## Test

```
orkestra test workflow.yaml workflow_test.yaml
```

Every HTTP call is mocked, so the test needs no network.

## Cassette

`test_recorded_conversation` replays `cassettes/conversation.yaml`: the recorded answers, with the token usage that came with them. That makes cost and cache assertions (`.usage.cost_usd`, `.usage.cache_read_tokens`) run offline and give the same result every time. `.trajectory` lists the run's tasks in order (`type` is `llm` for a model call).

Each entry also holds the hash of the request that produced it (`request_sha256`). With `match: strict`, the default, a request that differs fails the test: editing `system` or the question in `workflow.yaml` tells you the cassette is out of date. Record it again against the real API:

```
ORKESTRA_LLM_API_KEY=sk-ant-... orkestra test workflow.yaml workflow_test.yaml --record
```

`--record` sends the requests of cassette mocks to the real host and writes the file. Request headers (your key) are never stored, and bodies go through the secret redactor. `match: order` skips the hash check.

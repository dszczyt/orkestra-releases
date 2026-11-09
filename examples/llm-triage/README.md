# LLM triage

Asks Claude (model `claude-sonnet-5-5`) to classify an issue as `bug`, `feature` or `question`, then routes to a step that posts the label to `result_webhook`. Anything that is not clearly a bug or a feature goes to `question`.

## Run

```
orkestra run https://github.com/dszczyt/orkestra-releases/blob/main/examples/llm-triage/workflow.yaml
```

Or from a local checkout: `orkestra run examples/reference/llm-triage/workflow.yaml`.

## Inputs

JSON payload: `{"title": "App crashes on login", "body": "Stack trace ..."}`.

## Environment

`ORKESTRA_LLM_API_KEY`: your Anthropic API key. The secret `llm` references it as `${ORKESTRA_LLM_API_KEY}`.

Environment variables in `orkestra/secret` are expanded only when the CLI loads a LOCAL workflow file. Content fetched from a URL or git never has its environment expanded, so with `orkestra run <url>` the key would stay the literal string `${ORKESTRA_LLM_API_KEY}`. To use this workflow, download it and run the local copy. The API server and workers never expand them either.

The key is sent in the `x-api-key` header through `orkestra/auth/api_key`.

## Test

```
orkestra test workflow.yaml workflow_test.yaml
```

The test mocks every HTTP call, so it needs no network.

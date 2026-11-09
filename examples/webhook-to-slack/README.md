# Webhook to Slack

Posts the `text` of a JSON payload to a Slack incoming webhook, then records the HTTP status returned by Slack to `result_webhook`.

## Run

```
orkestra run https://github.com/dszczyt/orkestra-releases/blob/main/examples/webhook-to-slack/workflow.yaml
```

Or from a local checkout: `orkestra run examples/reference/webhook-to-slack/workflow.yaml`.

## Inputs

JSON payload: `{"text": "Deploy finished", "channel": "#ops"}`. Pass it with `--input` or `--input-file`. Set `slack_webhook_url` (and `result_webhook`) in the `globals` configmap, for example with `--set`.

## Environment

None. The Slack webhook URL is a configmap key, not an env var.

## Test

```
orkestra test workflow.yaml workflow_test.yaml
```

The test mocks every HTTP call, so it needs no network.

The workflow file also declares an `orkestra/trigger` document named `on_message` (kind `http`). The http trigger adapter takes no config keys. Its `workflow` field is loaded as a path or URL by the server, so point it at your own copy if you self-host. Once the trigger is registered on a server, fire it with:

```
curl -X POST http://localhost:8080/v1/triggers/on_message/fire \
  -H 'Content-Type: application/json' \
  -d '{"input": {"text": "Deploy finished", "channel": "#ops"}}'
```

(Add your API authentication header; the URL and port depend on your deployment.)
# Orkestra

[![Latest Release](https://img.shields.io/github/v/release/dszczyt/orkestra-releases?label=release)](https://github.com/dszczyt/orkestra-releases/releases/latest)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/dszczyt/orkestra-releases/blob/main/LICENSE)
[![Go 1.26+](https://img.shields.io/badge/go-1.26%2B-00ADD8)](https://go.dev)

Deterministic workflow orchestration on PostgreSQL alone: declarative YAML, every run pinned to a commit and replayable, started from the CLI or the HTTP API, on a cron schedule, or by firing a trigger. Self-hosted, MIT licensed.

Status: v0.2.0-alpha.2 (prerelease). Expect rough edges and breaking changes before 1.0.

---

## Install

```bash
# Homebrew
brew install dszczyt/tap/orkestra

# Container image (server and worker), tagged by release, for example
docker pull ghcr.io/dszczyt/orkestra:v0.2.0-alpha.2
```

Prebuilt binaries for Linux, macOS and Windows are on the [releases page](https://github.com/dszczyt/orkestra-releases/releases). Download the archive for your platform, extract it and put `orkestra` on your `PATH`.

---

## Quickstart in 5 minutes (local, no server)

The CLI runs a workflow on your machine and keeps its state in a context file. No database needed.

### 1. Run a workflow straight from a URL

```bash
orkestra --context ctx.json run \
  https://github.com/dszczyt/orkestra-releases/blob/main/examples/github-repo-health/workflow.yaml \
  --set globals.alert_webhook=https://webhook.site/<your-token>
```

This checks a public GitHub repository (no API key) and posts a report to the webhook. Get a free test URL at [webhook.site](https://webhook.site). Change the repository with `--set globals.owner=<owner> --set globals.repo=<repo>`.

github.com blob links are rewritten to raw content. Only https URLs are accepted. By default the server accepts remote workflows from github.com and raw.githubusercontent.com only; set `ORKESTRA_REMOTE_FETCH_HOSTS` (comma separated) to allow other hosts. From the CLI, any https host is accepted. Environment variables in `orkestra/secret` are never expanded for remote workflows; download the file and run a local copy for that.

### 2. Inspect the run

```bash
orkestra --context ctx.json context
```

The context file holds each step's input, output, status and timing. It is the source of truth for the run. Without a terminal, add `--format json`.

### 3. Resume from a step

```bash
orkestra --context ctx.json run \
  https://github.com/dszczyt/orkestra-releases/blob/main/examples/github-repo-health/workflow.yaml \
  --resume-from get_open_issues
```

Orkestra re-runs from that step, even if the run already completed, and reuses the stored results of the steps before it.

---

## Quickstart: server and console

Download `compose.yaml` from the root of the [public repository](https://github.com/dszczyt/orkestra-releases), then:

```bash
docker compose up
```

This starts PostgreSQL, the Orkestra server and a worker (the worker executes the tasks the server queues). The server applies database migrations at startup (`ORKESTRA_AUTO_MIGRATE=true`), listens on port 8080 (set `SERVER_PORT` to change it) and serves:

- the web console at http://localhost:8080
- the API under `/v1`
- a health check at `/healthz`

The image tag defaults to the release this file was published with; pin another with `ORKESTRA_VERSION=<tag>`. The binary is at `/orkestra` in the image (it is not on `PATH`). Create the first user, look up the id of the `Default` organization that the migrations create, and create an API key for it:

```bash
docker compose exec server /orkestra users create you@example.com
docker compose exec postgres psql -U orkestra -tAc \
  "SELECT id FROM organizations WHERE name = 'Default'"
docker compose exec server /orkestra keys create <org-id>
```

`users create` reads the password from stdin when it is not on a terminal, and prompts twice on a terminal. `keys create` prints the token once. Send it as `Authorization: Bearer <token>` to `/v1`.

Start a run by posting the workflow YAML itself (not a URL) and its input, then read the run back:

```bash
TOKEN=<token>
curl -fsS https://raw.githubusercontent.com/dszczyt/orkestra-releases/main/examples/github-repo-health/workflow.yaml \
  | jq -Rs '{workflow: ., input: {}}' \
  | curl -fsS -X POST http://localhost:8080/v1/runs \
      -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -d @-
# {"run_id":"<id>","status":"pending","created_at":"..."}

curl -fsS -H "Authorization: Bearer $TOKEN" http://localhost:8080/v1/runs/<id>
```

The workflow's own `globals` apply, so edit `alert_webhook` in the YAML first (the downloaded copy posts to a placeholder URL). Poll the run until `status` is `completed` or `failed`.

`orkestra serve` needs PostgreSQL. Without `ORKESTRA_DATABASE_URL` it prints a hint and exits with code 2.

---

## Reference workflows

Three workflows live in [`examples/`](https://github.com/dszczyt/orkestra-releases/tree/main/examples). Each has a README and a test with mocked HTTP calls (`orkestra test workflow.yaml workflow_test.yaml`).

**github-repo-health.** Checks a public GitHub repository (archived flag, open issues and pull requests, first 100) and posts a healthy or unhealthy report to a webhook. Uses the public GitHub API, no key.

```bash
orkestra --context ctx.json run https://github.com/dszczyt/orkestra-releases/blob/main/examples/github-repo-health/workflow.yaml \
  --set globals.alert_webhook=https://webhook.site/<your-token>
```

**webhook-to-slack.** Declares an HTTP webhook trigger and posts the `text` of the incoming JSON payload to a Slack incoming webhook whose URL is a key in the `globals` configmap. Pass `--input '{"text": "Deploy finished"}'` and `--set globals.slack_webhook_url=<url>` when running it by hand.

```bash
orkestra --context ctx.json run https://github.com/dszczyt/orkestra-releases/blob/main/examples/webhook-to-slack/workflow.yaml \
  --input '{"text": "Deploy finished"}' \
  --set globals.slack_webhook_url=<url> --set globals.result_webhook=https://webhook.site/<your-token>
```

**llm-triage.** Classifies an issue as `bug`, `feature` or `question` through the Anthropic Messages API, using an `orkestra/secret` and `orkestra/auth/api_key`, then posts the label to a webhook. It needs `ORKESTRA_LLM_API_KEY` in your environment. The CLI expands `${VAR}` in secrets only for local workflow files, so download the file and run the local copy.

```bash
ORKESTRA_LLM_API_KEY=<your-key> orkestra --context ctx.json run workflow.yaml \
  --input '{"title": "App crashes on login", "body": "Stack trace ..."}'
```

---

## Core concepts

- **Workflow** (`orkestra/workflow`): names the `initial_step`. Workflows live in Git next to your code.
- **Step** (`orkestra/step`): a named node that runs one operation. Steps are stateless; all state is in the context.
- **Operation** (`orkestra/operation/http`): the atomic unit of work. `url`, `headers` and `body` are JQ expressions over `.input`, `.globals` and earlier outputs (`task("step_name").output`).
- **Transition** (`orkestra/transition`): moves execution from one step to another. `on` is a JQ boolean; with several matches the highest `priority` wins; no `on` is a catch-all.
- **ConfigMap** (`orkestra/configmap`): key-value settings available as `.globals.<key>`. Override one key with `--set globals.key=value` or replace a whole map with `--configmap name=file.yaml`.
- **Secret and auth** (`orkestra/secret`, `orkestra/auth/*`): credentials for HTTP operations (bearer, basic, API key, OAuth2).
- **Context**: where a run's state is stored. `--context ctx.json` uses one file; `--context ./ctx/` (an existing directory) uses one file per task.

```yaml
---
kind: orkestra/workflow
name: hello
spec:
  initial_step: get_user
---
kind: orkestra/operation/http
name: get_user
spec:
  method: GET
  url: '"https://jsonplaceholder.typicode.com/users/1"'
---
kind: orkestra/step
name: get_user
spec:
  operation: get_user
---
kind: orkestra/transition
name: done
spec:
  from: get_user
  on: '.input.body.id != null'
  status: completed
```

---

## CLI reference

Global flags: `--context <path>` (context file or directory), `--config <file>` (default `$HOME/.orkestra.yaml`). Put them before the subcommand. Run `orkestra <command> --help` for the full text.

| Command | Purpose |
|---|---|
| `orkestra run <workflow>` | Run a workflow from a file or https URL |
| `orkestra validate <workflow>` | Validate a workflow against the schema and check its structure |
| `orkestra test <workflow-file> [test-file]` | Run operation, step and workflow tests with HTTP mocks |
| `orkestra serve` | Start the HTTP API server and web console |
| `orkestra worker` | Run a worker that executes pending and incoming tasks |
| `orkestra migrate` | Apply pending database migrations |
| `orkestra users create <email>` | Create a user account (bootstrap) |
| `orkestra keys create\|list\|revoke` | Manage API keys |

Also available: `orkestra describe <workflow>` (structure, `--graph` for an ASCII graph), `orkestra context` (inspect a run's context) and `orkestra auth login <workflow-file> <auth-resource-name>` (OAuth2 authorization code flow in the browser).

### run

Flags: `--input <json>`, `--input-file <path>`, `--resource <path>` (repeatable), `--resume-from <step>`, `--restart`, `--initial-step <step>` (`-i`), `--stop-at <step>`, `--step` (`-s`, pause after each step), `--dry-run`, `--mock-output <step=json|step=@file>` (implies `--dry-run`), `--timeout <duration>`, `--set <name.key=value>`, `--configmap <name=path>`, `--storage <file|folder|postgres>`, `--db-url <dsn>`, `--output <text|json>`, `--verbose` (`-v`), `--quiet` (`-q`).

Input is taken from `--input`, then `--input-file`, then piped stdin. The storage backend is chosen from `--context` (an existing directory selects `folder`, anything else `file`) unless `--storage` is set.

```bash
orkestra --context ctx.json run workflow.yaml \
  --mock-output 'create_user={"id": 42}'
```

### validate and test

```bash
orkestra validate workflow.yaml
orkestra test workflow.yaml workflow_test.yaml
```

`validate` takes `--verbose`. With one argument, `test` reads the workflow and its tests from the same file. Flags: `--step`, `--stop-at <phase>`, `--resume-from <phase>`, `--verbose`.

### serve, worker, migrate

All three read `ORKESTRA_DATABASE_URL` (a PostgreSQL connection string).

- `serve`: also reads `ORKESTRA_HTTP_ADDR` (default `0.0.0.0:8080`) and `ORKESTRA_AUTO_MIGRATE` (apply migrations at startup when `true`, default off). Exits with code 2 if the database URL is unset.
- `worker`: connects to PostgreSQL, drains tasks left pending by a crash, then executes tasks as they become due until stopped. Several workers can run against the same database; task claiming prevents double execution.
- `migrate`: applies the migrations embedded in the binary. Safe to run repeatedly; it prints `Applied <file>` per migration or `Database is up to date`.

### users and keys

Both need `ORKESTRA_DATABASE_URL`.

- `users create <email>`: bootstrap the first user on a fresh deployment. Later users join through organization invites.
- `keys create <org-id> [--owner <user-id>]`: creates an organization-wide key, or a personal access token with `--owner`. The token is printed once.
- `keys list <org-id>` and `keys revoke <key-id>`.

---

## What is new in v0.2

Compared with v0.1.0 (2026-05-06). See the [release notes](https://github.com/dszczyt/orkestra-releases/releases) for details.

- Web console: runs, triggers, task inspector, step input and output.
- Trigger resources: the DSL and adapter packages cover 20 kinds (cdc, cli, cron, email, form, git, graphql, grpc, http, log, mongodb, mqtt, otel, phone, queue, slack, sms, storage, webhook, websocket). In the server only `cron` runs on its own; the push kinds (http, webhook, git, slack, grpc, form, phone, sms, otel, log, cli) can be fired with `POST /v1/triggers/{name}/fire`. The eight kinds that need an external provider (cdc, email, graphql, mongodb, mqtt, queue, storage, websocket) are not wired into the server yet and cannot be fired.
- Multi-organization auth: users, sessions, organizations, memberships, invites, org-shared API keys and personal access tokens.
- Distributed worker with heartbeat and a stale-task reaper.
- Authentication resources for HTTP operations: `orkestra/secret`, `orkestra/auth/bearer`, `basic`, `api_key`, OAuth2 client credentials and authorization code.
- Run by URL: `orkestra run <https url>`.
- Run retention: `orkestra prune --older-than 30d` deletes finished runs, and `ORKESTRA_RETENTION=30d` on the worker does it hourly.
- Blob storage for large payloads: set `ORKESTRA_BLOB_DIR` (filesystem) or `ORKESTRA_BLOB_S3_BUCKET` with the other `ORKESTRA_BLOB_S3_*` variables (S3-compatible) on the server and worker.
- `orkestra migrate` and a public compose quickstart.

---

## License

MIT, copyright 2026 Damien Szczyt.

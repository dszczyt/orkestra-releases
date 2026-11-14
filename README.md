# Orkestra

[![Latest Release](https://img.shields.io/github/v/release/dszczyt/orkestra-releases?label=release)](https://github.com/dszczyt/orkestra-releases/releases/latest)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/dszczyt/orkestra-releases/blob/main/LICENSE)
[![Go 1.27+](https://img.shields.io/badge/go-1.27%2B-00ADD8)](https://go.dev)

Deterministic workflow orchestration on PostgreSQL alone: declarative YAML, every run pinned to a version and replayable, started from the CLI, the HTTP API, MCP, a cron schedule or a trigger. It runs HTTP calls, LLM calls and tool-using agents with budgets and human approval. Self-hosted, MIT licensed.

Status: v0.2.0-alpha.7 (prerelease). Expect rough edges and breaking changes before 1.0.

---

## Install

```bash
# Homebrew
brew install dszczyt/tap/orkestra

# Container image (server and worker), tagged by release, for example
docker pull ghcr.io/dszczyt/orkestra:v0.2.0-alpha.7
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
- **Context**: where a run's state is stored. `--context ctx.json` keeps the whole run in one file (JSON or YAML). Directory contexts are not supported yet.

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

Global flags: `--context <path>` (context file), `--config <file>` (default `$HOME/.orkestra.yaml`). Put them before the subcommand. Run `orkestra <command> --help` for the full text.

| Command | Purpose |
|---|---|
| `orkestra run <workflow>` | Run a workflow from a file or https URL |
| `orkestra validate <workflow>` | Validate a workflow against the schema and check its structure |
| `orkestra test <workflow-file> [test-file]` | Run operation, step and workflow tests with HTTP mocks |
| `orkestra serve` | Start the HTTP API server and web console |
| `orkestra worker` | Run a worker that executes pending and incoming tasks |
| `orkestra migrate` | Apply pending database migrations |
| `orkestra users create <email>` | Create a user account (bootstrap) |
| `orkestra keys create\|list\|revoke` | Manage API keys (`--scope runs:approve` lets a key approve an agent's tool calls) |
| `orkestra compile <workflow>` | Print the workflow with every `orkestra/agent` expanded into plain resources |
| `orkestra fork` | Fork a local run at a task into a new context file |
| `orkestra replay` | Re-evaluate a local run offline against its workflow |
| `orkestra signal` | Send a signal to a local run |
| `orkestra prune --older-than <30d>` | Delete finished runs |
| `orkestra mcp serve` | Serve Orkestra as an MCP server over stdio: run any workflow, read stored runs |
| `orkestra mcp pin <workflow>` | List the tools of the MCP servers a workflow declares and write them into the file (`--check` for CI) |

Also available: `orkestra describe <workflow>` (structure, `--graph` for an ASCII graph), `orkestra context` (inspect a run's context) and `orkestra auth login <workflow-file> <auth-resource-name>` (OAuth2 authorization code flow in the browser).

### run

Flags: `--input <json>`, `--input-file <path>`, `--resource <path>` (repeatable), `--resume-from <step>`, `--restart`, `--initial-step <step>` (`-i`), `--stop-at <step>`, `--step` (`-s`, pause after each step), `--dry-run`, `--mock-output <step=json|step=@file>` (implies `--dry-run`), `--timeout <duration>`, `--set <name.key=value>`, `--configmap <name=path>`, `--storage <file|folder|postgres>`, `--db-url <dsn>`, `--output <text|json>`, `--verbose` (`-v`), `--quiet` (`-q`).

Input is taken from `--input`, then `--input-file`, then piped stdin. With `--context`, the run is stored in that file unless `--storage` selects another backend.

```bash
orkestra --context ctx.json run workflow.yaml \
  --mock-output 'create_user={"id": 42}'
```

### validate and test

```bash
orkestra validate workflow.yaml
orkestra test workflow.yaml workflow_test.yaml
```

`validate` takes `--verbose`. With one argument, `test` reads the workflow and its tests from the same file. Flags: `--step`, `--stop-at <phase>`, `--resume-from <phase>`, `--record` (send the requests of cassette mocks to the real host and write the cassette files), `--verbose`. See Recorded test cassettes below.

### serve, worker, migrate

All three read `ORKESTRA_DATABASE_URL` (a PostgreSQL connection string).

- `serve`: also reads `ORKESTRA_HTTP_ADDR` (default `0.0.0.0:8080`) and `ORKESTRA_AUTO_MIGRATE` (apply migrations at startup when `true`, default off). Exits with code 2 if the database URL is unset. Flags: `--allow-mcp-stdio` (accept workflows that declare a stdio MCP server, which starts a command on the host) and `--mcp-allowed-origin <origin>` (repeatable, or `ORKESTRA_MCP_ALLOWED_ORIGINS`, comma separated). It also serves MCP over HTTP at `POST /mcp` and the trigger ingress at `/hooks/{name}`.
- `worker`: connects to PostgreSQL and executes tasks as they become due until stopped. Several workers can share a database; they claim tasks with `FOR UPDATE SKIP LOCKED`, up to `ORKESTRA_WORKER_CONCURRENCY` at once (default 16). A task whose worker dies goes back to the queue. Flags: `--event-mode <notify|wal>`, `--allow-mcp-stdio`, `--max-map-items <n>` (default 1000). Other settings: `ORKESTRA_HEARTBEAT_INTERVAL` (default 15s), `ORKESTRA_SHUTDOWN_GRACE` (default 30s).
- `migrate`: applies the migrations embedded in the binary. Safe to run repeatedly; it prints `Applied <file>` per migration or `Database is up to date`.

Workers wake on a PostgreSQL event, chosen by `ORKESTRA_EVENT_MODE` or `--event-mode`:

- `notify` (default): `LISTEN/NOTIFY`. No replication setup, so the worker does not need `wal_level=logical`. It needs a direct connection: a pooler in transaction mode drops `LISTEN`.
- `wal`: logical replication on a temporary slot per worker. Needs `wal_level=logical` and a `REPLICATION` role.

The event only wakes workers up. A 30 second fallback poll covers a lost event. Step execution is at least once and completion is exactly once, so make the operations behind your steps idempotent. `POST /v1/runs` accepts an `Idempotency-Key` header. `orkestra worker` and `orkestra serve` no longer use the old shared `orkestra_tasks` replication slot and drop it at startup. A trigger of kind `cdc` still needs `wal_level=logical` on the database it watches.

The worker refuses HTTP steps, MCP `http` servers and OAuth2 token requests to private, loopback or link-local addresses. If your workflows call internal services, set `ORKESTRA_ALLOW_PRIVATE_HTTP=1` on the worker.

### users and keys

Both need `ORKESTRA_DATABASE_URL`.

- `users create <email>`: bootstrap the first user on a fresh deployment. Later users join through organization invites.
- `keys create <org-id> [--owner <user-id>] [--scope <scope>] [--label <label>]`: creates an organization-wide key, or a personal access token with `--owner`. The token is printed once. `--scope runs:approve` lets the key approve an agent's tool calls, so leave it off keys you give to agent clients. `--label` (repeatable) lets the key read the steps whose `metadata.access` lists it, over REST and MCP.
- `keys list <org-id>` and `keys revoke <key-id>`.

### compile, fork, replay, signal

```bash
orkestra compile workflow.yaml
orkestra fork --context run.ctx --at <task-id> --out fork.ctx --input '{"city": "Lyon"}'
orkestra replay --context run.ctx --workflow workflow.yaml
orkestra signal --context run.ctx --name approval --key A-1042 --payload '{"approved": true}'
```

- `compile` prints the YAML a run is pinned to. The output is byte-stable and `${VAR}` stays unexpanded. A workflow with no agent is printed as written.
- `fork` (needs `--at <task-id>` and `--out <file>`) copies the completed top-level tasks before the `--at` task, with their outputs, and continues from a fresh copy of that task. The source run is not modified. `--input` replaces the input of the fork task, `--workflow` forks under another definition. Continue with `orkestra run <workflow> --context <out>`. On a server, use `POST /v1/runs/{run_id}/fork`.
- `replay` recomputes every transition and input transform from the recorded outputs. No operation runs. It exits 0 when the run follows the workflow, and 1 at the first difference, naming the task. With `--workflow`, the difference marks the task to fork from.
- `signal` (needs `--name`; `--key`, `--payload` and `--run` are optional) wakes a wait of a local run. Run `orkestra run` again on the same context to continue. On a server, use `POST /v1/runs/{run_id}/signals/{name}`.

### mcp serve

Register Orkestra once in the agent client. The client starts it over stdio, so no server runs, and the agent names the workflow and the context file at each call:

- `orkestra_run`: run a workflow (file path or https URL) with an input. With `context`, the run is kept in that context file; `restart` and `resume_from` work as in `orkestra run`.
- `orkestra_get_run`: the run stored in a context file and its tasks (step, status, timing, retries, error).
- `orkestra_get_task`: one task with its input and output.

Local workflow and context files must be under `--root`, which defaults to the directory where the client starts Orkestra (symlinks are resolved). https workflows are allowed. A context file must end in `.json`, `.yaml` or `.yml`, and an existing file is replaced only if it already holds an Orkestra run. Secret values of the workflow are redacted from every result.

```bash
claude mcp add orkestra -- orkestra mcp serve --root /path/to/project
```

A step can keep its data away from some agents, for example health data that only a local LLM may read:

```yaml
kind: orkestra/step
name: diagnose
metadata:
  access: [local-llm]
spec:
  operation: diagnose
```

Register each agent client with its own key, for example `orkestra mcp serve --key local-llm` for the local model and no key for a cloud assistant. A client whose key is not listed never gets that step's input, output or error text, nor the input or error text of any later task of the run, since restricted data flows into them. Resuming a context requires the workflow of its stored run. Task data is withheld when that workflow cannot be loaded, or is a remote URL whose rules may have changed since the run. This filters what goes through MCP only: an agent that can read the context file on disk is not stopped.

Flags of `mcp serve`: `--root`, `--key`, `--timeout` (maximum duration of one run, default 5m). `orkestra_get_run` also lists the tool calls waiting for approval. An approval is never given through MCP.

### mcp pin

```bash
orkestra mcp pin workflow.yaml
orkestra mcp pin --check workflow.yaml
```

`pin` connects to every `orkestra/mcp/server` of the file (or the one named by `--server`), calls `tools/list` once and writes the protocol revision and the tools into the file. At run time Orkestra never lists tools, so a server cannot add or change a tool under a running agent. A stdio server is started on your machine to be listed. Server annotations such as `readOnlyHint` are untrusted and decide nothing. `--check` writes nothing and exits non-zero when the live list differs from the pinned one, for CI.

---

## Workflow features

Each feature below is a resource in the workflow YAML. `orkestra validate` checks them and `orkestra test` runs them offline.

- **LLM operation** (`orkestra/operation/llm`): a model call through the Anthropic API with `model`, `max_tokens`, `system` and `user` (jq), and optional `effort`, `cache` and `pricing`. The output has `text`, `stop_reason`, `usage` and `cost_usd`. A refusal fails the task with `_error.type` `llm.refusal`. With PostgreSQL, every billed call writes a usage row in the same transaction as the task result. Set the key in `ORKESTRA_LLM_API_KEY` or an `orkestra/secret`.
- **Conversations**: `conversation: <name>` on LLM operations makes a call see the earlier turns of a run-scoped log, and `tool_results` answers the previous turn's tool calls. Operations that share a conversation must share `provider` and `system`.
- **Agents** (`orkestra/agent`): a tool-using agent in one resource (`provider`, `auth`, `model`, `prompt`, `budget`, `tools`). The loader expands it into plain steps, transitions, tools and a budget, so nothing is hidden: `orkestra compile` prints the result. Tools can be backed by an `operation`, by an MCP server (`mcp: <server>`), or by another workflow (`workflow: <path>`, a sub-agent whose budget counts against the caller's). Tool calls of one turn run in parallel by default (`parallel_tools`, `max_parallel_tools`).
- **Budgets** (`orkestra/budget`): limit the tasks of its `steps` by `turns`, `tokens`, `cost_usd` or `wall_clock`. A task over a limit fails without running with `_error.type` `limit`. `on_exhausted: <step>` routes it. `.budget` is available in jq. `orkestra validate` refuses a loop that calls a model unless a budget covers it.
- **Approvals**: a tool with `approval: required` stops the run until a person answers with the signal `approval`. The sender must hold the `runs:approve` scope (`orkestra keys create --scope runs:approve`). A denial or a timeout (`approval_timeout`, default 1h) gives the model an error result. Locally, answer with `orkestra signal`. On a server, `GET /v1/runs/{id}` lists `pending_approvals` and the answer goes to `POST /v1/runs/{run_id}/signals/approval`. MCP tools must say `approval: none` or `approval: required`.
- **Map and parallel**: `map` runs a step once per item of a jq array (with `concurrency`), `parallel` runs a fixed list of named branches. The output is the array, or the object keyed by branch name. `on_failure: fail` (default) or `continue` sets what a failed child does. A server caps a map at 1000 items.
- **Sub-workflows** (`orkestra/operation/workflow`): runs another workflow file as a child run. A failed child fails the step with `_error.type: child_run`. The hash of the parent covers the child files, so a child runs the definition its parent started with.
- **Timers and signals** (`orkestra/operation/wait`): `duration` is a timer. `signal`, `key` and `timeout` park the run until `POST /v1/runs/{run_id}/signals/{name}` (or `orkestra signal`) arrives.
- **Failure handling**: a `status: failed` transition routes a failed step. `on_failure: <step>` on the workflow starts a step after the first unhandled failure, for cleanup and notice. `compensate: <step>` on a step undoes it: when the run fails, each completed task gets a compensation task, newest first. The run ends `failed`.
- **Retries and timeouts**: a step accepts `retry` and `timeout` (default `5m`, per attempt). A retry is stored in the task and survives a restart.
- **Fork and replay**: `orkestra fork` restarts a run from a task, `orkestra replay` checks a recorded run against a workflow without running anything.

### Recorded test cassettes

An `orkestra/test/mock/http` with `cassette: <file>` replays recorded responses. `orkestra test --record` sends its requests to the real host and writes the file. Request headers are never stored and bodies go through the secret redactor. With `match: strict` (default) a changed request body fails the test, `match: order` ignores the body. A missing cassette is an error and the network is never used on replay. Expectations can read `.trajectory`, `.usage` and `.history`. The test clock is virtual, and a phase can move it with `advance_clock`. `orkestra/test/mock/mcp` stands in for an MCP server.

---

## MCP, both ways

- **Orkestra as an MCP server.** Over stdio, `orkestra mcp serve` runs locally with no server to deploy. Over HTTP, `orkestra serve` answers `POST /mcp` with a bearer API key and the same three tools (`orkestra_run`, `orkestra_get_run`, `orkestra_get_task`). There is no tool that sends a signal, so an agent cannot approve its own tool calls. A browser `Origin` must be allowed with `--mcp-allowed-origin`.
- **MCP servers as tools.** A workflow declares an `orkestra/mcp/server` (transport `http` or `stdio`), `orkestra mcp pin` writes its tools into the file, and an agent uses them with `mcp: <server>` in its tool list. A stdio server runs a command, so `orkestra serve` and `orkestra worker` refuse it unless started with `--allow-mcp-stdio`.

---

## Triggers

A trigger starts a workflow run from an outside event. Create one with `POST /v1/triggers` (API key) with `name`, `kind`, `workflow` and an optional `config`. The workflow is pinned when the trigger is created. `POST /v1/triggers/{name}/fire` starts a run from a push trigger by hand.

- **Push kinds** wait for `POST /hooks/{name}`, which needs no Orkestra API key: webhook, git, slack, http, grpc, form, phone, sms, otel, log, cli. The trigger authenticates the caller, either with a signature (`webhook` with `secret`, `git` and `slack` with `signing_secret`) or with an `ingress_token` of at least 16 characters, sent in `X-Orkestra-Token`, `Authorization: Bearer` or `?token=`. Any refusal is the same `404`. A `websocket` trigger takes `GET /hooks/{name}/ws` with a required `ingress_token`.
- **Listener kinds** run inside `orkestra serve`: cron, cdc (PostgreSQL), storage (S3), queue (Kafka, NATS), email (IMAP), mqtt, mongodb, graphql. A listener that stops is restarted with a backoff from 1 second to 1 minute.

Two operator settings on the server apply to listener kinds only:

- `ORKESTRA_TRIGGER_ALLOW_PRIVATE=true` lets listeners connect to private and loopback addresses. By default a trigger config cannot point at the server's own network. Use it only when all tenants are trusted.
- `ORKESTRA_TRIGGER_ALLOW_INSECURE_TLS=true` lets a trigger config turn TLS verification off or send credentials without TLS.

---

## What is new in v0.2

Compared with v0.1.0 (2026-05-06). See the [release notes](https://github.com/dszczyt/orkestra-releases/releases) for details.

- Web console: runs, triggers, task inspector, step input and output.
- LLM operation, agents (`orkestra/agent`, `orkestra compile`), conversations, tools and budgets, with usage and cost recorded per call.
- Approvals for an agent's tool calls, with the `runs:approve` key scope.
- Recorded test cassettes (`orkestra test --record`), trajectory and usage expectations, a virtual test clock and MCP mocks.
- `map`, `parallel`, sub-workflows, timers and signals, `on_failure` and compensation.
- `orkestra fork`, `orkestra replay` and `orkestra signal`.
- MCP both ways: `orkestra mcp serve` over stdio, `POST /mcp` on the server, and MCP servers as agent tools with `orkestra mcp pin`. Per-step access keys keep sensitive step data away from agents that are not authorized to read it.
- Triggers: 20 kinds. Push kinds are reachable at `POST /hooks/{name}`, listener kinds run inside the server, and two operator settings (`ORKESTRA_TRIGGER_ALLOW_PRIVATE`, `ORKESTRA_TRIGGER_ALLOW_INSECURE_TLS`) control what a trigger config may reach.
- Durable server mode: a step's output, status, next task and run close are written in one transaction. Step execution is at least once, completion exactly once. Several workers can share a database. A task whose worker dies is requeued, and retries survive a restart.
- The worker no longer needs `wal_level=logical`: the default event mode is `notify`, and `wal` is optional.
- The worker refuses HTTP calls to private addresses unless `ORKESTRA_ALLOW_PRIVATE_HTTP=1`.
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

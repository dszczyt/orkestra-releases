# Orkestra

[![Latest Release](https://img.shields.io/github/v/release/dszczyt/orkestra-releases?label=release)](https://github.com/dszczyt/orkestra-releases/releases/latest)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/dszczyt/orkestra-releases/blob/main/LICENSE)
[![Go 1.26+](https://img.shields.io/badge/go-1.26%2B-00ADD8)](https://go.dev)

**Deterministic workflow orchestration engine.** Define multi-step workflows in YAML. Run them locally, in CI, or from a cron job. Resume from any step. No server. No scheduler. No infrastructure.

---

## Install

### Homebrew

```bash
brew install dszczyt/tap/orkestra
```

### go install

```bash
go install github.com/dszczyt/orkestra/cmd/orkestra@latest
```

Binary is installed as `orkestra` in your `$GOPATH/bin`.

### Download binary

```bash
# Linux amd64 example (replace v0.1.0 with the actual version)
curl -L https://github.com/dszczyt/orkestra-releases/releases/latest/download/orkestra_v0.1.0_linux_amd64.tar.gz | tar xz
sudo mv orkestra /usr/local/bin/
```

Binaries for macOS (arm64/amd64) and Windows are available on the [releases page](https://github.com/dszczyt/orkestra-releases/releases/latest).

---

## Quickstart (5 minutes)

This example runs a minimal workflow against [JSONPlaceholder](https://jsonplaceholder.typicode.com), a free fake REST API — no signup required.

### 1. Save the workflow

Save the following as `user_onboarding.yaml`:

```yaml
---
kind: orkestra/workflow
name: user_onboarding
spec:
  initial_step: create_user
---
kind: orkestra/configmap
name: globals
spec:
  api_base: "https://jsonplaceholder.typicode.com"
  notify_webhook: "https://webhook.site/your-token-here"
---
kind: orkestra/operation/http
name: create_user
spec:
  method: POST
  url: '"\(.globals.api_base)/users"'
  headers: '{"Content-Type": ["application/json"]}'
  body: '{ name: .input.name, email: .input.email }'
---
kind: orkestra/step
name: create_user
spec:
  operation: create_user
---
kind: orkestra/transition
name: t_user_created
spec:
  from: create_user
  on: '.input.body.id != null'
  to: done
```

### 2. Run it

```bash
orkestra --context ctx.json run user_onboarding.yaml \
  --input '{"name": "Alice Smith", "email": "alice@example.com"}'
```

You should see output like:

```
[create_user] running...
[create_user] done (201 Created)
workflow finished: done
```

### 3. Inspect the context

```bash
cat ctx.json | jq '.tasks[] | {step: .step, status: .status, output: .output}'
```

Expected output:

```json
{
  "step": "create_user",
  "status": "completed",
  "output": {
    "content_type": "application/json",
    "data": {
      "status_code": 201,
      "body": {
        "id": 11,
        "name": "Alice Smith",
        "email": "alice@example.com"
      }
    }
  }
}
```

### 4. Resume from a step

If a step fails mid-run, or you want to re-run from a specific point without discarding prior results:

```bash
orkestra --context ctx.json run user_onboarding.yaml --resume-from create_user
```

Orkestra replays from `create_user`, using the existing context for any preceding steps.

---

## Core Concepts

### Workflow

A workflow is a YAML document that defines the entry point of your execution graph. It references steps by name and determines where execution begins.

Workflows are stored in Git alongside your code. Each run is associated with the workflow file path, making versioning and auditing straightforward.

```yaml
kind: orkestra/workflow
name: my_workflow
spec:
  initial_step: first_step
```

### Step

A step is a named execution unit that delegates to a single operation. Steps are the nodes in your workflow graph — transitions connect them.

Steps are stateless. All state is stored in the context. This makes steps safe to retry, replay, or reorder.

```yaml
kind: orkestra/step
name: fetch_data
spec:
  operation: get_user
```

### Operation

An operation is the atomic work unit — it performs exactly one thing. Currently, Orkestra ships with `orkestra/operation/http` for HTTP calls.

The `url`, `body`, and `headers` fields are JQ expressions evaluated against the current input and context. You can access outputs from previous steps with `task("step_name").output`. The `run()` function (no arguments) returns the current run record (`id`, `status`, `input`, `workflow_path`).

```yaml
kind: orkestra/operation/http
name: get_user
spec:
  method: GET
  url: '"\(.globals.api_base)/users/1"'
```

To chain outputs from a previous step:

```yaml
kind: orkestra/operation/http
name: create_todo
spec:
  method: POST
  url: '"\(.globals.api_base)/todos"'
  body: '{ userId: task("create_user").output.body.id, title: "Onboarding task" }'
```

### Transition

A transition defines how execution moves from one step to another. The `on` field is a JQ expression that must evaluate to a boolean. When multiple transitions match, the one with the highest `priority` wins.

Transitions support conditional routing, enabling branching logic without custom code.

```yaml
kind: orkestra/transition
name: t_success
spec:
  from: create_user
  priority: 10
  on: '.input.body.id != null'
  to: send_welcome_email
```

A transition with no `on` field (or `on: 'true'`) acts as a catch-all fallback when placed at lower priority.

### ConfigMap

A configmap is a named collection of key-value pairs accessible in every JQ expression as `.globals.<key>`. Use configmaps to externalize URLs, feature flags, thresholds, and other configuration.

Override individual keys at runtime with `--set`, or replace an entire configmap with `--configmap`.

```yaml
kind: orkestra/configmap
name: globals
spec:
  api_base: "https://api.example.com"
  retry_limit: 3
```

Access in an operation:

```jq
"\(.globals.api_base)/users"
```

Override at runtime:

```bash
orkestra --context ctx.json run workflow.yaml --set globals.api_base=https://staging.api.example.com
```

### Context

The context is a file or directory that Orkestra uses to persist each step's input, output, status, and timing. It is the source of truth for a run.

- **File**: `--context ctx.json` — single JSON file, suitable for short workflows. Auto-detected when the path does not point to an existing directory.
- **Folder**: `--context ./ctx/` — one file per task, better for long workflows or when you want per-step diffs in version control. Auto-detected when the path is a directory.

The context enables Orkestra's key capability: deterministic resumption. Any step can be re-run from any point without re-executing prior steps.

---

## Why Orkestra?

Orkestra occupies a unique position: it brings workflow orchestration to individual developers and small teams without requiring any infrastructure.

|  | Orkestra | Airflow | Temporal | Zapier / n8n |
|---|---|---|---|---|
| **Infrastructure** | none | scheduler + workers + DB | server + workers | SaaS / self-hosted |
| **Workflow definition** | YAML | Python DAG | SDK code | UI / JSON |
| **Local execution** | ✓ | ✗ | ✗ | ✗ |
| **Git-native versioning** | ✓ | partial | ✗ | ✗ |
| **Resume from any step** | ✓ | limited | ✓ (built-in) | ✗ |
| **Deterministic replay** | ✓ | ✗ | ✓ | ✗ |
| **Developer target** | backend / DevOps | data engineers | backend / distributed | non-technical |

Orkestra is not a replacement for Temporal or Airflow in large distributed systems. It is designed for the use cases those tools make prohibitively complex: scripted automation, API orchestration, data pipelines, and CI/CD workflows that benefit from durability and replayability without infrastructure overhead.

---

## Examples

The `examples/` directory in this repository contains ready-to-run workflows. Below are three highlights.

### currency_monitor.yaml

HTTP polling with conditional transitions. Fetches the EUR/USD exchange rate from [Frankfurter](https://www.frankfurter.app), posts an alert webhook if the rate crosses a configured threshold, and logs an OK message otherwise.

Demonstrates priority-based transitions and configmap-driven thresholds.

```bash
orkestra --context ctx.json run currency_monitor.yaml
```

To override the threshold at runtime:

```bash
orkestra --context ctx.json run currency_monitor.yaml --set globals.alert_threshold=1.10
```

### user_onboarding.yaml

Multi-step workflow with chained outputs. Creates a user, a todo item, a welcome post, and a notification — each step using `task("step_name").output.body.id` to reference the previous step's result.

Demonstrates output chaining and sequential API calls.

```bash
orkestra --context ctx.json run user_onboarding.yaml \
  --input '{"name": "Alice", "email": "alice@example.com", "username": "alice"}'
```

### github_repo_health.yaml

API aggregation with early exit. Fetches repository metadata and open issue count from the GitHub API. Short-circuits to an "unhealthy" terminal step if the repository is archived, skipping further checks.

Demonstrates parallel-style aggregation and conditional early exit via transitions.

```bash
orkestra --context ctx.json run github_repo_health.yaml
```

To check a different repository:

```bash
orkestra --context ctx.json run github_repo_health.yaml --set globals.repo=owner/repo
```

---

## CLI Reference

### Global flags

These flags apply to all commands and must be placed before the subcommand.

| Flag | Description |
|------|-------------|
| `--context <path>` | Path to context file or directory (auto-selects `file` or `folder` backend) |

### `orkestra run <workflow>`

Executes a workflow file. `<workflow>` is a path to a YAML file (or a multi-document YAML file).

```bash
orkestra --context ctx.json run <workflow> [flags]
```

#### Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--input <json>` | — | Inline JSON input for the workflow |
| `--input-file <path>` | — | Load input from a JSON file (stdin also accepted) |
| `--resource <path>` | — | Load extra resource file (repeatable); merges steps, operations, transitions |
| `--resume-from <step>` | — | Resume execution from a specific step name |
| `--restart` | false | Ignore existing context and start a fresh run |
| `--initial-step <step>` | — | Override `initial_step` from the workflow definition |
| `--stop-at <step>` | — | Halt execution before this step |
| `--step` / `-s` | false | Pause and prompt after each step (interactive mode) |
| `--dry-run` | false | Resolve transitions without executing operations |
| `--mock-output <step=json>` | — | Mock a step's output (implies `--dry-run`); use `step=@file` for file input |
| `--timeout <duration>` | 0 (none) | Global run timeout, e.g. `30s`, `5m` |
| `--set <name.key=value>` | — | Override a configmap field at runtime |
| `--configmap <name=path>` | — | Replace an entire configmap from a file |
| `--storage <backend>` | auto | Force storage backend: `file` or `folder` (auto-detected from `--context`) |
| `--db-url <dsn>` | — | PostgreSQL DSN — required for `--storage postgres` *(coming soon)* |
| `--output <format>` | `text` | Final result format: `text` or `json` |
| `--verbose` / `-v` | false | Print full inputs and outputs per step |
| `--quiet` / `-q` | false | Suppress all output except errors |

#### Input resolution order

When resolving the workflow input, Orkestra checks in this order:

1. `--input` (inline JSON string)
2. `--input-file` (path to a JSON file)
3. stdin (if not a terminal)

#### Storage auto-detection

When `--storage` is not set, Orkestra inspects the `--context` path:

- If the path points to an existing directory → `folder` backend
- Otherwise → `file` backend (creates the file if it does not exist)

Use `--storage` explicitly to force a backend regardless of the path.

#### Mock outputs

`--mock-output` lets you stub a step's output without executing its operation. The value can be an inline JSON string or a path prefixed with `@`:

```bash
# Inline JSON
orkestra --context ctx.json run workflow.yaml \
  --mock-output 'create_user={"id": 42, "name": "Alice"}'

# From file
orkestra --context ctx.json run workflow.yaml \
  --mock-output 'create_user=@fixtures/create_user.json'
```

`--mock-output` implies `--dry-run` — operations for non-mocked steps are also skipped.

---

## License

```
MIT © 2026 Damien Szczyt
```

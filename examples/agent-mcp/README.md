# Agent with an MCP tool

A Claude agent whose only tool is served by an MCP (Model Context Protocol) server. `mcp: docs` in the tool list is all the agent needs: the description and the input schema come from the pinned server resource, and the compiler generates the `orkestra/operation/mcp` that makes the call.

## Pin the server

The `url` in `workflow.yaml` is a placeholder. Point it at your server, then:

```
orkestra mcp pin workflow.yaml
```

This connects once, calls `tools/list`, and writes the protocol revision and the tools into the `orkestra/mcp/server` resource. Review the change like any other. At run time Orkestra never lists tools, so a server cannot add or redefine a tool under a running agent, and `orkestra validate` can name every callable tool offline. In CI, `orkestra mcp pin --check workflow.yaml` exits non-zero when the live list differs from the pinned one.

Server annotations such as `readOnlyHint` are untrusted. They appear as comments above each pinned tool and decide nothing.

## Approval

Every MCP tool an agent uses says `approval: none` or `approval: required`; there is no default, and `orkestra validate` fails without one. `required` means a person approves each call before it runs; see `agent-approval` for how. Server annotations such as `readOnlyHint` do not change that: they are untrusted.

## stdio servers

A server can also be a local process:

```yaml
spec:
  transport: stdio
  command: [npx, -y, "@modelcontextprotocol/server-filesystem", /data]
  env:
    API_TOKEN: {secret: fs, key: token}
```

`orkestra run` and `orkestra test` allow it. `orkestra serve` and `orkestra worker` refuse it unless started with `--allow-mcp-stdio`, because it runs a command on that host.

## Test it

```
ORKESTRA_LLM_API_KEY=test orkestra test workflow.yaml workflow_test.yaml
```

`orkestra/test/mock/mcp` stands in for the server named in `spec.server`: no process, no connection. A tool the mock does not list answers with a protocol error, and a server with no mock fails the test instead of reaching the real one.

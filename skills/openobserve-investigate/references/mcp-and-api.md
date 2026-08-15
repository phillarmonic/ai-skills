# OpenObserve MCP and API

## MCP connection

- Endpoint: `https://host/api/{org_id}/mcp`
- Transport: Streamable HTTP
- Authentication: `Authorization: Basic <base64(email:password)>`
- Content type: `application/json`
- Request plain JSON with `Accept: application/json`; clients may instead use
  `text/event-stream`, whose response contains `data: <json>` events.

OpenObserve supports the normal `initialize`, `ping`, `tools/list`, and
`tools/call` JSON-RPC methods. Its server is usable as a stateless request/
response endpoint, but configured MCP clients should perform their normal
handshake.

`tools/list` intentionally returns a small catalog: `tool_search`, `tools_call`,
and common pinned tools. Call `tool_search` through `tools/call`; its first text
content is a JSON string containing matching tool schemas. Execute a discovered
tool through `tools_call`. Pinned tools can be called directly by name.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "tool_search",
    "arguments": {"query": "trace DAG", "limit": 5}
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "tools_call",
    "arguments": {
      "tool": "GetTraceDAG",
      "args": {"org_id": "default", "stream_name": "traces", "trace_id": "..."},
      "detail": "summary"
    }
  }
}
```

Check both `result.structuredContent` and text blocks in `result.content`.
Text blocks frequently contain serialized JSON. Check JSON-RPC `error`, HTTP
status, and tool-level `isError`; HTTP 200 alone does not prove tool success.

## REST fallback

Use ordinary `curl` or another HTTP client only when the configured MCP
connection is unavailable or its transport is under investigation. Supply
credentials through the environment or the client's secret store instead of
embedding them in commands or files.

The core read-only endpoints are:

- List streams: `GET /api/{org}/streams?type=logs&fetchSchema=false`
- Get schema: `GET /api/{org}/streams/{stream}/schema?type=logs`
- Search: `POST /api/{org}/_search?type=logs`

Search body:

```json
{
  "query": {
    "sql": "SELECT _timestamp, level, message FROM \"app_logs\" ORDER BY _timestamp DESC",
    "start_time": 1760000000000000,
    "end_time": 1760003600000000,
    "from": 0,
    "size": 50
  },
  "agent_options": {
    "mode": "partition",
    "output_format": "csv"
  }
}
```

`start_time` and `end_time` are required epoch microseconds. Use `partition`
for wide top-N/aggregation searches and let the server partition one request.
Search errors may include `hint` and `suggestions`; use them before changing the
investigation strategy.

## Client configuration

Most clients need an HTTP MCP entry with the endpoint and authorization header.
Keep the token in environment-backed secret configuration. Register one server
per organization when investigating multiple environments so org boundaries
remain explicit.

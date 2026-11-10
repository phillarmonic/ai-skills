---
name: mcpdev
description: >-
  Test MCP servers with the mcpdev CLI: connect over stdio or Streamable HTTP,
  inspect tools/prompts/resources, call them, snapshot and semantically diff
  the public interface, run YAML/JSON scenarios, and watch a rebuilding binary
  without restarting an AI host. Use when a user has an MCP binary or --url,
  mcpdev.toml profiles, inspect/call/prompt/read/snapshot/diff/test/watch,
  --command argv, protocol auto/2025-11-25/2026-07-28, exit codes 2/10/20/30/40/60/70,
  incomplete catalogs, CI interface regression, or phillarmonic/setup-mcpdev.
  Do not use this skill to change mcpdev's own source tree or package layout.
---

# mcpdev

Use the **mcpdev** binary as a client against **someone else's** MCP server
(the binary, `go run` argv, or Streamable HTTP URL under test). It does not
open a browser Inspector, start a daemon, or reconfigure Cursor/Claude.

There is no `--transport grpc`. HTTP+SSE is `--allow-http-sse` only.

## Establish the client

```bash
mcpdev --format json version
```

Install if needed: `go install github.com/phillarmonic/mcpdev/cmd/mcpdev@latest`
(Go 1.27). Pin `@v0.1.1` when a pipeline needs a fixed snapshot schema and
protocol matrix. Release archives and identity rules:
<https://phillarmonic.github.io/mcpdev/reference/installation/>

`--format json version` reports `version`, `commit`, `build_time`,
`build_time_policy`, `snapshot_schema`, `goos`, and `goarch`. `build_time_policy`
is `git_commit` for release archives, `vcs_time` for `go install` of a module
version, and `unknown` for an unstamped local build.

Pick **exactly one** live target per command:

```bash
mcpdev inspect --command ./bin/server
mcpdev inspect --command go --command run --command ./cmd/server
mcpdev inspect --url http://127.0.0.1:8080/mcp
mcpdev inspect --profile local
```

`--command` / `-c` is an argv array (repeat the flag, or `--` then args). It is
never `sh -c` and never a string taken from server output. `--command` plus
`--url` is exit `2`.

Optional: nearest `mcpdev.toml` / `.mcpdev.toml`, then user
`~/.config/mcpdev/config.toml`. Flags beat `MCPDEV_*` env, which beat files.

```toml
protocol = "auto"
timeout = "10s"

[profiles.local]
command = ["go", "run", "."]
```

`command` in config is an argv array. `command = "go run ."` is invalid.
Put tokens in `header_env` or `${TOKEN:?}`, not committed literals.

## CI: install the released binary

In GitHub Actions, [`phillarmonic/setup-mcpdev`](https://github.com/phillarmonic/setup-mcpdev)
installs the published archive, verifies it against `SHA256SUMS`, caches it, and
adds it to `PATH` — no Go toolchain needed. The action tag and the CLI version
are separate.

```yaml
- uses: phillarmonic/setup-mcpdev@v1
  with:
    version: 'v0.1.1'   # default: latest; also accepts v0 or v0.1
- run: mcpdev version
```

Outputs are `version` (binary string, no `v` prefix), `path`, and `cache-hit`.
The step fails when the installed binary reports a version other than the
requested tag. Gate a committed baseline in the same job:

```yaml
- run: mcpdev diff testdata/mcpdev/baseline.json --command ./bin/server
```

Exit `40` is an interface change at or above `--fail-on`; exit `60` is an
incomplete capture. Both fail the step by default. Details:
<https://phillarmonic.github.io/mcpdev/guides/github-actions/>

## Default loop: inspect, then exercise

Every `inspect` / `call` / `test` case starts a **new** stdio child.

```bash
mcpdev inspect --command ./bin/server
mcpdev inspect --command ./bin/server tools
mcpdev inspect --command ./bin/server --name echo --expand
mcpdev --format json --command ./bin/server inspect | jq '{ok, incomplete, protocol}'
```

`--name` keeps matching names/URIs and `--expand` adds input schemas in human
output; neither rewrites collection status. `--max-bytes` bounds stdout bodies
(default `65536`) while `--save` still writes the complete body.

`--protocol auto` (default) probes `server/discover` (2026-07-28), then may
fall back to `initialize` (2025-11-25). Pin `--protocol 2025-11-25` or
`--protocol 2026-07-28` when you already know the era.

Human `inspect` labels `status=complete` (including zero items), `unsupported`,
`failed`, or `incomplete`. A stopped page walk is exit **60**, not an empty
success. `--allow-incomplete` keeps `incomplete: true` in JSON and exits `0`.

Then invoke:

```bash
mcpdev -c ./bin/server call echo --args '{"text":"hi"}'
mcpdev -c ./bin/server call echo --args @./args.json
mcpdev -c ./bin/server call echo --args-file ./args.json
mcpdev -c ./bin/server prompt greet --arg name=Ada
mcpdev -c ./bin/server read fixture://stable/note
mcpdev -c ./bin/server read --template 'items/{id}' --arg id=42
mcpdev --format json -c ./bin/server read bin://blob --save ./out.bin
```

`--arg` for `call` is `KEY=JSON`. For `prompt` and `read --template` it is a raw
string. Server `isError` is still RPC success (exit `0`) unless
`--fail-on-tool-error` (exit `30`).

stdout is **only** the result (`human` or one JSON object). Diagnostics, child
logs, `--trace`, `--verbose`, and watch status go to stderr. Pipe JSON; never
scrape stderr in CI.

## Lock the public interface

```bash
mcpdev snapshot --command ./bin/server --out testdata/mcpdev/baseline.json
mcpdev diff testdata/mcpdev/baseline.json --command ./bin/server
mcpdev --format json diff testdata/mcpdev/baseline.json testdata/mcpdev/after.json
```

Incomplete snapshot still writes a **new** `--out` path and exits `60`. It does
not replace an existing baseline. Diff outcomes: `0` no-change or below
`--fail-on` (default `additive`), `40` threshold-failure, `60` indeterminate.

Live-to-live without files:

```bash
mcpdev diff --from-command ./bin/old --to-command ./bin/new
mcpdev diff --from-url http://127.0.0.1:8080/mcp --to-url http://127.0.0.1:8081/mcp
mcpdev diff --from-profile old --to-profile new
```

`--ignore` drops metadata at a JSON Pointer. `--rename kind:from=to` maps
identities. `--no-exit-on-change` keeps exit `0` on `threshold-failure`.

## Declarative scenarios

```bash
mcpdev --command ./bin/server test ./scenarios/stdio.yaml
mcpdev --format json --jobs 2 --junit ./tmp/junit.xml test ./scenarios
```

CLI `--command` / `--url` / `--profile` override the file `target`. Invalid
files fail at load (exit `2`) before connect. Assertion failure is `30`;
unevaluable assertion is `60`. Write suites as in
[references/scenarios.md](references/scenarios.md).

Scenario steps use `call: {tool, arguments}` (not `name`), and assertion paths
read a normalized envelope: `$.ok`, `$.result.isError`, `$.result.text`,
`$.result.structured.*`, and flat `$.result.tools[*]` (not the `.items` nesting
of `--format json inspect`). A bare `$.isError` silently skips (exit `60`).
Copyable templates live in `assets/scenarios/` and `assets/mcpdev.toml`.

`--jobs` defaults to 4 (cases only; steps inside a case stay serial).
`--junit` maps assertion problems to JUnit `failure` and connection, protocol,
and cleanup problems to `error`.

`--replay PATH` plays a sanitized fixture (exclusive with a live target).
`--record PATH` writes one after a live command (path required; `-` rejected).

## Watch a rebuilding server

Foreground only. `--rebuild` is an argv array. Write to `--staging`, promote
onto `--artifact`. Failed builds keep the last good file and process.

```bash
mcpdev --command ./bin/server watch \
  --rebuild go --rebuild build --rebuild -o --rebuild ./bin/server.partial --rebuild ./cmd/server \
  --staging ./bin/server.partial --artifact ./bin/server --debounce 200ms
```

`--format json` emits JSON Lines events on stdout; human status stays on stderr.
`--fail-fast` / `--exit-on-change` can stop the process. Ctrl-C cancels in-flight
rebuilds and shuts the child down (stdin close, then interrupt, then kill).

Useful gates: `--smoke NAME` calls a tool after the inventory probe,
`--baseline PATH` seeds the comparison snapshot, `--readiness scenario` uses the
`--smoke` names as the readiness probe, `--keep-last-good` (default) keeps the
current child until the replacement probe succeeds. `--on-success none` only
rebuilds.

Do not point `--command` at the staging path while it is being written.

## Exit codes (script these)

| Code | Meaning |
| --- | --- |
| `0` | Complete success (`diff` unchanged or below threshold) |
| `2` | Bad flags, missing/conflicting target, invalid scenario or config |
| `10` | Spawn, HTTP/TLS, connect timeout, child died before handshake |
| `20` | Handshake/discover/JSON-RPC mismatch; stdio stdout was not JSON-RPC |
| `30` | Scenario assertion, or `call --fail-on-tool-error` |
| `40` | Public interface changed (`diff` / watch `--exit-on-change`) |
| `60` | Incomplete: pagination, discover skip/fail when needed, unevaluable assert, indeterminate diff |
| `70` | Internal / cleanup |

No command: help on stderr, `2`. `--help`: help on stdout, `0`.

A stdio child that exits before writing JSON-RPC is exit `10` even when the OS
closes the pipe before EOF. On Windows, an **absolute** `--command` path that
does not exist is validated up front and exits `2`, while a bare command name
fails at spawn with `10`; do not assume the same class across platforms.

HTTP `400` with `-32022` / `-32021` / `-32020` is a **modern** server (class
`20`). Do not enable `--allow-http-sse` for that. Empty or non-JSON `400` may
be legacy handshake. Details:
[references/protocol-and-failures.md](references/protocol-and-failures.md).

## Safety

- `--insecure` is never implied by `--url`.
- Do not exec server-returned strings.
- Credentials in `--header` are redacted in snapshots, traces, and logs.
- Child stdout is the MCP stream; logs belong on the server's stderr.

`--timeout` bounds RPCs. `--connect-timeout` bounds handshake including
`server/discover`. `--trace` / `--trace-format` log on stderr only;
`--verbose` adds full redacted bodies.

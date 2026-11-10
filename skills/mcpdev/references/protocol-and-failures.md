# Protocol eras and common failures

Use this when `inspect` / `call` / `watch` fail to connect, negotiate, or list.
Match the **exit number** in scripts. Do not scrape stderr.

## Eras

| `--protocol` | What the server speaks |
| --- | --- |
| `auto` (default) | Probe `server/discover` as 2026-07-28, then maybe `initialize` |
| `2026-07-28` | Stateless: `_meta` on every request, POST-only HTTP, no session id |
| `2025-11-25` | Stateful: `initialize` then `notifications/initialized` |

`--no-discover` records `discover: skipped`. That is not a successful discover
and can yield exit `60` when a catalog needed it.

Both first-class eras use **stdio** and **Streamable HTTP**. HTTP+SSE
(`2024-11-05`) needs `--allow-http-sse`. There is no default gRPC transport.

## Negotiation mistakes

| What you see | Typical exit | What to do |
| --- | --- | --- |
| HTTP 400 with `-32022`, `-32021`, or `-32020` | `20` | Modern peer. Fix version/headers/`_meta`. Do not treat as HTTP+SSE. |
| HTTP 400 empty or non-JSON | `10` / `20` | Auto may then send `initialize`. Opt in to HTTP+SSE only if that is really the transport. |
| stdio probe timeout or unrecognized error | `10` / `20` | Auto may send `initialize`. Pinned `2026-07-28` will not. |
| Child exits before writing JSON-RPC (stdout closed or EOF) | `10` | Treat as a dead child, even when the OS reports a closed pipe instead of EOF. |
| Banner or `go run` compile text on child **stdout** | `20` | First protocol byte must be `{` or `[`. Move logs to stderr. |
| JSON-RPC only on child stderr | usually not protocol | stderr is never parsed as MCP. |
| `--command` and `--url` together | `2` | Pick one. |

Diagnostics should show `requested`, `discovered`, `negotiated`, and `fallback`
(true only when initialize ran after a legacy-shaped discover failure).

## Incomplete catalogs and diffs

| Signal | Default exit |
| --- | --- |
| `pagination_stopped`, `cursor_loop`, `item_limit`, failed list | `60` |
| Diff `indeterminate` (a side incomplete) | `60` |
| Diff `threshold-failure` | `40` |
| Empty list with `status=complete` | `0` |

`--allow-incomplete` maps `60` to `0` and keeps `incomplete: true`. It does not
turn a truncated catalog into a successful empty list.

## Auth, TLS, timeouts, leftovers

| Symptom | Typical exit | What to do |
| --- | --- | --- |
| Non-loopback `http` or skipped TLS verify | `2` / `10` | Pass `--insecure` explicitly. |
| `401` / `403` | `10` | `--header 'Authorization: Bearer …'` or profile `header_env`. |
| Connect timeout | `10` | Raise `--connect-timeout`. |
| List canceled mid-page | `60` | Retry; Ctrl-C is not a complete catalog. |
| Port still bound after you killed mcpdev with `kill -9` | n/a | Stop the leftover **server** child; mcpdev is not a daemon. |

Watch: a failed rebuild keeps the last good binary and process. Events under
`.git`, `bin/`, and `dist/` do not rebuild.

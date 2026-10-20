# Network actions

Read this when writing readiness gates, health checks, or HTTP calls —
especially instead of `sleep` hacks before testing a freshly started service.

```drun
ping host "google.com" timeout "3s"
test connection to "localhost" on port 5432 timeout "5s"
wait for service at "http://localhost:8080/health" to be ready timeout "30s" retry "5s"

get "https://api.example.com/health" timeout "5s"
get "https://api.example.com/me" with auth bearer "{secret('api_token')}"
post "https://api.example.com/deploys" content type json with body "{\"ref\": \"main\"}"
```

## Branchable gates

`test connection` fails the task when nothing is listening. To branch instead,
use the condition forms, which never fail the run:

```drun
if port 5432 is open on "localhost":
  info "PostgreSQL is already running"

if port {$redis_port} is not open on "{$redis_host}" with timeout "2s":
  info "Redis is down, skipping the cache warmup"

when docker network "proxy" exists:
  info "Reusing the proxy network"
otherwise:
  run "docker network create proxy"
```

Ports accept Go durations or bare seconds and default to 5 seconds. In
`--dry-run` no connection is opened: a port reads as closed and a docker
network reads as missing, so the fallback branch runs. `check if port
<port> is open on "host"` is the statement spelling of `test connection`.

All HTTP verbs (`get`, `post`, `put`, `patch`, `delete`) accept
`with header "K: V"`, `with auth bearer "..."`, and `timeout "10s"`.
`wait for service ... to be ready` polls with HTTP GET (via curl) until a
success status or the timeout — the idiomatic gate before testing a freshly
started container.

Upstream example: `examples/37-network-actions.drun`.

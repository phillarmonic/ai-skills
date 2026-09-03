# Services and orchestration

Read this when a spec manages a fleet of services (the `service` block
subsystem) instead of single tasks.

A `service` block declares a long-running component with its own directory,
health check, compose file, environment, and optional git repository:

```drun
version: 2.0

project "platform" version "1.0.0":
  set environment to "development"

service "database" in "./services/database" means "PostgreSQL":
  health check:
    type "tcp"                       # tcp | http | dns
    endpoint "localhost:5432"
    timeout "5s"
    interval "10s"
    retries 3
  compose:
    file "docker-compose.yml"
  environment:
    POSTGRES_DB "myapp"
    POSTGRES_USER "app_user"

service "api" in "./services/api" means "API service":
  depends on ["database"]
  repository:
    url "https://github.com/{$globals.organization}/api.git"
    branch "main"
    clone true
    update_on_start false
  health check:
    type "http"
    endpoint "http://localhost:8080/health"
    condition "200"                  # expected HTTP status
  env_file:
    required true
    task "setup_api_env"             # task that provisions the env file
```

Declaring any `service` generates built-in lifecycle tasks (verified):
`start`, `stop`, `restart`, `health`, and `status`, which operate over the
declared services honoring `depends on` order. Health checks support `tcp`,
`http` (with expected status `condition`), and `dns` (with `domain`,
`record_type`, `expected_ip`) types.

Use this subsystem when a repo ships several compose-backed services that
must boot in dependency order; use plain tasks plus `wait for service`
(`network-actions.md`) for one-off readiness gates.

Upstream examples: `examples/64-microservices-basic.drun` through
`examples/69-microservices-complete.drun`,
`examples/70-multiline-build-commands.drun` (service `build:` blocks with
multi-line commands).

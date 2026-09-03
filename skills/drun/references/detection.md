# Detecting the environment

Read this when a task should adapt to the machine or CI environment instead
of hard-failing on a missing tool.

Declarative environment checks beat shelling out to `which`:

```drun
detect project type                       # reports what the repo looks like

if docker is available:
  info "container workflow available"

if go is available:
  if go version >= "1.21":
    info "modern Go"

when in ci environment:
  info "headless run"
when in local environment:
  info "interactive run"
```

`when in <name> environment:` knows `ci`, `local`, `production`, and
`staging`.

More verified forms:

```drun
# several tools at once (comma-separated, quoted when multi-word)
if git,go are available:
if docker,"docker compose" is not available:

# pick the first available spelling and remember it (DRY fallback)
detect available "docker compose" or "docker-compose" as $compose_cmd
run "{$compose_cmd} version"

# docker compose project state: usable / down / unusable / partial /
# unavailable / error
set $status to "{docker compose status}"
if $status is "usable":

# environment variable existence
if env HOME exists:
```

Tools whose names contain spaces must be quoted (`"docker compose"`). Any
other name — including hyphenated ones like `my-fake-tool` — may be written
bare or quoted in these conditions (verified).

This complements `requires tools:` — use `requires tools` when a missing tool
should fail the task, and `if <tool> is available` when the task should
adapt. For automatic installation of missing tools, see
`tool-provisioning.md`.

Upstream examples: `examples/26-smart-detection.drun`,
`examples/31-docker-tools-detection.drun`, `examples/32-dry-tool-detection.drun`,
`examples/40-docker-compose-status-check.drun`,
`examples/48-multi-tool-detection.drun`,
`examples/50-environment-variables.drun`.

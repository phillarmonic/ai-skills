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

This complements `requires tools:` — use `requires tools` when a missing tool
should fail the task, and `if <tool> is available` when the task should
adapt. For automatic installation of missing tools, see
`tool-provisioning.md`.

Upstream example: `examples/26-smart-detection.drun`.

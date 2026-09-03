# Automatic tool provisioning

Read this when a task needs a tool that may not be installed and failing is
worse than installing it — the house pattern for lint/security tooling.

Add `provision` to a tool requirement to have drun install a missing tool
instead of failing:

```drun
project "example" version "1.0":
  requires tools:
    golangci-lint provision
    gosec provision
    govulncheck provision

task "lint":
  requires tools:
    golangci-lint >= "1.64" provision     # version ranges work
  run "golangci-lint run ./..."
```

- Closed ranges (`>= "2.22" <= "2.22" provision`) pin the installed version;
  changing an installed tool's version needs `--allow-tool-version-changes`.
- A project may declare `provisioning sources:` with local YAML or
  `github:org/repo/path@ref` catalogs; project sources override the embedded
  first-party catalog.
- A task can inherit tool requirements from the tasks it orchestrates:

  ```drun
  task "ci" mode "ci" means "Run quiet CI checks":
    requires tools:
      from tasks:
        lint
        security
    call task lint
    call task security
  ```

Contrast with `if <tool> is available:` (see `detection.md`): provisioning
installs what is missing, detection adapts to what is present.

Upstream example: `examples/73-tool-provisioning.drun`.

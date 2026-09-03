# Code reuse: snippets, includes, and namespaces

Read this when sharing tasks/snippets across files or repos, or when a spec
is getting too big for one file.

## Includes

Declared in the `project` block; everything included lands under a namespace:

```drun
project "example" version "1.0":
  include "shared/docker.drun"                          # local file
  include "github:phillarmonic/drun/shared/x.drun"      # GitHub repo path
  include "https://example.com/shared/x.drun"           # arbitrary HTTPS
  include from drunhub "examples/drunhub-example-docker"         # drunhub registry
  include from drunhub "examples/drunhub-engine-validation" as validation
```

## Using included content

- Snippets (reusable statement blocks): `use snippet "docker.docker-cleanup"`
- Tasks: `call task "docker.docker-push" with image="myapp:v1" registry="{$registry}"`
- The namespace defaults to the included project's name; `as <name>` overrides
  it. References use dotted form: `<namespace>.<task-or-snippet>`.

## Project-level parameters and namespaced values

```drun
project "example" version "1.0":
  parameter $registry as string defaults to "ghcr.io"
```

- Read project parameters as `{$params.registry}`; included projects expose
  theirs as `{$params.<namespace>.registry}`.
- Included project settings appear under `{$globals.<namespace>.<key>}`.
- Variables set by an included snippet remain visible after `use snippet`.

Notes:

- Prefer `call task` over `use snippet` when the reused logic takes inputs —
  parameters make the contract explicit.
- Remote includes (`github:`, `https:`, drunhub) are fetched content: pin a
  ref when repeatability matters.

Upstream examples: `examples/53-code-reuse-demo.drun`,
`examples/54-include-demo.drun`, `examples/55-remote-github.drun`,
`examples/56-remote-https.drun`, `examples/61-test-drunhub-custom-namespace.drun`,
`examples/62-namespace-params-globals-test.drun`.

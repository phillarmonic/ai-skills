# Git policy and hooks

Read this when a project wants drun to enforce branch/commit conventions.

Projects can define git conventions with a `git policy:` block. When present,
use `xdrun cmd:hook install` to install drun-managed hooks that enforce them.
The Repertoire catalog does not install these hooks.

```drun
project "example" version "1.0":
  git policy:
    branch:
      default branches: "master", "main"
      protected branches: "master", "main"
      naming: "{type}/{identifier}-{description}"
      types: "feat", "fix", "chore"
    commit:
      messages: "{identifier}: {message}"
      extract identifier from branch
      enforce signed commits
```

For the `scm:` alias declaration used by semantic git actions such as
`git ensure ... from <alias>`, see `semantic-actions.md`.

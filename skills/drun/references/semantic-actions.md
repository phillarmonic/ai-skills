# Semantic actions worth knowing

Read this when touching files, structured config values, changelogs, project
versions, or git remotes — drun has structured statements that beat shelling
out. These are the ones used most across our repos:

```drun
# Files and folders
copy "env.example" to ".env"
create folder "caddy_data"
if file ".env" not exists:
if folder "build" is empty:
if file "./a.json" not matches file "./b.json":

# Structured file values (property, json, yaml, regex match)
get property "pluginVersion" from "gradle.properties" as $plugin_version
update property "pluginVersion" in "gradle.properties" to "{$version}" or fail
update match "(?m)^VERSION=(?P<value>[^\r\n]+)$" in "VERSION.txt" to "{$version}" or fail

# Project and changelog bookkeeping
check project version equals "{$plugin_version}"
update project version to "{$version}"
promote changelog "CHANGELOG.md" to version "{$version}"
# pin the date instead of today:
promote changelog "CHANGELOG.md" to version "{$version}" on "2026-09-01"

# Git/SCM: declare an alias in the project block, then reference it
#   scm:
#     git:
#       github:
#         my-repo:
#           default: https
#           https: "https://github.com/org/repo.git"
#           ssh: "git@github.com:org/repo.git"
git ensure $version is newer than latest version from my-repo

# Docker and git object operations (alternative to raw shell)
docker build image "myapp:latest" from "Dockerfile"
docker tag image "myapp:latest" as "myapp:dev"
docker push image "myapp:latest" to "ghcr.io/company"
docker compose up
git create branch "feat/x"
git checkout branch "feat/x"
git add files "."
git commit changes with message "wip"
git push to remote "origin" branch "feat/x"
git create tag "v1.0.0"
git show current branch

# Downloads with progress, permissions, and auth
download "https://example.com/install.sh" to ".downloads/install.sh" allow overwrite
download "https://api.github.com/zen" to "zen.txt" with auth bearer "{secret('gh_token')}" allow overwrite timeout "60s"

# Open a URL or file in the OS default handler (requires a trusted folder)
open url "https://example.com/docs"
open url "./coverage/index.html"
```

## Shell configuration

A project can pin the shell per OS (executable, args, extra environment) —
useful when tasks assume zsh/bash features:

```drun
project "example" version "1.0":
  shell config:
    linux:
      executable: "/bin/zsh"
      args:
        - "-l"
      environment:
        TERM: "xterm-256color"
```

Upstream examples: `examples/12-simple-file-ops.drun`,
`examples/23-docker-actions.drun`, `examples/24-git-actions.drun`,
`examples/30-shell-customization.drun`, `examples/47-download-complete.drun`,
`examples/74-file-values.drun`, `examples/75-changelog-promotion.drun`,
`examples/76-open-url.drun`.

Shell-related modifiers:

- `run "! go list -deps ./cmd/app | rg '^bad/import/'"` — because `run`
  executes through the shell, a leading `!` is POSIX negation: the step passes
  when the command fails. Handy for boundary and absence checks.
- `use workdir "docs"` scopes a task to a subdirectory instead of `cd`.
- `run "pnpm run test:watch" attached` gives the command the terminal (stdin,
  TTY) — use for watchers, shells, and anything interactive; only with
  single-line `run`.

Upstream examples: `examples/12-simple-file-ops.drun`,
`examples/74-file-values.drun`, `examples/75-changelog-promotion.drun`.

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

# Structured file values (property, json, yaml, toml, regex match)
get property "pluginVersion" from "gradle.properties" as $plugin_version
update property "pluginVersion" in "gradle.properties" to "{$version}" or fail
get json "/version" from "package.json" as $package_version
update json "/version" in "package.json" to "{$version}" or add as string
get yaml "chart.appVersion" from "Chart.yaml" as $chart_version
get toml "package.version" from "Cargo.toml" as $crate_version
update toml "project.version" in "pyproject.toml" to "{$version}" or fail
update match "(?m)^VERSION=(?P<value>[^\r\n]+)$" in "VERSION.txt" to "{$version}" or fail

# Project and changelog bookkeeping
check project version equals "{$plugin_version}"
update project version to "{$version}"
promote changelog "CHANGELOG.md" to version "{$version}"
# pin the date instead of today:
promote changelog "CHANGELOG.md" to version "{$version}" on "2026-09-01"
# re-running for a version that already has a section merges new entries into it

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

## Opening things and folder trust

`open url` refuses to launch anything from a folder that is not trusted on this
machine. Trust it once with `xdrun cmd:trust` (or `xdrun cmd:trust /path/to`) and
revoke with `xdrun cmd:untrust`; trust is stored in `~/.drun/trusted.yml` and a
trusted parent covers its children. On a headless host, over SSH, or in CI it
prints the target instead of failing, and a dry run skips the check entirely.

## Structured file values

Every format supports `get`, `check` and `update`:

```text
get <format> <selector> from <file> as <variable>
check <format> <selector> in <file> (equals <value> | differs from <value>)
update <format> <selector> in <file> to <value> (or fail | or add [as string|number|boolean])
```

| Format | Selector | Missing key |
| --- | --- | --- |
| `property` | exact key | `or add` adds a string |
| `json` | RFC 6901 pointer, object members | `or add as ...` |
| `yaml` | dot-separated mapping path | `or add as ...` |
| `toml` | TOML dotted key | `or add as ...` |
| `match` | Go regex with one `value` capture | `or fail` only |

`property`, `json` and `match` edit in place and keep the surrounding bytes.
`yaml` and `toml` decode the document and write it back through a serializer,
so comments and the original layout are lost: TOML comes back with tables and
keys sorted, single quotes, and arrays collapsed onto one line. Both are fine
for a version bump, but expect a formatting diff on the first write.

A `match` pattern has to match exactly once; the statement fails when it matches
zero or several times, so a table-blind pattern is brittle next to a structured
selector.

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

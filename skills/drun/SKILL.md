---
name: drun
description: >-
  Use when working with drun specs or xdrun automation: tasks, parameters,
  call task argument passing, platforms, interpolation, built-in functions,
  semantic actions, tool provisioning, CI, and hooks.
---

# drun

Use this skill when the task mentions drun, xdrun, `.drun/spec.drun`, task
automation, or repository workflows implemented in drun.

Drun is a fluent automation DSL. The interpreter binary is `xdrun`
("execute drun"). Prefer readable specs that make intent obvious at a glance.

## What to know first

- Main spec location: `.drun/spec.drun`
- Initialize a repo: `xdrun --init`
- List tasks: `xdrun --list`
- Run a task: `xdrun <task>`
- Pass task parameters as `key=value`, for example
  `xdrun deploy environment=production`
- Keep CLI behavior flags separate, for example
  `xdrun deploy environment=production --dry-run`
- Upstream docs and examples: https://github.com/phillarmonic/drun

## Recommended workflow

1. Read the existing drun file before making changes.
2. If there is no spec yet, initialize one with `xdrun --init`.
3. Use `xdrun --list` to inspect task names instead of guessing.
4. For platform-specific workflows, prefer separate declarations with
   `@platform(...)` instead of mixing OS branches into one task when behavior
   differs substantially.
5. Use canonical platform names in new specs: `linux`, `mac`, `windows`.
   Legacy `darwin` still parses, but prefer `mac` in new code and examples.
6. If a task family includes both platform-tagged variants and one unannotated
   task, drun resolves the exact platform variant first and uses the
   unannotated task as the fallback.
7. When adding hard dependencies, declare them with `requires tools:`.
8. Prefer small, readable tasks that explain intent with `means`, `info`, and
   `step`.
9. For AI-driven CI or noisy checks, prefer `mode "ci"` so successful shell
   stdout/stderr stays buffered and only failure output is emitted.
10. After editing a spec, run the narrowest relevant `xdrun` command to verify.

## Project AI guidance

Repositories can install a managed cross-agent guide with:

```bash
xdrun cmd:skill install drun-basics
```

That writes `.drun/ai/drun-basics.md` plus light entrypoints in common agent
files. Prefer that for explicit per-repository onboarding. A Repertoire project
bootstrap can install this complete portable skill globally while managing only
compact activation pointers in the repository.

## Calling tasks and passing arguments

Use `call task` to compose tasks instead of duplicating steps:

```drun
call task setup                                    # single word / snake_case, unquoted
call task "run-tests"                              # quote kebab-case, spaces, special chars
call task "run-tests" with test_type="unit"
call task fuzz with iterations=100                 # bare numbers are fine
call task "deploy" with environment="prod" replicas=3
```

Rules for `with` arguments (these are the ones that bite):

- All `key=value` pairs go on the same line as the call, space-separated.
- Values must be **double-quoted strings or bare numbers** — nothing else.
  `with env=prod`, `with env=$env`, `with env={$env}`, and `with env='prod'`
  are all parse errors.
- `with` values are **not** evaluated at the call site. They are stored raw
  and interpolated lazily, in the **callee's** context, when the called task
  uses `{$param}`.
- The callee inherits the caller's `set` variables, but **not** the caller's
  parameters. So `with env="{$environment}"` fails at runtime with
  `undefined variable: {$environment}` when `$environment` is a caller
  `given`/`requires` parameter. Forward parameters through a `set` variable
  first — this is the canonical pattern:

  ```drun
  task "caller":
    requires $environment
    set $env to "{$environment}"          # resolves in the caller's context
    call task "deploy" with environment="{$env}"
  ```

- The same applies to ternary, `if/then/else`, and `secret(...)` inside a
  `with` value: they resolve in the callee's context, so compute them with
  `set` in the caller first, then pass `with flag="{$flag}"`.
- Constraint validation on the callee's parameter (`from [...]`,
  `matching pattern`, ranges) sees the **raw, un-interpolated** value, so an
  interpolated `with` value like `with env="{$env}"` fails a constrained
  parameter even when the resolved value would be valid. Prefer validating in
  the caller, or pass literal values into constrained parameters.
- Bare numbers arrive as strings; declare `as number` on the callee's
  parameter when numeric validation matters.
- Passed values override `given $x defaults to ...`. A missing `requires $x`
  fails at runtime with `required parameter 'x' not provided`.
- Undeclared `with` keys are silently ignored. When a value "doesn't take",
  check the callee's `given`/`requires` parameter names for a mismatch.
- Variables set inside the called task propagate back into the caller.
- `depends on a and b` takes no arguments; use `call task` when you need to
  pass values.

Ordering note: compute values with `set` **before** the call, not after —
this is also the required order for parameter forwarding:

```drun
set $tag to "v1"
call task deploy with environment="prod" tag="{$tag}"
```

(On drun ≤ 2.28.0 there is also a parser bug: a `set $x to ...` statement on
the line immediately after a `call task ... with ...` is consumed as another
parameter name and fails with `expected '=' after parameter name`. Fixed in
later releases; the `set`-before-`call` order avoids it on every version.)

CLI parity: `xdrun deploy environment=prod replicas=5` is the shell equivalent
of `with environment="prod" replicas=5`. Keep behavior flags after the
`key=value` args: `xdrun deploy environment=prod --dry-run`.

Useful examples in the upstream repo:

- `examples/46-task-calling.drun`
- `examples/53-code-reuse-demo.drun`

## Interpolation and variables

- Task and project variables commonly use `{$name}` inside strings.
- Environment variables use shell-style syntax such as `${USER}` or
  `${HOME:-/tmp}`.
- Conditional interpolation supports ternary and `if/then/else`, for example
  `{$debug ? '--debug' : ''}` or
  `{if $environment is 'production' then 'prod.yml' else 'dev.yml'}`.
- Secrets can be read with `secret(...)`, for example `{secret('api_key')}` or
  `{secret('webhook_url', 'https://default.example')}`.
- Undefined drun variables are strict by default. Prefer
  `requires $name` or `given $name defaults to ...`.
- If an expression is hard to scan inline, compute it with
  `set $name to "..."` first.
- Multi-line `run "..."` strings support interpolation and are often cleaner
  than one huge shell line.

Useful examples in the upstream repo:

- `examples/03-interpolation.drun`
- `examples/51-env-var-interpolation.drun`
- `examples/52-conditional-interpolation.drun`
- `examples/62-secrets-interpolation.drun`
- `examples/63-multiline-strings.drun`

## Tool checks

Prefer declarative requirements when a task depends on a binary or minimum
version:

```drun
project "example" version "1.0":
  requires tools:
    go >= "1.21"
    docker
```

Task-level checks are also valid:

```drun
task "test" means "Run the test suite":
  requires tools:
    go
  run "go test ./..."
```

### Automatic tool provisioning

Add `provision` to have drun install a missing tool instead of failing.
This is the house pattern for lint/security tooling:

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

Upstream example: `examples/73-tool-provisioning.drun`.

## Parameters in practice

Typed parameters with constraints catch bad input before any command runs:

```drun
requires $version as string matching semver_optional_v
requires $port as number between 1000 and 9999
requires $environment from ["dev", "staging", "production"]
requires $email matching email format
given $label as boolean defaults to false
```

Available pattern macros: `semver` (strict, requires the `v` prefix),
`semver_optional_v`, `semver_extended`, `uuid`, `url`, `ipv4`, `slug`,
`docker_tag`, `git_branch`. Prefer macros over hand-written `matching pattern`
regexes when one fits.

Common idioms:

- Optional value: `given $path defaults to empty` (equivalent to `""`), then
  branch with `if $path is empty:` / `if $path is not empty:`.
- Boolean switch: `given $volumes as boolean defaults to false`, then
  `if $volumes is true:` or inline
  `run "docker compose down {$volumes ? '--volumes' : ''}"`.
- Parameter defaults can call built-in functions, including pipes:
  `given $tag defaults to "{current git branch | replace '/' by '-' | lowercase}"`.

Upstream examples: `examples/11-typed-parameters.drun`,
`examples/35-advanced-parameter-validation.drun`.

## Built-in functions and pipe transforms

Interpolation `{...}` supports built-ins, chainable with `|`:

| Expression | Yields |
| --- | --- |
| `{current git commit}` | short commit hash (`a72091f`) |
| `{current git branch}` | branch name (`feature/new-api`) |
| `{now.format('2006-01-02-15-04-05')}` | formatted time (Go layout) |
| `{pwd}`, `{hostname}` | working directory, host |
| `{os}`, `{shell}` | `windows`/`linux`/`darwin`; `bash`/`pwsh`/... |
| `{env('VAR')}` | environment variable |
| `{available tasks(', ')}` | OS-available task names, joined; extra args omit names |

Pipe transforms: `replace "from" by "to"`, `without prefix "text"`,
`without suffix "text"`, `uppercase`, `lowercase`, `trim`,
`normalized for shell` (path separators/escapes for the active shell).

```drun
set $release_version to "{$version without prefix 'v'}"
set $docker_tag to {current git branch | replace "/" by "-" | lowercase}
run "go build -o {$path normalized for shell} ./cmd/app"
```

Note: `{os}` reports `darwin`, while `@platform(...)` prefers `mac` in new
specs — the two vocabularies differ on purpose.

Upstream example: `examples/08-builtin-functions.drun`.

## Semantic actions worth knowing

Drun has structured statements that beat shelling out; the ones used most
across our repos:

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

# Git/SCM: declare an alias in the project block, then reference it
#   scm:
#     git:
#       github:
#         my-repo:
#           default: https
#           https: "https://github.com/org/repo.git"
#           ssh: "git@github.com:org/repo.git"
git ensure $version is newer than latest version from my-repo
```

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

## House conventions

Patterns that repeat across Phillarmonic specs — follow them in our repos:

- A `default` task that prints `info "Available tasks: {available tasks(', ')}"`.
- A `ci` task in `mode "ci"` that narrates with `step` and orchestrates one
  `call task` per check (`vuln`, `lint`, `test`, `sec`, ...). Keep the
  individual checks as their own tasks so they can run standalone.
- Shell logic longer than a line or two lives in `.drun/scripts/*.sh` and the
  task does `run ".drun/scripts/ci-postgres.sh {$port}"`.
- Release tasks (`prepare-release`) chain: `requires $version matching
  semver_optional_v` → preflight checks → `call task ci` →
  `promote changelog` → `update project version to`.
- `xdrun ci --task-mode=normal` overrides buffering at runtime when you need
  to watch a ci-mode task stream; `--task-mode=ci` does the reverse.

## Writing good specs

- Keep the file readable at a glance.
- Prefer task names that match user intent.
- For the same user-facing workflow across platforms, use duplicate task names
  with disjoint `@platform(...)` annotations so `xdrun <task>` resolves the
  correct variant.
- A task family may include one unannotated task as fallback when no
  platform-specific variant matches.
- Use `given $name defaults to ...` for optional parameters.
- Use `requires $name` for values that must be supplied at runtime.
- Prefer interpolation for task inputs and shell env expansion for true process
  environment values.
- For complex command assembly, compute intermediate values with `set` before
  the `run` step.
- Use `call task ... with key="value"` instead of duplicating steps; see
  "Calling tasks and passing arguments" for the strict value rules.
- Use `mode "ci"` for noisy validation when you want to save output tokens on
  success.
- Keep shell commands explicit inside `run "..."`.

## Lifecycle hooks

Hooks are declared inside the `project` block — not per task:

```drun
project "myapp" version "1.0":
  on drun setup:
    info "Starting pipeline for {$globals.project} v{$globals.version}"
    if file ".env" not exists:
      copy ".env.example" to ".env"

  before any task:
    info "Starting task: {$globals.current_task}"
    capture task_start_time from now

  after any task:
    capture task_end_time from now
    info "Task '{$globals.current_task}' finished"

  on drun teardown:
    info "Pipeline completed"
```

Execution order: `on drun setup` (once) → `before any task` → task body →
`after any task` → `on drun teardown` (once).

Semantics that surprise people (verified against the engine):

- `before any task` / `after any task` wrap **only the invoked target task**.
  Dependency tasks and `call task` callees do **not** trigger them, despite
  the "any task" name.
- `after any task` and `on drun teardown` are **skipped when the task fails**
  — teardown is not a `finally`. Put critical cleanup inside the task itself
  (or use `try/catch/finally`) if it must run on failure.
- A failing `on drun setup` or `before any task` hook aborts the run;
  `after any task` and `on drun teardown` hooks are best-effort — their
  failures print a warning but don't fail the run.
- Inside hooks, `$globals.current_task`, `$globals.project`,
  `$globals.version`, and `$globals.drun_version` are available; the
  `capture $t from now` + `let $d be {end} - {start}` idiom gives timing.

Typical uses: provisioning `.env` from `.env.example` in `on drun setup`,
per-run timing/logging in the task-level hooks, pipeline metrics in
`on drun teardown`.

Upstream example: `examples/39-drun-lifecycle-hooks.drun`.

## Lifecycle basics

- Bootstrap with `xdrun --init`.
- Treat `.drun/spec.drun` as the source of truth for project automation.
- Use `xdrun --list` to discover available workflows.
- `mode "ci"` buffers normal shell stdout/stderr and only prints that buffer
  when a command fails.
- Validate with targeted runs such as `xdrun test` or
  `xdrun build --dry-run`.

## Example starter spec

```drun
version: 2.0

project "example" version "1.0":
  requires tools:
    go

task "default" means "Show available automation":
  info "Run xdrun --list to inspect tasks"

task "test" means "Run tests":
  run "go test ./..."

@platform("linux", "mac")
task "shell" means "Open a Unix shell":
  run "bash" attached

@platform("windows")
task "shell" means "Open PowerShell":
  run "pwsh.exe" attached

task "ci" mode "ci" means "Run noisy checks with buffered output":
  run "go test ./..."
```

## Git policy and hooks

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

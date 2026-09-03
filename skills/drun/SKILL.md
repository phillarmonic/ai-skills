---
name: drun
description: >-
  Use when working with drun specs or xdrun automation: tasks, parameters,
  call task argument passing, platforms, interpolation, built-in functions,
  control flow and loops, environment detection, semantic actions, tool
  provisioning, lifecycle hooks, CI, and hooks.
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
  `{secret('webhook_url', 'https://default.example')}`; for managing secrets
  from tasks see `references/secrets.md`.
- Undefined drun variables are strict by default. Prefer
  `requires $name` or `given $name defaults to ...`.
- If an expression is hard to scan inline, compute it with
  `set $name to "..."` first.
- Multi-line `run "..."` strings support interpolation and are often cleaner
  than one huge shell line. For multi-command scripts there is also a block
  form, plus a capture variant that stores stdout in a variable:

  ```drun
  run:
    echo "first command"
    echo "second command"

  capture from shell "git rev-parse --short HEAD" as $commit
  info "commit: {$commit}"
  ```

Useful examples in the upstream repo:

- `examples/03-interpolation.drun`
- `examples/51-env-var-interpolation.drun`
- `examples/52-conditional-interpolation.drun`
- `examples/62-secrets-interpolation.drun`
- `examples/63-multiline-strings.drun`
- `examples/29-multiline-shell-commands.drun`

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

To have drun auto-install missing tools instead of failing, add `provision` —
see `references/tool-provisioning.md`.

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

## Deep references

Load these only when the task touches their topic — each file starts with the
traps to know before writing code:

- `references/parameters.md` — typed parameters, pattern macros
  (`semver_optional_v`, `docker_tag`, ...), `defaults to empty` and boolean
  idioms, validation traps.
- `references/builtins-and-transforms.md` — built-in function table, pipe
  transforms, `let`/`set`/`transform`, and the two `capture` syntaxes
  (shell-capture vs. evaluated expression; `now` epoch seconds).
- `references/control-flow.md` — `for each` loops (comma/whitespace string
  splitting), filters, matrices, `in parallel`, and the real range/line/
  pattern-match loop forms.
- `references/lifecycle-hooks.md` — `on drun setup/teardown`,
  `before/after any task`; their failure semantics and the timer recipe.
- `references/detection.md` — `if <tool> is available`, `detect project
  type`, `when in ci/local/production/staging environment`.
- `references/network-actions.md` — `ping`, `test connection`,
  `wait for service ... to be ready`, HTTP verbs.
- `references/semantic-actions.md` — file/property/changelog/project-version
  statements, SCM aliases, `use workdir`, `attached`, `!` negation.
- `references/tool-provisioning.md` — `provision`, version ranges,
  provisioning catalogs, `from tasks:` inheritance.
- `references/progress-timers.md` — progress indicators and named timers.
- `references/git-policy.md` — `git policy:` blocks and drun-managed git
  hooks (`xdrun cmd:hook install`).
- `references/code-reuse.md` — includes (local, `github:`, HTTPS, drunhub),
  namespaces, `use snippet`, project-level `parameter`, `$params.*`.
- `references/secrets.md` — `secret(...)` interpolation, `secret
  set/exists/list/delete`, namespaces.
- `references/error-handling.md` — `try`/`catch as $e`/`finally`, `throw`,
  typed catches, `rethrow`, and where `ignore` actually works.
- `references/orchestration.md` — `service` blocks with health checks,
  compose, repository cloning, and the generated start/stop/health tasks.

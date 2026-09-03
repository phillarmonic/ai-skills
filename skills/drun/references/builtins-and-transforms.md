# Built-in functions, pipes, and variable operations

Read this when transforming strings, using git/time/env values in
interpolation, or deciding between `let`, `set`, `transform`, and `capture`.

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

## Variables: let, set, and transform

- `let $x = "value"` declares; `set $x to "value"` assigns. Both interpolate
  their right-hand side in the current context; treat them as
  interchangeable in modern specs (`set ... to ...` reads best).
- `transform $x with <op>` mutates a variable in place:
  `trim`, `uppercase`, `lowercase`, `replace "from" "to"`, `concat "..."`,
  `length`, `slice 0 7`, `split ","`, `join " | "`.
- `transform` only works on `let`/`set` variables — applying it to a
  `given`/`requires` parameter fails with `variable '$x' not found`. Copy the
  parameter first: `let $tag = "{$version}"` then transform.
- Prefer pipe transforms in interpolation (`{$x | lowercase}`) for one-off
  use, and `transform` when the value is built up over several steps.

## Capture

Two syntaxes with **different** semantics (verified against the engine):

- `capture from shell "cmd" as $name` — runs the command and stores its
  (interpolated, trimmed) stdout. This is the useful one:

  ```drun
  capture from shell "git rev-parse --short HEAD" as $commit
  info "commit: {$commit}"
  ```

- `capture name from <expr>` — interpolates/evaluates the expression like
  `let`/`set` and stores the resolved text. In particular a bare `from now`
  stores the `now` builtin's **Unix time in seconds**:

  ```drun
  capture start_time from now
  info "started at {start_time}s"
  ```

  For human-readable timestamps use `{now.format('2006-01-02 15:04:05')}`
  and for durations use the timer built-ins (`start timer`/`show elapsed
  time`/`stop timer`, see `progress-timers.md` and `lifecycle-hooks.md`).

Upstream examples: `examples/08-builtin-functions.drun`,
`examples/28-variable-operations.drun`,
`examples/39-drun-lifecycle-hooks.drun`.

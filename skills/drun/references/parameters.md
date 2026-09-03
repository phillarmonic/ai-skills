# Parameters in practice

Read this when declaring task parameters (`requires`/`given`), adding input
validation, or when a parameter value "doesn't take".

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

Traps:

- Constraint validation sees the **raw, un-interpolated** value when the
  parameter is passed via `call task ... with` — an interpolated value like
  `with env="{$env}"` fails a constrained parameter even when the resolved
  value would be valid. Validate in the caller, or pass literal values into
  constrained parameters.
- A missing `requires $x` fails at runtime with
  `required parameter 'x' not provided`.
- Undeclared `with` keys are silently ignored — when a value "doesn't take",
  check the callee's `given`/`requires` names for a mismatch.
- `transform` cannot mutate a parameter (`variable '$x' not found`); copy it
  to a `let`/`set` variable first. See `builtins-and-transforms.md`.

Upstream examples: `examples/11-typed-parameters.drun`,
`examples/35-advanced-parameter-validation.drun`.

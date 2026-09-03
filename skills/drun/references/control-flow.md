# Control flow: loops and matrices

Read this when writing loops, build/test matrices, or parallel fan-out.

## Traps first (verified against the engine at HEAD)

- `for $i in range 1 to 5`, `for each line text in file "f"`, and
  `for each match m in pattern "..."` are **unimplemented stubs**: the engine
  ignores the bounds/file/pattern and iterates hardcoded sample items
  (a range loop literally runs 0–10). Do not use them until implemented.
- `for each $x in $text` splits `$text` on **whitespace only**. A
  space-separated string iterates word by word; a comma-separated string
  iterates as ONE item. Prefer array literals or `as list` settings.
  (Several upstream examples imply comma-splitting; they only "work" because
  the filter happens to match the whole string.)

## The reliable form

`for each` over an **array literal** or a project-level list:

```drun
project "example" version "1.0":
  set platforms as list to ["linux", "mac", "windows"]

task "matrix":
  # project lists are reachable as $globals.platforms (or just $platforms)
  for each $platform in $globals.platforms:
    for each $arch in ["amd64", "arm64"]:
      step "Building {$platform}-{$arch}"

  # filters
  for each $item in ["a_test", "b.js", "c_test"] where $item contains "test":
    info "matched {$item}"

  # parallel fan-out (outer loop parallel, inner sequential is a good default)
  for each $env in ["dev", "staging"] in parallel:
    step "Deploying {$env}"

  # loop control
  for each $item in ["one", "skip", "two"]:
    continue if $item == "skip"
    break when $item == "two"
```

Filter operators include `contains`, `starts with`, `ends with`, `==`, `!=`.
Interpolate loop variables with `{$item}`.

## when / otherwise

`when <condition>:` is the multi-branch sibling of `if`, with `otherwise:` as
the fallthrough:

```drun
when $environment is "production":
  warn "production deploy"
when $environment is not "dev":
  info "review carefully"
otherwise:
  info "safe environment"
```

The matrix pattern — nested `for each` over array literals, with the outer
loop `in parallel` and inner loops sequential — is the house way to do
build/test/deploy matrices.

Upstream examples: `examples/42-matrix-sequential.drun`,
`examples/43-matrix-parallel.drun` (real), `examples/27-advanced-control-flow.drun`
(filters and loop control are real; its range/line/pattern loops hit the stubs
above).

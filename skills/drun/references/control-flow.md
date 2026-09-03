# Control flow: loops and matrices

Read this when writing loops, build/test matrices, or parallel fan-out.

## Trap: string splitting in `for each`

`for each $x in $text` splits a **string** iterable as follows (verified
against the engine at HEAD):

- values containing **commas** iterate per comma-separated item (whitespace
  around each item is trimmed) — `"a, b, c"` yields `a`, `b`, `c`;
- values **without commas** split on whitespace — `"a b c"` yields `a`, `b`, `c`.

Array literals and `as list` parameters/settings iterate their elements
directly, which is usually clearer than embedding a list in a string:

```drun
for each $env in ["dev", "staging"]:
  step "Deploying {$env}"
```

## Range, file-line, and pattern-match loops

These loop forms are implemented (verified against the engine at HEAD):

```drun
# Real bounds and step (negative steps iterate downwards)
for $i in range 1 to 9 step 2:
  info "Item {$i}"

# Reads the file line by line (dry runs report without reading)
for each line text in file "logs/out.txt":
  info "Line: {text}"

# Regex matches over a subject variable/parameter (subject clause required)
for each match result in pattern "[0-9]+" of $log:
  info "Number: {result}"
```

A range `step 0`, a non-integer bound, a missing file, an invalid pattern, or
an undefined subject each produce a clear error. Pattern-match loops require
`of $subject` — the parser rejects a match loop without it.

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
(comma-separated `$items` filters, range/line/match demos all run for real).

# Error handling

Read this when a task may fail recoverably, or when you need cleanup that
must run even on failure (remember: `on drun teardown` does not — see
`lifecycle-hooks.md`).

```drun
task "deploy":
  try:
    run "./deploy.sh"
  catch as $e:
    warn "Deploy failed: {$e}"
    run "./rollback.sh"
    rethrow                     # optional: keep the task failing
  finally:
    run "./cleanup.sh"          # always runs
```

Verified semantics:

- `try:` / `catch as $e:` / `finally:` work as expected; `$e` binds the error
  message. `finally` runs either way, and a failing `finally` overrides the
  original error.
- `catch <Type> as $e:` narrows handling by error type
  (`catch FileNotFoundError as $e:`). Types match by message content, not a
  real type hierarchy — prefer catching all and branching on `{$e}` when in
  doubt.
- `throw "message"` fails the task with a custom error — use it for preflight
  validation with actionable messages:

  ```drun
  if file "config.yml" not exists:
    throw "config.yml missing — run xdrun setup first"
  ```

- `rethrow` inside a catch keeps the task failing, but note it raises a
  generic `rethrown error` — the original message is lost (engine
  simplification). Log `{$e}` yourself before rethrowing.
- `ignore` only makes sense **inside a catch block**: it suppresses the
  handled error. A bare `ignore` elsewhere does not stop a failing `run` from
  failing the task (verified).

Upstream examples: `examples/19-error-handling.drun`,
`examples/13-simple-error-test.drun`, `examples/16-simple-throw.drun`.

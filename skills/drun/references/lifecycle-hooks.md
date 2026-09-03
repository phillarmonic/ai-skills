# Lifecycle hooks

Read this when adding per-run setup/teardown or per-task logging and timing.

Hooks are declared inside the `project` block — not per task:

```drun
project "myapp" version "1.0":
  on drun setup:
    info "Starting pipeline for {$globals.project} v{$globals.version}"
    if file ".env" not exists:
      copy ".env.example" to ".env"

  before any task:
    info "Starting task: {$globals.current_task}"
    info "{start timer('task')}"

  after any task:
    info "Task {$globals.current_task} took {show elapsed time('task')}"
    info "{stop timer('task')}"

  on drun teardown:
    info "Pipeline completed"
```

Execution order: `on drun setup` (once) → `before any task` → task body →
`after any task` → `on drun teardown` (once).

## Semantics that surprise people (verified against the engine)

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
  `$globals.version`, and `$globals.drun_version` are available.

## Timing inside hooks

Use the timer built-ins — they work across hook boundaries (verified), as in
the example above.

`capture name from now` does store a real Unix-time timestamp (the `now`
builtin, in seconds), so capturing `start_time`/`end_time` around a hook works
for recording when events happened — see
`examples/39-drun-lifecycle-hooks.drun`. The engine has no arithmetic
operator, though, so do not try `let d be {end} - {start}` to compute a
duration; for human-readable durations use the timer built-ins. See
`builtins-and-transforms.md` for the two capture syntaxes.

## Typical uses

- `on drun setup`: provisioning `.env` from `.env.example`, announcing the
  pipeline.
- `before`/`after any task`: per-run logging and timing.
- `on drun teardown`: pipeline metrics, farewell summaries.

Upstream example: `examples/39-drun-lifecycle-hooks.drun`.

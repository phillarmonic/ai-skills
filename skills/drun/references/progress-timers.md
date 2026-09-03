# Progress and timers

Read this when adding progress indicators or elapsed-time reporting to long
tasks.

Side-effecting interpolation functions for UX polish:

```drun
info "{start timer('build')}"
info "{start progress('Building', 'build')}"
info "{update progress('50', 'Compiling', 'build')}"
info "{finish progress('Done', 'build')}"
info "took {show elapsed time('build')}"
info "{stop timer('build')}"
```

Timers and progress indicators are keyed by name, can run several at once,
and work inside lifecycle hooks (see `lifecycle-hooks.md`). Invalid calls
(unknown timer, out-of-range percentage) render as placeholders instead of
failing the task.

Upstream example: `examples/38-progress-and-timers.drun`.

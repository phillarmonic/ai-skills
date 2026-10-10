# CI mode: buffered output and the step summary

Read this when a task runs noisy checks whose logs only matter on failure —
linters, test suites, security scanners. `mode "ci"` keeps successful command
output quiet, and the run ends with a per-step verdict so a passing gate carries
its own evidence instead of leaving the exit code as the only signal.

## Traps to know first

- `mode "ci"` only buffers **shell** output (`run`, `exec`, `shell`). Action
  statements (`step`, `info`, `success`, ...) still print, so the banners you
  keep are the ones you wrote.
- The buffer is flushed **only when a command fails**, so a green run stays
  quiet (a command marked `attached` is never buffered and always streams).
- `--task-mode=normal` turns buffering off for the whole invocation **and**
  suppresses the step summary; `--task-mode=ci` forces both onto a task that
  does not declare the mode.
- The summary is printed only when the run declared at least one `step`. A ci
  task with no `step` statements is unchanged — no summary line appears.
- A step passes when execution reaches the next step (or the end of the run);
  it fails when the run aborts inside it. There is no "step finished" syntax —
  the next step, or the end of the run, is what closes the previous one.

## Declaring a ci task

Narrate the pipeline with `step` and orchestrate one `call task` per check:

```drun
task "ci" mode "ci" means "Runs the pipeline quietly":
  step "Lint"
  call task lint
  step "Unit tests"
  call task test
  step "Security"
  call task sec
  success "CI executed successfully end-to-end"
```

Steps from a called task that inherits ci mode are included, in order, so
`call task test` can add its own steps between the caller's markers.

## The step summary

Every ci run that used at least one `step` ends with a compact, grep-friendly
summary:

```log
CI STEP SUMMARY
STEP=dependency-vulnerability-checks STATUS=passed DURATION_MS=1358
STEP=linter STATUS=passed DURATION_MS=907
STEP=unit-tests STATUS=failed DURATION_MS=418
CI_STATUS=failed STEPS=3 PASSED=2 FAILED=1 DURATION_MS=2731
```

- `STEP=` is a slug of the step message: lowercased, with every run of
  non-alphanumeric characters collapsed to `-` (`"Unit tests"` → `unit-tests`).
  It is stable and safe to grep or parse; a message that slugs to nothing
  becomes `step`.
- `STATUS` is `passed` or `failed`. `DURATION_MS` is that step's wall time in
  milliseconds.
- The final `CI_STATUS=` line aggregates the run: overall verdict, total step
  count, pass/fail counts, and total duration.
- The failing command's own stdout/stderr is still printed above the summary, so
  the verdict and the explaining logs line up in one captured log.

## Choosing a format

```bash
xdrun ci                       # text summary (default)
xdrun ci --ci-summary=json     # one machine-readable line
xdrun ci --ci-summary=off      # banners only, no summary
```

`--ci-summary=json` emits a single JSON object as the last line:

```json
{"task":"ci","status":"failed","duration_ms":2731,"steps":[{"step":"linter","status":"passed","duration_ms":907},{"step":"unit-tests","label":"Unit tests","status":"failed","duration_ms":418}]}
```

`label` is present only when it differs from the slug. `--ci-summary=off`
restores banner-only output for callers that only want the exit code.

## Why it exists

Buffering saves input tokens and log-ingestion cost, but it used to leave the
exit code as the only success signal, detached from the output that explains a
failure. The summary keeps the token savings while making each run
self-evident — exit code plus a machine-greppable verdict per step.
Example: drun's own `.drun/spec.drun` `ci` task.

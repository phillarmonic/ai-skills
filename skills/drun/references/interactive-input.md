# Interactive input: confirm and prompt

Read this when a task should ask the person running it for a yes/no answer
(`confirm`) or free-form text (`prompt`). Both are native drun statements:
they ask from a real terminal and resolve deterministically everywhere else —
CI, piped stdin, agents, dry runs — so they never hang an automated run.

## Traps to know first

- A bare `confirm` (no `as $var`) is a **gate**, not an assignment: a no
  answer stops the run gracefully with exit code 0 and later statements don't
  execute. Use `as $var` when you want to branch on the answer instead.
- `as $var` never aborts. The answer is stored as drun's string boolean —
  `"true"` or `"false"` — so test it with `if $migrate is "true":`, not a
  bare truthiness check.
- `prompt` requires `as $var`; without it drun fails at runtime before asking.
- A `confirm` `defaults to` value must parse as yes/no. Accepted spellings:
  `yes`, `no`, `"true"`, `"false"`, `1`, `0` (quoted or bare). Anything else
  is a runtime error. `prompt` defaults are free text.
- Optional clauses come in a fixed order: `defaults to <value>` first, then
  `as $var`.
- The question string is required, and both question and default interpolate
  `{$var}` at execution time.
- A bare `confirm` with no declared default that runs off a terminal (and
  without `--yes`/`--no`) is an **error**, never a hang: drun says so and
  stops. Decide explicitly with `defaults to` or a flag.

## Surface forms

```drun
confirm "Deploy to production?"                    # gate: no -> graceful stop (exit 0)
confirm "Run database migrations?" as $migrate     # stores "true" / "false"
confirm "Delete build cache?" defaults to "no"     # quoted or bare yes/no
confirm "Continue?" defaults to "yes" as $go
prompt "Which environment?" as $environment
prompt "Release notes?" defaults to "n/a" as $notes
```

`confirm` renders `❓ <question> [y/N]` (`[Y/n]` when the default is yes) and
accepts y/yes/n/no case-insensitively (plus the `true`/`false` and `1`/`0`
spellings). An empty line takes the declared default. `prompt` renders
`❓ <question> [default]: ` and stores the typed text, falling back to its
default on an empty line.

## Resolution when there is no terminal

drun treats a run as non-interactive when stdin is not a real TTY or when it
detects a CI environment. It never waits for input there. Resolution order:

1. the declared `defaults to` value;
2. the global `--yes`/`-y`/`--no` assumption;
3. otherwise a clear error (for `confirm`, "no interactive terminal and no
   default answer").

So an unattended task should declare a default or be invoked with a flag:

```drun
confirm "Delete the build cache?" defaults to "no"
prompt "Release notes?" defaults to "n/a" as $notes
```

```bash
xdrun deploy --yes            # -y also works
xdrun deploy --no
```

## Flags and dry run

- `--yes`/`-y`/`--no` answer every interactive statement without prompting —
  even on a TTY — and on a normal (non-dry) run they override a declared
  default for `confirm`.
- `--dry-run` never asks. It prints `[DRY RUN] would ask: "<question>"` and
  resolves to the declared default; without one, a `confirm` assumes yes (or
  the `--yes`/`--no` assumption) and a `prompt` resolves to an empty answer.
  A bare `confirm` never declines in a dry run unless `--no` was passed.

## Composition and scope

- Gates compose with control flow. A decline inside an `if`/`when` body or a
  sequential `for each` loop still stops the whole run gracefully (the
  decline is not scoped to the loop iteration).
- Inside a `try:` block the decline is an ordinary statement error: a matching
  `catch` intercepts it and the run continues. Put gates where a decline
  should end the run — don't bury them in `try`/`catch`.
- Variables written by `as $var` live for the rest of the current task and
  propagate back to the caller of a `call task`, exactly like other
  assignments.
- Shell commands that prompt on their own still need `run "..." attached`;
  `confirm` and `prompt` statements are the alternative that falls back
  cleanly when there is no terminal.

Example: `examples/77-confirmations.drun`.

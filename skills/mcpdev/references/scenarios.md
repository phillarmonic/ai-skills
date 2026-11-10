# Scenario files for `mcpdev test`

YAML or JSON. `schema_version` must be `1`. Extra properties are errors. Quote
YAML strings that contain `{` (URI templates, `${...}`) so they are not flow
mappings. There are no loops, branches, includes, or shell steps.

Each **case** opens its own session, runs **steps** in order, then closes.
After the first unexpected failure, remaining steps in that case skip; cleanup
still runs. Other cases continue up to `--jobs` (default 4).

## Copyable templates

Verified starting points live in `assets/` — copy and adapt the target, tool
names, and paths:

- `assets/scenarios/smoke.yaml` — inspect + happy-path `call` with the correct
  `$.result.*` envelope, secrets, and a `latency` assertion.
- `assets/scenarios/errors.yaml` — `expect_failure` for `tool_error` (a tool
  returning `isError`) vs `protocol` (an unknown tool the server never runs).
- `assets/mcpdev.toml` — `stdio` and `http` profiles with a memory-safe fixture
  target.

## Minimal suite

```yaml
schema_version: 1
name: weather-smoke
target:
  command: ["./bin/weather"]
protocol: auto
cases:
  - name: echo
    steps:
      - call:
          tool: echo            # the step key is `tool`, never `name`
          arguments:
            text: hi
        assert:
          - equal:
              path: $.result.isError   # not $.isError — see "Assertion data model"
              value: false
          - contains:
              path: $.result.text
              value: hi
```

`target` is exactly one of `profile`, `command` (argv array), or `url`.
CLI `--command` / `--url` / `--profile` override this when you pass them.

Secrets: `{from_env: ENV_NAME}` only (no `value`). Interpolate as
`${secrets.name}`. Do not put tokens in `variables`.

A server that indexes or builds on connect can miss the default handshake
window (exit `10`, connection). Raise `defaults.connect_timeout` (and
`defaults.timeout` for slow tools), and point the target at a small fixture
workspace rather than a huge repo.

## One operation per step

| Key | Meaning |
| --- | --- |
| `inspect` | List `tools`, `prompts`, `resources`, `templates`, or `all` |
| `call` | `tools/call`: `tool` name plus a JSON `arguments` object |
| `prompt` | `prompts/get` with string arguments |
| `read` | `resources/read`: exactly one of `uri` or `template` |
| `snapshot` | Compare live interface to a baseline file (`fail_on`, `ignore`, `rename`) |

Optional: `id`, `expect_failure.category`, `capture` `{name, path}`, `assert`.

`expect_failure.category` values: `usage`, `connection`, `protocol`,
`assertion`, `interface_change`, `incomplete`, `internal`, `tool_error`.

Interpolation in strings: `${variable}`, `${capture}`, `${secrets.name}`.
This covers step arguments and the value-bearing assertion fields
`equal.value`, `contains.value`, and string values inside an inline `schema`;
paths and closed-vocabulary fields stay literal. Missing capture paths are
errors, not empty strings, and unknown references in assertion values fail at
load like step arguments.

## Assertion data model

Every assertion `path` is a JSON Path over the **step response envelope**, not
the raw `--format json` CLI shape. `$` is that envelope; the useful fields are:

| Path | Step | Meaning |
| --- | --- | --- |
| `$.ok` | any | transport/RPC success (bool) |
| `$.result.tools[*]` | `inspect` | flat tool array; also `prompts[*]`, `resources[*]`, `templates[*]` |
| `$.result.tools[*].name` | `inspect` | subfields: `.name`, `.description`, `resources[*].uri`, `templates[*].uriTemplate` |
| `$.result.isError` | `call` | `false` on success, `true` on a tool error |
| `$.result.text` | `call` | concatenated text content |
| `$.result.structured` | `call` | the tool's `structuredContent` object, e.g. `$.result.structured.code` |
| `$.result.content[0].text` | `call` | one content block |
| `$.result.contents[0].text` | `read` | resource body; pair with a `content_type` assertion |
| `$.result.messages[0].content.text` | `prompt` | rendered prompt message |

Two traps that silently produce a `skip` (exit `60`, "unevaluable"), never a
clean pass:

- A bare `$.isError` or `$.tools[*]` resolves to nothing. Prefix with
  `$.result.`.
- `inspect` lists are flat arrays here (`$.result.tools[*]`), even though
  `mcpdev --format json inspect` nests them under `.result.tools.items`. Do not
  write `.items` in a scenario path.

## Assertions (one kind each)

Each entry below is a **list item under a step's `assert:`** — including
`latency`. There is no step-level `latency`, `schema`, or `timeout` key; putting
one beside `assert` fails at load with `unknown field` (exit `2`).

| Kind | Check |
| --- | --- |
| `equal` | JSON Path equals `value` or a variable/capture |
| `contains` | String or array at `path` contains `value` |
| `json_path` | `exists` and/or `count` |
| `schema` | Value matches inline JSON Schema or `file` |
| `content_type` | MIME `equals` |
| `latency` | Step duration `max` (Go duration) |
| `error_category` | Same vocabulary as `expect_failure` |

`equal.value` and `contains.value` accept `${...}` interpolation, so a prior
capture can be asserted without duplicating its literal (inline `schema` string
values interpolate too). Prefer `variable:` on `equal` when the whole value is a
binding.

```yaml
        assert:
          - json_path:
              path: $.result.tools[*]
              exists: true
              count: 12
          - latency:
              max: 30s
```

Unavailable assertions are skip or fail, never a silent pass.

## Hooks

`setup` / `cleanup` may use `exec` as an **argv array** only. Do not interpolate
server-returned captures into hook argv.

## Report exits

| Outcome | Exit |
| --- | --- |
| All required assertions passed | `0` |
| Assertion failed or `expect_failure` missed | `30` |
| Assertion could not be evaluated | `60` |
| Connect / child shutdown | `10` |
| Handshake / unsupported primitive | `20` |
| Cleanup / close failed | `70` |
| Invalid document | `2` (no connect) |

`--junit PATH`: assertion problems are JUnit `failure`; connection, protocol,
and cleanup are JUnit `error`.

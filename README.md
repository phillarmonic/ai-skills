# Phillarmonic AI Skills

A catalog of portable AI agent skills maintained by Phillarmonic.

## Available skills

| Skill               | Description                                                                                                       |
|---------------------|-------------------------------------------------------------------------------------------------------------------|
| `repertoire`        | Install and manage portable AI agent skills with Repertoire.                                                      |
| `zensical`          | Create, configure, build, and troubleshoot Zensical sites.                                                        |
| `zensical-glossary` | Create and maintain rich glossary pages for Zensical sites.                                                       |
| `common-stubs`      | Supply small, reusable starter files for common project conventions.                                              |
| `drun`              | Use when working with drun specs or xdrun automation: tasks, parameters, platforms, interpolation, CI, and hooks. |
| `openobserve-investigate` | Investigate logs, metrics, traces, incidents, and stream metadata through OpenObserve MCP or its HTTP API. |

## OpenObserve investigation

Install the skill with Repertoire:

```shell
repertoire add openobserve-investigate --catalog phillarmonic
```

The skill guides schema-first, read-only incident investigation through an
OpenObserve MCP server. It also exposes verified MCP configuration templates
for Cursor, VS Code, and Claude Desktop:

```shell
repertoire stub list openobserve-investigate
repertoire stub get openobserve-investigate/cursor-mcp
repertoire stub get openobserve-investigate/vscode-mcp
repertoire stub get openobserve-investigate/claude-desktop-mcp
```

`stub get` returns the installed template path and client-specific merge
instructions. Templates contain placeholders only; keep real authorization
values out of version control.

## Repertoire

The root `repertoire.yaml` makes this repository consumable by
[Repertoire](https://github.com/phillarmonic/repertoire-ai).

This is Repertoire's official vendored skill set. The hosted repository is available through the built-in `phillarmonic`
catalog and does not need to be declared separately. When a skill has a single visible definition, these commands are
equivalent:

```shell
repertoire add zensical
repertoire add zensical --catalog phillarmonic
```

If another visible catalog also defines `zensical`, Repertoire lists every matching definition and requires
`--catalog phillarmonic` to select this one.

For local development, override the built-in catalog with a checkout:

```shell
repertoire catalog add /path/to/ai-skills --name phillarmonic --force
repertoire add zensical --catalog phillarmonic
```

Every listed path must be contained in this repository and contain an Agent Skills-compatible `SKILL.md`.

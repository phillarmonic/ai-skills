# Phillarmonic AI Skills

A catalog of portable AI agent skills maintained by Phillarmonic.

## Repertoire

The root `repertoire.yaml` makes this repository consumable by
[Repertoire](https://github.com/phillarmonic/repertoire-ai).

The hosted catalog is available by default:

```shell
repertoire list --available --catalog phillarmonic
repertoire add zensical --catalog phillarmonic
```

For local development, override the built-in catalog with a checkout:

```shell
repertoire catalog add /path/to/ai-skills --name phillarmonic --force
repertoire add zensical --catalog phillarmonic
```

Every listed path must be contained in this repository and contain an Agent
Skills-compatible `SKILL.md`.

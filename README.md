# Phillarmonic AI Skills

A catalog of portable AI agent skills maintained by Phillarmonic.

## Repertoire

The root `repertoire.yaml` makes this repository consumable by
[Repertoire](https://github.com/phillarmonic/repertoire-ai).

This is Repertoire's official vendored skill set. The hosted repository is
available through the built-in `phillarmonic` catalog and does not need to be
declared separately. When a skill has a single visible definition, these
commands are equivalent:

```shell
repertoire add zensical
repertoire add zensical --catalog phillarmonic
```

If another visible catalog also defines `zensical`, Repertoire lists every
matching definition and requires `--catalog phillarmonic` to select this one.

For local development, override the built-in catalog with a checkout:

```shell
repertoire catalog add /path/to/ai-skills --name phillarmonic --force
repertoire add zensical --catalog phillarmonic
```

Every listed path must be contained in this repository and contain an Agent
Skills-compatible `SKILL.md`.

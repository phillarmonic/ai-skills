# File stubs

A skill may expose small file-backed stubs through an optional `stubs.yaml` in
its root. Read this when you are authoring stub definitions or when an agent
needs the exact `stub list` / `stub get` workflow.

Loose catalogs cannot ship stubs. A real `repertoire.yaml` is required before
`stubs.yaml` is recognized.

## Define stubs

```yaml
schema: 1
stubs:
  editorconfig:
    description: Ensure text files end with a newline.
    path: assets/.editorconfig
    instructions: |
      Create or merge the repository-root .editorconfig while preserving
      existing settings.
```

Each stub points to one contained regular file and includes non-empty
description and instructions.

## Fetch a stub

Install the containing skill, then ask Repertoire for a verified local asset
path:

```bash
repertoire stub list
repertoire stub list common-stubs
repertoire stub get common-stubs/editorconfig
```

Repertoire prints the path and instructions for the agent. By default it does
not copy, merge, execute, or print the asset itself. When the stub instructions
call for a wholesale file creation, add `--raw` to write only the asset bytes to
stdout, which is safe to redirect:

```bash
repertoire stub get --raw common-stubs/gitattributes > .gitattributes
```

Never redirect the default (advisory) output into a file: it emits the
`Stub`/`Description`/`Asset`/`Instructions` header, not the asset content. Use
the advisory form only to locate the `Asset:` path when the instructions require
merging into an existing file.

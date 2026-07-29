# Configuration and behavior

Read this reference when changing `zensical_glossary` configuration or
diagnosing term extraction, merging, matching, links, or tooltip text.

## Configuration

All options live under
`[project.markdown_extensions.zensical_glossary]`.

| Option               | Type    | Default         | Behavior                                                                                           |
|----------------------|---------|-----------------|----------------------------------------------------------------------------------------------------|
| `glossary_file`      | string  | `"glossary.md"` | Single source path relative to `docs_dir`.                                                         |
| `glossary_files`     | list    | `[]`            | Multiple sources (paths or globs). Wins over `glossary_file` when non-empty.                       |
| `inline_definitions` | boolean | `false`         | Scan every page for `<!-- zensical-glossary: Term -->` markers.                                    |
| `heading_level`      | integer | `2`             | Exact Markdown heading level treated as a term in glossary files.                                  |
| `first_only`         | boolean | `true`          | Annotate only the first occurrence of each term per page.                                          |
| `case_sensitive`     | boolean | `false`         | Require matching capitalization when true.                                                         |
| `min_length`         | integer | `2`             | Ignore terms shorter than this many characters.                                                    |
| `max_definition`     | integer | `280`           | Truncate flattened tooltip text to this length.                                                    |
| `base_url`           | string  | derived         | Single file: full glossary URL override. Multiple files: prefix joined with each derived page URL. |
| `language`           | string  | site language   | Tooltip interface language: `en`, `fr`, `es`, or `pt`.                                             |
| `labels`             | table   | `{}`            | Override supported tooltip labels, currently `more`.                                               |
| `docs_dir`           | string  | `"docs"`        | Fallback when Zensical context does not provide `docs_dir`.                                        |

`enabled = false` disables the extension when passed by the Markdown extension
loader.

## Multiple glossary sources

- `glossary_files` accepts plain paths and glob patterns (`*`, `?`, `[`),
  resolved relative to `docs_dir`. Globs are expanded and sorted for
  deterministic builds; plain paths are kept in configuration order even when
  the file is missing (missing files contribute no entries).
- All sources are merged into one term index at build time. Files are
  processed in configuration order and, within each file, in document order;
  the first definition of a term wins (compared case-insensitively).
- Each term links to the page that defines it, so a term from
  `glossary/api.md` links to `/glossary/api/#term`.
- Every glossary source page is skipped during annotation, so glossary pages
  never link to themselves.

## Inline definitions

With `inline_definitions = true`, every Markdown page under `docs_dir` (dot
directories excluded) is scanned for `<!-- zensical-glossary: Term -->`
markers.

- The definition runs from the marker to the next heading of any level, the
  next marker, or the end of the file.
- The marker is replaced by an invisible anchor,
  `<a id="<slug>" class="zensical-glossary-def"></a>`, and the page becomes
  the link target for that term.
- Markers are merged into the same index after glossary files. When a term is
  defined in both places, the glossary file wins; the inline marker still
  renders its anchor on the page.
- Inline definitions ignore `heading_level`; the term comes from the marker
  text.
- A page never links to the terms it defines itself, whether defined by
  headings or markers. In single-file mode, explicit `base_url` applies only
  to glossary-file entries; inline entries always derive their own page URL.

## Entry extraction

- Strip initial YAML front matter before parsing.
- Glossary files: treat only headings at `heading_level` as terms; end a
  definition at the next heading whose level is less than or equal to
  `heading_level`; include deeper headings and their content in the current
  definition.
- Flatten images, Markdown links, reference links, inline code, common emphasis
  markers, and whitespace for tooltip text.
- Ignore terms below `min_length`, entries with empty definitions, and later
  case-insensitive duplicates.
- Sort terms longest-first so a multi-word term wins over its shorter
  substring.
- Generate anchors with Python-Markdown's TOC slugification.

Because duplicate and empty entries are ignored silently, inspect them during
content review instead of depending on the build to reject them.

Parsed sources are cached by modification time: unchanged files are never
re-parsed, and the merged index and term regex are built once per build, so
watch-mode rebuilds stay fast.

## Matching

Matching uses word-like boundaries that also treat hyphens as part of a word.
It therefore avoids matching a term inside a larger identifier or hyphenated
word. Matching is case-insensitive by default.

The extension skips these rendered elements:

```text
a, abbr, code, pre, script, style, kbd, h1, h2, h3, h4, h5, h6
```

It also skips elements already carrying `glossary-term`, every glossary source
page, and each term's own defining page.

## Derived glossary URLs

URLs are derived per defining page. With directory URLs enabled:

- `glossary.md` becomes `/glossary/`.
- `reference/glossary.md` becomes `/reference/glossary/`.
- `index.md` or `README.md` becomes `/`.
- A nested `index.md` or `README.md` becomes its containing directory route.

With directory URLs disabled, `glossary.md` becomes `/glossary.html`.

These defaults are root-relative. For a GitHub Pages project site or another
deployment below a path prefix, set an explicit `base_url`. Its meaning
depends on the mode:

```toml
[project.markdown_extensions.zensical_glossary]
# Single file: base_url IS the glossary URL; term links append #<slug>.
base_url = "https://OWNER.github.io/REPOSITORY/glossary/"

# Multiple files: base_url is a prefix joined with each derived page URL.
glossary_files = ["glossary/*.md"]
base_url = "https://OWNER.github.io/REPOSITORY"
# A term defined in glossary/api.md links to
# https://OWNER.github.io/REPOSITORY/glossary/api/#term
```

In multi-file mode the prefix is joined with a slash, so omit the trailing
slash from `base_url`. In single-file mode it is used verbatim, so include the
trailing slash when appropriate.

## Tooltip language

Resolve tooltip interface language in this order:

1. Extension `language`.
2. Zensical theme `language`.
3. English.

Normalize regional forms such as `pt-BR` and `pt_BR` to `pt`; unsupported
languages fall back to English. `labels.more` overrides the built-in
call-to-action. Glossary definitions remain exactly as authored.

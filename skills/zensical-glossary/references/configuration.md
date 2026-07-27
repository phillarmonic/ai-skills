# Configuration and behavior

Read this reference when changing `zensical_glossary` configuration or
diagnosing term extraction, matching, links, or tooltip text.

## Configuration

All options live under
`[project.markdown_extensions.zensical_glossary]`.

| Option | Type | Default | Behavior |
| --- | --- | --- | --- |
| `glossary_file` | string | `"glossary.md"` | Source path relative to `docs_dir`. |
| `heading_level` | integer | `2` | Exact Markdown heading level treated as a term. |
| `first_only` | boolean | `true` | Annotate only the first occurrence of each term per page. |
| `case_sensitive` | boolean | `false` | Require matching capitalization when true. |
| `min_length` | integer | `2` | Ignore terms shorter than this many characters. |
| `max_definition` | integer | `280` | Truncate flattened tooltip text to this length. |
| `base_url` | string | derived | Override the glossary page URL used in generated links. |
| `language` | string | site language | Tooltip interface language: `en`, `fr`, `es`, or `pt`. |
| `labels` | table | `{}` | Override supported tooltip labels, currently `more`. |
| `docs_dir` | string | `"docs"` | Fallback when Zensical context does not provide `docs_dir`. |

`enabled = false` disables the extension when passed by the Markdown extension
loader.

## Entry extraction

- Strip initial YAML front matter before parsing.
- Treat only headings at `heading_level` as terms.
- End a definition at the next heading whose level is less than or equal to
  `heading_level`.
- Include deeper headings and their content in the current definition.
- Flatten images, Markdown links, reference links, inline code, common emphasis
  markers, and whitespace for tooltip text.
- Ignore terms below `min_length`, entries with empty definitions, and later
  case-insensitive duplicates.
- Sort terms longest-first so a multi-word term wins over its shorter
  substring.
- Generate anchors with Python-Markdown's TOC slugification.

Because duplicate and empty entries are ignored silently, inspect them during
content review instead of depending on the build to reject them.

## Matching

Matching uses word-like boundaries that also treat hyphens as part of a word.
It therefore avoids matching a term inside a larger identifier or hyphenated
word. Matching is case-insensitive by default.

The extension skips these rendered elements:

```text
a, abbr, code, pre, script, style, kbd, h1, h2, h3, h4, h5, h6
```

It also skips elements already carrying `glossary-term` and skips the glossary
page itself.

## Derived glossary URLs

With directory URLs enabled:

- `glossary.md` becomes `/glossary/`.
- `reference/glossary.md` becomes `/reference/glossary/`.
- `index.md` or `README.md` becomes `/`.
- A nested `index.md` or `README.md` becomes its containing directory route.

With directory URLs disabled, `glossary.md` becomes `/glossary.html`.

These defaults are root-relative. For a GitHub Pages project site or another
deployment below a path prefix, set an explicit absolute or correctly prefixed
`base_url`, including the trailing slash when appropriate:

```toml
[project.markdown_extensions.zensical_glossary]
base_url = "https://OWNER.github.io/REPOSITORY/glossary/"
```

Generated term links append `#<slug>` directly to `base_url`.

## Tooltip language

Resolve tooltip interface language in this order:

1. Extension `language`.
2. Zensical theme `language`.
3. English.

Normalize regional forms such as `pt-BR` and `pt_BR` to `pt`; unsupported
languages fall back to English. `labels.more` overrides the built-in
call-to-action. Glossary definitions remain exactly as authored.

---
name: zensical
description: >-
  Generate, maintain, diagnose, and extend Zensical static documentation sites.
  Use this skill whenever the task involves the `zensical` CLI (new/build/serve),
  authoring Markdown docs, editing zensical.toml or mkdocs.yml configuration,
  configuring themes/navigation/Markdown extensions/plugins, migrating a
  Material for MkDocs project, fixing build warnings and broken links, or
  deploying a Zensical site. Zensical is the static site generator by the
  creators of Material for MkDocs.
---

# Zensical

Zensical is a modern static site generator by the creators of Material for
MkDocs. It writes documentation in Markdown and produces a searchable,
customizable static site. Internally it is a Rust build engine (crates
`zensical`, `zensical-serve`, `zensical-watch`) with a Python front end (the
`zensical` package) that provides the CLI, config parsing, Markdown rendering,
and MkDocs compatibility. Agents usually interact through the CLI and by editing
`zensical.toml` and Markdown files.

## When to use this skill

Use it for any of these:

- Scaffolding a new site (`zensical new`).
- Building or previewing a site (`zensical build`, `zensical serve`).
- Editing configuration (`zensical.toml`, or an existing `mkdocs.yml`).
- Configuring theme, colors, fonts, navigation, features, Markdown extensions.
- Authoring content (admonitions, code blocks, tabs, diagrams, footnotes, math).
- Diagnosing build warnings, broken links, and invalid anchors.
- Migrating a Material for MkDocs / MkDocs project.
- Deploying to GitHub Pages or any static host.

## Prerequisites and setup

Zensical is published on PyPI as `zensical` and requires Python >= 3.10. The CLI
entry point is `zensical` (mapped to `zensical.main:cli`).

```bash
# Preferred: run without installing globally
uvx zensical --help
pipx run zensical --help

# Or install into an environment
pip install zensical
zensical --version # prints the bare version string, e.g. 0.1.0
```

If `zensical` is not on `PATH` but the package is installed, use
`python -m zensical`. When working inside this repository (a maturin project),
build the native extension first with `uv sync` then `uv run maturin develop`,
and invoke via `uv run zensical ...`.

Always confirm the tool is available before giving build/serve instructions:
`zensical --version`. Do not assume a global install.

## CLI reference

There are exactly three commands. All are thin wrappers; the real work happens
in the Rust runtime with Python callbacks for config parsing and Markdown.

### `zensical new [DIRECTORY]`

Scaffolds a template project into the current directory (or `DIRECTORY`).

- Creates `zensical.toml`, `docs/index.md`, `docs/markdown.md`, and a GitHub
  Pages workflow at `.github/workflows/docs.yml`.
- Fails if the target is a file, or if `zensical.toml` already exists there.
- Never overwrites existing template files; missing ones are added, so it is
  safe to re-run to restore a deleted starter file.

```bash
zensical new # scaffold into current folder
zensical new my-docs # scaffold into ./my-docs
```

### `zensical serve [OPTIONS]`

Builds the site and serves it locally with live reload (watches sources and
rebuilds incrementally).

| Option                 | Alias | Description                                                                    |
|------------------------|-------|--------------------------------------------------------------------------------|
| `--config-file PATH`   | `-f`  | Path to config file. Must exist.                                               |
| `--dev-addr <IP:PORT>` | `-a`  | Bind address (default `localhost:8000`).                                       |
| `--open`               | `-o`  | Open the preview in the default browser.                                       |
| `--strict`             | `-s`  | Accepted but currently unsupported for serve; prints a warning and is ignored. |

```bash
zensical serve
zensical serve -a 0.0.0.0:8080 -o
```

Serve runs until interrupted (Ctrl+C). When running it from an agent, start it
as a detached/background process and poll the address with `curl`, then stop it
by PID. Do not block a foreground session on it.

### `zensical build [OPTIONS]`

Builds the static site into the output directory (`site_dir`, default `site/`).

| Option               | Alias | Description                                                             |
|----------------------|-------|-------------------------------------------------------------------------|
| `--config-file PATH` | `-f`  | Path to config file. Must exist.                                        |
| `--clean`            | `-c`  | Clean the cache/output before building. Use for reproducible CI builds. |
| `--strict`           | `-s`  | Strict mode: abort the build if any enabled warning is emitted.         |

```bash
zensical build # incremental build into ./site
zensical build --clean # clean rebuild (recommended for CI/deploy)
zensical build --strict # fail on warnings (recommended for link checking)
```

### Config file auto-discovery

When `--config-file` is omitted, both `build` and `serve` search the current
directory in this order and use the first match:

1. `zensical.toml`
2. `mkdocs.yml`
3. `mkdocs.yaml`

If none exists, the command exits with "No config file found in the current
folder." Run the CLI from the project root, or pass `-f`.

## Configuration

Zensical's native format is `zensical.toml`. It also reads MkDocs `mkdocs.yml`
for compatibility. The file extension decides the parser: `.toml` is parsed as
Zensical config, anything else as MkDocs YAML.

### zensical.toml structure

Top-level settings live under a `[project]` table (a bare top level without
`[project]` also works; if `project` is present it is unwrapped). Nested areas
use TOML sub-tables such as `[project.theme]` and
`[project.markdown_extensions.*]`.

```toml
[project]
site_name = "Documentation"          # REQUIRED; build fails without it
site_description = "..."
site_author = "..."
site_url = "https://example.com/"     # set for production (canonical URLs, sitemap)
copyright = "Copyright &copy; 2026 The authors"   # HTML allowed

# Directories (must stay inside the project root; site_dir != docs_dir)
docs_dir = "docs"                     # default "docs"; must exist
site_dir = "site"                     # default "site"; build output
use_directory_urls = true             # pretty URLs (default true)

# Optional extra assets, relative to docs_dir
extra_css = ["stylesheets/extra.css"]
extra_javascript = ["javascripts/extra.js"]

# Repository integration (enables edit/view buttons via features)
repo_url = "https://github.com/user/repo"
repo_name = "user/repo"               # auto-derived for github/gitlab/bitbucket

# Live-reload watch paths beyond docs/ and theme (strings only)
watch = ["includes", "macros.py"]

[project.theme]
name = "material"                     # "material"/"zensical" -> built-in theme
variant = "modern"                    # "modern" (default) or "classic"
language = "en"                       # 60+ languages supported
features = ["navigation.instant", "content.code.copy"]
# custom_dir = "overrides"            # template overrides directory
# logo = "assets/logo.png"            # relative to docs_dir
# favicon = "assets/images/favicon.png"

[project.theme.icon]
logo = "lucide/book-open"

[project.theme.font]
text = "Inter"                        # "modern" defaults: Inter / JetBrains Mono
code = "JetBrains Mono"               # "classic" defaults: Roboto / Roboto Mono

# Color schemes (array of tables => palette toggle)
[[project.theme.palette]]
scheme = "default"                    # light
toggle.icon = "lucide/sun"
toggle.name = "Switch to dark mode"

[[project.theme.palette]]
scheme = "slate"                      # dark
toggle.icon = "lucide/moon"
toggle.name = "Switch to light mode"

# Social links
[[project.extra.social]]
icon = "fontawesome/brands/github"
link = "https://github.com/user/repo"
```

Key rules enforced by the parser (`python/zensical/config.py`):

- `site_name` is required. Missing it raises a configuration error.
- `docs_dir` must exist and be inside the project root.
- `site_dir` must be non-empty, inside the project root, and different from
  `docs_dir`.
- `theme.name` set to `material` or `zensical` uses the built-in theme. Setting
  it to `null` opts out of extending the Material theme. Other names must be an
  installed MkDocs theme entry point.
- `variant = "modern"` is the default look; `"classic"` reproduces the
  traditional Material for MkDocs appearance and switches default fonts/icons.

### Navigation

Without a `nav`, Zensical derives navigation from the `docs_dir` folder
structure. To define it explicitly in TOML, use inline tables (title = path)
and nested arrays for sections:

```toml
nav = [
  { "Home" = "index.md" },
  { "Getting started" = "start.md" },
  { "Guides" = [
    { "Install" = "guides/install.md" },
    { "Configure" = "guides/configure.md" },
  ] },
]
```

`index.md` and `README.md` are treated as section index pages (pair with the
`navigation.indexes` feature).

### Markdown extensions

Each extension is a TOML sub-table under `[project.markdown_extensions.*]`; its
keys are the extension options. When `markdown_extensions` is omitted, Zensical
applies a sensible default set (abbr, admonition, attr_list, def_list,
footnotes, md_in_html, toc with permalinks, and many `pymdownx.*` extensions
including superfences with a mermaid fence, tabbed, highlight, emoji, tasklist).
Defining `markdown_extensions` yourself replaces the defaults, so re-add what
you need.

```toml
[project.markdown_extensions.toc]
permalink = true

[project.markdown_extensions.pymdownx.highlight]
anchor_linenums = true
line_spans = "__span"
pygments_lang_class = true

[project.markdown_extensions.pymdownx.superfences]
custom_fences = [
  { name = "mermaid", class = "mermaid", format = "pymdownx.superfences.fence_code_format" }
]
```

Function-valued options are given as dotted import strings and resolved at load
time (for example `emoji_generator = "zensical.extensions.emoji.to_svg"`).

### Plugins and compatibility shims

Zensical is not a plugin host, but it maps several common MkDocs plugins onto
built-in Markdown extensions when it sees them in `plugins`: `autorefs`,
`mkdocstrings`, `macros`, `mkdocs-glightbox`, and `markdown-exec`. Requirements:

- `mkdocstrings` and `markdown-exec` must be installed or the build errors out
  with a clear message.
- `material.extensions` and the legacy `materialx` namespaces in YAML are
  automatically remapped to `zensical.extensions`.

## Authoring content

Content is standard Markdown plus the enabled extensions. The scaffolded
`docs/markdown.md` and `docs/index.md` are the canonical cheat sheet. Common
constructs:

- Admonitions: `!!! note`, `!!! warning`; collapsible with `??? info "Title"`.
- Code blocks: fenced with language, `title="..."`, `hl_lines="2 3"`, and
  annotations via `# (1)!` markers referencing a numbered list below.
- Content tabs: `=== "Tab title"` blocks.
- Diagrams: ```` ```mermaid ```` fenced blocks.
- Footnotes: `text[^1]` with `[^1]: definition` (rendered as tooltips).
- Formatting: `==highlight==`, `^^insert^^`, `~~delete~~`, `H~2~O`, `A^T^A`,
  keys `++ctrl+alt+del++`.
- Icons/emoji: `:material-icon:`, `:rocket:` shortcodes.
- Math: `$$ ... $$` (requires MathJax to be added on the page or globally).
- Task lists: `- [x]` / `- [ ]`.

Per-page front matter (YAML at the top of a `.md` file) sets metadata such as
the page icon:

```markdown
---
icon: lucide/rocket
---

# Page title
```

## Diagnosing and fixing builds

The Rust runtime emits `ariadne`-style diagnostics (source snippet with a
caret) to stderr. Diagnostic categories and the config that gates them (under
`[project.validation]`, MkDocs `validation` is also mapped):

| Issue                               | Default   | Config key              |
|-------------------------------------|-----------|-------------------------|
| Invalid link (broken internal link) | warn (on) | `invalid_links`         |
| Invalid link anchor                 | warn (on) | `invalid_link_anchors`  |
| Unresolved link reference           | off       | `unresolved_references` |
| Unresolved footnote reference       | off       | `unresolved_footnotes`  |
| Unused link definition              | off       | `unused_definitions`    |
| Unused footnote                     | off       | `unused_footnotes`      |
| Shadowed link definition            | off       | `shadowed_definitions`  |
| Shadowed footnote                   | off       | `shadowed_footnotes`    |

```toml
[project.validation]
invalid_links = true
invalid_link_anchors = true
unresolved_references = true   # opt in to stricter checks
```

Diagnostic workflow for agents:

1. Reproduce with a clean build: `zensical build --clean`.
2. To gate CI on warnings, add `--strict` so warnings abort the build
   (exit non-zero with "Aborted because --strict flag is set").
3. Read the file path, line, and message. Fix broken links by correcting the
   relative path (links are validated against `docs_dir`) or the `#anchor`
   (anchors come from heading slugs / `toc`).
4. Re-run until clean.

Common failure messages and fixes:

- "No config file found in the current folder." Run from the project root or
  pass `-f path/to/zensical.toml`.
- "Missing required setting: site_name." Add `site_name` under `[project]`.
- "Docs directory does not exist." Create `docs/` or fix `docs_dir`.
- "site_dir and docs_dir must be different" / "must be within project root."
  Adjust the paths.
- "Theme 'X' is not installed." Install the theme package or use `material`.
- "mkdocstrings/markdown-exec ... is not installed." Install the dependency or
  remove the plugin.

## Extending

- Theme overrides: set `theme.custom_dir` to a directory (for example
  `overrides`) holding template partials/blocks that override the built-in
  theme. Custom themes may also `extends` another theme via `mkdocs_theme.yml`.
- Extra styling/scripts: `extra_css` and `extra_javascript` (paths relative to
  `docs_dir`).
- Icons and logos: use bundled icon sets (`lucide/*`, `fontawesome/*`,
  `material/*`, `simple/*`) via `theme.icon.*`, or a custom image via
  `theme.logo` / `theme.favicon`.
- Macros/variables: enable the `macros` plugin/extension and provide a
  `main.py` (or configured module) plus optional `include_yaml` variables;
  add these to `watch` so live reload picks up changes.
- Snippets: `pymdownx.snippets` `auto_append`/`base_path` files are watched and
  trigger rebuilds.

## Deploying

`zensical new` scaffolds `.github/workflows/docs.yml`, which builds with
`zensical build --clean` and publishes `site/` to GitHub Pages on pushes to
`master`/`main`. For any other host, run `zensical build --clean` and serve the
resulting `site/` directory. Set `site_url` for correct canonical URLs and
sitemap generation. Track an empty `site/` in git by adding a dotfile such as
`site/.gitkeep`; the cleaner preserves paths beginning with `.`.

## Quick task recipes

- New site: `zensical new my-docs && cd my-docs && zensical serve -o`.
- Preview existing: `cd <project> && zensical serve`.
- Production build: `zensical build --clean`, optionally `--strict`.
- Link-check a project: enable validation keys, then `zensical build --strict`.
- Migrate MkDocs: keep `mkdocs.yml` and run `zensical build`; it will parse the
  YAML, remap namespaces, and shim supported plugins. Move to `zensical.toml`
  incrementally for the native experience.

---
name: zensical-glossary
description: >-
  Create, configure, maintain, reorganize, and verify enriched glossary pages
  with the zensical-glossary Python-Markdown extension for Zensical sites. Use
  when adding hover definitions and canonical term links, editing
  docs/glossary.md or another glossary source, spreading terms across multiple
  glossary pages or glob patterns, defining terms inline with comment markers,
  changing term heading levels or matching behavior, renaming or removing
  glossary terms, localizing tooltip labels, fixing glossary links or tooltips,
  or upgrading zensical-glossary.
---

# Zensical Glossary

Treat glossary sources as the canonical term database. The `zensical_glossary`
extension extracts entries from one or more Markdown pages — and optionally
from `<!-- zensical-glossary: Term -->` markers on regular pages — merges them
into a single index, then annotates matching prose elsewhere in the rendered
site with tooltips and links back to the page that defines each term.

## Establish context

1. Locate `zensical.toml` and read `[project]`, `docs_dir`, `nav`,
   `use_directory_urls`, `site_url`, and
   `[project.markdown_extensions.zensical_glossary]`.
2. Resolve the glossary sources: `glossary_files` (paths or glob patterns,
   relative to `docs_dir`) wins over `glossary_file`; the default is
   `docs/glossary.md`. Expand any glob to know which pages participate.
3. Read every glossary source before editing it. When `inline_definitions` is
   enabled, also find marker-defined terms with:

   ```bash
   rg -n 'zensical-glossary:' docs
   ```

4. Inspect nearby documentation for terminology and existing explicit links
   with:

   ```bash
   rg -n -i 'term|related phrase' docs
   ```

5. Confirm the installed versions without changing the environment:

   ```bash
   uv run python -c 'import importlib.metadata as m; print("zensical", m.version("zensical")); print("zensical-glossary", m.version("zensical-glossary"))'
   ```

   Adapt the command to the project's package manager. If the package is
   absent and installation is in scope, add it with `uv add
   zensical-glossary` or the project's equivalent.

Read [references/configuration.md](references/configuration.md) when changing
options, debugging matching or merging, or reasoning about generated URLs and
anchors.

## Create a glossary

1. Choose the glossary shape: a single page for a small site, several
   domain-focused pages for a large one, or inline markers for terms best
   defined where they are used. The shapes compose — files stay the curated
   canonical source and markers the lightweight fallback.
2. Choose one term heading level for glossary files. Use `##` for a flat
   glossary or `###` when `##` headings group terms into sections.
3. Create the glossary inside `docs_dir`. Give every term a non-empty
   definition:

   ```markdown
   # Glossary

   ## Authoring

   ### Canonical term

   A short, self-contained definition that works well in a tooltip.

   Add examples, caveats, and deeper headings after the opening definition
   when the full glossary entry needs more detail.
   ```

4. Enable the extension in `zensical.toml`, preserving every existing Markdown
   extension. Single page:

   ```toml
   [project.markdown_extensions.zensical_glossary]
   glossary_file = "glossary.md"
   heading_level = 3
   first_only = true
   ```

   Multiple pages (plain paths in configuration order, or globs expanded
   against `docs_dir` and sorted for deterministic builds):

   ```toml
   [project.markdown_extensions.zensical_glossary]
   glossary_files = ["glossary/core.md", "glossary/api.md"]
   # or: glossary_files = ["glossary/**/*.md"]
   heading_level = 3
   ```

5. Add glossary pages to explicit `nav` when the project defines one. Do not
   add duplicate navigation when Zensical derives it from the directory tree.
6. Build and inspect the result using the verification workflow below.

## Define terms inline

When a term is best explained where it is used, enable
`inline_definitions = true` and mark a definition directly on any regular
page:

```markdown
<!-- zensical-glossary: Widget -->

A widget is a reusable UI unit that...
```

- The paragraphs after the marker — up to the next heading or the next marker
  — become the definition. Keep them self-contained; they are the tooltip
  text.
- The marker renders as an invisible anchor (`#widget`); every other mention
  of the term on the site links to it.
- All pages are scanned once per build (cached by modification time) and
  merged with any glossary files into one hybrid index. When a term is
  defined both inline and in a glossary file, the glossary file wins.
- Inline entries derive their link target from their own page; `base_url`
  never rewrites them in single-file mode.

## Maintain entries

- Keep the heading text canonical and the first definition sentence concise.
  Tooltip text is flattened to plain text and truncated by `max_definition`.
- Use exact term headings for phrases that should match. Longer terms win over
  shorter overlapping terms.
- Treat terms as unique case-insensitively across the whole merged index.
  `case_sensitive` changes prose matching, not duplicate detection. With
  multiple sources the first file in configuration order wins; with inline
  markers the glossary file wins. Remove accidental duplicates instead of
  relying on precedence.
- Keep category headings shallower than `heading_level`. Deeper headings become
  part of the current term's definition.
- Avoid very broad words that create noisy links. Increase `min_length`, rename
  the term more precisely, or enable case-sensitive matching when appropriate.
- Prefer `first_only = true` for normal prose. Set it to `false` only when every
  occurrence adds value.
- Remember that headings, code, preformatted text, existing links,
  abbreviations, keyboard markup, scripts, styles, glossary source pages, and
  each term's own defining page are deliberately not annotated.
- Sources are cached by modification time, so watch-mode rebuilds only re-parse
  changed files. When a page seems stale during `zensical serve`, edit and save
  the source file rather than assuming a configuration problem.

## Update or reorganize a glossary

1. Inspect the current entry and every use of the term, including capitalization
   and explicit `#anchor` links.
2. Decide whether the change affects only the definition or also the canonical
   term. Definition edits preserve links; heading edits usually change the
   generated anchor.
3. For a rename, update the glossary heading or marker text and intended prose
   occurrences. Update hand-authored links to the old anchor. Do not blindly
   replace code, identifiers, URLs, or historical text.
4. For a move between sections of one page, preserve the exact term heading
   unless an anchor change is intended. Moving a term to a different glossary
   page changes its link target to that page — check explicit links.
5. For deletion, search for remaining prose and explicit links. Decide whether
   each occurrence should become plain text, point to a replacement term, or
   remain historical wording.
6. Rebuild cleanly and inspect both the glossary pages and representative pages
   that use changed terms.

When upgrading the dependency, read the installed package metadata and project
lockfile first, use the project's package manager, review the lockfile diff,
then run the same clean verification. Do not assume configuration compatibility
across versions.

## Configure links and language

Let the extension derive glossary URLs for ordinary sites. With multiple
sources or inline definitions, each term derives the URL of the page that
defines it. Set `base_url` when the deployed site uses a repository subpath,
custom route, or another URL that the derived root-relative path cannot
represent. Its meaning depends on the mode: with a single `glossary_file` it
is the full glossary URL (inline definitions still derive their own page
URLs); with `glossary_files` it is a prefix joined with each derived page URL.

Tooltip interface labels follow the explicit extension `language`, then the
Zensical theme language, then English. Glossary content is never translated
automatically. Use `labels = { more = "..." }` only to override the tooltip
call-to-action.

## Verify changes

Run the project's normal checks, then perform a clean strict build when the
installed Zensical version supports it:

```bash
uv run zensical build --clean --strict
```

Verify all of the following:

- The build succeeds without glossary-related warnings or broken links.
- Every glossary page appears at its expected route and each term has the
  expected heading anchor.
- A representative prose occurrence renders with `class="glossary-term"`, a
  useful `data-glossary` value, and an `href` pointing at the page that
  defines the term.
- With multiple sources, terms from different pages link to their own defining
  page, and the first-listed file wins on duplicate terms.
- With inline definitions, each marker renders as
  `<a id="<slug>" class="zensical-glossary-def">`, and the defining page does
  not link to its own terms.
- Repeated terms follow `first_only`.
- Code, headings, existing links, and the glossary pages remain unannotated.
- Tooltip presentation and the localized action label work in a browser.

Use `rg -n 'class="glossary-term"|data-glossary=' site` for a quick generated
HTML check. Do not accept this alone when link routing, styling, JavaScript, or
responsive behavior changed; preview the built site in a browser.

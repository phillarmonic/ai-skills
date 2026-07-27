---
name: zensical-glossary
description: >-
  Create, configure, maintain, reorganize, and verify enriched glossary pages
  with the zensical-glossary Python-Markdown extension for Zensical sites. Use
  when adding hover definitions and canonical term links, editing
  docs/glossary.md or another glossary source, changing term heading levels or
  matching behavior, renaming or removing glossary terms, localizing tooltip
  labels, fixing glossary links or tooltips, or upgrading zensical-glossary.
---

# Zensical Glossary

Treat the glossary Markdown page as the canonical term database. The
`zensical_glossary` extension extracts entries from one exact heading level,
then annotates matching prose elsewhere in the rendered site with tooltips and
links back to those entries.

## Establish context

1. Locate `zensical.toml` and read `[project]`, `docs_dir`, `nav`,
   `use_directory_urls`, `site_url`, and
   `[project.markdown_extensions.zensical_glossary]`.
2. Resolve `glossary_file` relative to `docs_dir`; default to
   `docs/glossary.md`.
3. Read the whole glossary before editing it. Inspect nearby documentation for
   terminology and existing explicit links with:

   ```bash
   rg -n -i 'term|related phrase' docs
   ```

4. Confirm the installed versions without changing the environment:

   ```bash
   uv run python -c 'import importlib.metadata as m; print("zensical", m.version("zensical")); print("zensical-glossary", m.version("zensical-glossary"))'
   ```

   Adapt the command to the project's package manager. If the package is
   absent and installation is in scope, add it with `uv add
   zensical-glossary` or the project's equivalent.

Read [references/configuration.md](references/configuration.md) when changing
options, debugging matching, or reasoning about generated URLs and anchors.

## Create a glossary

1. Choose one term heading level. Use `##` for a flat glossary or `###` when
   `##` headings group terms into sections.
2. Create the glossary inside `docs_dir`. Give every term a non-empty
   definition:

   ```markdown
   # Glossary

   ## Authoring

   ### Canonical term

   A short, self-contained definition that works well in a tooltip.

   Add examples, caveats, and deeper headings after the opening definition
   when the full glossary entry needs more detail.
   ```

3. Enable the extension in `zensical.toml`, preserving every existing Markdown
   extension:

   ```toml
   [project.markdown_extensions.zensical_glossary]
   glossary_file = "glossary.md"
   heading_level = 3
   first_only = true
   ```

4. Add the page to explicit `nav` when the project defines one. Do not add
   duplicate navigation when Zensical derives it from the directory tree.
5. Build and inspect the result using the verification workflow below.

## Maintain entries

- Keep the heading text canonical and the first definition sentence concise.
  Tooltip text is flattened to plain text and truncated by `max_definition`.
- Use exact term headings for phrases that should match. Longer terms win over
  shorter overlapping terms.
- Treat terms as unique case-insensitively. `case_sensitive` changes prose
  matching, not duplicate entry detection. Remove accidental duplicates
  instead of relying on the first entry winning.
- Keep category headings shallower than `heading_level`. Deeper headings become
  part of the current term's definition.
- Avoid very broad words that create noisy links. Increase `min_length`, rename
  the term more precisely, or enable case-sensitive matching when appropriate.
- Prefer `first_only = true` for normal prose. Set it to `false` only when every
  occurrence adds value.
- Remember that headings, code, preformatted text, existing links,
  abbreviations, keyboard markup, scripts, styles, and the glossary page itself
  are deliberately not annotated.

## Update or reorganize a glossary

1. Inspect the current entry and every use of the term, including capitalization
   and explicit `#anchor` links.
2. Decide whether the change affects only the definition or also the canonical
   term. Definition edits preserve links; heading edits usually change the
   generated anchor.
3. For a rename, update the glossary heading and intended prose occurrences.
   Update hand-authored links to the old anchor. Do not blindly replace code,
   identifiers, URLs, or historical text.
4. For a move between sections, preserve the exact term heading unless an
   anchor change is intended.
5. For deletion, search for remaining prose and explicit links. Decide whether
   each occurrence should become plain text, point to a replacement term, or
   remain historical wording.
6. Rebuild cleanly and inspect both the glossary page and representative pages
   that use changed terms.

When upgrading the dependency, read the installed package metadata and project
lockfile first, use the project's package manager, review the lockfile diff,
then run the same clean verification. Do not assume configuration compatibility
across versions.

## Configure links and language

Let the extension derive the glossary URL for ordinary sites. Set `base_url`
when the deployed site uses a repository subpath, custom route, or another URL
that the derived root-relative path cannot represent.

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
- The glossary page appears at the expected route and each term has the
  expected heading anchor.
- A representative prose occurrence renders with `class="glossary-term"`, a
  useful `data-glossary` value, and the correct `href`.
- Repeated terms follow `first_only`.
- Code, headings, existing links, and the glossary page remain unannotated.
- Tooltip presentation and the localized action label work in a browser.

Use `rg -n 'class="glossary-term"|data-glossary=' site` for a quick generated
HTML check. Do not accept this alone when link routing, styling, JavaScript, or
responsive behavior changed; preview the built site in a browser.

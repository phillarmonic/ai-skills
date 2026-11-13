---
name: exporting
description: Export presentations to PDF, PPTX, PNG, or Markdown
---

# Exporting Slides

Export presentations to PDF, PPTX, PNG, or Markdown.

## Browser Exporter

Access at `http://localhost:3030/export`:
- Select format and options
- Preview and download

## CLI Export

Requires playwright:
```bash
pnpm add -D playwright-chromium
```

### PDF Export

```bash
slidev export
slidev export --output my-slides.pdf
```

### PowerPoint Export

```bash
slidev export --format pptx           # each slide as an image
slidev export --format pptx-editable  # native shapes, selectable text
```

`pptx-editable` measures the rendered slides and rebuilds them as PowerPoint shapes. SVG (including Mermaid), canvas, iframes, KaTeX formulas, gradients and CSS filters stay pictures, and any slide that cannot be rebuilt falls back to the image export on its own. Fonts are named, not embedded. `--per-slide` is not supported with it.

### Review exported layout

Open the generated file in the target application or render it with a compatible office renderer, then visually inspect every slide. Browser screenshots validate the browser layout only. A successful export, a valid ZIP package or selectable text does not confirm that the PowerPoint layout matches it.

Confirm that the viewer opened the exact regenerated artifact, using its path and modification time or digest. If the application may retain an older open document, open a uniquely named copy of the latest export. Keep the original untouched and verify that any PDF or images used for review were actually produced from that copy; an acknowledged save command alone does not establish this.

Treat repair, recovery or unreadable-content warnings as failed validation. Investigate the export or post-processing step and regenerate the file until it opens without repair. Check that the expected text, bullets and shapes survived, since a repaired presentation can silently lose content. When diagnosing generated DrawingML, check that geometry coordinates are integers and that required XML structures remain valid.

Editable text may wrap differently because of font substitution, font metrics, text-box margins and PowerPoint's line-spacing rules. Check that the intended fonts are available in the viewing environment. Pay particular attention to cover titles and subtitles near artwork, long bullets near the right edge, nested list indentation and text above footers.

Inspect inline code and other highlighted text in editable exports, especially when a postprocessor merges text boxes or bullet lists. Backgrounds may become separate shapes that obscure text, and independently positioned runs may no longer align with the merged text. Check the exported shapes and their stacking order before attributing the problem to the source layout. If confirmed, correct the conversion or use transparent export styling; plain text is also suitable when code typography is not essential. Preserve command syntax and paths exactly, regenerate the file and review the affected slide in the target application. Do not flatten the slide merely to hide the defect when editability is required.

Review spacing as well as clipping: line height, paragraph space before and after, bullet gaps, indentation, text-box insets and clearance above footers. A slide can fit and still have uneven or excessive gaps. Compare short and wrapped bullets in the target application with the browser reference. When a converter derives paragraph spacing from source positions, check that it does not count wrapped-line height again as extra paragraph space. Correct the conversion or local layout rather than inserting blank lines or shrinking the entire deck. Regenerate and review both the affected slides and the final deck after any spacing change.

Check for text crossing decorative elements, overlapping another text box, clipping at slide boundaries or acquiring an extra line that crowds the content below. Compare slide images from the exported file with the browser reference. Confirm that representative headings and body lists remain editable, since individual slides may fall back to images.

Correct the source layout or export conversion, regenerate the file and inspect the affected slides again. Record which slides and output formats were checked, then review the complete final artifact before reporting it as validated. If no compatible renderer is available, state that visual validation of the exported file remains unverified; do not report the layout as checked based only on the browser preview.

### PNG Export

```bash
slidev export --format png
slidev export --format png --range 1-5
```

### Markdown Export

```bash
slidev export --format md
```

## Export Options

### With Click Steps

Export each click as separate page:
```bash
slidev export --with-clicks
```

### Dark Mode

```bash
slidev export --dark
```

### Slide Range

```bash
slidev export --range 1,4-7,10
```

### Table of Contents

PDF with clickable outline:
```bash
slidev export --with-toc
```

### Timeout

For slow-rendering slides:
```bash
slidev export --timeout 60000
```

### Wait

Wait before capture:
```bash
slidev export --wait 2000
```

### Wait Until

Wait condition:
```bash
slidev export --wait-until networkidle   # Default
slidev export --wait-until domcontentloaded
slidev export --wait-until load
slidev export --wait-until none
```

### Transparent Background

```bash
slidev export --omit-background
```

### Custom Browser

```bash
slidev export --executable-path /path/to/chrome
```

## Headmatter Options

```yaml
---
exportFilename: my-presentation
download: true              # Add download button in build
export:
  format: pdf
  timeout: 30000
  withClicks: false
---
```

## Troubleshooting

### Missing Content

Increase wait time:
```bash
slidev export --wait 3000 --timeout 60000
```

### Wrong Global Layer State

Use `--per-slide` or use `slide-top.vue` instead of `global-top.vue`.

### Broken Emojis

Use system fonts or install emoji font on server.

### CI/CD Export

Install playwright browsers:
```bash
npx playwright install chromium
```

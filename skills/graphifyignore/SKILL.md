---
name: graphifyignore
description: >-
  Author, review, or debug a .graphifyignore file and scope a graphify corpus.
  Use whenever the task involves writing or fixing .graphifyignore patterns,
  graphify detecting or extracting the wrong set of files (too many, zero, or
  missing sources), excluding tests/fixtures/translations/generated content
  from a knowledge graph, choosing between LLM-free and LLM extraction phases,
  or day-to-day graphify maintenance (update, watch, cluster, label, query).
---

# .graphifyignore and graphify quality of life

How graphify's ignore file actually works (verified against the graphify
source, `graphify/detect.py`), plus the practical commands that keep a graph
useful. Read this before writing patterns — the merge order and the
directory-only rule are not obvious, and several intuitions from `.gitignore`
carry over only partially.

## How .graphifyignore is loaded

- Full **gitignore syntax**: `#` comments, `!` negation, `dir/` directory-only
  patterns, anchored patterns (any `/` in the pattern anchors it to the ignore
  file's directory), `*` within a path segment, `**` across segments, and
  **last-match-wins** — the final matching pattern decides.
- **Merge order per directory** (lowest to highest priority):
  `$GIT_DIR/info/exclude` → `.gitignore` → `.graphifyignore`. The
  `.graphifyignore` is read last, so its patterns — including `!` negations —
  win conflicts.
- **Ancestor chain**: ignore files are loaded from the nearest VCS root down
  to the scan root, and *nested* ignore files below the scan root are honored
  too (picked up live as the walk reaches them). Inner files beat outer ones.
- **Git-tracked files are preserved from `.gitignore`-only pruning** — but
  `.graphifyignore` (and `--exclude`) rules stay authoritative even for
  tracked files. This is the tool's escape hatch for "tracked in git, but keep
  it out of the graph."
- **Directory-only patterns (`foo/`) check `is_dir()` on disk.** A `dir/`
  pattern does not match a path that doesn't exist. When testing patterns
  programmatically, test against real files — hypothetical paths give false
  negatives for every `dir/` rule.
- Filenames are normalized to **Unicode NFC** on both sides, so patterns are
  safe against macOS NFD filename drift.
- UTF-16 (BOM'd) ignore files are decoded, with a warning — re-save as UTF-8.

**Built-in noise pruning happens independently of ignore files**: dependency
and cache dirs, virtualenvs (only with real venv markers like `pyvenv.cfg`),
`coverage/` (only with coverage artifacts present), `snapshots/` dirs (only
when they contain `.snap` files or sit under a JS test root), and `worktrees/`
nested inside dotted dirs. Check whether a rule is even needed before adding
one.

**Consequence: never duplicate `.gitignore` content** (`node_modules/`,
`lib/`, `dist/`, `coverage/`, `*.tsbuildinfo`, …) in `.graphifyignore`. It is
already merged. The file exists for *graph-scoping* decisions only: what
belongs in the knowledge graph, not what belongs in the build.

## What belongs in a project's .graphifyignore

Typical graph-scoping exclusions for a code repository:

```gitignore
# Translation duplicates — one canonical language is enough
**/*.zh.md
**/*.i18n.yaml

# Generated/projected content (doc sites rendered from sources, build reports)
website/

# Recorded test outputs & replay fixtures (large, derived, low signal)
**/tests/snapshots/
**/tests/fixtures/
**/__snapshots__/
*.snap
*.jsonl

# Vendored upstream test suites (upstream behavior, not this project's)
vendor/**/test/
vendor/**/tests/
```

Common gotchas:

- **A bare `**` ignores everything.** A placeholder `.graphifyignore`
  containing `**` makes detection report `found 0 code, 0 docs` — the
  extraction then "succeeds" with a 0-node graph. If detection finds nothing,
  read the ignore files first.
- **Enumerate every test extension.** `**/*.spec.ts` does not match
  `*.spec.tsx`; snapshot/replay harnesses often use distinct suffixes
  (`*.e2e.ts`, `*.snapshot.ts`). Missing one leaks hundreds of test files into
  the graph as top-level nodes.
- Excluding `**/*.md` also excludes READMEs and docs — fine for a code-only
  phase (below), but say so in a comment so the next person knows it's
  deliberate and temporary.

## LLM-free vs LLM phases

Graphify has a hard split between extraction modes — control it explicitly:

- **Code is extracted structurally (AST). No LLM, no API key, ever.**
- **Docs, papers, and images go through semantic (LLM) extraction** — Gemini
  if `GEMINI_API_KEY`/`GOOGLE_API_KEY` is set, otherwise the host agent.
- **Community labeling** is a separate LLM step whose backend is auto-detected
  from *any* provider key in the environment — including `DEEPSEEK_API_KEY`
  and `MOONSHOT_API_KEY`. A stray key in a repo `.env` can turn a "structural"
  run into billed LLM calls.

The robust pattern for a large repo is two phases:

1. **Phase 1 — code-only, zero LLM.** Add a clearly marked temporary block to
   `.graphifyignore` (`**/*.md` plus every test-file extension), then:

   ```sh
   graphify extract . --code-only --no-cluster   # fresh structural build
   graphify cluster-only . --no-label            # communities, placeholder names
   ```

2. **Phase 2 — fold in prose.** Delete the temporary block, run the
   `/graphify` skill (or `graphify extract` with a chosen `--backend`) so
   docs/notes are semantically extracted, then name communities with
   `graphify label . --missing-only`.

## Everyday commands

| Task | Command |
|---|---|
| Fresh headless build (CI/scripts) | `graphify extract . [--code-only] [--no-cluster]` |
| Incremental rebuild after code edits | `graphify update .` (LLM-free) |
| Rebuild after deletions/refactors | `graphify update . --force` |
| Continuous rebuilds | `graphify watch .` |
| Re-cluster without LLM naming | `graphify cluster-only . --no-label` |
| Name communities later | `graphify label . --missing-only` |
| Ask the graph | `graphify query "…" [--budget N]`, `explain "X"`, `path "A" "B"`, `affected "X"`, `god-nodes` |

Quality-of-life details worth knowing:

- **Ignore-rule changes are self-healing**: `graphify update .` reports
  `pruned N node(s) from M newly-ignored file(s)` and evicts them — no manual
  cleanup or full rebuild needed.
- **Staleness is recorded**: `graphify-out/GRAPH_REPORT.md` names the commit
  the graph was built from; compare with `git rev-parse HEAD`.
- **Big graphs get an aggregated view**: above 5,000 nodes, `graph.html` is a
  community-level map (use `--obsidian` for node-level detail, `--no-viz` in
  CI).
- **`graphify update` cannot create the first graph** — it is incremental and
  fails with "No code files found" on a repo with no `graphify-out/`. The
  first build is `graphify extract`.
- Some extractors have grammar gaps (e.g. the TypeScript parser rejects
  `export type * from`, so pure re-export barrels extract no symbols). The
  extract step prints `N file(s) had syntax errors` with names — check whether
  the listed files carry real logic before worrying.

## Debugging pattern behavior

When detection surprises you, evaluate patterns directly against graphify's
own matcher instead of trial-and-error rebuilds — and use **real, existing
paths** (directory-only rules need `is_dir()` to be true):

```sh
python3 - <<'EOF'
import sys; sys.path.insert(0, '/path/to/graphify')   # repo containing graphify/detect.py
from pathlib import Path
from graphify.detect import _load_graphifyignore, _is_ignored
root = Path('/path/to/project').resolve()
pats = _load_graphifyignore(root)
for rel in ['src/index.ts', 'src/x.spec.ts', 'docs/guide.md']:
    print(rel, '->', 'IGNORED' if _is_ignored(root / rel, root, pats) else 'kept')
EOF
```

To find the installed package source for the import path:
`graphify --version`, then resolve the editable-install mapping in the uv
tool's `site-packages` (or the site-packages of whatever environment provides
the `graphify` command).

# Authoring efficient Repertoire skills

Guidance for writing a `SKILL.md` package that agents load reliably and cheaply.
Repertoire distributes these packages; the quality of the package decides how
well an agent triggers and follows it. Read this when creating or revising a
skill in a catalog.

## Contents

- [Why structure matters](#why-structure-matters)
- [Anatomy of a skill](#anatomy-of-a-skill)
- [Catalog manifests versus loose repositories](#catalog-manifests-versus-loose-repositories)
- [Progressive disclosure](#progressive-disclosure)
- [Write the frontmatter](#write-the-frontmatter)
- [Write the description that triggers](#write-the-description-that-triggers)
- [Write the body](#write-the-body)
- [Organize references, assets, and scripts](#organize-references-assets-and-scripts)
- [Organize multi-domain skills by variant](#organize-multi-domain-skills-by-variant)
- [Test the skill before publishing](#test-the-skill-before-publishing)
- [Authoring checklist](#authoring-checklist)

## Why structure matters

An agent sees every installed skill's name and description on every turn, but
only pulls the body into context when it decides the skill is relevant, and only
reads a reference file when the body tells it to. A well-structured skill keeps
the always-loaded surface tiny, makes the trigger decision obvious, and defers
detail until it is needed. A poorly-structured skill either fails to trigger or
floods context with detail the agent did not need.

## Anatomy of a skill

A skill is a directory whose name matches the catalog key and the frontmatter
`name`:

```text
skill-name/
├── SKILL.md          # required: frontmatter + body
├── references/       # optional: deep docs read on demand
├── assets/           # optional: files used in output (templates, configs, icons)
├── scripts/          # optional: executable helpers for deterministic work
└── stubs.yaml        # optional: Repertoire file-backed stubs (see references/stubs.md)
```

Only `SKILL.md` is required. Keep every supporting file inside the skill
directory so Repertoire installs the complete package. Paths must stay contained
within the directory.

## Catalog manifests versus loose repositories

A Git repository of `SKILL.md` directories can be installed as a loose catalog
without a `repertoire.yaml`. That is enough to discover, lock, and copy skills.

Variants, always-on instructions, optional hooks, and `stubs.yaml` require a
real catalog `repertoire.yaml`. `repertoire catalog init` writes that starting
point (catalog name, skill keys, placeholder `SKILL.md` files) into the current
directory without running Git. Fill in the placeholders, then test with a
local override before you push.

## Progressive disclosure

Skills load in three levels. Design each level for its cost:

1. **Metadata** (`name` + `description`) — always in context, for every skill
   the agent has installed. Budget ~100 words. This is the only thing that
   decides whether the skill triggers.
2. **`SKILL.md` body** — loaded when the skill triggers. Keep it under ~500
   lines. This holds the core workflow and pointers to references.
3. **Bundled resources** (`references/`, `assets/`, `scripts/`) — loaded or
   executed only when the body directs the agent to them. Effectively unlimited;
   scripts can run without their source entering context.

The rule of thumb: move anything the agent does not need on the common path down
a level. If the body is approaching 500 lines, add a layer of hierarchy — split
detail into `references/` and leave a one-line pointer that says when to read it.

## Write the frontmatter

Required keys are `name` and `description`. Keep the block minimal:

```yaml
---
name: skill-name
description: >-
  One or two sentences: what the skill does AND the concrete situations that
  should trigger it.
---
```

- `name` must exactly match the catalog key and the skill directory name.
- Use a folded scalar (`>-`) for multi-line descriptions so it stays one logical
  string.
- Add other keys only when the catalog schema needs them; do not invent
  frontmatter fields.

## Write the description that triggers

The description is the primary triggering mechanism. Agents tend to
*under-trigger* — they skip a useful skill because the description felt narrow or
optional. Counter that:

- State **what it does** and **when to use it** together. All "when to use"
  information belongs here, not buried in the body.
- Be a little pushy about triggering. Enumerate concrete nouns, commands, file
  names, and situations so a keyword or intent match is likely.
- Cover casual phrasings, not just the formal name. Users rarely name the skill.

**Weak:** `Manage skills with the CLI.`

**Strong:** `Install, declare, update, and troubleshoot portable AI agent
skills with the Repertoire CLI. Use when working with repertoire commands,
repertoire.yaml, catalogs, bootstrap or sync workflows, or catalog ambiguity.`

Note how the strong version names the tool, the config files, the verbs, and the
situations — each is a trigger hook.

## Write the body

Write for an agent that has already decided to use the skill and now needs to
act.

- **Use the imperative.** "Run `repertoire list` before mutating state," not
  "You could list first."
- **Explain why, briefly.** A short rationale ("so newly published skills appear
  without a separate update") helps the agent generalize instead of pattern-
  matching one example. Prefer this over stacks of bare `MUST`s.
- **Lead with the common path.** Put the establish-context / core-workflow steps
  first; push edge cases and recovery toward the end.
- **Show exact commands and outputs.** Fenced command blocks and explicit output
  templates remove ambiguity.
- **Prefer general phrasing over narrow examples.** One concrete example plus the
  underlying rule beats five examples that the agent might overfit to.
- **Point to references explicitly.** When you defer detail, say what the file
  covers and when to open it: "For the full resolution order, see
  `references/resolution.md`."

For a fixed output shape, pin it with a template the agent must reuse:

```markdown
## Report structure
Use this exact template:
# [Title]
## Summary
## Findings
```

## Organize references, assets, and scripts

- **`references/`** holds documentation the agent reads on demand: edge-case
  tables, full option lists, troubleshooting trees, schema definitions. Each
  file should stand alone — an agent may open it without the surrounding body.
  For any reference over ~300 lines, add a table of contents at the top so the
  agent can jump to the relevant part instead of loading the whole file.
- **`assets/`** holds files that appear in the *output*: templates, starter
  configs, icons, fonts. In Repertoire, prefer exposing single starter files
  through `stubs.yaml` (see `references/stubs.md`) so
  agents fetch a verified path instead of pasting content.
- **`scripts/`** holds executable helpers for deterministic, repetitive, or
  error-prone work. A script runs without its source entering context, which
  saves tokens and avoids the agent re-deriving fragile logic. Keep scripts
  self-contained and document their invocation in the body.

## Organize multi-domain skills by variant

When one skill spans several frameworks, platforms, or targets, split the
detail so the agent reads only the branch it needs:

```text
deploy/
├── SKILL.md        # shared workflow + how to pick a variant
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

The body carries the selection logic and the common steps; each reference
carries one variant. This keeps the triggered context small regardless of how
many variants exist.

## Test the skill before publishing

Validate structure and triggering from a disposable checkout before pushing.

A read-only resolution check is fully side-effect-free — an override only
redirects where a catalog is read from, so `list --available` installs nothing
and needs no cleanup:

```bash
REPERTOIRE_OVERRIDES="phillarmonic=/path/to/ai-skills" repertoire list --available
```

An actual install test **does** mutate state: `add` and `install` write a
managed copy into the selected scope and (for `add`) record a requirement in the
manifest, even under an override. At the default global scope that touches your
home-directory agent roots. To keep the test contained and trivially reversible,
install into a throwaway `--project` worktree instead of global scope:

```bash
scratch=$(mktemp -d); git -C "$scratch" init; cd "$scratch"
REPERTOIRE_OVERRIDES="phillarmonic=/path/to/ai-skills" \
  repertoire --project install skill-name --target codex
```

Then sanity-check triggering and instructions: write 2–3 realistic prompts a
real user would type — casual phrasing, concrete file names and context, not the
skill's own name — and confirm the agent both reaches for the skill and can
follow the body to a correct result. If it under-triggers, strengthen the
description; if it goes off-track mid-task, tighten the body or move detail into
a clearly-pointed reference.

**Clean up when done.** Remove the throwaway worktree with `rm -rf "$scratch"`;
the override needs no cleanup once the command exits (unset the env var if you
exported it). If you instead installed at global scope, undo it explicitly with
`repertoire remove skill-name` — a stray global copy and requirement otherwise
linger in every target you added.

## Authoring checklist

- [ ] Directory name, catalog key, and frontmatter `name` all match.
- [ ] Description states what it does AND when to trigger, with concrete hooks.
- [ ] Body is imperative, leads with the common path, under ~500 lines.
- [ ] Detail beyond the common path lives in `references/`, each with a pointer.
- [ ] References over ~300 lines start with a table of contents.
- [ ] Deterministic or fragile work is delegated to `scripts/`.
- [ ] Output files ship via `assets/` or `stubs.yaml`, not pasted inline.
- [ ] Every path stays inside the skill directory.
- [ ] Triggering and instructions verified from a local override checkout, and
      any throwaway worktree or global test install cleaned up afterward.

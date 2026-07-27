---
name: repertoire
description: >-
  Install, declare, update, remove, and troubleshoot portable AI agent skills
  with the Repertoire CLI. Use when working with repertoire commands,
  repertoire.yaml, repertoire.lock.json, .repertoire.yaml, skill catalogs,
  project or global skill installation, multi-agent targets, bootstrap or sync
  workflows, catalog ambiguity, or managed-skill safety errors.
---

# Repertoire

Use Repertoire to manage portable `SKILL.md` packages across Codex, Claude
Code, Cursor, Gemini CLI, Windsurf, Cline, Roo Code, Kiro, Junie, Kimi Code,
OpenCode, GitHub Copilot, OpenClaw, and shared `.agents` setups.

## Establish context

Confirm the executable and inspect existing state before making changes:

```bash
repertoire --version
repertoire list
repertoire catalog list
```

Commands default to user-global scope and install into home-directory skill
roots. Use `--project` when a skill should live in the current Git worktree.
`--global` makes the default explicit. Never combine `--project` and
`--global`.

Shell completion suggests catalogs Repertoire already knows (built-in, global
and project registrations, bootstrap catalogs, lock sources, and cached
remotes), including source URLs for `catalog add`. Prefer tab-completing those
known catalogs instead of inventing catalog names.

Use Repertoire commands instead of manually editing generated
`repertoire.lock.json` files or managed skill copies.

## Discover and select skills

List visible definitions:

```bash
repertoire list --available
repertoire list --available --catalog phillarmonic
```

`list --available` refreshes visible catalog caches before reading manifests, so
newly published tracking-branch skills should appear without a separate catalog
update.

The official `phillarmonic` catalog is built in and does not need registration.
An unqualified skill name works when exactly one visible catalog defines it:

```bash
repertoire add zensical
```

If multiple catalogs define the name, Repertoire lists every matching catalog
and source. Do not choose silently; repeat the command with the catalog the user
intends:

```bash
repertoire add zensical --catalog phillarmonic
```

## Install skills

Use `add` to install a skill and declare it as a requirement:

```bash
repertoire add zensical --target codex
repertoire add github.com/phillarmonic/ai-skills/zensical --target codex
repertoire add zensical --target codex --target claude
repertoire add zensical --target all
```

Prefer source-qualified IDs (`github.com/phillarmonic/ai-skills/zensical`) when
a short name could be ambiguous. For catalog skill keys that intentionally
qualify a generic name, use owner-prefixed kebab-case names such as
`phillarmonkey-code`.
This is especially useful for broad action skills such as `code`, `review`,
`docs`, or `test`, where the owner-prefixed key tells agents which behavior
definition they should apply.

Use `install <skill>` for a one-off tracked installation when the skill is not
declared. With no skill name, `install` installs or repairs every declared
requirement:

```bash
repertoire install zensical --target agents
repertoire install github.com/phillarmonic/ai-skills/zensical --target agents
repertoire install
repertoire install --target all
```

Without `--target`, Repertoire detects existing agent configuration or skill
directories. Use `--target all` only when copies should be created for every
supported target even if its directory does not yet exist. Pass `--project` to
install into the Git worktree instead of the home directory.

## Configure catalogs

Register non-built-in catalogs explicitly:

```bash
repertoire catalog add git@github.com:example/company-skills.git --name company --ref main
repertoire catalog add /path/to/skills --name local
repertoire catalog update
```

Private catalog authentication belongs in normal Git credential helpers or SSH
agents, never in manifests. An explicit registration named `phillarmonic`
overrides the built-in source and is useful for local catalog development.

Before removing a catalog, check whether installed or declared skills still
reference it:

```bash
repertoire list
repertoire catalog remove company
```

## Author a private catalog

A private catalog is an ordinary Git repository with a `repertoire.yaml` at
its root. Keep each skill in its own directory and list that contained relative
path in the catalog manifest:

```text
company-skills/
├── repertoire.yaml
└── skills/
    ├── code-reviewer/
    │   ├── SKILL.md
    │   └── references/
    └── release-helper/
        └── SKILL.md
```

```yaml
schema: 1
catalog:
  name: company
  description: Private skills maintained by Example Company
  skills:
    code-reviewer:
      path: skills/code-reviewer
    release-helper:
      path: skills/release-helper
```

Catalog names must contain 1–64 lowercase letters, digits, or single hyphens.
Skill keys use the same kebab-case rule. Use owner-prefixed names such as
`phillarmonkey-code` for generic action-oriented skills such as `code`,
`review`, `docs`, or `test`; bare names are harder for agents to distinguish
when several personal or vendor catalogs are enabled.
Every path must be relative, remain inside the repository, and point to a
directory containing `SKILL.md`. Each `SKILL.md` needs YAML frontmatter whose
`name` exactly matches its catalog key and whose `description` is non-empty. The
skill directory name must also match the skill key. Keep supporting scripts,
references, and assets inside that skill directory so Repertoire installs the
complete package.

## Author and use file stubs

A skill may expose small file-backed stubs through an optional `stubs.yaml` in
its root:

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
description and instructions. Install the containing skill, then ask Repertoire
for a verified local asset path:

```bash
repertoire stub list
repertoire stub list common-stubs
repertoire stub get common-stubs/editorconfig
```

Repertoire prints the path and instructions for the agent. It does not copy,
merge, execute, or print the asset itself.

Validate locally before publishing from a disposable Git worktree. Project
scope keeps both the test registration and installed copy out of the user's
global Repertoire state:

```bash
scratch=$(mktemp -d)
git -C "$scratch" init
cd "$scratch"
repertoire --project catalog add /absolute/path/to/company-skills --name company-dev
repertoire --project list --available --catalog company-dev
repertoire --project install code-reviewer --catalog company-dev --target agents
```

Commit and push the repository to a private Git remote, grant users read access,
then register the remote using SSH or credential-helper-backed HTTPS:

```bash
git ls-remote git@github.com:example/company-skills.git
repertoire catalog add git@github.com:example/company-skills.git --name company --ref main
repertoire list --available --catalog company
```

Do not embed tokens, passwords, or other credentials in the catalog URL or any
manifest. Repertoire delegates authentication to Git. Omit `--ref` to track the
remote default branch; use a branch for updateable releases or a tag/full commit
for an immutable catalog snapshot.

## Bootstrap a project

Use `.repertoire.yaml` in a project root for repeatable onboarding. Prefer
source-qualified skill IDs and `scope: global` so the manifest stays in the repo while
skills install under home-directory agent roots:

```yaml
schema: 1

catalogs:
  company:
    source: git@github.com:example/company-skills.git
    ref: main

skills:
  github.com/phillarmonic/ai-skills/zensical:
    scope: global
    targets: [codex]

  github.com/example/company-skills/phillarmonkey-code:
    scope: project
    targets: [agents]
```

Then run:

```bash
repertoire bootstrap
```

If `.repertoire.yaml` is missing, `bootstrap` creates a starter that lists every
built-in `phillarmonic` skill with source-qualified IDs and `scope: global`, then
installs them. `sync` does not create a missing file. Omitted `scope` defaults
to `global`. Use `scope: project` only when the skill should be installed inside
the Git worktree. `bootstrap` installs or repairs declared copies using current
catalog state. Run `repertoire sync` when catalogs should be refreshed before
synchronizing the bootstrap declarations. Removing a declaration does not
uninstall an existing skill; use `repertoire remove <skill>` explicitly.

## Update, repair, and remove

```bash
repertoire update zensical
repertoire update company
repertoire update
repertoire install
repertoire remove zensical
```

`update` refreshes tracking catalogs and reinstalls selected managed skills. If
the argument names a visible catalog rather than an installed skill, it refreshes
that catalog and stops.
`install` without a name repairs declared requirements from current catalog
state. `remove` only removes content that Repertoire can verify it manages.

## Handle safety failures

Repertoire tracks catalog source, commit, digest, targets, and installed
locations. If a target is unmanaged or locally modified, inspect it before
using `--force`. Never discard local changes merely to make a command pass.

When a global skill is already managed from another source or ref, confirm that
replacing the shared installation is intended before using `--force`.

For catalog clone failures, verify the same source with Git:

```bash
git ls-remote <source>
```

Repair Git authentication or connectivity, then retry the Repertoire command.

## Verify the result

After a mutating workflow, inspect state and confirm the intended target:

```bash
repertoire list
```

When working in a repository, review `repertoire.yaml`,
`repertoire.lock.json`, or `.repertoire.yaml` changes before reporting
completion. Report the selected scope, catalog, and targets.

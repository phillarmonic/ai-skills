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

Inside a Git worktree, Repertoire uses project state by default. Outside a
worktree, it uses user-global state. Pass `--project` or `--global` when the
intended scope is not obvious; never combine them.

Use Repertoire commands instead of manually editing generated
`repertoire.lock.json` files or managed skill copies.

## Discover and select skills

List visible definitions:

```bash
repertoire list --available
repertoire list --available --catalog phillarmonic
```

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
repertoire add zensical --target codex --target claude
repertoire add zensical --target all
```

Use `install <skill>` for a one-off tracked installation when the skill is not
declared. With no skill name, `install` installs or repairs every declared
requirement:

```bash
repertoire install zensical --target agents
repertoire install
repertoire install --target all
```

Without `--target`, Repertoire detects existing agent configuration or skill
directories. Use `--target all` only when copies should be created for every
supported target even if its directory does not yet exist.

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

## Bootstrap a project

Use `.repertoire.yaml` in a project root for repeatable onboarding:

```yaml
schema: 1

catalogs:
  company:
    source: git@github.com:example/company-skills.git
    ref: main

skills:
  zensical:
    scope: project
    targets: [codex]

  code-reviewer:
    catalog: company
    scope: global
    targets: [codex, claude]
```

Then run:

```bash
repertoire bootstrap
```

`bootstrap` installs or repairs declared copies using current catalog state.
Run `repertoire sync` when catalogs should be refreshed before synchronizing
the bootstrap declarations. Removing a declaration does not uninstall an
existing skill; use `repertoire remove <skill>` explicitly.

## Update, repair, and remove

```bash
repertoire update zensical
repertoire update
repertoire install
repertoire remove zensical
```

`update` refreshes tracking catalogs and reinstalls selected managed skills.
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

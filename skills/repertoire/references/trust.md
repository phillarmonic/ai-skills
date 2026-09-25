# Catalog trust

Read this when a catalog registration has a `trust` block, when install fails
with a commit or digest signature error, or when you are publishing signatures
for a catalog others will trust.

Trust is opt-in. A registration with no `trust` block installs, updates, and
bootstraps as before. Repertoire does not call `gpg`, and the lock omits
fingerprint fields. The manifest and lock schema stay at 1.

## Declare keys

Put armored public keys next to the manifest that registers the catalog
(project `repertoire.yaml`, or the user-global manifest). List each file and
the fingerprint it must match:

```yaml
catalogs:
  company:
    source: git@github.com:example/company-skills.git
    ref: main
    trust:
      keys:
        - path: keys/company.asc
          fingerprint: ABCDEF0123456789ABCDEF0123456789ABCDEF01
```

`path` is relative to the manifest directory and must stay inside it. An empty
path, an empty fingerprint, an absolute path, or a path that leaves that
directory is rejected. An empty `keys` list is rejected. Fingerprints are
compared as hex; spaces and letter case do not matter. When a subkey signs,
declare the primary key fingerprint.

Export only the public key (`gpg --armor --export <fingerprint>`). Do not
commit private keys, and do not put key material in the catalog URL. Repertoire
does not fetch keys from a keyserver.

## What gets checked

Materializing the catalog runs `git verify-commit` on the checked-out commit.
Before a skill is written into the lock, each skill directory must contain
`REPERTOIRE.digest.asc`: a detached ASCII signature over that directory's
content digest. Variant directories are checked the same way, on their own
digest. A loose catalog uses the same file in each discovered skill directory
and has no variants.

Either declared key may sign the commit, and either declared key may sign a
digest. The two checks can succeed with different keys from the same list.

Verification imports only those public keys into a temporary keyring, after
each file's fingerprint matches the registration. The user's GnuPG home is
left untouched. The temporary keyring is deleted when the check finishes.

`add`, `install`, `update`, and `bootstrap` share both checks. A failure names
the catalog, the skill, and the commit, and says whether the commit signature
or the digest signature failed. The lock is left unchanged.

These do not skip the check:

- a local catalog path
- `--override` or `REPERTOIRE_OVERRIDES`
- `--force` (that flag is for locally modified or unmanaged copies)
- `--dry-run` when the catalog is already cached, or when the source is local
  or overridden (the check runs, and a failure is a real failure)

`--dry-run` against an uncached remote catalog stops at "would clone" and does
not prove the signatures are valid.

`gpg` must be on `PATH` when a trust block is present. A missing `gpg` fails
the command. Do not remove the trust block, point the catalog at an unsigned
checkout, or edit `repertoire.lock.json` to make the command pass. Ask the user
before changing who is allowed to sign.

## What the lock records

A successful trusted install records `commit_fingerprint` and
`digest_fingerprint` on the skill entry and on a project-artifact entry for
that skill. `repertoire show` does not print those fields. Read them from the
lock when you need them. Older locks that omit them still load. A catalog with
no trust block omits them.

`digest_fingerprint` is the key that signed the primary skill directory. Each
variant directory is still verified.

## Publish signatures

Sign the catalog commit with a key consumers will list (`git commit -S`).

`REPERTOIRE.digest.asc` sits in the skill directory beside `SKILL.md`, and in
each variant directory. It is not part of the content digest, so adding the
file does not change the digest you sign.

Repertoire has no sign command. The signed bytes are the exact lowercase hex
digest, with no trailing newline:

```bash
printf '%s' "$DIGEST" | gpg --detach-sign --armor --output skills/code-reviewer/REPERTOIRE.digest.asc
```

Get `$DIGEST` from an install that used a registration without a trust block:

- `repertoire show <skill>` prints the primary skill digest.
- For a target that installs a variant, that target's `target_digests` value
  in `repertoire.lock.json` is the variant directory's digest. Sign that hex
  into the variant directory. Do not edit the lock.

Then commit the signature files with a signed commit, and only then add the
trust block on the consumer registration.

A changed skill, a missing or non-regular `REPERTOIRE.digest.asc`, a digest
signed by any other key, or an unsigned commit fails closed.

## Failure phrases

- **commit signature failed.** The checked-out commit is unsigned, or no
  declared key signed it.
- **digest signature failed.** The skill, or a variant named `skill (target)`,
  has no valid `REPERTOIRE.digest.asc` over its content digest.
- **gpg is not available.** A trust block is present and `gpg` is not on
  `PATH`.
- **fingerprint does not match.** The armored file is not the key named in the
  registration.

The error also names the catalog, the skill, and the commit.

# How `gsh doctor` works

```
gsh doctor
```

`gsh doctor` reads the repository and reports five classes of problem. **It
never modifies anything** — not `.gitignore`, not the index, not your files. It
prints the command that would fix each finding and leaves running it to you.

It exits `1` when it finds a problem, which is what makes it usable as a gate.

## The check that is the point of the command

> **A `.gitignore` does nothing for files git already tracks.**

This is the single most common `.gitignore` misunderstanding, and it is the
reason doctor exists. `.gitignore` decides which *untracked* files git should
stop showing you. Once a file is in the index, git tracks it forever, and adding
a matching rule changes nothing at all: `node_modules/` keeps arriving in every
clone, `.env` keeps being pushed, and `git status` stays silent about it because
the file is tracked and unchanged.

Nothing warns you. `gsh auto` will happily write a perfect `.gitignore` on top of
a repository where all of it is already too late. The first doctor check is what
tells you:

```
Tracked files that should be ignored
✗ 3 tracked files matching .gitignore but still in the index
  • .env
  • node_modules/leftpad/index.js
  • node_modules/leftpad/package.json

  .gitignore does nothing for files git already tracks.
  Remove them from the index (they stay on disk):
    git rm -r --cached <path>
```

`git rm --cached` removes the file from the index and leaves it on disk. After
that commit, the `.gitignore` rule finally applies. Everyone else's next pull
deletes their copy — say so before you push it.

The list is capped at ten entries, with `... and N more` after it.

Underneath this is `git ls-files -i -c --exclude-standard`: files git is
tracking that the current exclude rules say should be ignored. It is exact —
there is no pattern-matching of gsh's own involved — and it is the first thing
to run on a repository somebody else started.

## The other four checks

### 1. gitignore

Reports whether a `.gitignore` exists, how many actual rules it holds (non-blank,
non-comment lines), and which templates gsh can see installed in it:

```
gitignore
✓ .gitignore found (165 rules)
  • templates: macos node python visualstudiocode
```

The template list comes from the `### gsh:start … ###` markers, so it only shows
templates gsh installed — see [the block format](block-format.md). A missing
`.gitignore` is a **warning**, not a problem:

```
! no .gitignore in this repository
  fix: gsh auto
```

If the markers in the file are malformed — a start with no end, two starts in a
row — doctor says so and ignores them:

```
! .gitignore has malformed gsh block markers, ignoring them
```

### 3. Secrets

Every file git can see (tracked, plus untracked-and-not-ignored) is checked by
**basename** against a fixed pattern list:

```
.env  .env.*  *.pem  *.key  *.p12  *.pfx
id_rsa  id_dsa  id_ecdsa  id_ed25519
credentials.json  service-account*.json  .npmrc  .pypirc
```

Findings are split into two, because the fix is different for each.

**Committed** — already in git history. Ignoring it now changes nothing; the
credential is out:

```
✗ 2 sensitive files are COMMITTED to this repo
  • .env
  • deploy.pem

  these are already in git history - rotate the credentials, then:
    git rm --cached <path>
```

Note the order: *rotate first*. `git rm --cached` takes the file out of future
commits; it does not remove it from history, and anyone who has ever cloned the
repository still has it.

**Not ignored** — on disk, not committed yet, and nothing is stopping the next
`git add .`:

```
✗ 2 sensitive files are not ignored
  • .env
  • id_rsa

  fix: gsh add dotenv   →   or add the pattern by hand
```

Both count as problems and both exit `1`.

A file that matches a pattern but is already ignored is not reported — git does
not list it, so doctor does not see it. That is the intended behaviour, and it
is also how you silence a false positive.

**Known false positive:** the `.env.*` pattern matches `.env.example`,
`.env.sample` and `.env.template`, which are meant to be committed. doctor will
flag them:

```
$ ls -a
.env.example  server.key

$ gsh doctor
...
✗ 2 sensitive files are not ignored
  • .env.example
  • server.key
```

There is no allowlist. Until there is one, either ignore the file (doctor then
stops seeing it) or accept the noise.

### 4. Large files

Files git can see that are over **5 MB** and not ignored. A warning, not a
problem — big files are sometimes exactly what you meant:

```
! 1 file over 5MB not ignored
  • 7MB  assets/demo.mp4
```

Sizes are floored to whole megabytes, and at most five are listed. This check
reads every visible file to size it, which is where most of doctor's runtime
goes.

### 5. Duplicates

Identical non-comment, non-blank lines in `.gitignore`. A warning:

```
! 2 duplicated rules
  • *.log
  • target/
```

Duplicates are harmless to git but are usually a symptom — two templates
covering the same ground, or rules pasted in by hand that gsh later added again
inside a block. At most five are listed. This check is skipped when there is no
`.gitignore`.

## The summary line and the exit code

| Result | Line | Exit |
|---|---|---|
| Nothing found | `✓ all good` | `0` |
| Warnings only | `! 1 warning, nothing critical` | `0` |
| Any problem | `✗ 2 problems, 1 warning` | `1` |

Problems: tracked-but-ignored files, committed secrets, unignored secrets.
Warnings: no `.gitignore`, files over 5 MB, duplicated rules.

Only warnings still exits `0` on purpose — a 6 MB test fixture should not fail
your build.

## In CI

Because it exits `1` on problems and changes nothing, doctor drops into a
pipeline as-is. `NO_COLOR` and `GSH_ASCII` are honoured, and gsh suppresses
colour and spinners on non-terminals anyway.

GitHub Actions:

```yaml
- name: gitignore audit
  run: |
    git clone --depth 1 https://github.com/MatteoKrsticDev/gsh.git /tmp/gsh
    bash /tmp/gsh/gsh/main doctor
```

There is no package to install yet, so CI runs the script out of a clone
directly — `bash <clone>/gsh/main` needs no `setup` step.

As a pre-commit hook, in `.git/hooks/pre-commit`:

```bash
#!/bin/sh
exec gsh doctor
```

Be deliberate about this one. doctor reports the *state of the repository*, not
the state of your commit, so a pre-existing committed secret blocks every commit
until it is dealt with. That is arguably the right behaviour; it is also
annoying on a repository you have just inherited. Running it in CI, or on a
pre-push hook, is the gentler placement.

## Performance

Measured on a 1,001-file repository (Apple silicon):

```
$ time gsh doctor
1.27s user 1.92s system 127% cpu 2.501 total
```

Roughly 2–3 seconds. The cost is dominated by the large-file check, which sizes
every visible file, and by the per-file `git ls-files --error-unmatch` used to
decide tracked vs. untracked for each secret candidate. It scales with the
number of files git can see, so a repository whose bulk is already ignored is
much cheaper than the file count suggests.

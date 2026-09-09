# Command reference

```
gsh <command> [args] [options]
```

Commands, and whether they need to be run inside a git repository:

| Command | Aliases | Needs a git repo | Writes `.gitignore` |
|---|---|---|---|
| [`gsh auto`](#gsh-auto) | — | yes | yes |
| [`gsh add`](#gsh-add) | `a` | yes | yes |
| [`gsh init`](#gsh-init) | `i` | yes | creates it empty |
| [`gsh doctor`](#gsh-doctor) | `doc` | yes | never |
| [`gsh list`](#gsh-list) | `l` | no | no |
| [`gsh show`](#gsh-show) | — | no | no |
| [`gsh version`](#gsh-version) | `v` | no | no |
| [`gsh help`](#gsh-help) | `h` | no | no |

## Exit codes

gsh uses two:

| Code | Meaning |
|---|---|
| `0` | Success. For `doctor`: no problems (warnings alone still exit `0`). |
| `1` | Failure, aborted by the user, or `doctor` found a problem. |

`gsh-update` is a separate script with a third code; see
[Install and update](install.md).

Answering `n` at a confirmation prompt exits `1`. That is deliberate: a CI job
that forgot `-y` must not look like it did the work.

## Global options

These are recognised anywhere on the command line — before the command, after
it, or in between.

| Flag | Effect |
|---|---|
| `-y`, `--yes` | Assume yes; never prompt. |
| `--refresh` | Ignore the cache and refetch the catalog and any template it needs. |
| `--no-color` | Disable colour. `NO_COLOR` in the environment does the same. |
| `--ascii` | ASCII icons instead of Unicode. `GSH_ASCII` does the same. |
| `-h`, `--help` | Help for the command it accompanies. Short-circuits: nothing is written. |
| `-v`, `--version` | Print the version and exit `0`. Short-circuits. |
| `--` | Everything after it is a positional argument, never a flag. |

`-h` and `-v` are parsed as flags, not as positionals, so `gsh add --help`
prints help for `add` rather than complaining about a template called `help`:

```
$ gsh add --help
Usage: gsh add <template> [template...] [-y]

  add templates to .gitignore

  gsh help   for all commands, options and environment variables
```

Colour and Unicode are decided per stream. `use_color` gates stdout, a separate
check gates stderr, so a terminal on one says nothing about the other:
`gsh doctor > report.txt` run from a terminal leaves the file free of escape
codes while the prompts on stderr stay coloured.

---

## `gsh auto`

```
gsh auto [-y]
```

Scans the whole repository, works out which stacks it contains, and appends the
matching templates to `.gitignore`. The rules and the reasoning behind them are
in [How `gsh auto` decides](auto.md).

```
$ gsh auto

Detected
  ✓ node               web/package.json +1 more
  ✓ python             api/pyproject.toml
  ✓ visualstudiocode   .vscode/settings.json
  ✓ macos              you are on macOS

? Add node,python,visualstudiocode,macos to .gitignore? [y/N] y
✓ created .gitignore
✓ added node,python,visualstudiocode,macos (165 rules)
```

The second column is *why* — the shallowest path that triggered the rule.
`+1 more` means other files matched the same template; only the winner is shown.

Templates already installed are listed but not re-added:

```
$ gsh auto -y

Detected
  • node               web/package.json +1 more - already present
  • python             api/pyproject.toml - already present
  • visualstudiocode   .vscode/settings.json - already present
  • macos              you are on macOS - already present

› everything already in .gitignore, nothing to do
```

Exit codes:

| Situation | Code |
|---|---|
| Templates added, or everything already present | `0` |
| You answered `n` at the prompt | `1` |
| Nothing detected | `1` |
| Not a git repository | `1` |

Declining:

```
$ gsh auto
...
? Add node,python,visualstudiocode,macos to .gitignore? [y/N] n
› aborted
```

`gsh auto -y` on a repository with no `.gitignore` creates one without a second
prompt — creating the file is part of the write you already agreed to. This is
what makes `echo y | gsh auto` work.

## `gsh add`

```
gsh add <template> [template...] [-y]
```

Adds named templates. All of them go out as one request to gitignore.io and
land in one block:

```
$ gsh add rust go
✓ added rust,go (13 rules)
```

The count is non-blank, non-comment lines — actual ignore rules, not lines of
file.

Templates already installed are skipped, and gsh says so:

```
$ gsh add rust
  • rust already present, skipped
› nothing to do
```

That exits `0`; nothing was wrong. An unknown name exits `1` and nothing is
written:

```
$ gsh add nope
✗ unknown template(s): nope
  try: gsh list nope
```

Every name is validated against the catalog *before* anything is written, so
`gsh add node bogus` writes nothing at all rather than half the request.

With no `.gitignore` present, `gsh add` asks before creating one (`-y` skips the
question).

Exit codes: `0` on success or nothing-to-do; `1` for an unknown template, no
arguments, not a git repository, an unwritable `.gitignore`, or a failed
download with no cache and no local fallback.

## `gsh init`

```
gsh init
```

Creates an empty `.gitignore`. Fails if one already exists — it will not
truncate your file.

```
$ gsh init
✓ created .gitignore
  next: gsh auto

$ gsh init
✗ you already have a .gitignore file
```

Exit codes: `0` created; `1` if the file exists, is a symlink, is not a regular
file, cannot be created, or you are not in a git repository.

Most of the time you want `gsh auto`, which creates the file itself.

## `gsh doctor`

```
gsh doctor
```

Five read-only checks. It changes nothing and prints the command that would.
Each check, and why it is there, is in [How `gsh doctor` works](doctor.md).

```
$ gsh doctor

gitignore
✓ .gitignore found (165 rules)
  • templates: macos node python visualstudiocode

Tracked files that should be ignored
✓ none

Secrets
✓ no exposed secrets found

Large files
✓ none over 5MB

Duplicates
✓ no duplicated rules

✓ all good
```

Exit codes:

| Result | Summary line | Code |
|---|---|---|
| No findings | `✓ all good` | `0` |
| Warnings only | `! 1 warning, nothing critical` | `0` |
| Any problem | `✗ 2 problems, 1 warning` | `1` |

Problems are: tracked-but-ignored files, committed secrets, unignored secrets.
Warnings are: no `.gitignore`, files over 5 MB, duplicated rules.

## `gsh list`

```
gsh list [filter]
```

Lists the catalog. The filter is a **substring, case-insensitive, not a regex** —
`gsh list [` searches for a literal bracket rather than dying inside grep.

```
$ gsh list rust
› 2 templates matching rust
rust
rust-analyzer
```

```
$ gsh list | head -5
› 571 templates available (gsh list <filter> to search)
1c
1c-bitrix
a-frame
actionscript
```

The count line goes to stderr and the names to stdout, so `gsh list | grep`
sees only names. On a terminal the names are laid out in columns via `column(1)`
if it is available; piped, they are one per line.

Exit codes: `0`; `1` if the filter matches nothing.

```
$ gsh list zzz
✗ no template matches 'zzz'
```

## `gsh show`

```
gsh show <template> [template...]
```

Prints template content on stdout without touching `.gitignore`. Works outside a
git repository.

```
$ gsh show rust
# Created by https://www.toptal.com/developers/gitignore/api/rust
# Edit at https://www.toptal.com/developers/gitignore?templates=rust

### Rust ###
# Generated by Cargo
# will have compiled files and executables
debug/
target/

# Remove Cargo.lock from gitignore if creating an executable, leave it for libraries
# More information here https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html
Cargo.lock

# These are backup files generated by rustfmt
**/*.rs.bk

# MSVC Windows builds of rustc generate these, which store debugging information
*.pdb

# End of https://www.toptal.com/developers/gitignore/api/rust
```

Several names are joined into one request, exactly as `gsh add` would:

```
$ gsh show rust go | head -3
# Created by https://www.toptal.com/developers/gitignore/api/rust,go
# Edit at https://www.toptal.com/developers/gitignore?templates=rust,go
```

Because only content goes to stdout, `gsh show go > .gitignore` produces a clean
file.

Exit codes: `0`; `1` with no arguments or an unknown template.

## `gsh version`

```
$ gsh version
gsh 1.2
```

Reads the `Version:` line from `current.txt` in the install directory. Prints
`gsh unknown` if that file is missing. `gsh -v` and `gsh --version` are the same
thing and short-circuit any command on the line.

## `gsh help`

`gsh help` prints the full listing: commands, options, environment variables,
exit codes and examples. `gsh help` is also what an unrecognised command gets,
after an error, with exit `1`:

```
$ gsh frobnicate
✗ unknown command: frobnicate

gsh - gitignore.sh
...
```

`gsh <command> --help` prints one command's usage instead. An unrecognised
command name after `--help` falls through to the full help.

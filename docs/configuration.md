# Configuration

gsh has no config file. Everything is an environment variable or a flag, and a
flag always beats the environment.

> Output on this page is real. A few absolute paths from throwaway test installs
> have been shortened where they would otherwise run off the line; nothing else
> has been edited.

| Variable | Default | Read by | What it does |
|---|---|---|---|
| [`GSH_HOME`](#gsh_home) | *(searched)* | `gsh`, `gsh-update` | The directory holding `resources/` and `current.txt` |
| [`GSH_API`](#gsh_api) | `https://www.toptal.com/developers/gitignore/api` | `gsh` | Base URL of the template API |
| [`GSH_CACHE_DIR`](#gsh_cache_dir) | `${XDG_CACHE_HOME:-~/.cache}/gsh` | `gsh` | Where the catalog and bundles are cached |
| [`GSH_CACHE_TTL`](#gsh_cache_ttl) | `86400` | `gsh` | Cache lifetime in seconds |
| [`GSH_REMOTE_URL`](#gsh_remote_url) | GitHub `raw` `current.txt` | `gsh-update` | Where the published version is read from |
| [`GSH_ASCII`](#gsh_ascii) | *(unset)* | `gsh`, `gsh-update`, `setup` | Any non-empty value: ASCII icons instead of Unicode |
| [`NO_COLOR`](#no_color) | *(unset)* | `gsh`, `gsh-update`, `setup` | Set at all, even to empty: no colour |

Three more are read indirectly: `XDG_CACHE_HOME` (below), `TERM` (`dumb`
disables colour) and the locale (`LC_ALL`, `LC_CTYPE`, `LANG` — a non-UTF-8
locale disables Unicode icons). `COLUMNS`, if it is exported, sets the width
`gsh list` lays its columns out to.

`gsh help` lists only the first four. The rest are documented here.

---

## `GSH_HOME`

The directory that holds `resources/` (local templates and the offline fallback)
and `current.txt` (the version). Set it and every search below is skipped:

```
$ gsh version
gsh 1.2
$ gsh list
c
go
java
linux
...

$ GSH_HOME=/tmp/althome gsh version
gsh 0.0-alt
$ GSH_HOME=/tmp/althome gsh list
alt-only
```

*(both `gsh list` runs above are offline with no catalog cached, so they show
only local templates — see [Templates](templates.md#offline-and-the-fallback-order).)*

With `GSH_HOME` unset, gsh resolves its own path — following symlinks by hand,
because macOS `readlink` has no `-f` — and takes the **first** of these that
contains a `resources/` directory:

| # | Candidate | The layout it means |
|---|---|---|
| 1 | `<script>/..` | The clone: `<install>/gsh/main` beside `<install>/resources` |
| 2 | `<script>/../share/gsh` | A prefix install: `<prefix>/bin/gsh` + `<prefix>/share/gsh` |
| 3 | `/usr/local/share/gsh` | A system install |
| 4 | `/usr/share/gsh` | A system install |

`gsh-update` uses the same list with its own directory prepended, because
`update` sits at the root of the clone rather than one level down.

The clone is tried first so a checkout always wins over a stale system install,
and the relative prefix path is tried before the absolute ones so a
`--prefix=$HOME/.local` install is not shadowed by something in `/usr`. Both
layouts work today:

```
$ bash prefix/bin/gsh version
gsh 1.2
```

Symlinks are followed to the real file first, so putting a link on your `PATH`
resolves to the clone the link points at, not to the directory holding the link.
Chains are followed up to 40 hops, which is what stops a symlink cycle from
hanging the shell.

If nothing matches, gsh stops and tells you what it looked at:

```
$ bash orphan/bin/gsh version
✗ cannot find gsh's data directory (the one holding resources/)
  looked in:
    /path/to/orphan/bin/..
    /path/to/orphan/bin/../share/gsh
    /usr/local/share/gsh
    /usr/share/gsh
  fix: set GSH_HOME to the directory holding resources/ and current.txt
```

Two things to know about the variable itself:

- **It is not validated.** `GSH_HOME=/nope` is accepted; gsh then finds no
  `current.txt` (`gsh version` prints `gsh unknown`) and no local templates,
  while remote templates keep working normally. The error above only appears
  when `GSH_HOME` is unset *and* no candidate matched.
- **It is used verbatim, not resolved.** A relative `GSH_HOME` is relative to
  wherever you happen to be, so it works from one directory and silently does
  not from another. Use an absolute path.

## `GSH_API`

Base URL for the catalog (`$GSH_API/list`) and for template bundles
(`$GSH_API/<comma,list>`). Point it at a mirror, or at a local directory with
`file://` when you want gsh to run with no network at all.

The response is validated whatever the source: names are filtered on ingest, and
a body that is markup or that contains gsh's own block markers is refused. See
[Templates](templates.md#what-a-response-has-to-look-like). Redirects are limited
to five and forced to stay on `https`.

## `GSH_CACHE_DIR`

Where `list.txt` and `templates/` live. The default follows the XDG base
directory spec:

```
$ XDG_CACHE_HOME=$PWD/xdg gsh list rust >/dev/null
$ find xdg
xdg
xdg/gsh
xdg/gsh/list.txt
xdg/gsh/templates
```

so `~/.cache/gsh` unless `XDG_CACHE_HOME` says otherwise, and `GSH_CACHE_DIR`
overrides both. Deleting the directory is always safe.

The directory has to be creatable; gsh does not check that it was. Point it at
somewhere unwritable and you get raw shell errors and a bogus warning rather
than a clean message — see [Rough edges](templates.md#rough-edges).

## `GSH_CACHE_TTL`

Seconds before a cached file is considered stale, default `86400` — 24 hours.
It applies to the catalog and to every bundle independently.

Stale does not mean deleted. A stale entry is refetched when the network allows
and used as-is when it does not, so raising or lowering this changes how often
gsh talks to gitignore.io, never whether it works offline.

```
$ gsh list rust                       # catalog cached an hour ago
› 2 templates matching rust
rust
rust-analyzer

$ GSH_CACHE_TTL=0 gsh list rust       # same cache, now stale, API unreachable
! cannot reach gitignore.io, using the cached catalog
› 2 templates matching rust
rust
rust-analyzer
```

`GSH_CACHE_TTL=0` forces a fetch attempt on every command, which is `--refresh`
made permanent. There is no "never expire" value; a very large number is the way
to say it.

## `GSH_REMOTE_URL`

Read by `gsh-update` only. The URL of the published `current.txt`, default:

```
https://raw.githubusercontent.com/MatteoKrsticDev/gsh/main/current.txt
```

`gsh-update` compares that file with your local `current.txt` **byte for byte**,
not by version number, so any difference at all — including one in the two lines
of preamble — counts as "not up to date". See
[Install and update](install.md#gsh-update).

## `GSH_ASCII`

Any **non-empty** value swaps the Unicode icons for ASCII: `✓ ✗ ! › • →` become
`+ x ! > * ->`, and the braille spinner becomes `| / - \`. `--ascii` does the
same for one run.

```
$ gsh doctor
gitignore
✓ .gitignore found (33 rules)
  • templates: go macos rust

$ GSH_ASCII=1 gsh doctor
gitignore
+ .gitignore found (33 rules)
  * templates: go macos rust
```

`GSH_ASCII=` (empty) is treated as unset and changes nothing — the opposite of
`NO_COLOR` below, which is deliberate on both sides.

You do not usually need it. gsh already falls back to ASCII when the locale is
not UTF-8:

```
$ LC_ALL=C gsh doctor
gitignore
+ .gitignore found (33 rules)
  * templates: go macos rust
```

With no locale variable set at all, gsh assumes UTF-8.

## `NO_COLOR`

[no-color.org](https://no-color.org) semantics: the variable disables colour by
its **presence**, whatever its value. `NO_COLOR=0` and `NO_COLOR=` both turn
colour off — that is the standard, not a bug, and it is why `GSH_ASCII` does not
work the same way.

Counting output lines that carry an ANSI escape, on a real terminal:

| Run | Lines with escapes |
|---|---|
| `gsh doctor` | 12 |
| `NO_COLOR= gsh doctor` | 0 |
| `NO_COLOR=0 gsh doctor` | 0 |
| `TERM=dumb gsh doctor` | 0 |
| `gsh doctor --no-color` | 0 |

Colour is decided per stream, so a terminal on stdout says nothing about stderr:
`gsh doctor > report.txt` from a terminal writes a clean file while the prompts
on stderr stay coloured. Spinners are on stderr and are suppressed when stderr
is not a terminal, so nothing in a CI log ever contains a spinner frame.

There is no flag or variable that forces colour **on**.

---

## Flags

Recognised anywhere on the command line — before the command, after it, or in
between. `gsh -y --no-color add macos` and `gsh add macos --no-color -y` are the
same command.

| Flag | Effect |
|---|---|
| `-y`, `--yes` | Assume yes; never prompt |
| `--refresh` | Ignore the cache and refetch the catalog and any bundle needed |
| `--no-color` | Disable colour (`NO_COLOR` does the same) |
| `--ascii` | ASCII icons (`GSH_ASCII` does the same) |
| `-h`, `--help` | Help for the command it accompanies; short-circuits, nothing is written |
| `-v`, `--version` | Print the version and exit `0`; short-circuits |
| `--` | Everything after it is a positional argument, never a flag |

`-h` and `-v` are parsed as flags rather than as positionals, which is why
`gsh add --help` prints help for `add` instead of complaining about a template
called `help`.

`gsh-update` takes only `--no-color`, `--ascii` and `-h`/`--help`, and rejects
anything else:

```
$ gsh-update --frobnicate
gsh-update: unknown option: --frobnicate
gsh-update: run 'gsh-update --help' for usage
```

`gsh` itself does not: an unknown `--flag` is treated as a positional argument,
so `gsh add --frobnicate` reports an unknown *template* — and, on the way there,
a page of `grep` usage, because the name reaches `grep` as an option. Cosmetic,
but it is why an argument starting with `-` is worth avoiding even after `--`.

## Exit codes

`gsh` uses two:

| Code | Meaning |
|---|---|
| `0` | Success. For `doctor`: no problems — warnings alone still exit `0`. |
| `1` | Failure, aborted by the user, or `doctor` found a problem. |

Answering `n` at a confirmation prompt is exit `1`, on purpose: a CI job that
forgot `-y` must not look like it did the work. Per-command detail is in the
[command reference](commands.md#exit-codes).

`gsh-update` adds a third:

| Code | Meaning |
|---|---|
| `0` | Already up to date, or updated and verified |
| `1` | Nothing changed: remote unreachable, bad reply, not a clone, or `git pull` failed |
| `2` | `git pull` succeeded but the new version did not land on disk |

`setup` uses `0` and `1`. Note that a stale `gsh` alias found elsewhere in your
rc file is reported loudly but still exits `0` — the install did happen. See
[Install and update](install.md).

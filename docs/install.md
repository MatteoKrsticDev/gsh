# Install and update

gsh is three files of bash and a directory of fallback templates. It needs
**bash, git and curl** and nothing else — no node, no python, no package
manager.

**There is no Homebrew, apt or pacman package.** Not "not yet documented":
they do not exist. Today the install is a clone.

**Verified on macOS and Linux. Windows is unverified** — gsh should work under
Git Bash or WSL, and it has a `windows` detection branch, but nobody has
confirmed it end to end. Treat it as unsupported until someone does.

macOS ships bash 3.2.57 from 2007 and gsh runs on it deliberately, so you do not
need a newer bash from Homebrew. See [CONTRIBUTING.md](../CONTRIBUTING.md) if you
are going to write code against that constraint.

> Every block on this page is real output. Absolute paths in it come from test
> installs and have been shortened to `~/gsh` and `~/.zshrc` so the examples stay
> readable; nothing else has been edited.

## Install

```bash
git clone https://github.com/MatteoKrsticDev/gsh.git ~/gsh
bash ~/gsh/setup
source ~/.zshrc        # or ~/.bashrc
```

```
$ bash ~/gsh/setup

gsh - gitignore.sh

› zsh detected
✓ alias gsh
✓ alias gsh-update

✓ installed in ~/gsh
  rc file: ~/.zshrc

› run source ~/.zshrc (or open a new terminal), then:
    gsh auto      # in any project
    gsh doctor    # audit the repo
```

Then check it:

```
$ gsh version
gsh 1.2
```

The clone location is yours to choose. `setup` records wherever it is, and
nothing else in gsh cares.

## What `setup` writes

A **delimited managed block**, always at the end of the rc file:

```bash
# >>> gsh >>>
# managed by the gsh installer - re-run setup to update this block
alias gsh='bash "/Users/you/gsh/gsh/main"'
alias gsh-update='bash "/Users/you/gsh/update"'
# <<< gsh <<<
```

Everything else in the file is preserved verbatim. Re-running `setup` strips the
old block, keeps the rest, and appends a fresh one — so a second run is
byte-identical to the first, and moving the clone somewhere else is just
`bash <new-path>/setup`.

It also `chmod +x`s `gsh/main` and `update`. The aliases do not need that — they
run `bash <path>` explicitly — but running the scripts directly, or through a
symlink on your `PATH`, does.

### Which rc file

From the basename of `$SHELL`:

| `$SHELL` | rc file |
|---|---|
| `zsh` | `~/.zshrc` |
| `bash` | `~/.bashrc`, or `~/.bash_profile` when that exists and `~/.bashrc` does not |
| anything else | none — the two alias lines are printed for you to paste, exit `1` |

```
$ SHELL=/usr/bin/fish bash ~/gsh/setup

gsh - gitignore.sh

✗ unsupported shell: /usr/bin/fish
  add these lines to your shell rc file by hand:
    alias gsh='bash "~/gsh/gsh/main"'
    alias gsh-update='bash "~/gsh/update"'
```

The rc file is created if it does not exist, and `setup` checks it is writable
with `-w` before it starts — `touch` succeeds on a mode-444 file you own, which
used to make the installer report success after an append that never happened.

### When it finds a stale alias

An `alias gsh=` or `alias gsh-update=` line elsewhere in the file that is **not**
byte-identical to the ones being installed — an older installer's copy, a
hand-written one, a clone you moved — is reported:

```
✗ ~/.zshrc has gsh alias line(s) that do not point at this install:
      alias gsh="bash /opt/old-gsh/gsh/main"
  the managed block is written last, so 'gsh' still resolves here,
  but the stale line will confuse the next person who reads ~/.zshrc.
  fix: delete it from ~/.zshrc - the block above already provides it.
```

The install still happened and `setup` still exits `0`. Because the managed block
is written **last**, the shell reads it last and `gsh` resolves to this install;
the stale line is a readability problem, not a functional one. gsh does not
delete lines you might have written on purpose.

Lines that *are* byte-identical to what gsh installs get folded into the managed
block instead of being reported — that is how an install predating the block
format upgrades cleanly.

### When it refuses

One marker without its pair, usually from a half-finished hand edit:

```
✗ ~/.zshrc has one gsh block marker without its pair
  setup will not guess where the block ends.
  fix: delete the leftover marker line from ~/.zshrc, then re-run bash setup
```

Exit `1`, nothing written. With only one marker the strip step cannot tell where
the block ends and would swallow the rest of your rc file. Delete the orphan
line and re-run.

### The alias, and when you want something else

Aliases exist only in interactive shells. `gsh` in a shell script, a Makefile, a
`git` hook or a CI job will not resolve. Two alternatives, both verified:

```bash
# a symlink on your PATH - resolved through to the clone, so this works
ln -s ~/gsh/gsh/main ~/.local/bin/gsh

# or just run the script
bash ~/gsh/gsh/main doctor
```

The second is what [CI](doctor.md#in-ci) should do: clone and run, no install
step at all.

## Install layouts

`gsh/main` and `update` find `resources/` and `current.txt` by resolving their
own path — through symlinks, by hand, because macOS `readlink` has no `-f` — and
taking the first candidate that contains a `resources/` directory:

| Layout | Executable | Data |
|---|---|---|
| Clone *(what `setup` installs)* | `<install>/gsh/main` | `<install>/resources`, `<install>/current.txt` |
| Prefix | `<prefix>/bin/gsh` | `<prefix>/share/gsh/` |
| System | anywhere | `/usr/local/share/gsh` or `/usr/share/gsh` |
| Explicit | anywhere | wherever [`GSH_HOME`](configuration.md#gsh_home) points |

The clone is checked first, so a checkout you are working in always wins over a
system install; the relative `../share/gsh` is checked before the absolute
`/usr/*` paths, so a `--prefix` install is not shadowed by one in `/usr`.

The prefix layout works today even though nothing packages gsh into it yet:

```
$ bash prefix/bin/gsh version
gsh 1.2
```

If no candidate matches, gsh stops with the list it tried and tells you to set
`GSH_HOME` — see [Configuration](configuration.md#gsh_home).

`gsh/main` re-execs itself under `bash` when it is started by `sh` or `dash`, or
under bash in POSIX mode, because it genuinely needs bash: process substitution,
`$'…'`, arrays, `IFS=$'\t'`. All of these work:

```
$ sh ~/gsh/gsh/main version
gsh 1.2
$ dash ~/gsh/gsh/main version
gsh 1.2
$ bash --posix ~/gsh/gsh/main version
gsh 1.2
```

With no bash on the system at all there is nothing to re-exec into, and gsh says
so instead of failing strangely:

```
gsh: needs bash (not sh or dash, and not POSIX mode)
```

## `gsh-update`

```
$ gsh-update
✓ already up to date (1.2)
```

It does four things, in this order:

1. `curl` the published `current.txt`
   ([`GSH_REMOTE_URL`](configuration.md#gsh_remote_url)).
2. Compare it with the local `current.txt`. Identical → exit `0`, nothing done.
3. `git -C <install> pull --ff-only`.
4. Compare the two files **again**, and only then report success.

Step 4 is the interesting one. A pull can succeed and still not bring the new
version — wrong branch, a remote behind its own raw endpoint, a fast-forward that
landed nothing — so `gsh-update` trusts the file on disk rather than git's exit
status. Reproduced here by pointing a real clone at a published version its
remote does not carry:

```
$ GSH_REMOTE_URL=file:///tmp/published-9.9.txt gsh-update
✗ update did not take, gsh is still at 1.2
  git pull succeeded but ~/gsh/current.txt does not match the published 9.9.
  fix: cd ~/gsh && git status && git branch -vv
       make sure the branch you track is the one carrying 9.9
```

That is the only thing that exits `2`.

Two properties of step 2 worth knowing: the comparison is **byte for byte, not a
version comparison**, so any difference in `current.txt` — including in its two
lines of preamble — counts as "an update is available"; and it has no notion of
newer or older, so a remote that has gone *backwards* also triggers a pull.

### Why it refuses on a packaged install

```
$ gsh-update
✗ gsh at /usr/local/share/gsh was not installed from a git clone, update aborted
  gsh-update updates by pulling, which is not how this copy got here.
  fix: update it the way you installed it -
    Homebrew     brew upgrade gsh
    Arch/AUR     sudo pacman -Syu gsh
    Debian       sudo apt update && sudo apt upgrade gsh
  or install a clone you own:
    git clone https://github.com/MatteoKrsticDev/gsh.git && bash gsh/setup
```

The check is `git rev-parse --is-inside-work-tree` in the install directory, not
a test for a `.git` **directory** — `.git` is a regular file in a worktree or a
submodule, and both of those are still clones.

A package manager owns the files it installed. `git pull` there has no remote to
pull from, and would be the wrong tool even if it did: it would fight the package
database, and on a system install it would need root to do it. So `gsh-update`
stops and hands you back to whatever installed it.

Read that message with today's reality in mind: **no such package exists yet**.
If you see it, either your install is a copy rather than a clone, or `GSH_HOME`
points somewhere that is not a checkout. The `git clone … && bash gsh/setup`
line at the bottom is the working answer.

The other two failures, both exit `1` and both leave everything untouched. An
unreachable endpoint (reproduced against a dead local port, so the `tried:` line
shows that instead of the GitHub default):

```
$ GSH_REMOTE_URL=http://127.0.0.1:9/current.txt gsh-update
✗ cannot reach GitHub, update aborted
  tried: http://127.0.0.1:9/current.txt
  fix: check your connection, then run gsh-update again
```

A reply with no `Version:` line — a captive portal, a 404 page, a moved file —
is treated the same way rather than being parsed into an empty version:

```
✗ unexpected reply from the version endpoint, update aborted
  <url> returned no 'Version:' line
  fix: open that URL in a browser; if it looks wrong, report it
```

And a pull that git itself refuses. git prints its own explanation first; the
last two lines are gsh's:

```
$ gsh-update
✗ git pull failed, update aborted
  you may have local changes: cd ~/gsh && git status
```

Updating by hand is always available and is exactly what `gsh-update` would have
done:

```bash
cd ~/gsh && git pull --ff-only
```

No re-run of `setup` is needed after an update; the alias points at the path, and
the path did not move.

## Uninstalling

There is no uninstall script. Three steps, none of them clever:

```bash
# 1. delete the managed block from your rc file
#    everything between  # >>> gsh >>>  and  # <<< gsh <<<  inclusive

# 2. delete the clone
rm -rf ~/gsh

# 3. optional - the cache
rm -rf ~/.cache/gsh
```

gsh writes nothing else outside a repository. Inside repositories, the blocks it
added to your `.gitignore` files stay; they are ordinary text and
[safe to delete by hand](block-format.md).

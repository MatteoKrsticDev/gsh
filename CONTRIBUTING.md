# Contributing to gsh

gsh is three bash scripts and a directory of fallback templates. There is no
build step, no test framework and no dependency file, and the aim is to keep it
that way.

Read [docs/](docs/README.md) first — especially
[How `gsh auto` decides](docs/auto.md) if you are touching detection, and
[The block format](docs/block-format.md) if you are touching anything that
writes to a `.gitignore`.

## Running from a clone

No install step. The scripts find their own data directory, so a checkout runs
as-is:

```bash
git clone https://github.com/MatteoKrsticDev/gsh.git
cd gsh
bash gsh/main version        # gsh 1.2
bash gsh/main doctor
```

Point the cache somewhere disposable while you work, so you are not fighting —
or poisoning — the one your installed gsh uses:

```bash
export GSH_CACHE_DIR=/tmp/gsh-dev-cache
```

Test in throwaway repositories, never in this one:

```bash
mkdir /tmp/t && cd /tmp/t && git init
touch package.json
bash ~/path/to/gsh/gsh/main auto -y
```

Every environment variable is in [docs/configuration.md](docs/configuration.md).
`GSH_API` accepts a `file://` URL, which is the cheapest way to test offline
behaviour and malformed responses without unplugging anything.

## The bash 3.2.57 constraint

**gsh must run on bash 3.2.57.** That is what macOS ships — released in 2007,
frozen at GPLv2 — and it is the shell most of gsh's users will run it under
whether they know it or not. `/bin/bash` on a Mac:

```
$ /bin/bash --version
GNU bash, version 3.2.57(1)-release (arm64-apple-darwin25)
```

If your bash is 5.x from Homebrew, your patch can be broken and you will not find
out. **Test with `/bin/bash` explicitly.**

What that rules out, with the actual failure each one gives on 3.2:

| Feature | Since | On bash 3.2 |
|---|---|---|
| `mapfile` / `readarray` | 4.0 | `/bin/bash: mapfile: command not found` |
| Associative arrays (`declare -A`) | 4.0 | `declare: -A: invalid option` |
| `wait -n` | 4.3 | `wait: -n: invalid option` |
| Case conversion (`${x^^}`, `${x,,}`) | 4.0 | `${x^^}: bad substitution` |
| `EPOCHSECONDS`, `BASH_ARGV0`, `SRANDOM` | 5.0+ | empty, silently |
| `&>>`, `\|&` | 4.0 | `syntax error near unexpected token` |
| Negative array indices (`${a[-1]}`) | 4.3 | `bad array subscript` |

Note the two failure modes. The first three are loud. `${EPOCHSECONDS}` is
**silent** — it expands to nothing and your arithmetic quietly becomes zero — and
that is the class of bug that reaches a Mac user and not you.

What to use instead:

| Instead of | Write |
|---|---|
| `mapfile -t arr < file` | `while IFS= read -r line; do arr[n++]=$line; done < file` |
| An associative array | A parallel indexed array plus a linear scan (see `tpl_index` in `gsh/main`), or a `case` over a string |
| `wait -n` | `wait "$pid"` — gsh runs one background job at a time |
| `${x^^}` | `tr '[:lower:]' '[:upper:]'`, or a `case` with `[Yy]` classes |
| `$EPOCHSECONDS` | `date +%s` |

Two more house rules that come from the same place:

- **Process substitution and `$'…'` are fine** — they are bash 3.2 features. But
  they are why `gsh/main` re-execs itself when it is started by `sh` or `dash`.
  Anything you add above that re-exec block, at the top of the file, must be
  POSIX: `sh` parses and runs the file one command at a time and would die on a
  bash-only construct before ever reaching the re-exec.
- **`set -u` is on.** Use `${var:-}` for anything that may be unset, and
  `${arr[@]+"${arr[@]}"}` for a possibly-empty array — bash 3.2 treats an empty
  array as unset.

## No new dependencies

gsh requires **bash, git and curl**. That is the whole list, and adding to it is
the one change that will be turned down on principle. No jq, no python, no node,
no downloading a helper at runtime.

Beyond those three, only utilities that exist on a stock macOS *and* a minimal
Linux: `sed grep tr sort uniq wc cat head tail cut basename dirname mktemp date
stat uname mkdir rm touch cmp chmod readlink printf`.

They have to work in **both** the BSD and GNU flavours, which is not free:

- `stat` — the flags differ entirely. gsh runs `stat -f %m` and falls back to
  `stat -c %Y`.
- `readlink` — macOS has no `-f`. gsh follows symlink chains by hand, with a hop
  limit.
- `sed -i` — needs a backup suffix argument on BSD. Do not use it in the scripts.
- `grep -P` — not on macOS. Do not use it.
- `date -d` / `date -v` — both platform-specific. `date +%s` is the portable one.

`column` is the one optional utility: `gsh list` uses it when stdout is a
terminal and it exists, and prints one name per line when it does not.

Two habits for anything the user typed. Use `-F` — `gsh list [` must search for a
literal bracket, not die inside a regex engine — and pass `--` before the
argument, so a name starting with `-` cannot reach `grep` as an option.
`cmd_list` does both; `known_template`, `fetch_bundle` and `install_templates`
are missing the `--`, which is why `gsh add -- -y` prints a page of `grep` usage
before it reports the unknown template. Worth fixing if you are in there.

## Adding a detection rule

Rules live in `SCAN_RULES` in `gsh/main`, one per line:

```
<template>[ <template>...]|<strength>|<glob>
```

Five things to get right:

1. **The template must exist in the gitignore.io catalog.** Check with
   `bash gsh/main list <name>`. If it does not, `gsh auto` warns and drops the
   detection at runtime, which is a bug in the table rather than in anybody's
   repository:

   ```
   $ gsh auto -y
   Detected
   ! skipping 'bun' (bun.lockb): no such template available
     ✓ macos              you are on macOS
   ```

   (There is no `bun` template upstream today. That output is what a wrong entry
   looks like.)

2. **At most one `*`, at the start or at the end of the glob.** This is a hard
   constraint, not a style preference: `detect_stack` compiles each glob down to
   its longest literal run and hands the whole set to one `grep -F` pass, which
   sieves 50,000 paths before bash sees any of them. A glob with a `*` in the
   middle would sieve wrong.

3. **`strong` or `weak`.** `strong` is a project manifest — somebody wrote
   `Cargo.toml` on purpose, so it counts wherever it sits. `weak` is a bare
   extension; a weak match found *only* under a test, fixture, example or docs
   directory is discarded. If your marker is `*.<ext>`, it is almost certainly
   weak.

4. **Put it next to its ecosystem.** The table order decides the order of the
   `Detected` list, so a polyglot repository should read top-down like a stack
   list.

5. **The glob matches the tail of a path**, so `package.json` matches
   `frontend/package.json` too. If your marker only makes sense at a specific
   depth, include the directory: `config/application.rb`, not `application.rb`.

Then test it in a throwaway repository — this is a real run of a rule added for
this document:

```
$ grep -n '^bazel' gsh/main
711:bazel|strong|MODULE.bazel

$ mkdir /tmp/bz && cd /tmp/bz && git init -q .
$ mkdir src && touch MODULE.bazel src/BUILD
$ bash ~/gsh/gsh/main auto -y

Detected
  ✓ bazel              MODULE.bazel
  ✓ macos              you are on macOS

✓ created .gitignore
✓ added bazel,macos (22 rules)
```

Check both directions before you send it: that the marker is detected where it
should be, and that a copy of it under `tests/fixtures/` does **not** drag the
template in if the rule is weak.

Then update [docs/auto.md](docs/auto.md): the rule table, and the rule/template
counts in the opening paragraph and in `README.md`. A rule that is not in the
table is a rule nobody can explain to a confused user.

## Adding a local template

Drop a file in `resources/`:

```bash
printf '# my company\n*.secret\n' > resources/myco.gitignore
bash gsh/main list myco          # it should appear
```

One rule decides whether this is worth doing at all: **check that the name does
not exist upstream first.** A name in the gitignore.io catalog always wins over
a local file — the local copy is then only reachable when the network is down,
which is the worst possible moment to discover it says something different from
the online version. The full precedence is in
[docs/templates.md](docs/templates.md#a-local-name-never-overrides-the-catalog).

The ten templates already in `resources/` are the deliberate exception: they all
exist upstream and are there purely as the offline fallback for the most common
stacks.

Names must be `[A-Za-z0-9._+-]`, at most 64 characters — they become cache path
components and block-marker tokens.

## Running the checks

There is no test suite and no CI. The checks are these, and they are all manual:

```bash
# 1. syntax, under the oldest bash that matters
for f in gsh/main setup update; do /bin/bash -n "$f" && echo "$f: syntax OK"; done

# 2. shellcheck, if you have it (it is not required and not installed by default)
shellcheck -s bash gsh/main setup update
```

Then a smoke test, in throwaway repositories, with a throwaway cache:

```bash
export GSH_CACHE_DIR=/tmp/gsh-dev-cache
G="bash $PWD/gsh/main"

mkdir /tmp/t1 && cd /tmp/t1 && git init -q .
$G auto -y                       # detects, creates .gitignore, exits 0
$G auto -y                       # 'everything already in .gitignore', exits 0
$G add rust go -y                # one block, one request
$G add nope                      # unknown template, exits 1
$G doctor                        # exits 0 here
$G list rust                     # filter is a substring
$G show rust | head -1           # stdout only, no colour when piped
```

The paths through the code that are easy to break and easy to forget:

| What to try | What should happen |
|---|---|
| `GSH_API=http://127.0.0.1:9/api $G show node` | falls back to cache, then to `resources/`, and says which |
| `rm -rf $GSH_CACHE_DIR` then the same | `only local templates are available`, still exits `0` |
| `$G show rails` with no network and no cache | `cannot download: rails`, exits `1` |
| `$G auto` answering `n` | `aborted`, exits **`1`** |
| `$G doctor` in a repo with a committed `.env` | reports it, exits `1` |
| `$G show go > out && grep -c $'\033' out` | `0` — no escapes when stdout is not a terminal |
| `sh gsh/main version` | re-execs under bash, prints the version |
| `LC_ALL=C $G doctor` | ASCII icons |
| a `.gitignore` with one `### gsh:start` and no end | warns once, changes nothing |

Check the exit codes, not just the output. They are a documented contract —
[docs/configuration.md](docs/configuration.md#exit-codes) — and `gsh doctor`
exiting `1` on problems is what makes it usable in CI.

## Sending a patch

- **One concern per patch.** A detection rule and a doctor check are two patches.
- **Comments explain why, not what.** The existing code is dense with the reason
  a thing is done the way it is — a prune list that is correctness rather than
  speed, a `grep -F` that exists because of what a user might type. Keep that up;
  it is the most valuable thing in the file.
- **Every error message names the fix.** `err` says what went wrong, `dim` says
  what to run. Do not add an error that leaves the user with nowhere to go.
- **Nothing destructive without a prompt**, and `-y` is the only way to skip one.
  `gsh doctor` in particular must never modify anything: it prints the command
  and lets the user run it.
- **Update the docs in the same patch.** A behaviour change that is not in
  [docs/](docs/README.md) will be described wrongly by somebody within a month.

Two things to know before you open a PR:

- **`docs/` is currently listed in this repository's own `.gitignore`**, so
  documentation changes do not show up in `git status` and are not staged by
  `git add .`. Use `git add -f docs/…` until that line is removed.
- **This repository has no LICENSE file.** Until one is added, the terms under
  which contributions are accepted are undefined. Worth asking about before you
  spend a weekend on something large.

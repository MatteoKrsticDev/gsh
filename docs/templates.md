# Templates

Everything gsh writes into a `.gitignore` comes from one of two places: the
[gitignore.io](https://www.toptal.com/developers/gitignore) catalog, or the
`resources/` directory of your gsh install. This page is about where a template
comes from, what happens when the network is not there, and how to add one of
your own.

## The catalog

571 templates, fetched from `GET $GSH_API/list` and kept in
`$GSH_CACHE_DIR/list.txt`, one name per line.

```
$ gsh list | head -5
› 571 templates available (gsh list <filter> to search)
1c
1c-bitrix
a-frame
actionscript
```

`gsh list <filter>` searches it as a case-insensitive substring, and
[`gsh show`](commands.md#gsh-show) prints a template without installing it.

Names are validated on the way in and on the way out. A catalog entry that is
not `[A-Za-z0-9._+-]{1,64}` is dropped — the cache outlives the response it came
from, so a bad name is never allowed onto disk in the first place — and the
count of what was dropped is reported:

```
! ignored 1 malformed name(s) in the catalog
```

Today's catalog does not trigger it — all 571 names pass — and the longest
genuine name is `openframeworks+visualstudio`, 27 characters.

## One request per bundle

`gsh add`, `gsh auto` and `gsh show` join every name they need with commas and
ask for all of them at once. Twelve templates cost one HTTP request, not twelve,
and the response says so in its first line:

```
$ gsh show rust go | head -3
# Created by https://www.toptal.com/developers/gitignore/api/rust,go
# Edit at https://www.toptal.com/developers/gitignore?templates=rust,go
```

The bundle also lands in `.gitignore` as **one block** with one marker line —
see [the block format](block-format.md).

Two consequences worth knowing:

- The order you type is the order gitignore.io gets, and `rust,go` and `go,rust`
  are two different cache entries holding the same rules. Harmless, just a
  duplicated file.
- `gsh add` validates every name against the catalog *before* it writes
  anything, so a typo in one name means nothing is written at all rather than
  half a bundle.

## The cache

| | |
|---|---|
| Location | `$GSH_CACHE_DIR`, default `${XDG_CACHE_HOME:-~/.cache}/gsh` |
| Catalog | `list.txt` |
| Bundles | `templates/<key>.gitignore` |
| Lifetime | `GSH_CACHE_TTL` seconds, default `86400` (24 hours) |

```
$ ls ~/.cache/gsh
list.txt
templates

$ ls ~/.cache/gsh/templates
java-858639778904.gitignore
macos-a8d0f9c4854d.gitignore
node-32b6e49583c2.gitignore
node_python_rust_go_java_kotlin_swift_dart_elixir_haskell_julia_nim_zig_ruby_rai-9e58f9b256d1.gitignore
rust_go-8f1e59edbae9.gitignore
```

A file counts as fresh when it is **non-empty** and its mtime is less than
`GSH_CACHE_TTL` seconds old. Both halves matter: the zero-byte `list.txt` gsh
writes when it cannot reach the API is never treated as fresh, so the next run
tries the network again rather than believing an empty catalog for a day.

Nothing expires by deletion. A stale entry is refetched if the network allows
and served as-is if it does not, which is what makes gsh work on a plane.

`--refresh` ignores freshness for both the catalog and every bundle the command
needs, and refetches. Deleting `~/.cache/gsh` is equivalent and safe: the next
command rebuilds it.

### Why the filenames look like that

The cache key is the joined name list with commas turned into underscores,
truncated to 80 characters, plus a 12-character digest of the *full* list:

```
node_python_rust_go_java_kotlin_swift_dart_elixir_haskell_julia_nim_zig_ruby_rai-9e58f9b256d1.gitignore
```

That is 103 characters for a 30-template bundle, and it stays 103 characters for
a 300-template one. Joining the names verbatim would overflow `NAME_MAX` (255)
at around 26 templates — reachable in a polyglot monorepo — and both the write
and the later read would fail with a raw errno. The readable prefix is there for
whoever browses the cache; the digest carries the identity.

The digest comes from `git hash-object`. With no git on `PATH` the key falls
back to the prefix plus the length of the name list.

## Offline, and the fallback order

For each name you ask for, gsh decides where it comes from **before** touching
the network:

1. The name is in the cached catalog → it is a **remote** template.
2. Otherwise `resources/<name>.gitignore` exists → it is a **local** template.
3. Otherwise: `✗ unknown template: <name>`, exit `1`.

Local templates are emitted first, then the remote bundle is resolved:

1. Bundle cached and fresh, and no `--refresh` → use the cache, no request.
2. Otherwise download, then validate the body (below). On success it is cached
   and used.
3. Download failed, but a cached copy exists **at any age** → use it and say so:

   ```
   ! offline: using the cached copy of node
   ```

4. No cached copy, but some of the names have a local file in `resources/` →
   use those:

   ```
   $ gsh show python
   ! offline: falling back to the local copy of python

   #### python ####
   # Byte-compiled / cache
   __pycache__/
   ...
   ```

5. Names with neither a cached copy nor a local file → error, exit `1`:

   ```
   $ gsh show rails
   ✗ cannot download: rails
   ```

The catalog follows the same shape. Fresh cache: no request. Stale cache and no
network:

```
! cannot reach gitignore.io, using the cached catalog
```

No cache and no network — the catalog is empty, so every name falls to step 2
and only local templates exist:

```
$ gsh list
! cannot reach gitignore.io, only local templates are available
› 11 templates available (gsh list <filter> to search)
c
go
java
linux
macos
myco
node
python
rust
test
windows
```

Note what happens to `gsh show node` in that state: `node` is not in the (empty)
catalog, so it resolves as a local template and prints `resources/node.gitignore`
without a single request. The bundled `resources/` set —
`c go java linux macos node python rust test windows` — exists for exactly this
moment.

### What a response has to look like

The body is checked before it is written anywhere. Two bodies are refused:

```
! the API returned markup, not a gitignore template
! the API response contains gsh block markers, refusing it
```

The first catches an HTML error page served with HTTP 200, which would otherwise
land in your `.gitignore` verbatim and then be cached. The second matters more:
`### gsh:start … ###` in a downloaded body would be committed into your
`.gitignore` and read back later as "these templates are installed" — a forged
marker survives a fresh clone and a healthy API. A refused body falls through to
the same fallback chain as a failed download.

Genuine gitignore.io bodies contain neither. They open with
`# Created by https://…`.

CRLF is stripped on the way into the cache and on the way out of a local file,
so a `.gitignore` written on Windows does not arrive full of `^M`.

## Local templates

Anything in `resources/` named `<name>.gitignore` is a template. Adding one is
the whole procedure:

```
$ printf '# my company\n*.secret\nbuild-out/\n' > ~/gsh/resources/myco.gitignore

$ gsh list myco
› 1 templates matching myco
myco

$ gsh add myco -y
✓ created .gitignore
✓ added myco (2 rules)

$ cat .gitignore
### gsh:start myco ###

#### myco ####
# my company
*.secret
build-out/
### gsh:end ###
```

The name must be `[A-Za-z0-9._+-]`, at most 64 characters — it becomes a cache
path component and a marker-line token, so its shape is gsh's to decide, not
yours. `gsh doctor` reads it back like any other template:

```
gitignore
✓ .gitignore found (2 rules)
  • templates: myco
```

### A local name never overrides the catalog

This is the part that surprises people, so it is worth being blunt:

> **If the name exists in the cached catalog, the catalog wins.** A local file
> with the same name is only reached when the catalog does not have that name,
> or when the download fails and nothing is cached.

All ten bundled templates — including `node`, `python` and `test` — exist
upstream, so in normal online use *none of them is ever read*. They are the
offline fallback, nothing more:

```
$ cat ~/gsh/resources/test.gitignore
# test template - used to verify that "gsh add test" works
test-artifact.txt

$ gsh show test
# Created by https://www.toptal.com/developers/gitignore/api/test
# Edit at https://www.toptal.com/developers/gitignore?templates=test

### Test ###
### Ignore all files that could be used to test your code and
### you wouldn't want to push
```

So `resources/` is for templates that **do not exist upstream**. Before naming
one, check:

```
$ gsh list <name>
```

If gitignore.io already has that name, pick a different one — otherwise your file
is dead weight that only appears when the network is down, which is the worst
possible time to discover it says something different from the online version.

Local templates are also install-local: they live in the directory
[`GSH_HOME`](configuration.md#gsh_home) resolves to, and a `gsh-update` pull will
not carry a file you never committed. Keep a copy somewhere if it matters.

## Rough edges

- **A cache directory that cannot be created is not handled.** `mkdir -p` is not
  checked, so pointing `GSH_CACHE_DIR` at an unwritable path produces raw shell
  errors, a nonsense `ignored 571 malformed name(s)` warning, and an exit status
  of `0` with the catalog silently reduced to the local templates. Point it
  somewhere writable.
- **Whitespace inside a catalog name is squashed, not rejected.** All spaces are
  stripped before validation, so a catalog entry `evil name` becomes the
  perfectly valid name `evilname` rather than being counted as malformed. Only
  matters against a hostile `GSH_API`.

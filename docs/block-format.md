# The block format

Everything gsh writes into a `.gitignore` goes inside a pair of marker lines:

```
### gsh:start node,python ###
# Created by https://www.toptal.com/developers/gitignore/api/node,python
...
### gsh:end ###
```

Both markers are `#` comments, so git treats them as nothing. They exist for one
reason: they tell gsh which lines are its own, and everything outside them is
yours.

This page is for people who edit `.gitignore` by hand — which you should feel
free to do.

## The exact syntax

| | |
|---|---|
| Start | `### gsh:start ` + a comma-separated template list + ` ###` |
| End | `### gsh:end ###` |

Each has to be the **whole line**, with nothing before or after it. Indentation
breaks the match, and so does a trailing word. This is not a marker and gsh will
never touch what follows it:

```
  ### gsh:start rust ###
```

The template list is comma-separated; spaces around the commas are tolerated on
read (`rust, go` reads back as `rust` and `go`).

## What gsh writes

`gsh add` and `gsh auto` **append**. Never insert, never rewrite, never reorder:

```
$ cat .gitignore
my own rules
*.bak

$ gsh add rust go -y
✓ added rust,go (13 rules)

$ cat .gitignore
my own rules
*.bak

### gsh:start rust,go ###
# Created by https://www.toptal.com/developers/gitignore/api/rust,go
# Edit at https://www.toptal.com/developers/gitignore?templates=rust,go
...
# End of https://www.toptal.com/developers/gitignore/api/rust,go
### gsh:end ###
```

One command writes one block, however many templates it asked for — they are
[fetched as a single bundle](templates.md#one-request-per-bundle). A second
command writes a second block:

```
$ gsh add macos -y
✓ added macos (18 rules)

$ grep -n 'gsh:' .gitignore
4:### gsh:start rust,go ###
48:### gsh:end ###
50:### gsh:start macos ###
87:### gsh:end ###
```

A blank line is inserted before each block when the file is not empty, and a
missing final newline on your last line is added first, so a file that did not
end in a newline does not get the marker welded onto it:

```
$ printf 'no-newline-here' > .gitignore
$ gsh add macos -y
$ head -4 .gitignore
no-newline-here

### gsh:start macos ###
# Created by https://www.toptal.com/developers/gitignore/api/macos
```

A [local template](templates.md#local-templates) carries its own sub-header
inside the block, which is how you can tell offline content from gitignore.io
content at a glance:

```
### gsh:start myco ###

#### myco ####
# my company
*.secret
build-out/
### gsh:end ###
```

After writing, gsh checks that the last line of the file is the end marker. That
is the only proof the whole block landed — a write cut short by a full disk or a
signal dies inside the copy, where the shell pipeline never sees it. If the check
fails you get an error, not a success message — `✗ could not write the block to
.gitignore, nothing was added`, followed by `check permissions and free space,
then re-run`.

## What gsh reads back

**Only the start-marker lines.** The content of a block is never parsed.

The names on those lines are what `gsh auto` means by `already present`, and what
`gsh doctor` lists under `templates:`. Blocks written by gsh 1.x
(`### <name>.gitignore ###`) are recognised too, so an old file keeps working:

```
$ cat .gitignore
### node.gitignore ###
node_modules/

$ gsh doctor
gitignore
✓ .gitignore found (1 rule)
  • templates: node
```

Because only the marker is read, the marker is the truth. An empty block still
counts as installed:

```
$ cat .gitignore
### gsh:start rust, go ###
### gsh:end ###

$ gsh doctor
gitignore
✓ .gitignore found (0 rules)
  • templates: go rust

$ gsh add rust go -y
  • rust already present, skipped
  • go already present, skipped
› nothing to do
```

And a name gsh never wrote counts too:

```
$ cat .gitignore
### gsh:start totally-made-up ###
### gsh:end ###

$ gsh doctor
gitignore
✓ .gitignore found (0 rules)
  • templates: totally-made-up
```

Neither is a bug to route around — it is what makes hand editing predictable. If
you want gsh to stop thinking a template is installed, remove its marker line. If
you want it to keep thinking so, keep the marker.

## What gsh will not touch

- **Your own lines.** Anything outside a block is copied through untouched, in
  place. gsh has no reformatter and no deduplicator.
- **Existing blocks.** Nothing rewrites, reorders or removes one. There is no
  `gsh remove`; deleting a block is a hand edit, and deleting it by hand is the
  supported way.
- **Rules you pasted in yourself.** gsh cannot see them, so if you hand-pasted
  the Node rules and later run `gsh auto`, it adds its own copy. `gsh doctor`
  then reports the duplicated rules — that is the intended signal, not a
  failure.
- **A `.gitignore` that is not a regular file.** A symlink is refused outright,
  because `>>` follows it and git stores and restores symlinks, so a repository
  shipping `.gitignore -> ~/.zshrc` would get gsh's appends written outside the
  work tree:

  ```
  $ gsh add macos -y
  ✗ .gitignore is a symlink, refusing to write through it
    replace it with a regular file, or run gsh where it points
  ```

  A directory named `.gitignore` is refused for the same reason:
  `✗ .gitignore exists but is not a regular file`.

## Removing a template

Delete the block, marker lines included:

```bash
# in $EDITOR: delete from '### gsh:start rust,go ###'
# through '### gsh:end ###' inclusive
```

gsh then no longer considers those templates installed and will re-add them if
asked. Deleting only the *rules* and leaving the markers behind does the
opposite: the templates stay "installed" and gsh will not re-add them.

Deleting individual rules from inside a block is fine as well — nothing verifies
block contents — but the next `gsh add <same template>` will not restore them,
because the marker still says it is there.

## Malformed blocks

A file's markers are malformed when they do not pair up cleanly:

| In the file | Result |
|---|---|
| A start with no end | malformed |
| Two starts before an end | malformed, both dropped |
| An end with no start | malformed |
| `### gsh:end ###` after a start line that has trailing text | malformed — the start was never a marker |
| Several correctly paired blocks | fine, any number |

gsh says so once, on stderr, and then **ignores every marker in the file**:

```
$ cat .gitignore
### gsh:start rust ###
target/

$ gsh doctor
gitignore
✓ .gitignore found (1 rule)
! .gitignore has malformed gsh block markers, ignoring them
```

Nothing is repaired and nothing is deleted; your file is left exactly as it is.
The practical consequence is that gsh forgets which templates are installed, so
the next `gsh auto` re-adds them in a fresh block and `gsh doctor` then reports
the duplicates. That direction is deliberate: failing this way can only
under-report. It can never claim a template is installed when it is not.

Which matters more than it sounds. The payload of a forged marker lives in the
*committed* `.gitignore`, so it survives a fresh clone and a healthy API — which
is also why a downloaded template body containing gsh markers is
[refused outright](templates.md#what-a-response-has-to-look-like). The same rule
retires an orphaned start marker left behind by a half-finished write, which
would otherwise make gsh refuse that template forever.

To fix a malformed file, delete the orphan marker line — or delete the whole
block and re-run `gsh auto`.

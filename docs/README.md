# gsh documentation

gsh version 1.2. Every command, flag and environment variable on these pages was
run against this version before it was written down.

## Pages

| Page | Read it when |
|---|---|
| [Command reference](commands.md) | You want the exact syntax, exit code or output of a command |
| [How `gsh auto` decides](auto.md) | Something was detected that you did not expect, or was not detected that you did |
| [How `gsh doctor` works](doctor.md) | You want to know what each check means, or to run doctor in CI |
| [Templates](templates.md) | You want to know where templates come from, how the cache behaves offline, or how to add your own |
| [The block format](block-format.md) | You edit `.gitignore` by hand and want to know what gsh will touch |
| [Configuration](configuration.md) | You need to point gsh at a different API, cache or install, or you want the flags and exit codes in one place |
| [Install and update](install.md) | You are installing, updating or removing gsh, or `gsh-update` refused to run |
| [Contributing](../CONTRIBUTING.md) | You are sending a patch: running from a clone, the bash 3.2 constraint, adding a rule, the checks |

## The one-paragraph version

`gsh auto` scans the repository through git's index, matches every path against
59 detection rules, and appends the matching [gitignore.io](https://www.toptal.com/developers/gitignore)
templates to `.gitignore` inside `### gsh:start … ###` / `### gsh:end ###`
markers. `gsh doctor` reads the repository and reports five classes of problem —
most importantly files that git already tracks despite matching `.gitignore`,
which no `.gitignore` change can fix. Templates are cached under
`~/.cache/gsh` for 24 hours and gsh keeps working from that cache offline.

## Conventions in these pages

- `$ ` marks a command; everything after it is real output from that command.
- Output is shown as gsh prints it with `--no-color`; on a terminal the same
  text is coloured.
- Where a page says "verified", it means the behaviour was reproduced against
  gsh 1.2, not read out of the source.
- Absolute paths inside sample output come from throwaway test installs and are
  shortened to `~/gsh` and `~/.zshrc` where they would otherwise be unreadable.
  Where that has been done, the page says so.

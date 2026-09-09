# gsh — gitignore.sh

> A bash CLI that reads your repository, works out what it is, and writes the
> `.gitignore` it needs — then tells you which files git is already tracking that
> the `.gitignore` cannot help with.

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

Requires **bash, git and curl**. Nothing else. Version **1.2**.

## Why this exists

Most `.gitignore` tools look at the files in your current directory and stop
there. That finds nothing in a monorepo, where the `package.json` is in `web/`
and the `pyproject.toml` is in `api/`.

`gsh auto` scans the **whole tree** through git's index — 0.4 s on a
50,000-file repository — and applies 59 detection rules covering 44 stacks. It
knows the difference between a project marker and a piece of test data, so a
stray `.tf` under `tests/fixtures/` will not drag in a Terraform template. See
[How `gsh auto` decides](docs/auto.md).

`gsh doctor` covers the other half of the problem. A `.gitignore` does nothing
for files git already tracks — the rule matches, git ignores the rule, and
`node_modules/` stays in every clone. doctor finds those files, and the secrets
that are already committed, and prints the command that fixes it. It exits `1`
on problems, so it works as a CI step or a pre-commit hook.

## Install

There is no Homebrew, apt or pacman package yet. Today the install is a clone:

```bash
git clone https://github.com/MatteoKrsticDev/gsh.git ~/gsh
bash ~/gsh/setup
source ~/.zshrc        # or ~/.bashrc
```

`setup` writes a managed block of `gsh` / `gsh-update` aliases into your shell
rc file (zsh and bash; anything else prints the two lines to add by hand).
Re-running it rewrites that block in place rather than appending a second copy.

**Verified on macOS and Linux. Windows is untested** — gsh should run under Git
Bash or WSL, and it has a `windows` detection branch, but nobody has confirmed
it, so treat it as unsupported until someone does.

Full detail, including `gsh-update` and uninstalling, is in
[Install and update](docs/install.md).

## A short walkthrough

```
$ cd webapp && git init
$ gsh auto -y

Detected
  ✓ node               web/package.json +1 more
  ✓ python             api/pyproject.toml
  ✓ visualstudiocode   .vscode/settings.json
  ✓ macos              you are on macOS

✓ created .gitignore
✓ added node,python,visualstudiocode,macos (165 rules)
```

Later, on a repository somebody else started:

```
$ gsh doctor

gitignore
✓ .gitignore found (79 rules)
  • templates: macos node

Tracked files that should be ignored
✗ 3 tracked files matching .gitignore but still in the index
  • .env
  • node_modules/leftpad/index.js
  • node_modules/leftpad/package.json

  .gitignore does nothing for files git already tracks.
  Remove them from the index (they stay on disk):
    git rm -r --cached <path>

Secrets
✗ 2 sensitive files are COMMITTED to this repo
  • .env
  • deploy.pem

  these are already in git history - rotate the credentials, then:
    git rm --cached <path>


Large files
! 1 file over 5MB not ignored
  • 7MB  assets/demo.mp4

Duplicates
✓ no duplicated rules

✗ 2 problems, 1 warning
```

doctor never changes anything. It prints the command and exits `1`.

Adding something specific, without the detection:

```
$ gsh add rust go
✓ added rust,go (13 rules)
```

Everything gsh writes goes inside markers, so your own lines are never touched:

```
### gsh:start rust,go ###
...
### gsh:end ###
```

## Commands

| Command | What it does |
|---|---|
| `gsh auto` | Detect the stack from the whole tree and write the templates for it |
| `gsh doctor` | Audit the repo; exits `1` on problems |
| `gsh add <tpl...>` | Add named templates in a single request |
| `gsh init` | Create an empty `.gitignore` |
| `gsh list [filter]` | List or search the 571-template catalog |
| `gsh show <tpl...>` | Print a template on stdout without installing it |
| `gsh version` | Print the installed version |
| `gsh help` | Usage |

Short forms: `i`, `a`, `doc`, `l`, `v`, `h`. Flags: `-y`, `--refresh`,
`--no-color`, `--ascii`, `-h`, `-v`.

Colour and spinners appear only when the relevant stream is a terminal, so
`gsh show go > .gitignore` and `gsh list | grep rust` stay clean, and
`gsh doctor > report.txt` produces a plain file.

Every flag, every exit code and real output for each command:
[Command reference](docs/commands.md).

## Documentation

| | |
|---|---|
| [Command reference](docs/commands.md) | Every command and flag, with exit codes and real output |
| [How `gsh auto` decides](docs/auto.md) | The 59 rules, the tree scan, pruning, strong vs weak markers |
| [How `gsh doctor` works](docs/doctor.md) | The five checks, and using it in CI or a pre-commit hook |
| [Templates](docs/templates.md) | The catalog, the cache, offline behaviour, local templates |
| [The block format](docs/block-format.md) | What gsh will and will not touch in a file you edit by hand |
| [Configuration](docs/configuration.md) | Every environment variable |
| [Install and update](docs/install.md) | Installing, `gsh-update`, uninstalling |
| [CONTRIBUTING.md](CONTRIBUTING.md) | Running from a clone, the bash 3.2 constraint, adding a rule |

Templates come from [gitignore.io](https://www.toptal.com/developers/gitignore).

## License

This repository does not currently carry a LICENSE file. Until one is added,
default copyright applies and you do not have permission to redistribute it.

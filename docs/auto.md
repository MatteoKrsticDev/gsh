# How `gsh auto` decides

`gsh auto` answers one question — *what is this repository made of* — and it
answers it from the files, not from the directory you happen to be standing in.
This page is the reference for why something was detected, and why something
else was not.

Short version: gsh lists every path in the repository through git, throws away
paths inside a fixed set of vendor directories, matches the rest against 59
rules covering 44 templates, discards weak (extension-based) matches that live
only under test or fixture directories, and adds your operating system.

## The scan

```
git ls-files -z -c -o --exclude-standard
```

is the whole file listing. Three things follow from that choice:

- **The whole tree, not the current directory.** `web/package.json` and
  `services/api/pyproject.toml` count exactly as much as a manifest in the root.
  A glob in the rule table is matched against the *tail* of every path, so
  `package.json` matches `package.json` and `frontend/package.json` alike. This
  is the reason the scan exists: looking only at the front door of a monorepo
  finds nothing.
- **Committed *and* not-yet-committed files** (`-c -o`), so `gsh auto` works on a
  repository whose first commit has not happened yet.
- **Nothing you already ignore** (`--exclude-standard`). If a path is covered by
  an existing `.gitignore`, git does not list it and gsh does not see it.

Cost, measured on a 50,002-file repository (Apple silicon, warm cache):

```
$ time gsh auto        # answering n
0.41s user 0.10s system 106% cpu 0.474 total
```

That is three processes in total — the listing, the prune, and a `grep -F`
sieve that reduces 50,000 paths to the handful bash actually loops over.

## Pruned directories

Paths matching this pattern are dropped before any rule is applied:

```
(^|/)(\.git|node_modules|vendor|target|dist|build|\.venv|venv|__pycache__|Pods|\.next|\.nuxt|bower_components)/
```

This is not a speed optimisation, it is correctness. Right after `git init`
nothing is ignored yet, so `git ls-files -o` happily lists all 40,000 files
under `node_modules/` — including several hundred `package.json` files and
whatever `Cargo.toml` or `pyproject.toml` a dependency ships. Without the prune,
gsh would detect *somebody else's* dependency tree as if it were your project.

The prune matches directory components anywhere in the path, so
`packages/ui/node_modules/react/package.json` is dropped too.

A consequence worth knowing: the prune list is fixed. A vendored tree under a
name gsh does not know (`third_party/`, `deps/`) is still scanned, and its
manifests can produce detections. If that happens, the fix is either to ignore
that directory first (git will then hide it from gsh) or to add the templates
you want by hand with `gsh add`.

## Strong and weak markers

Every rule is tagged `strong` or `weak`, and the tag is the whole noise policy.

**strong** — a project manifest. Somebody wrote `Cargo.toml` on purpose. It is
a Rust crate wherever it sits, including under `tests/`.

**weak** — a bare file extension. `*.tf` says as much about somebody's test data
as it does about the project. A weak marker found **only** under a test or
fixture directory is discarded.

The directory names that count as test material:

```
test tests spec specs fixture fixtures testdata
example examples sample samples demo demos doc docs
```

Only *directory* components are checked; the file's own name is not. So
`infra/test.tf` is a Terraform file that happens to be called `test`, and
`tests/x.tf` is test data.

Verified, in the same repository, with one file moved:

```
$ ls tests/fixtures/main.tf
$ gsh auto

Detected
  ✓ node               frontend/package.json +1 more
  ✓ python             api/pyproject.toml
  ✓ visualstudiocode   .vscode/settings.json
  ✓ macos              you are on macOS
```

```
$ cp tests/fixtures/main.tf infra/main.tf
$ gsh auto

Detected
  ✓ node               frontend/package.json +1 more
  ✓ python             api/pyproject.toml
  ✓ terraform          infra/main.tf
  ✓ visualstudiocode   .vscode/settings.json
  ✓ macos              you are on macOS
```

Terraform appears only when a `.tf` exists somewhere that is not test material.
"Discarded" means discarded per-path: one weak match outside a fixture directory
is enough, however many fixture copies exist.

Note that `doc` and `docs` are on the list, which catches a real case —
`docs/paper.tex` is documentation, not a LaTeX project — and one that surprises
people: a genuine LaTeX paper living in `docs/` is not detected. Move it, or run
`gsh add latex`.

The eight weak rules are marked in the table below. Everything else is strong.

## The rule table

59 rules, 44 templates. Read in source order — the order also decides how the
`Detected` list is grouped, so a polyglot repository reads top-down like a stack
list.

| Marker | Template(s) | |
|---|---|---|
| `package.json` | node | |
| `package-lock.json` | node | |
| `yarn.lock` | node | |
| `pnpm-lock.yaml` | node | |
| `next.config.*` | nextjs | |
| `nuxt.config.*` | nuxtjs | |
| `angular.json` | angular | |
| `svelte.config.*` | svelte | |
| `astro.config.*` | astro | |
| `react-native.config.*` | reactnative | |
| `metro.config.*` | reactnative | |
| `deno.json` | deno | |
| `deno.jsonc` | deno | |
| `deno.lock` | deno | |
| `pyproject.toml` | python | |
| `requirements.txt` | python | |
| `setup.py` | python | |
| `Pipfile` | python | |
| `manage.py` | django | |
| `*.ipynb` | jupyternotebooks | **weak** |
| `Cargo.toml` | rust | |
| `go.mod` | go | |
| `pom.xml` | maven, java | |
| `build.gradle` | gradle, java | |
| `build.gradle.kts` | gradle, java, kotlin | |
| `settings.gradle.kts` | kotlin | |
| `build.sbt` | scala, sbt | |
| `mix.exs` | elixir | |
| `*_web/router.ex` | phoenix | |
| `stack.yaml` | haskell | |
| `*.cabal` | haskell | |
| `Manifest.toml` | julia | |
| `Project.toml` | julia | **weak** |
| `*.Rproj` | r | |
| `renv.lock` | r | |
| `*.nimble` | nim | |
| `build.zig` | zig | |
| `Gemfile` | ruby | |
| `config/application.rb` | rails | |
| `composer.json` | composer | |
| `artisan` | laravel | |
| `pubspec.yaml` | dart, **or** flutter — see below | |
| `CMakeLists.txt` | cmake | |
| `Package.swift` | swift | |
| `Podfile` | cocoapods | |
| `project.pbxproj` | xcode | |
| `*.sln` | visualstudio | **weak** |
| `*.csproj` | visualstudio | **weak** |
| `ProjectSettings/ProjectVersion.txt` | unity | |
| `project.godot` | godot | |
| `*.tf` | terraform | **weak** |
| `ansible.cfg` | ansible | |
| `playbook.yml` | ansible | **weak** |
| `Chart.yaml` | helm | |
| `*.tex` | latex | **weak** |
| `.vscode/settings.json` | visualstudiocode | |
| `.vscode/launch.json` | visualstudiocode | |
| `.idea/workspace.xml` | jetbrains | |
| `*.iml` | jetbrains | **weak** |

A rule may list several templates; `pom.xml` adds both `maven` and `java`.

Every glob has at most one `*`, at the start or the end. That is a hard
constraint, not a coincidence — see [CONTRIBUTING.md](../CONTRIBUTING.md) if you
are adding a rule.

### `pubspec.yaml`: dart or flutter

`pubspec.yaml` is Dart's manifest and Flutter's. Only one of them depends on the
Flutter SDK, and the file says which: if it contains a `flutter:` key or
`sdk: flutter`, gsh reports `flutter`; otherwise `dart`.

```
$ cat pubspec.yaml
name: myapp
dependencies:
  flutter:
    sdk: flutter

$ gsh auto
Detected
  ✓ flutter            pubspec.yaml
```

```
$ cat pubspec.yaml
name: mycli
dependencies:
  args: ^2.0.0

$ gsh auto
Detected
  ✓ dart               pubspec.yaml
```

## Directories that git cannot show

An ignored directory is invisible to `git ls-files` by design — which is exactly
the problem, because a repository that already ignores `node_modules/` gives the
scan nothing to find. Five directories are therefore checked directly, in the
working directory root:

| Directory | Template |
|---|---|
| `node_modules/` | node |
| `.venv/` | python |
| `venv/` | python |
| `.vscode/` | visualstudiocode |
| `.idea/` | jetbrains |

These never override a real path found by the scan; they only fill a gap.

```
$ ls -a
.  ..  .git  .idea  .venv  docs  node_modules

$ gsh auto

Detected
  ✓ node               node_modules/
  ✓ python             .venv/
  ✓ jetbrains          .idea/
  ✓ macos              you are on macOS
```

## Your operating system

One template is always added, from `uname -s`:

| `uname -s` | Template |
|---|---|
| `Darwin` | macos |
| `Linux` | linux |
| `MINGW*`, `MSYS*`, `CYGWIN*` | windows |

It is reported as `you are on macOS` rather than as a path, because there is no
path to point at.

This is why "nothing detected" is effectively unreachable on a supported
platform: even an empty repository detects your OS. The `no known stack detected`
message and its exit `1` only appear where `uname -s` returns something else.

## Which path gets shown

Several files often match the same template. gsh reports one line per template
and picks the path to show like this:

1. **Shallowest wins.** `frontend/package.json` explains a monorepo better than
   `frontend/packages/ui/vendored/package.json`.
2. **Ties go to the earlier rule.** `node` reports `package.json`, not the
   `package-lock.json` sitting beside it.
3. **Then first in sort order** — not "whichever git listed first", because git
   lists untracked files before tracked ones and the line you see would
   otherwise depend on what happens to be committed.

`+N more` on the end counts the other matches.

Paths longer than 60 characters are shown as `...` plus the last 57, and tabs,
newlines and carriage returns inside a path are replaced with `?`.

## Already-installed templates

Before writing, gsh reads back which templates are already in `.gitignore` (see
[the block format](block-format.md)) and marks them:

```
  • node               web/package.json +1 more - already present
```

It only recognises templates *it* installed, via the block markers. If you
pasted the Node rules in by hand, gsh does not know that and will add its own
copy. `gsh doctor` will then report the duplicated rules.

## When a detection is wrong

- **Something was detected that is not your project.** It came from a vendored
  or generated tree gsh does not prune. Ignore that directory first, then re-run
  — git will hide it from the next scan.
- **Something was not detected.** Either the marker file is inside an ignored
  directory (git never showed it to gsh), or the match was weak and lived only
  under a test/fixture/doc directory, or there is no rule for it. `gsh add
  <template>` covers all three, and a missing rule is worth
  [contributing](../CONTRIBUTING.md).
- **A detection names a template the catalog does not have.** gsh says so rather
  than dropping it silently:
  `! skipping 'foo' (path): no such template available`. That is a bug in the
  rule table, not in your repository — please report it.

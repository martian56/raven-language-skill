# rvpm and the `raven` CLI (3.0)

## Project layout

```text
app/                    rvpm new app
├── rv.toml
├── src/main.rv         fun main() lives here
└── src/*_test.rv       tests (any *_test.rv in the package)

mylib/                  rvpm new mylib --lib
├── rv.toml
└── lib.rv              what other packages import
```

## rv.toml

```toml
[package]
name = "app"
version = "0.1.0"
edition = "v3"

[dependencies]
"github.com/martian56/raven-csv" = "v0.2.0"

# [fmt]
# indent_width = 4
# wrap_width = 100
```

`edition = "v3"` is the default and the only accepted edition. A package
with `edition = "v2"` is Raven 2 syntax and is rejected with a message
saying so.

## rvpm commands

| Command | What it does |
|---------|--------------|
| `rvpm new <name> [--lib]` / `rvpm init [--lib]` | scaffold a package |
| `rvpm build` | compile `src/main.rv` to `target/raven-out/<name>`, or type check a library |
| `rvpm run [args]` | build, then run |
| `rvpm test [NAME]` | run `fun test_*()` in `*_test.rv`; with `NAME`, only tests whose name contains it |
| `rvpm fmt [paths] [--check]` | format sources in place |
| `rvpm add github.com/user/repo@v1.2.0` | add a dependency and lock it |
| `rvpm install` / `rvpm update [pkg]` | fill the cache from `rv.lock` / re-resolve |
| `rvpm doc` | Markdown API docs into `target/doc` |
| `rvpm dist` | package the app (tar, zip, deb, rpm, msi) |
| `rvpm fetch`, `rvpm lock`, `rvpm cache dir/list/clean`, `rvpm workspace` | plumbing |

Each test runs in its own process, so one failing assertion does not stop
the others.

## raven commands

| Command | What it does |
|---------|--------------|
| `raven run file.rv [args]` | compile and run one file |
| `raven build file.rv -o out` | compile to a native binary |
| `raven check file.rv` | type check only |
| `raven fmt file.rv [--write]` | print formatted source, or rewrite in place |
| `raven --version` | print the version |

`RAVEN_TIMINGS=1` prints how long each compiler phase took.

## Imports and module resolution

- `import std.fs` / `from std.fs import read`: a bundled stdlib module.
- `from "./models" import User`: `models.rv` next to the importing file;
  `"../shared/util"` works too.
- `from "github.com/user/repo" import Thing`: the package's `lib.rv`;
  `"github.com/user/repo/sub"` loads `sub.rv` inside it. The package must be
  in `[dependencies]`.
- Every top-level item is importable from another file; `pub` is accepted
  on declarations but not enforced yet.

## Common workflows

```bash
rvpm new app && cd app
rvpm run
rvpm add github.com/martian56/raven-csv@v0.2.0
rvpm test
rvpm fmt
```

Publishing a library is pushing a git tag: dependents name the repository
and the tag in `[dependencies]`.

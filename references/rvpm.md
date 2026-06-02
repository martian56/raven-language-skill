# rvpm and the `raven` CLI (v2)

`raven` is the compiler and build driver. `rvpm` is the package manager and project runner. Both ship in the v2 release and should be on `PATH`. A C linker must also be available (`link.exe` on Windows via the MSVC build tools, `cc`/`clang` on Unix) so the compiler can link the runtime into your program.

## Project layout

```
my_project/
├── rv.toml          # manifest
├── rv.lock          # resolved dependency versions + tree hashes (commit this)
└── src/
    └── main.rv      # entry point: defines fun main()
```

A library package that is meant to be imported uses a `lib.rv` at the repo root instead of `src/main.rv` (a bare `import "github.com/user/repo"` loads `lib.rv`; `import "github.com/user/repo/sub"` loads `sub.rv`).

## `rv.toml`

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "v2"

[dependencies]
"github.com/user/raven-json" = "1.0"

# [ffi]                 # optional native link pass-through
# libs = ["sqlite3"]    # (parsed; linking of external libs is still limited)

# [fmt]
# indent_width = 4
# wrap_width = 100
```

- `[package]` — `name`, `version`, `edition` (`v2`).
- `[dependencies]` — GitHub-direct packages keyed by `github.com/<user>/<repo>`, value is a version constraint (a git tag like `1.0`/`v1.0.0`).
- `[ffi]` — optional `libs` / `link_args` for native linking (schema exists; full wiring is limited in this release).
- `[fmt]` — `indent_width` (default 4), `wrap_width` (default 100).

## `rvpm` subcommands

| Command                         | What it does                                                        |
| ------------------------------- | ------------------------------------------------------------------- |
| `rvpm init [name]`              | Scaffold `rv.toml` + `src/main.rv`.                                  |
| `rvpm add <pkg>[@version]`      | Add a GitHub dependency, resolve it, write `rv.lock`, fill the cache.|
| `rvpm install`                  | Resolve `rv.toml` against `rv.lock` and populate the cache.          |
| `rvpm update [pkg]`             | Re-resolve and rewrite `rv.lock` for one package or all.             |
| `rvpm build`                    | Compile `src/main.rv` to `target/raven-out/<name>`.                  |
| `rvpm run [args]`               | Build then run, forwarding `args`.                                   |
| `rvpm fmt [paths]`              | Format in place (defaults to `src/`; pass paths for a library). `--check` for CI. |
| `rvpm fetch <pkg>`              | Fetch `github.com/<user>/<repo>@<version>` into the shared cache.    |
| `rvpm lock`                     | Generate or validate `rv.lock`.                                      |

Package arguments use the `github.com/<user>/<repo>` form; append `@<tag>` to pin (for example `rvpm add github.com/user/raven-uuid@v0.1.0`).

## The `raven` CLI

```bash
raven build file.rv -o out    # compile a single .rv file to a native binary
raven build file.rv           # compile to a default output path
raven --version
raven --help
```

There is no `raven file.rv` direct-run, no `-c` type-check-only flag, and no REPL in v2. Compiling is the check: if `raven build` succeeds, the program type-checked, lowered, and linked. Run the produced binary directly (`./out`).

## Module resolution

`import` targets resolve by form:

1. `import std/<module>` — the bundled standard library, compiled into the `raven` binary (`include_str!`), always available.
2. `import "./rel"` or `import "../rel"` — a sibling/relative `.rv` file resolved against the importing file.
3. `import "github.com/<user>/<repo>[/<sub>]"` — a package from the rvpm cache, fetched via `git` into `~/.rvpm/cache/...` and pinned by `rv.lock`. A bare path loads `lib.rv`; a subpath loads `<sub>.rv`.

A consumed dependency may itself import std modules and use their types, methods, and (as of 2.0.2) free functions.

## Common workflows

### Start a project

```bash
rvpm init my_thing
cd my_thing
rvpm run
```

### Add and use a GitHub package

```bash
rvpm add github.com/martian56/raven-uuid@v0.1.0
```

```raven
import "github.com/martian56/raven-uuid" { Uuid }

fun main() {
    let id = Uuid.v4()
    print("${id.to_string()}")
}
```

### Add a local module

Drop `src/util.rv` next to `main.rv`, then:

```raven
import "./util" { helper }
```

### Build, run, format

```bash
raven build src/main.rv -o app && ./app
rvpm run
rvpm fmt
rvpm fmt --check        # CI
```

### Publish a package

Push the repo (with `rv.toml` and a root `lib.rv`) to GitHub and tag a semver release (`git tag v0.1.0 && git push --tags`). Pushing the tag is the publish; consumers then `rvpm add github.com/<user>/<repo>@v0.1.0`.

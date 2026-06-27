---
name: raven-language-skill
description: Reference and patterns for writing Raven programming language source (.rv files). Use this whenever the user is writing, editing, debugging, or reviewing Raven code, working in an rvpm project (rv.toml), discussing Raven syntax, importing from Raven's stdlib (std/io, std/string, std/fmt, std/collections, std/math, std/iter, std/fs, std/time, std/json, std/net, std/http, std/tls, std/ffi, std/sync, std/cmp, std/random, std/env, std/encoding, std/hash, std/path, std/process, std/regex), or running raven/rvpm commands. Raven v2 is a statically-typed compiled language (Cranelift backend, tracing GC) with generics, traits, sum types, pattern matching, concurrency, a C FFI, and metaprogramming. It has a few syntax pitfalls (no semicolons, PascalCase types, string-method name is `.length()` not `.len()`, enum construction is qualified) that this skill helps Claude avoid. Trigger even when the user just mentions a `.rv` file or rvpm. Do NOT trigger for the Raven compiler source itself (the `.rs` files) or unrelated languages.
metadata:
  author: martian56
  version: 2.3.1
---

# Raven Language (v2)

Raven v2 is a statically-typed, ahead-of-time compiled language implemented in Rust. Source lowers through a resolver, type checker, HIR, and a monomorphizing MIR, then a Cranelift backend emits a native binary. Memory is managed by a tracing garbage collector. The syntax reads like a blend of Rust and Swift, with local type inference, no semicolons, traits, generics, sum types, and pattern matching. The single most useful thing this skill does is steer you away from the non-obvious pitfalls so the first version of your code parses, type-checks, and links.

> v2 is a clean break from v1. If you find old material mentioning `elseif`, C-style `for` loops, `int`/`string` lowercase types, `format("{}", x)`, `import math;`, or `main();` at the bottom of a file, that is v1 and does not apply here.

## When to read what

- **This file**: read every time. The pitfalls and templates cover most tasks.
- **`references/builtins.md`**: the always-available functions and the methods on `String`, `List`, `Map`, `Set`.
- **`references/stdlib.md`**: per-module API for `std/io`, `std/string`, `std/collections`, `std/math`, `std/iter`, `std/fs`, `std/time`, `std/json`, and more.
- **`references/rvpm.md`**: starting a project, `rv.toml`, GitHub-direct packages, the `raven` and `rvpm` CLIs.
- **`references/grammar.md`**: exact lexical and grammar rules when something parses surprisingly.

The stdlib source lives in `stdlib/std/*.rv` in the Raven repo and is written in Raven itself, so reading it is the authoritative answer for any signature.

## Pitfalls to internalise

These are the mistakes that ruin first-try compilation. Burn them in.

1. **No semicolons.** Statements end at the newline. Do not terminate lines with `;`.
2. **Types are PascalCase.** `Int`, `Float`, `Bool`, `String`, `Char`, `Unit`. There is no `int`/`string`. `Unit` is the no-value type (the implicit return).
3. **Type annotations are optional.** Local inference works: `let x = 5` infers `Int`. Annotate when you want to be explicit or when inference can't tell (`let xs: List<Int> = []`). Function parameters and return types are still written out.
4. **`let` is mutable; there is no `let mut`.** Reassign freely. Compound assignment works: `+= -= *= /= %=`. `const` is for immutable compile-time constants and works both as a local (`const X = 5`) and at module level (`const MAX: Int = 100`).
5. **Logical operators are `&&`, `||`, `!`.** The words `and`/`or`/`not` are NOT operators.
6. **`else if` is two words** (v1's `elseif` is gone). `if` is also an expression: `let s = if x > 0 { "pos" } else { "neg" }`.
7. **`for` is range/iterator based.** `for i in 0..10 { }` (exclusive), `for i in 1..=10 { }` (inclusive), `for item in list { }`. `while cond { }` and `loop { }` exist. `break` and `continue` work.
8. **String interpolation is `"${expr}"`.** The expression can be arbitrary: a nested string literal (`"${a.concat("y")}"`), a macro call (`"${square!(5)}"`), and a struct value that derives `ToString` (`"${p}"`) all interpolate directly.
9. **`String` length is `.length()`, not `.len()`.** `.len()` type-checks but fails in codegen. `List`/`Map`/`Set` use `.len()`. String methods (`.to_upper()`, `.trim()`, `.split()`, `.concat()`, …) require `import std/string`.
10. **No `null`.** Absence is `Option<T>` (sugar `T?`) with `Some(x)`/`None`. Fallible results are `Result<T, E>` with `Ok(x)`/`Err(e)` and the `?` operator. `Some`, `None`, `Ok`, `Err` are in scope without imports.
11. **Enum variants are constructed qualified:** `Shape.Circle(2.0)`, `Color.Red`. In `match`, the patterns are bare: `Circle(r) -> ...`. `match` is exhaustive.
12. **No visibility modifiers.** Every top-level `fun`/`struct`/`enum`/`trait` is importable. There is no `export` and no `pub`.
13. **`fun main()` is the entry point.** Define it; do NOT call `main()` yourself. There is no top-level statement execution.
14. **Prefer selector imports for free functions.** `import std/fs { write }` then `write(path, data)` is the form that always works. Module-qualified access (`import std/fs` then `fs.exists(...)`) works for some runtime-backed modules but not all (`import std/math` then `math.sqrt(...)` fails), so reach for the selector form. Types (`Map`) and methods (`.to_upper`) come in through `import std/collections { Map }` / `import std/string`.
15. **`match` on `String` literal patterns works.** `match s { "a" -> ..., _ -> ... }` matches as expected, and strings support `<`/`>` ordering.
16. **Reading module-level bindings works; mutating them does not.** A top-level `let`/`const` is readable from any function (`let greeting: String = "hi"`), but reassigning a module-level binding from a function mis-compiles (`binop lhs used a Unit value`). For mutable state, keep it inside functions, thread it through parameters, or hold it in a struct you pass around.

## Anatomy of a Raven file

```raven
// imports: std modules by name, locals and packages by quoted path
import std/io { println }
import std/collections { Map }
import "./board"                              // local ./board.rv
import "github.com/user/raven-json" { parse } // GitHub-direct package

// a trait
trait Display {
    fun show(self) -> String
}

// a struct + an impl of the trait + inherent methods
struct Point {
    x: Float,
    y: Float,
}

impl Display for Point {
    fun show(self) -> String = "(${self.x}, ${self.y})"
}

impl Point {
    fun magnitude(self) -> Float = sqrt(self.x * self.x + self.y * self.y)
}

import std/math { sqrt }

// a free function (single-expression body)
fun midpoint(a: Point, b: Point) -> Point =
    Point { x: (a.x + b.x) / 2.0, y: (a.y + b.y) / 2.0 }

// entry point: defined, never called by you
fun main() {
    let p = Point { x: 3.0, y: 4.0 }
    println("mag = ${p.magnitude()}")
    println(p.show())
}
```

Implicit rules:

- **Imports**: `import std/<module> { names }` for the bundled stdlib; `import "./rel"` for a sibling file; `import "github.com/user/repo" { names }` for a package fetched by rvpm. Selectors bring the names into scope unqualified.
- **No entry call**: the runtime calls `main` for you.
- **Single-expression bodies**: `fun f(...) -> T = <expr>` is shorthand for `{ return <expr> }`.
- **Trailing commas** are allowed in struct definitions/literals and collection literals.

## Templates

Working code you can adapt. These compile.

### Variables, control flow

```raven
fun main() {
    let total = 0
    for i in 1..=10 {
        if i % 2 == 0 {
            total += i
        }
    }
    let label = if total > 20 { "big" } else { "small" }
    print("${total} is ${label}")
}
```

### Collections and iterators

```raven
import std/io { println }
import std/collections { Map }
import std/iter { collect, fold }

fun main() {
    let nums = [1, 2, 3, 4, 5, 6]
    println("len = ${nums.len()}, first = ${nums[0]}")

    // lazy pipeline: consume with collect / fold / count
    let doubledEvens: List<Int> =
        collect(nums.iter().filter(fun(x: Int) -> Bool = x % 2 == 0).map(fun(x: Int) -> Int = x * 2))
    println("count = ${doubledEvens.len()}")

    let sum = fold(nums.iter(), 0, fun(acc: Int, v: Int) -> Int = acc + v)
    println("sum = ${sum}")

    let tally: Map<String, Int> = Map.new()
    tally.set("a", 1)
    match tally.get("a") {
        Some(n) -> println("a = ${n}"),
        None -> println("missing"),
    }
}
```

### Struct, trait, dynamic dispatch

```raven
trait Shape {
    fun area(self) -> Float
}

struct Circle { r: Float }
struct Square { side: Float }

impl Shape for Circle {
    fun area(self) -> Float = 3.14159 * self.r * self.r
}
impl Shape for Square {
    fun area(self) -> Float = self.side * self.side
}

fun describe(s: dyn Shape) {
    print("area = ${s.area()}")
}

fun main() {
    describe(Circle { r: 2.0 })
    describe(Square { side: 3.0 })
}
```

Note: `List<dyn Trait>` (a heterogeneous list of trait objects) is not supported in this release. Pass trait objects to `dyn`-typed parameters or assign to a `dyn`-typed local.

### Enums, match, Option, Result, ?

```raven
enum Shape {
    Circle(Float)
    Rectangle(Float, Float)
}

fun area(s: Shape) -> Float =
    match s {
        Circle(r) -> 3.14159 * r * r,
        Rectangle(w, h) -> w * h,
    }

fun checked_div(a: Int, b: Int) -> Result<Int, String> {
    if b == 0 {
        return Err("divide by zero")
    }
    return Ok(a / b)
}

fun halve_then_div(a: Int, b: Int) -> Result<Int, String> {
    let q = checked_div(a, b)?    // ? propagates Err, unwraps Ok
    return Ok(q / 2)
}

fun main() {
    print("${area(Shape.Circle(2.0))}")
    match halve_then_div(20, 5) {
        Ok(v) -> print("ok ${v}"),
        Err(e) -> print("err ${e}"),
    }
}
```

### Generics with a trait bound

```raven
trait Describe {
    fun describe(self) -> String
}

struct Dog {}
impl Describe for Dog {
    fun describe(self) -> String = "a dog"
}

fun announce<T: Describe>(x: T) {
    print("this is ${x.describe()}")
}

fun main() {
    announce(Dog {})
}
```

### defer (LIFO, runs at function exit)

```raven
fun main() {
    let log = [1]
    defer log.push(3)    // runs second
    defer log.push(2)    // runs first
    print("len now ${log.len()}")
}
```

### Concurrency: spawn + channels

```raven
import std/sync { channel, channel_buffered, yield_now }

fun main() {
    let ch = channel()
    spawn(fun() -> Unit {
        let i = 1
        while i <= 5 {
            ch.send(i)
            i = i + 1
        }
    })
    let sum = 0
    let n = 0
    while n < 5 {
        sum += ch.recv()
        n += 1
    }
    print("sum = ${sum}")   // 15
}
```

Goroutines run in **parallel** on a pool of worker threads (one per core), so `spawn` is true parallelism, not cooperative time-slicing. A goroutine only ever suspends at a blocking point (a full/empty channel, `yield_now`, `sleep_millis`), and the scheduler may resume it on a different worker than it ran on before. Channels carry `Int` and block when full (send) or empty (recv).

`std/sync` also has a `Mutex`, a `WaitGroup`, and `select`:

```raven
import std/sync { mutex, wait_group, channel, select_recv }

fun main() {
    // Mutex: lock() blocks until free, unlock() releases (call only while held).
    let m = mutex()
    m.lock()
    m.unlock()

    // WaitGroup: add() before spawning, done() as each finishes, wait() blocks to zero.
    let wg = wait_group()
    wg.add(1)
    spawn(fun() -> Unit {
        wg.done()
    })
    wg.wait()

    // select_recv: block on several channels, lowest ready index wins. The
    // SelectResult is { index, value } (index -1 if the list was empty).
    let a = channel()
    let b = channel()
    spawn(fun() -> Unit { a.send(7) })
    let r = select_recv([a, b])
    print("chan ${r.index} -> ${r.value}")
}
```

Channels, wait groups, and select sets hold runtime registry entries with no destructor; call `.free()` (or `select_recv` frees its own set) when you are done with one to avoid leaking the entry.

### Metaprogramming: derive + macros + reflection

```raven
@derive(Eq, Hash, ToString, Debug)
struct Point { x: Int, y: Int }

macro square { ($x:expr) => { ($x) * ($x) } }

fun main() {
    let p = Point { x: 1, y: 2 }
    print(p.to_string())          // Point { x: 1, y: 2 }
    print("${square!(5)} fields=${field_names<Point>().len()}")
}
```

Derivable traits: `Eq`, `Hash`, `ToString`, `Debug`, and `Ord`. `@derive(Ord)` adds `compare(self, other) -> Int` (negative / zero / positive), comparing structs field-by-field in declaration order and enums by variant order then payload. Pair it with `import std/cmp { sort }` to sort a `List`:

```raven
import std/cmp { sort }

@derive(Ord)
struct Version { major: Int, minor: Int }

fun main() {
    let vs = sort([Version { major: 1, minor: 2 }, Version { major: 1, minor: 0 }])
    print("${vs[0].minor}")   // 0
}
```

### C FFI

```raven
import std/ffi { alloc, free, load, store }

extern "C" {
    fun abs(x: CInt) -> CInt
    fun strlen(s: CStr) -> CSize
}

fun main() {
    print("${abs(-7)}")
    let len = strlen(c"hello")     // c"..." is a C string literal
    print("${len}")
    let buf = alloc<CInt>(2)
    store<CInt>(buf, 42)
    print("${load<CInt>(buf)}")
    free<CInt>(buf)
}
```

## Built-in cheatsheet

Always available, no import. See `references/builtins.md` for details.

| Need                        | Use                                                  |
| --------------------------- | ---------------------------------------------------- |
| Print a line                | `print("...")` (interpolate with `${expr}`)          |
| Print via stdlib            | `println(...)` after `import std/io { println }`     |
| List length / element       | `xs.len()`, `xs[i]`, `xs.push(v)`                    |
| Iterator from a list        | `xs.iter()` then `.map`/`.filter`, consumed by `std/iter` |
| Option / Result values      | `Some(x)`, `None`, `Ok(x)`, `Err(e)`                 |
| Goroutine                   | `spawn(fun() -> Unit { ... })`                       |
| Compile-time reflection     | `type_name<T>()`, `field_names<T>()`                 |
| Raw memory byte             | `__str_byte_at(s, i)` (low-level; prefer `std/string`) |

## Stdlib at a glance

Bundled into the compiler; import with `import std/<module> { names }`. See `references/stdlib.md`.

| Module            | What's there                                                            |
| ----------------- | ----------------------------------------------------------------------- |
| `std/io`          | `print`, `println`                                                      |
| `std/string`      | merges `String` methods: `to_upper`, `to_lower`, `trim`, `split`, `concat`, `contains`, `replace`, `substring`, `length`, … |
| `std/fmt`         | string formatting: `pad_left`, `pad_right`, `center`, `repeat`, `join`, `to_hex`/`to_binary`/`to_octal`/`to_radix`, `from_hex`/`from_radix`, `format_float`, `pad_int` |
| `std/collections` | `Map<K, V>`, `Set<T>` with constructors and methods                     |
| `std/iter`        | `collect`, `fold`, `count` over `.iter().map(...).filter(...)` pipelines |
| `std/math`        | `sqrt`, `pow`, `pow_int`, `abs`, `min`, `max`, `pi`, `e`, trig, …       |
| `std/fs`          | `read`, `write`, `append`, `exists`, `remove_file`, `list_dir`, `split_lines` (all return `Result` where fallible) |
| `std/time`        | `now`, `now_millis`, `format_timestamp`, `parse_timestamp`              |
| `std/json`        | `JsonValue` enum, `parse`, `stringify`                                  |
| `std/sync`        | `channel`, `channel_buffered`, `send`/`recv`, `yield_now`, `sleep_millis`, `Mutex` (`mutex`, `lock`/`unlock`), `WaitGroup` (`wait_group`, `add`/`done`/`wait`), `select_recv` |
| `std/ffi`         | `alloc`, `free`, `load`, `store`, `offset`, `is_null`, `null_ptr`       |
| `std/random`      | `Rng` (`new`, `from_entropy`, `next_int`, `gen_range`)                  |
| `std/cmp`         | `min`, `max`, `clamp`, `sort`, `sorted_by` (pairs with `@derive(Ord)`)  |
| `std/http`        | `get`, `post`, `put`, `delete`, `patch`, `request`; `serve_connection` for a basic server |
| `std/net`         | `connect`, `listen`, `dns_lookup`, `reachable`                          |
| `std/tls`         | client TLS: `connect(addr, server_name)` (verified), `connect_with` + `config()` builder (`add_ca_file`, `client_cert`, `insecure_skip_verify`), `upgrade(tcp_stream, server_name)` for STARTTLS (Postgres/MySQL); `TlsStream` read/write/close. `std/http` already does `https://` |
| `std/encoding`    | `hex_encode`, `hex_decode`, `url_encode`, base64 helpers                |
| `std/env`         | `get_env`, `has_env`, `get_env_or`, `args`, `arg_count`, `arg_at`, `exit`, `os_name`, `arch` |
| `std/hash`        | `fnv`, `djb`, `crc`, `checksum`, `combine`                              |
| `std/path`        | `join`, `basename`, `dirname`, `extension`, `stem`, `normalize`, `is_absolute` |
| `std/process`     | `run`, `run_with_input`                                                 |
| `std/regex`       | `compile` and match against compiled patterns                          |
| `std/test`        | assertions for `rvpm test`: `assert`, `assert_msg`, `assert_true`/`assert_false`, `assert_eq`/`assert_ne` (generic), `assert_eq_int`/`assert_eq_str`/`assert_eq_float`, `assert_some`/`assert_none`/`assert_ok` |

If a name isn't here, read `stdlib/std/<module>.rv` in the Raven repo. Note: a dependency package can also import std free functions (fixed in 2.0.2).

## Running and project layout

```
my_project/
├── rv.toml          # [package] / [dependencies] / optional [ffi], [fmt]
└── src/
    └── main.rv      # entry point — defines fun main(), not called manually
```

```bash
rvpm init my_project        # scaffold in the current dir (--lib for a library, lib.rv)
rvpm new my_project         # scaffold in a fresh my_project/ dir (--lib)
rvpm run                    # build src/main.rv and run it (forwards args: rvpm run -- a b)
rvpm build                  # compile to target/raven-out/<name>, or type-check a lib.rv
rvpm test                   # run fun test_*() tests in *_test.rv files (assert with import std/test)
rvpm doc                    # generate Markdown API docs into target/doc
rvpm fmt                    # format .rv files in place (pass paths for a library: rvpm fmt lib.rv)
rvpm fmt --check            # CI: non-zero if anything would change

rvpm add github.com/user/repo@v1.0.0   # add a GitHub-direct dependency, write rv.lock
rvpm install                # resolve rv.toml against rv.lock and fill the cache
rvpm update [pkg]           # re-resolve and rewrite rv.lock for one package or all
rvpm fetch github.com/user/repo@v1.0.0 # fetch a package into the shared cache
rvpm lock                   # generate or validate rv.lock
rvpm cache list             # inspect the shared cache (also dir / clean)
rvpm version                # print the rvpm version (also --version, -V)

raven build file.rv -o out  # compile a single file to a native binary
./out                       # run it
```

There is no `raven file.rv` direct-run, no `-c` type-check flag, and no REPL in v2; compiling is the check. A C linker must be on PATH (`link.exe` on Windows, `cc`/`clang` on Unix) so the compiler can link the runtime into your program.

## Recommended workflow

1. Sketch types first: structs, enums, traits.
2. Write small free functions; group methods in `impl` blocks.
3. Write `fun main()` (do not call it).
4. `raven build src/main.rv -o app` (or `rvpm run`). Compile early and often; the type checker catches the bulk of mistakes.
5. If a parse error appears, the usual cause is a pitfall above: a stray `;`, lowercase type name, `.len()` on a `String`, or an unqualified enum constructor.
6. `rvpm fmt` before committing.

## Style

- 4-space indent, no semicolons.
- Order: imports → traits → structs/enums → impls → free functions → `fun main`.
- `snake_case` for functions and variables, `PascalCase` for types and enum variants.
- Prefer single-expression bodies (`fun f() -> T = expr`) for one-liners.
- Use string interpolation (`"${x}"`) instead of manual concatenation.
- Keep `match` arms exhaustive; add a `_ -> ...` arm only when a catch-all is truly intended.

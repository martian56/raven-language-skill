---
name: raven-language-skill
description: Reference and patterns for writing Raven programming language source (.rv files). Use this whenever the user is writing, editing, debugging, or reviewing Raven code, working in an rvpm project (rv.toml), discussing Raven syntax, importing from Raven's stdlib (std.io, std.string, std.fmt, std.collections, std.math, std.iter, std.fs, std.time, std.json, std.net, std.http, std.tls, std.ffi, std.sync, std.cmp, std.random, std.env, std.encoding, std.hash, std.path, std.process, std.regex, std.test), or running raven/rvpm commands. Raven 3 reads like Python with braces and compiles to native code (Cranelift backend, tracing GC), with classes, traits, generics, sum types, pattern matching, concurrency, a C FFI, and metaprogramming. This skill steers Claude away from the syntax traps (lowercase types, `let` is immutable, `case` arms, square-bracket generics, `from x import y`). Trigger even when the user just mentions a `.rv` file or rvpm. Do NOT trigger for the Raven compiler source itself (the `.rs` files) or unrelated languages.
metadata:
  author: martian56
  version: 3.0.1
---

# Raven Language (3.0)

Raven reads like Python with braces: blocks use `{ }`, statements end at a newline, and types are checked at compile time but mostly inferred. Source compiles ahead of time through Cranelift to a native binary, and a tracing garbage collector manages memory. This skill's main job is to keep the first version of your code compiling.

> Raven 3 is not source compatible with Raven 2. Old material with `struct`, `const`, `Int`/`String`, `List<T>`, `&&`/`||`/`!`, `Point { x: 1 }`, `Ok(x) -> ...` match arms, `fun(x) { }` lambdas, or `import std/io { println }` is Raven 2, and the 3.0 compiler rejects each of those forms with an error naming the replacement.

## When to read what

- **This file**: read every time. The pitfalls and templates cover most tasks.
- **`references/builtins.md`**: builtins (`print`, `len`, `range`, ...) and the methods on `str`, `list`, `dict`, `set`, `Option`, `Result`.
- **`references/stdlib.md`**: per-module API for the `std.*` modules.
- **`references/rvpm.md`**: starting a project, `rv.toml`, GitHub-direct packages, the `raven` and `rvpm` CLIs.
- **`references/grammar.md`**: exact lexical and grammar rules when something parses surprisingly.

The stdlib source lives in `stdlib/std/*.rv` in the Raven repo, written in Raven, so reading it is the authoritative answer for any signature.

## Pitfalls to internalise

1. **`let` cannot be reassigned; `var` can.** `var count = 0` then `count += 1`. Assigning to a `let` is a compile error. `let` freezes the binding, not the value: a `let` list can still be pushed to.
2. **Types are lowercase:** `int`, `float`, `bool`, `str`, `char`, `list[int]`, `dict[str, int]`, `set[str]`, `(int, str)`. User types and `Option[T]` / `Result[T, E]` are capitalized.
3. **Generics use square brackets:** `fun first[T](xs: list[T]) -> Option[T]`, bounds inline `[T: Eq + Hash]`, `class Box[T] { value: T }`.
4. **`and`, `or`, `not`.** `&&`, `||`, `!` are errors.
5. **No semicolons needed.** A newline ends a statement.
6. **Classes, not structs:** `class Point { x: float = 0.0  fun len(self) -> float { ... } }` holds fields, defaults, and methods together. Build values with call syntax: `Point(1.0, 2.0)`, `Point(x: 1.0)`, `Point(..p, y: 5.0)`. Every class value is a reference.
7. **`match` uses `case` blocks:** `match s { case Shape.Circle(r) { ... } case _ { ... } }`. Patterns nest (`case Some(Ok(0))`), take guards (`case n if n > 0`), and are checked for exhaustiveness. Enum values are built qualified: `Shape.Circle(2.0)`.
8. **Lambdas are `=>`:** `x => x * 2`, `(a, b) => a + b`, `() => { ... }`. Parameter types are inferred from the call site or from how the lambda is used.
9. **Imports are Python style:** `import std.fs`, `from std.json import parse, stringify`, `from "./models" import User`, `from "github.com/user/repo" import Thing`. Every top-level item is importable; `pub` is accepted but not required.
10. **Python builtins work with no import:** `print(a, b)`, `len`, `range`, `sum`, `min`, `max`, `sorted`, `reversed`, `enumerate`, `zip`, `abs`, `round`, `str`, `int`, `float`. `x in xs`, `xs[-1]`, slices `xs[1..3]`, `xs[2..]`, `xs[..2]`.
11. **Collections:** `[1, 2]` list, `{"a": 1}` dict, `{1, 2}` and `{x}` sets, `{}` empty dict, `set()` empty set. Comprehensions: `[x * 2 for x in xs if x > 0]`, `{k: v for ...}`. `d[k]`, `d[k] = v`, `for (k, v) in d`.
12. **No `null`.** Absence is `Option[T]` (`Some(x)` / `None`); fallible results are `Result[T, E]` (`Ok` / `Err`) with `?`. A failing `?` inside `main` stops the program with the error's message.
13. **`if` / `match` are statements; for a value use `a if cond else b`.** `elif` works as `else if`, and `x ** y` is power. Functions return with `return`, or use a one-line body `fun sq(x: int) -> int = x * x`.
14. **`fun main()` is the entry point.** Define it; never call it.
15. **A trait used as a type is a trait object:** `fun total(shapes: list[Shape])` takes any class implementing `Shape`, and a list literal of mixed classes converts automatically.
16. **Strings:** `"hello ${name}"` interpolation, format specs `"${x:.2}"`, `"${s:>8}"`, `"${n:05}"`. `len(s)`, `s.split(",")`, `s.strip()`, `s.upper()`, `", ".join(parts)`, `"ab" * 3`. No import needed.

## Anatomy of a Raven file

```rust
from std.math import sqrt

trait Shape {
    fun area(self) -> float

    fun describe(self) -> str {
        return "shape with area ${self.area():.2}"
    }
}

class Circle: Shape {
    r: float = 1.0

    fun area(self) -> float = 3.14159 * self.r * self.r
}

class Point {
    x: float
    y: float

    fun dist(self, o: Point) -> float {
        let dx = self.x - o.x
        let dy = self.y - o.y
        return sqrt(dx * dx + dy * dy)
    }
}

fun main() {
    print(Circle(r: 2.0).describe())
    print(Point(0.0, 0.0).dist(Point(3.0, 4.0)))
}
```

## Templates

### Variables and control flow

```rust
fun classify(n: int) -> str {
    if n < 0 {
        return "negative"
    } else if n == 0 {
        return "zero"
    }
    return "big" if n > 100 else "small"
}

fun main() {
    var total = 0
    for i in range(10) {
        if i % 2 == 0 {
            continue
        }
        total += i
    }
    var n = 3
    while n > 0 {
        n -= 1
    }
    print(total, classify(total))
}
```

### Collections and comprehensions

```rust
fun main() {
    let words = ["apple", "bob", "cherry"]
    let lengths = {w: len(w) for w in words}
    let long = [w for w in words if len(w) > 3]
    print(lengths["bob"], long, sorted(words, reverse: true)[0])
    let seen = {"apple"}
    seen.add("kiwi")
    print("kiwi" in seen, len(seen))
    for (i, w) in enumerate(words) {
        print(i, w)
    }
    let (first, rest) = (words[0], words[1..])
    print(first, rest)
}
```

### Enums, match, Option, Result, ?

```rust
enum Shape {
    Circle(r: float),
    Rect(w: float, h: float),
    Dot,
}

fun area(s: Shape) -> float {
    match s {
        case Shape.Circle(r) { return 3.14 * r * r }
        case Shape.Rect(w, h) if w == h { return w * w }
        case Shape.Rect(w, h) { return w * h }
        case Shape.Dot { return 0.0 }
    }
}

fun half(n: int) -> Result[int, str] {
    if n % 2 == 1 {
        return Err("${n} is odd")
    }
    return Ok(n / 2)
}

fun quarter(n: int) -> Result[int, str] {
    let h = half(n)?
    return half(h)
}

fun main() {
    print(area(Shape.Rect(2.0, 3.0)), quarter(8), quarter(6).is_err())
    let found: Option[int] = Some(3)
    if let Some(v) = found {
        print("found", v)
    }
    print(quarter(12)?)
}
```

### Generics and traits

```rust
trait Describe {
    fun describe(self) -> str
}

class Dog: Describe {
    name: str

    fun describe(self) -> str = "dog ${self.name}"
}

fun largest[T: Ord](xs: list[T]) -> Option[T] = xs.max()

fun all_text[T: Describe](items: list[T]) -> list[str] = [i.describe() for i in items]

fun main() {
    print(largest([3, 9, 2]), all_text([Dog("rex")]))
}
```

### Lambdas and higher-order functions

```rust
fun apply(f: fun(int) -> int, x: int) -> int = f(x)

fun main() {
    let add = (a, b) => a + b
    print(apply(x => x * 10, 4), add(1, 2))
    let evens = [1, 2, 3, 4].filter(x => x % 2 == 0)
    print(evens, [1, 2, 3].fold(0, (acc, x) => acc + x))
}
```

### Concurrency: spawn and channels

```rust
from std.sync import Channel, channel_buffered

fun main() {
    let results: Channel[int] = channel_buffered(4)
    for k in range(4) {
        spawn {
            results.send(k * k)
        }
    }
    var total = 0
    for _ in range(4) {
        total += results.recv()
    }
    print(total)
    let task = spawn {
        print("in task")
    }
    task.join()
}
```

### Tests

```rust
// math_test.rv, run with `rvpm test` (or `rvpm test add` for matching names)
from std.test import assert_eq_int

fun add(a: int, b: int) -> int = a + b

fun test_add() {
    assert add(2, 3) == 5
    assert_eq_int(add(-1, 1), 0)
}
```

### Derive, JSON, defer

```rust
@derive(Ord, ToJson, FromJson)
class Item {
    name: str
    qty: int
}

fun main() {
    defer print("done")
    let items = sorted([Item("b", 2), Item("a", 1)])
    print(items[0].name, items[0] == Item("a", 1))
}
```

## Running

```bash
raven run file.rv          # compile and run one file
raven check file.rv        # type check only
raven fmt file.rv --write  # format in place
rvpm new app               # new project (rvpm new lib --lib for a library)
rvpm run / rvpm build / rvpm test [NAME] / rvpm fmt
```

`rv.toml` carries `edition = "v3"`. A package with `edition = "v2"` is Raven 2 syntax and is rejected.

## Recommended workflow

1. Write the code using the templates above.
2. `raven check file.rv` (or `rvpm build`) and fix what the compiler reports; its errors name the 3.0 form when you write a Raven 2 one.
3. `raven fmt --write` to normalise layout.
4. Add `*_test.rv` files with `test_*` functions and run `rvpm test`.

## Style

Four-space indentation, one blank line between top-level items, snake_case functions and variables, PascalCase classes, enums, and traits. Prefer `let` unless the binding must change, comprehensions over manual loops for building lists, and `?` over nested `match` for propagating errors.

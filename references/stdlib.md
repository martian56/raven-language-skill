# Raven standard library (v2)

The stdlib is bundled into the compiler (it is no longer shipped as separate files), and each module is written in Raven. The source is `stdlib/std/*.rv` in the Raven repo, and reading it is the authoritative answer for any signature.

Import a module and select the names you need:

```raven
import std/io { println }
import std/collections { Map, Set }
import std/string                      // method-only module: merges String methods
```

Selecting free functions by name (and calling them unqualified) is the form that always works. A `module.function()` qualified call works for some runtime-backed modules (`fs.exists(...)`) but not all (`math.sqrt(...)` is rejected), so prefer selectors.

Available modules: `io`, `string`, `collections`, `iter`, `math`, `cmp`, `hash`, `fmt`, `encoding`, `random`, `env`, `fs`, `time`, `net`, `http`, `json`, `regex`, `process`, `ffi`, `error`, `path`, `test`, `sync`, plus the always-in-scope `core` prelude.

---

## std/io

```raven
import std/io { print, println }
```

`print(s)` writes without a trailing newline; `println(s)` adds one. The global `print` (no import) writes a line.

## std/string

```raven
import std/string
```

Method-only module: importing it merges methods onto `String` (`length`, `to_upper`, `to_lower`, `trim`, `substring`, `concat`, `repeat`, `index_of`, `contains`, `starts_with`, `replace`, `char_at`, …). See `references/builtins.md` for the table. Use `.length()`, not `.len()`.

## std/collections

```raven
import std/collections { Map, Set }
```

### Map<K, V>  (K: Eq + Hash)

```raven
let m: Map<String, Int> = Map.new()      // or a literal: ["a": 1, "b": 2]
m.set("a", 1)
match m.get("a") { Some(v) -> ..., None -> ... }   // get returns Option<V>
let present = m.has("a")                  // Bool
let n = m.len()
let ks = m.keys()                         // List<K>
let vs = m.values()                       // List<V>
m.remove("a")                             // Bool
```

### Set<T>  (T: Eq + Hash)

```raven
let s: Set<Int> = Set.new()              // or a literal: {1, 2, 3}
s.add(1)
let has = s.contains(1)                  // Bool
let n = s.len()
s.remove(1)                              // Bool
```

Mutators act in place (unlike v1, you do not reassign the result).

## std/iter

```raven
import std/iter { collect, fold, count }
```

Lazy adapters over `xs.iter()`: `.map(f)`, `.filter(pred)`, `.take(n)`, `.skip(n)`, `.enumerate()`. Drive them with a consumer:

| Consumer                       | Returns      |
| ------------------------------ | ------------ |
| `collect(it)`                  | `List<T>`    |
| `fold(it, init, f)`            | accumulator  |
| `count(it)`                    | `Int`        |

```raven
let kept: List<Int> = collect(xs.iter().map(fun(x: Int) -> Int = x * 10).filter(fun(y: Int) -> Bool = y > 20))
let total = fold(xs.iter(), 0, fun(a: Int, v: Int) -> Int = a + v)
```

## std/math

```raven
import std/math { sqrt, pow, pow_int, abs, abs_int, min, max, min_int, max_int, clamp, clamp_int, pi, e, tau, ln, sin, cos }
```

`Float` functions: `sqrt`, `pow`, `abs`, `min`, `max`, `clamp`, `ln`, `sin`, `cos`, and constants via `pi()`, `e()`, `tau()` (functions, not constants). `Int` helpers: `abs_int`, `min_int`, `max_int`, `clamp_int`, `pow_int`.

## std/fs

```raven
import std/fs { read, write, append, exists, remove_file, create_dir, remove_dir, list_dir, size, is_file, is_dir, split_lines }
```

| Function                      | Returns                      |
| ----------------------------- | ---------------------------- |
| `read(path)`                  | `Result<String, Error>`      |
| `write(path, contents)`       | `Result<Bool, Error>`        |
| `append(path, contents)`      | `Result<Bool, Error>`        |
| `remove_file(path)`           | `Result<Bool, Error>`        |
| `create_dir(path)`            | `Result<Bool, Error>`        |
| `list_dir(path)`              | `Result<List<String>, Error>`|
| `size(path)`                  | `Result<Int, Error>`         |
| `exists(path)`                | `Bool`                       |
| `is_file(path)` / `is_dir(path)` | `Bool`                    |
| `split_lines(s)`              | `List<String>`               |

```raven
match read("data.txt") {
    Ok(body) -> print("${split_lines(body).len()} lines"),
    Err(e) -> print("read failed"),
}
```

`std/path` provides `join`, `basename`, `dirname`, `extension`, `stem`, `is_absolute`.

## std/time

```raven
import std/time { now, now_millis, from_timestamp, format_timestamp, parse_timestamp, weekday, sleep_millis }
```

`now()` is Unix seconds, `now_millis()` is Unix milliseconds (both `Int`). `from_timestamp(ts)` builds a `DateTime`; `format_timestamp(ts, pattern)` and `parse_timestamp(text, pattern) -> Result<Int, Error>` convert.

## std/json

```raven
import std/json { JsonValue, parse, stringify }
```

```raven
enum JsonValue {
    Null,
    Bool(Bool),
    Number(Float),
    Str(String),
    Array(List<JsonValue>),
    Object(Map<String, JsonValue>),
}
```

`parse(text) -> Result<JsonValue, Error>`, `stringify(value) -> String`. Build values with the qualified constructors: `JsonValue.Str("x")`, `JsonValue.Number(2.0)`, `JsonValue.Object(map)`. `@derive(ToJson, FromJson)` on your own types gives `to_json()` / `Type.from_json(v)`.

## std/sync

```raven
import std/sync { channel, channel_buffered, yield_now }
```

`channel()` is unbuffered, `channel_buffered(cap)` holds up to `cap`. Methods `ch.send(Int)` and `ch.recv() -> Int` block (yielding to the scheduler) when full/empty. `yield_now()` hands control to another goroutine. Channels carry `Int` in this release.

## std/cmp

```raven
import std/cmp { min, max, clamp, sort, sorted_by }
```

`min`/`max`/`clamp` over orderable values, `sort(list) -> List<T>` and `sorted_by(list, key)` return sorted copies. `sort` relies on ordering, so it works on numbers, `String`, and any `struct`/`enum` that derives `@derive(Ord)` (which provides `compare(self, other) -> Int`, comparing structs field-by-field in declaration order and enums by variant order then payload).

```raven
import std/cmp { sort }

@derive(Ord)
struct Version { major: Int, minor: Int }

fun main() {
    let vs = sort([Version { major: 1, minor: 2 }, Version { major: 1, minor: 0 }])
    print("${vs[0].minor}")   // 0
}
```

## std/random

```raven
import std/random { Rng }
```

`Rng.new(seed)` is deterministic; `Rng.from_entropy()` seeds from a runtime source (distinct on every call as of 2.0.1). Methods: `next_int()` (full i64), `gen_range(lo, hi)` (half-open), `next_float()` (`[0,1)`), `next_bool()`, `choice(list) -> Option<T>`, `shuffle(list)`. Hold one `Rng` and draw many values from it.

## std/ffi

```raven
import std/ffi { alloc, free, load, store, offset, is_null, null_ptr }
```

Raw memory for the C FFI: `alloc<T>(n)`, `free<T>(p)`, `store<T>(p, v)`, `load<T>(p) -> T`, `offset<T>(p, i)`, `is_null<T>(p)`, `null_ptr<T>()`. C types (`CInt`, `CStr`, `CPtr<T>`, `CFnPtr`, …) are built in; declare foreign functions in an `extern "C"` block.

## std/net, std/http, std/process, std/env, std/regex, std/encoding, std/test

Convenience modules over sockets, HTTP, subprocesses, environment variables, regular expressions, byte/string encoding, and test assertions. The APIs are small; read `stdlib/std/<module>.rv` for the current set (for example `import std/http`, `import std/net { connect }`, `import std/env { get_env }`, `import std/regex { compile }`, `import std/test { assert, assert_eq_int }`).

---

## When you don't see it here

Read the source in the Raven repo:

```bash
ls stdlib/std/            # all modules
cat stdlib/std/<module>.rv  # exact functions, structs, and methods
```

Every top-level `fun`, `struct`, `enum`, and `trait` in those files is importable.

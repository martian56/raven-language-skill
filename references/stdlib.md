# Raven standard library

The stdlib lives in `lib/*.rv` in the Raven repository. Each file is itself written in Raven, so when in doubt, read the source — it's short and authoritative.

Three import forms work for any of these:

```raven
import math;                              // namespace — call as math.sqrt(...)
import str from "str";                    // alias — call as str.trim(...)
import { trim, contains } from "str";     // selective — call directly
```

---

## math

```raven
import math;
```

### Constants

| Name      | Value                  |
| --------- | ---------------------- |
| `math.PI` | `3.141592653589793`    |
| `math.E`  | `2.718281828459045`    |
| `math.TAU`| `6.283185307179586`    |

### Functions

| Function | Signature | Notes |
| --- | --- | --- |
| `math.abs(x)` | `(float) -> float` | Absolute value |
| `math.min(a, b)` | `(float, float) -> float` |  |
| `math.max(a, b)` | `(float, float) -> float` |  |
| `math.floor(x)` | `(float) -> float` | Toward `-infinity` |
| `math.ceil(x)` | `(float) -> float` | Toward `+infinity` |
| `math.round(x)` | `(float) -> float` | Half-up |
| `math.pow(b, e)` | `(float, float) -> float` | Handles negative exponent via `1/result` |
| `math.sqrt(x)` | `(float) -> float` | Newton's method, ~10 iterations |
| `math.sin(x)` | `(float) -> float` | Radians, Taylor series |
| `math.cos(x)` | `(float) -> float` | Radians |
| `math.tan(x)` | `(float) -> float` | Radians |
| `math.log(x)` | `(float) -> float` | Natural log |
| `math.log10(x)` | `(float) -> float` |  |
| `math.random()` | `() -> float` | `[0.0, 1.0)` |
| `math.random_int(min, max)` | `(int, int) -> int` | Inclusive on both ends |
| `math.clamp(v, lo, hi)` | `(float, float, float) -> float` |  |
| `math.lerp(a, b, t)` | `(float, float, float) -> float` | Linear interpolate |
| `math.distance(x1, y1, x2, y2)` | `(float, float, float, float) -> float` | 2D Euclidean |
| `math.degrees_to_radians(d)` | `(float) -> float` |  |
| `math.radians_to_degrees(r)` | `(float) -> float` |  |

---

## str

```raven
import str from "str";
```

These are *function-style* on top of the built-in string methods (which already cover the common cases). Use the built-in methods (`s.to_upper()`, `s.contains(x)`, …) when they suffice, and reach for `str` when you need things the methods don't cover (padding, repeat, reverse, casing helpers, comparison).

| Function | Returns | Notes |
| --- | --- | --- |
| `str.to_upper(s)` | `string` |  |
| `str.to_lower(s)` | `string` |  |
| `str.trim(s)` | `string` |  |
| `str.trim_left(s)` | `string` |  |
| `str.trim_right(s)` | `string` |  |
| `str.contains(s, sub)` | `bool` |  |
| `str.starts_with(s, p)` | `bool` |  |
| `str.ends_with(s, p)` | `bool` |  |
| `str.index_of(s, sub)` | `int` | -1 if missing |
| `str.last_index_of(s, sub)` | `int` | -1 if missing |
| `str.pad_left(s, width, pad)` | `string` | Pads with `pad` on the left |
| `str.pad_right(s, width, pad)` | `string` |  |
| `str.pad_center(s, width, pad)` | `string` |  |
| `str.repeat(s, n)` | `string` |  |
| `str.reverse(s)` | `string` |  |
| `str.capitalize(s)` | `string` | First letter upper, rest lower |
| `str.title_case(s)` | `string` | "hello world" → "Hello World" |
| `str.is_empty(s)` | `bool` |  |
| `str.is_blank(s)` | `bool` | Empty or whitespace |
| `str.is_numeric(s)` | `bool` | Digits and at most one `.` |
| `str.is_alpha(s)` | `bool` |  |
| `str.is_alphanumeric(s)` | `bool` |  |
| `str.compare(a, b)` | `int` | `-1`/`0`/`1` |
| `str.compare_ignore_case(a, b)` | `int` |  |

---

## collections

```raven
import collections;
```

Provides four data structures, each as a struct with methods.

### Map (string → string)

```raven
let m: Map = collections.new_map();
m = m.set("name", "Alice");
m = m.set("role", "engineer");
let n: string = m.get("name");        // "Alice"
let has: bool = m.has("role");        // true
m = m.remove("role");
let size: int = m.len();              // 1
let ks: string[] = m.keys();
let vs: string[] = m.values();
```

The setters return a new `Map` — assign back. There are also bare functions (`collections.map_set`, etc.) but the method form is idiomatic.

### Set (string)

```raven
let s: Set = collections.new_set();
s = s.add("apple");
s = s.add("apple");        // no-op
let has: bool = s.contains("apple");
let n: int = s.len();
let u: Set = s.union(other);
let i: Set = s.intersection(other);
```

### Stack and Queue

```raven
let st: Stack = collections.new_stack();
st = st.push("a");
let top: string = st.peek();
let popped: string = st.pop();
let empty: bool = st.is_empty();

let q: Queue = collections.new_queue();
q = q.enqueue("first");
let front: string = q.peek();
let next: string = q.dequeue();
```

All collections are immutable-style: methods return updated copies. Always reassign.

---

## filesystem

```raven
import filesystem;
```

Wraps the built-in file functions with friendlier helpers.

### Path manipulation

| Function | Returns |
| --- | --- |
| `filesystem.join_path(parts: string[])` | `string` |
| `filesystem.split_path(path)` | `string[]` |
| `filesystem.dirname(path)` | `string` |
| `filesystem.basename(path)` | `string` |
| `filesystem.extension(path)` | `string` (without the dot) |

### Listing and finding

| Function | Returns |
| --- | --- |
| `filesystem.list_files(dir)` | `string[]` (files only) |
| `filesystem.list_directories(dir)` | `string[]` (subdirs only) |
| `filesystem.list_directory(dir)` | `string[]` (both) |
| `filesystem.find_files(dir, pattern)` | `string[]` (recursive substring match) |
| `filesystem.find_files_by_extension(dir, ext)` | `string[]` |

### File I/O

| Function | Returns |
| --- | --- |
| `filesystem.read_lines(path)` | `string[]` |
| `filesystem.write_lines(path, lines: string[])` | `void` |
| `filesystem.append_line(path, line: string)` | `void` |
| `filesystem.copy_file(src, dst)` | `bool` |
| `filesystem.move_file(src, dst)` | `bool` |
| `filesystem.get_file_info(path)` | `FileInfo` struct |
| `filesystem.files_are_equal(a, b)` | `bool` |

### Validation

| Function | Returns |
| --- | --- |
| `filesystem.is_valid_filename(name)` | `bool` |
| `filesystem.sanitize_filename(name)` | `string` (replaces invalid chars with `_`) |

---

## json

```raven
import "json";    // note: quoted form
```

Raven's JSON support is limited. Validation, basic parse/access, and pretty-printing.

| Function | Returns | Notes |
| --- | --- | --- |
| `json.validate(raw: string)` | `string` | `"ok"` on success, error message otherwise |
| `json.parse(raw: string)` | `string[]` | Path-indexed entries |
| `json.get_value(entries: string[], path: string)` | `string` | E.g. `"$[name]"` |
| `json.minify(raw: string)` | `string` | Strips whitespace |
| `json.pretty(raw: string, indent: int)` | `string` | Pretty-prints |

```raven
let raw: string = "{\"name\":\"Alice\",\"age\":30}";
if (json.validate(raw) == "ok") {
    let pretty: string = json.pretty(raw, 2);
    print(pretty);
}
```

For complex JSON work, parse into your own structs by hand or read the file as text and `split` — Raven's JSON layer is intentionally thin.

---

## time

```raven
import time;
```

Convenience helpers on top of `sys_time()`, `sys_date()`, `sys_timestamp()`. The exact API evolves; read `lib/time.rv` for the current list.

---

## web / network

```raven
import web;
import network;
```

Convenience wrappers around `http_fetch`, `tcp_*`, `dns_lookup`. Read the lib files; the API is small.

---

## testing

```raven
import "testing";
```

Assertion helpers for writing tests. Inspect `lib/testing.rv` for the current set (typically `assert_equals`, `assert_true`, etc., plus a runner).

A test file can be invoked directly with `raven path/to/test.rv`.

---

## When you don't see it here

The stdlib evolves. To check what's actually available right now:

```bash
ls lib/                # see all modules
cat lib/<module>.rv    # read its exports
```

Every `export fun` and `export let` in those files is callable from your code.

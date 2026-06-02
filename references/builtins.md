# Raven built-ins and core methods (v2)

What is available without an `import`, plus the methods on the core collection and string types. Anything beyond this lives in a stdlib module (see `references/stdlib.md`).

## Always-available functions

### `print(value)`
Writes to stdout followed by a newline. The argument is usually an interpolated string. There are no format specifiers; interpolate values with `${expr}`.

```raven
print("hello")
print("x = ${x}, y = ${y}")
```

For a print without the trailing newline, or to print explicitly via the stdlib, use `import std/io { print, println }`.

> There is no global `panic`, `format`, `input`, `len(x)`, or `read_file` in v2. Use `${...}` interpolation instead of `format`, the `.len()` method on collections, and the `std/fs`/`std/io` modules for I/O.

## Prelude values (no import)

`Some(x)`, `None`, `Ok(x)`, `Err(e)` construct `Option<T>` and `Result<T, E>`. The `?` operator unwraps `Ok`/`Some` and early-returns `Err`/`None`.

## Concurrency

### `spawn(closure)`
Starts a goroutine on the cooperative scheduler. The closure has type `fun() -> Unit`.

```raven
spawn(fun() -> Unit { ch.send(1) })
```

Channels come from `std/sync`.

## Compile-time reflection (no import)

### `type_name<T>() -> String`
The name of the type argument, resolved at compile time (per monomorphization for a generic `T`).

### `field_names<T>() -> List<String>`
A struct type's field names, in declaration order.

```raven
fun introspect<T>() {
    print("type ${type_name<T>()}")
    for f in field_names<T>() {
        print("  field ${f}")
    }
}
```

Runtime reflection (`to_any`, `type_name_of`, `field_names_of`, `get_field`, `cast<T>`) is also available; see the reflection module/source for exact signatures.

## `List<T>` methods (built-in, no import)

| Form                | Result        | Notes                                             |
| ------------------- | ------------- | ------------------------------------------------- |
| `xs[i]`             | `T`           | Direct element access; the common accessor        |
| `xs[i] = v`         | mutate        | In-place element assignment                        |
| `xs.len()`          | `Int`         | Element count                                      |
| `xs.push(v)`        | mutate        | Append; mutates the list in place                  |
| `xs.get(i)`         | `Option<T>`   | Bounds-checked; a `match` on it may need a `_` arm |
| `xs.iter()`         | iterator      | Bridge into `std/iter` (`.map`, `.filter`, …)      |

List literals: `[1, 2, 3]`; an empty list needs a type annotation (`let xs: List<Int> = []`).

`Map<K, V>` and `Set<T>` are not built-in; import them from `std/collections`.

## `String` methods (require `import std/string`)

Importing `std/string` merges these methods onto every `String`. The length method is `.length()` (the `.len()` you would expect type-checks but fails in codegen, so always use `.length()` on strings).

| Method                                 | Returns    | Notes                              |
| -------------------------------------- | ---------- | ---------------------------------- |
| `s.length()`                           | `Int`      | Byte length                        |
| `s.substring(start, end)`              | `String`   | Half-open `[start, end)`           |
| `s.concat(other)`                      | `String`   | Concatenation                      |
| `s.to_upper()` / `s.to_lower()`        | `String`   |                                    |
| `s.trim()`                             | `String`   | Strip leading/trailing whitespace  |
| `s.repeat(n)`                          | `String`   |                                    |
| `s.index_of(needle)`                   | `Int`      | `-1` if absent                     |
| `s.contains(needle)`                   | `Bool`     |                                    |
| `s.starts_with(prefix)`                | `Bool`     |                                    |
| `s.replace(from, to)`                  | `String`   | Replace all occurrences            |
| `s.char_at(i)`                         | `String`   | One-character string at byte `i`   |

Strings are immutable: every method returns a new string. Compare with `==` (there are no `<`/`>` ordering operators on `String`, and `match` on string-literal patterns is currently broken; use `==`/`if` chains).

### Low-level string intrinsics (no import)

Used by the stdlib; occasionally handy for byte-level work.

| Intrinsic                    | Returns  | Notes                              |
| ---------------------------- | -------- | ---------------------------------- |
| `__str_len(s)`               | `Int`    | Byte length (what `.length()` calls) |
| `__str_byte_at(s, i)`        | `Int`    | ASCII byte at index `i`            |
| `__str_concat(a, b)`         | `String` |                                    |
| `__str_substring(s, a, b)`   | `String` |                                    |
| `__str_from_byte(b)`         | `String` | One-character string from a byte   |

Prefer the `std/string` methods; reach for the intrinsics only when parsing bytes.

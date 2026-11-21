# Raven built-ins and core methods (3.0)

Everything on this page is in scope with no import.

## Builtin functions

| Call | Meaning |
|------|---------|
| `print(a, b, ...)` | print values separated by spaces, then a newline |
| `len(x)` | length of a `str` (bytes), list, dict, or set |
| `range(n)`, `range(a, b)`, `range(a, b, step)` | the ints `0..n`, `a..b`, stepped |
| `sum(xs)`, `min(xs)`, `max(xs)` | over a list (`min`/`max` panic on an empty list) |
| `sorted(xs)`, `sorted(xs, reverse: true)`, `reversed(xs)` | new lists |
| `enumerate(xs)`, `zip(a, b)` | lists of `(index, item)` and `(a, b)` tuples |
| `str(x)`, `int(x)`, `float(x)` | conversions; `int("42")` parses and panics on bad input |
| `abs(x)`, `round(x)` | numbers |
| `panic(msg)` | stop the program with a message |
| `assert cond` / `assert cond, msg` | statement; fails with the condition's text or `msg` |
| `set()` | an empty set |
| `type_name[T]()`, `field_names[T]()` | compile-time reflection |

Casts: `x as int` (truncates), `x as float`, `x as str`.

## Values in the prelude

`Some(x)`, `None`, `Ok(x)`, `Err(e)`, `true`, `false`. The `Error` type from
`std.error` and the core traits `ToString`, `Eq`, `Ord`, `Hash`, and
`Iterator[T]` are in scope too.

## Operators on collections and strings

- `x in xs`, `x not in xs` for lists, sets, dict keys, and substrings.
- `xs[i]`, `xs[-1]`, `xs[a..b]`, `xs[a..]`, `xs[..b]`, `xs[a..=b]`; negative
  bounds count from the end. Slicing works on `str` by byte offset.
- `d[k]` reads (panics when missing), `d[k] = v`, `d[k] += 1`.
- `list + list`, `list * n`, `str + str`, `str * n`.
- `for x in xs`, `for (k, v) in d`, `for c in s` (characters), `for i in 0..n`,
  `for i in (0..n).step(2)`.

## list[T] methods

`len`, `is_empty`, `push(x)`, `pop()`, `reserve(n)`, `extend(other)`,
`concat(other)`, `contains(x)`, `index_of(x)`, `count(f)`, `first()`,
`last()` (both `Option`), `slice(a, b)`, `take(n)`, `skip(n)`, `reversed()`,
`sort()`, `sorted()`, `sorted_by(cmp)`, `min()`, `max()` (both `Option`),
`sum()`, `join(sep)` (for `list[str]`), `map(f)`, `filter(f)`,
`fold(init, f)`, `each(f)`, `any(f)`, `all(f)`, `find(f)`, `position(f)`,
`enumerate()`, `zip(other)`, `step(k)`.

## str methods

`len()`, `is_empty()`, `split(sep)`, `split_whitespace()`, `lines()`,
`strip()`, `trim_start()`, `trim_end()`, `upper()`, `lower()`,
`replace(a, b)`, `startswith(p)`, `endswith(p)`, `contains(p)`,
`index_of(p)`, `last_index_of(p)`, `count(p)`, `slice(a, b)`, `repeat(n)`,
`reverse()`, `join(parts)` (`", ".join(xs)`), `char_at(i)`, `byte_at(i)`,
`parse_int()`, `parse_float()` (both `Option`), `is_blank()`, `iter()`
(characters).

## dict[K, V] and set[T] methods

dict: `len`, `is_empty`, `get(k)` (`Option`), `get_or(k, default)`,
`set(k, v)`, `contains(k)`, `remove(k)`, `keys()`, `values()`, `items()`
(`(k, v)` tuples), `clear()`. set: `len`, `is_empty`, `add(x)`,
`contains(x)`, `remove(x)`, `to_list()`, `union(o)`, `intersection(o)`,
`difference(o)`, `is_subset(o)`. Iteration follows insertion order. Keys and elements
must implement `Eq` and `Hash` (`int`, `bool`, `str`, tuples of those, and
any plain class).

## Option[T] and Result[T, E] methods

`unwrap()`, `expect(msg)`, `unwrap_or(x)`, `unwrap_or_else(f)`, `map(f)`,
`and_then(f)`, `or_else(f)`, `is_some()`, `is_none()`, `is_ok()`,
`is_err()`, `ok_or(e)`, `ok()`, `err()`, `map_err(f)`, `unwrap_err()`.

## Concurrency

`spawn { ... }` starts a goroutine and returns a task with `join()`.
Channels come from `std.sync`: `channel()`, `channel_buffered(n)`, typed
`Channel[T]` with `send(x)` and `recv()`.

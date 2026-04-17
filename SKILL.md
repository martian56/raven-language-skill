---
name: raven-language-skill
description: Reference and patterns for writing Raven programming language source (.rv files). Use this whenever the user is writing, editing, debugging, or reviewing Raven code, working in an rvpm project (rv.toml), discussing Raven syntax, importing from Raven's stdlib (math, str, collections, json, filesystem, time, web, network, testing), or running raven/rvpm commands. Raven is a small statically-typed interpreted language with several syntax pitfalls (`elseif` not `else if`, no comments before `else`, no `const`, mandatory type annotations, C-style for loops only) that this skill helps Claude avoid. Trigger even when the user just mentions a `.rv` file or rvpm without asking explicitly for help with syntax. Do NOT trigger for the Raven compiler/interpreter source itself (the `.rs` files in this repo) or unrelated languages.
metadata:
  author: martian56
  version: 1.6.4
---

# Raven Language

Raven is a small statically-typed, tree-walking interpreted language implemented in Rust. Its syntax looks like a hybrid of Rust and TypeScript but the rules are stricter and there are several non-obvious pitfalls. The single most useful thing this skill does is steer you away from those pitfalls so the first version of your code parses and type-checks.

## When to read what

- **This file**: read every time. It contains the gotchas that catch people on their first program and the templates you'll mimic for most tasks.
- **`references/builtins.md`**: when you need a built-in function (`print`, `format`, `read_file`, HTTP, time, etc.) and want the exact signature.
- **`references/stdlib.md`**: when importing from `math`, `str`, `collections`, `filesystem`, `json`, `time`, `web`, `network`, `testing` — full per-module API.
- **`references/rvpm.md`**: when initialising a project, configuring `rv.toml`, formatting, or running.
- **`references/grammar.md`**: when something parses surprisingly and you need the exact rules.

If unsure what symbol exists, you can also read `lib/<module>.rv` directly from the Raven repo — the stdlib is written in Raven itself.

## Pitfalls to internalise

These are the mistakes that ruin first-try compilation. Burn them in.

1. **`elseif`, not `else if`.** One word. `} elseif (cond) { ... }`. Two-word `else if` will not parse.
2. **No comments between `}` and `else`/`elseif`.** The next token after the closing brace must be `else`/`elseif`. A `// remark` in between turns into a parse error.
3. **`const` does nothing.** It's lexed but not parsed. Use `let` for everything.
4. **For loops are C-style only.** `for (let i: int = 0; i < n; i = i + 1) { ... }`. The `let`, the type, the explicit `i = i + 1` are all required. There is no `for x in arr`, no `i++`, no `+=`.
5. **No `break` or `continue`.** Restructure with `while` and a flag.
6. **Type annotations are mandatory.** Every `let`, every function parameter, every function return type, every struct field needs `: type`. Type inference exists in some places but assume it doesn't.
7. **No implicit conversions and no cast syntax.** `let s: string = 5;` is a type error. There is no `x as string`, no `(int)x`, no `string(x)`. Stringify with `format("{}", x)`. Parse with `parse_int(s)` (returns 0 on failure — there's no exception).
8. **No null/None.** Initialise everything. Primitives must have an initialiser at declaration; structs may declare without one and start with default fields.
9. **String concat with `+` works but only with strings on the left in some contexts.** Safer: always use `format("{} {}", a, b)`.
10. **Arrays passed to functions don't reliably mutate the caller.** Inside a function, `arr.push(x)` may modify only the local copy. Idiom: have helpers **return** their array, then concatenate in the caller. (See "Array mutation" below.)
11. **Methods need `self` as the first parameter** in `impl` blocks. Without it, the parser/type-checker rejects it.
12. **Negative literals are unary minus.** `-5` is `-` applied to `5`. Usually transparent, but matters when you're reasoning about the AST or precedence.
13. **`&&`/`||` do NOT short-circuit.** Both sides are always evaluated. Don't write `if (ptr != null && ptr.field)` style patterns — though "null" doesn't exist anyway, this matters for things like `if (i < len(a) and check(a[i]))` (a guard followed by an indexed access). If `i` is out of range, `check(a[i])` still runs and crashes. Restructure.
14. **`enum` variants have no data and no pattern matching.** Compare with `==`. Construct with `EnumName::Variant`. Convert from string with `enum_from_string("EnumName", "Variant")`.
15. **Statement terminators**: every declaration, assignment, expression statement, `return`, `print(...)`, and `import` ends with `;`. Block statements (`if`, `while`, `for`, function bodies, etc.) do not.
16. **Stdlib collection mutators return new values — reassign.** `Map.set`, `Map.remove`, `Set.add`, `Set.remove`, `Stack.push`, `Stack.pop`, `Queue.enqueue`, `Queue.dequeue` all return a new collection. `map.set("k","v");` looks fine but throws away the result. Write `map = map.set("k", "v");`. Same for sets, stacks, queues. Constructors are `collections.new_map()`, `collections.new_set()`, `collections.new_stack()`, `collections.new_queue()` — not `Map.new()` or `HashMap.new()`. Membership is `map.has(k)`, not `contains_key`.
17. **`fun main()` must be invoked.** Define it at the top of the file, then call it on the last line: `main();`. Top-level statements run, but a function defined and never called doesn't.

## Anatomy of a Raven file

```raven
// imports first
import "board";              // local file (./board.rv or src/board.rv)
import math;                 // stdlib module — call as math.sqrt(...)
import str from "str";       // stdlib alias — call as str.to_upper(...)
import { trim, contains } from "str";  // selective — bare names

// types
struct Point {
    x: float,
    y: float,
}

// methods on a struct
impl Point {
    fun distance_from_origin(self) -> float {
        return math.sqrt(self.x * self.x + self.y * self.y);
    }
}

// exports for other modules
export fun midpoint(a: Point, b: Point) -> Point {
    return Point {
        x: (a.x + b.x) / 2.0,
        y: (a.y + b.y) / 2.0,
    };
}

// program entry — call main() at top level
fun main() -> void {
    let p: Point = Point { x: 3.0, y: 4.0 };
    print(format("distance = {}", p.distance_from_origin()));
}
main();
```

A few rules implicit in this:

- **Modules**: imported by file stem; `import "board"` finds `./board.rv` or in `src/`. `import math` reaches into the stdlib (`lib/math.rv`).
- **Top-level execution**: top-level statements run top-to-bottom. The convention is to put the program in `fun main()` and call `main();` on the last line.
- **Trailing commas** are allowed inside struct definitions/instantiations and array literals.
- **Exported items** become available in the importer's scope; with `import "x"`, names are merged as if declared locally (so `starting_cells()` defined in `rules.rv` is callable as `starting_cells()` from another file that did `import "rules";`). With `import x;` (stdlib-style), exports live under the namespace `x.fn()`.

## Templates

Working code you can adapt. These compile.

### Iterate an array

```raven
let xs: int[] = [10, 20, 30];
for (let i: int = 0; i < len(xs); i = i + 1) {
    print(format("xs[{}] = {}", i, xs[i]));
}
```

### Build an array

```raven
fun ints_up_to(n: int) -> int[] {
    let out: int[] = [];
    for (let i: int = 0; i < n; i = i + 1) {
        out.push(i);
    }
    return out;
}
```

### Array mutation pitfall

This works (push on a local):

```raven
fun seven() -> int[] {
    let m: int[] = [];
    m.push(7);
    return m;
}
```

This may **not** mutate the caller's array, depending on context:

```raven
fun add_seven(m: int[]) -> void {
    m.push(7);   // may mutate only the local copy
}
```

Safe pattern — return and concat:

```raven
fun pawn_moves(board: Cell[][], r: int, c: int) -> int[] {
    let m: int[] = [];
    m.push(r); m.push(c); /* … */
    return m;
}

fun all_moves(board: Cell[][]) -> int[] {
    let out: int[] = [];
    for (let r: int = 0; r < 8; r = r + 1) {
        let part: int[] = pawn_moves(board, r, 0);
        for (let i: int = 0; i < len(part); i = i + 1) {
            out.push(part[i]);
        }
    }
    return out;
}
```

### Struct with methods

```raven
struct Counter {
    n: int,
}

impl Counter {
    fun inc(self) -> void {
        self.n = self.n + 1;
    }

    fun get(self) -> int {
        return self.n;
    }
}

let c: Counter = Counter { n: 0 };
c.inc();
c.inc();
print(c.get());   // 2
```

`self` mutations persist when the method is called on a **variable**. They don't persist when called on a freshly-constructed expression like `Counter { n: 0 }.inc()`.

### Enums

```raven
enum Status {
    Pending,
    Active,
    Done,
}

let s: Status = Status::Active;
if (s == Status::Active) {
    print("running");
}
let parsed: Status = enum_from_string("Status", "Done");
```

No payloads, no pattern matching. Compare with `==`.

### If / elseif / else

```raven
if (x < 0) {
    print("negative");
} elseif (x == 0) {
    print("zero");
} else {
    print("positive");
}
```

Keep `} elseif (` and `} else {` together. No comments in the gap.

### Read input, parse, branch

```raven
let line: string = input("> ");
let n: int = parse_int(line);   // returns 0 if not a number
if (n == 0) {
    print("not a positive number");
} else {
    print(format("got {}", n));
}
```

### Reading and writing files

```raven
if (file_exists("data.txt")) {
    let body: string = read_file("data.txt");
    let lines: string[] = body.split("\n");
    print(format("{} lines", len(lines)));
}
write_file("out.txt", "hello\nworld\n");
append_file("out.txt", "more\n");
```

### Pseudo-randomness without `import math` if you can't

Raven's `math.random()` exists, but if you need a quick non-cryptographic random pick without an import, the fractional part of `sys_timestamp()` works:

```raven
let ts: float = sys_timestamp();
let frac: int = parse_int(format("{}", ts).split(".")[1]);
if (frac < 0) { frac = -frac; }
let pick: int = frac % len(options);
```

### A small program with modules and a CLI loop

```raven
// src/greet.rv
export fun salutation(name: string) -> string {
    return format("Hello, {}!", name);
}

// src/main.rv
import "greet";

fun main() -> void {
    let name: string = input("name: ");
    print(salutation(name));
}
main();
```

Run with `rvpm run` from the project root (the directory containing `rv.toml`).

## Built-in cheatsheet

These are always available — no import needed. See `references/builtins.md` for full signatures.

| Need                     | Use                                                     |
| ------------------------ | ------------------------------------------------------- |
| Print to stdout          | `print(template, args...)` — `{}` placeholders          |
| Build a string           | `format(template, args...)`                             |
| Read a line              | `input(prompt)`                                         |
| Length                   | `len(arr)` or `len(string)`                             |
| Type as string           | `type(value)`                                           |
| Parse integer            | `parse_int(s)` (returns 0 on failure, no exception)     |
| ASCII code               | `char_code("A")` → 65                                   |
| Crash with a message     | `panic("...")`                                          |
| Files                    | `read_file`, `write_file`, `append_file`, `file_exists` |
| Dirs                     | `list_directory`, `is_dir`, `create_directory`          |
| Time / date / timestamp  | `sys_time()`, `sys_date()`, `sys_timestamp()`           |
| HTTP                     | `http_fetch(method, url, headers, body)` → response     |
| TCP                      | `tcp_listen`, `tcp_accept`, `tcp_read`, `tcp_write`     |
| Enum from string         | `enum_from_string("Type", "Variant")`                   |

## Stdlib at a glance

These live in `lib/*.rv` and are imported with `import math;` etc. See `references/stdlib.md` for full per-function signatures.

| Module        | What's there                                                           |
| ------------- | ---------------------------------------------------------------------- |
| `math`        | `PI`, `E`, `TAU`; `sqrt`, `pow`, `sin`, `cos`, `random`, `random_int`, `clamp`, `lerp`, … |
| `str`         | `to_upper`, `trim`, `contains`, `starts_with`, `pad_left`, `repeat`, … |
| `collections` | `Map`, `Set`, `Stack`, `Queue` with constructors and methods           |
| `filesystem`  | path helpers (`join_path`, `dirname`, `basename`, `extension`), recursive listing, line-based file I/O |
| `json`        | `validate`, `parse`, `get_value`, `minify`, `pretty`                   |
| `time`        | time/date helpers beyond the built-ins                                 |
| `web`/`network` | HTTP and TCP convenience wrappers                                    |
| `testing`     | assertion helpers for tests                                            |

If a function you want isn't here, **read the actual `.rv` file** under `lib/` — the source is short and the truth.

## Running and project layout

```
my_project/
├── rv.toml          # [package] / [dependencies] / optional [fmt]
└── src/
    └── main.rv      # entry point — must call main() at the bottom
```

```bash
rvpm init my_project   # scaffold
rvpm run               # finds nearest rv.toml and runs src/main.rv
rvpm fmt               # format src/ in place
rvpm fmt --check       # CI: exit non-zero if anything would change

raven file.rv          # run directly
raven file.rv -c       # type-check only (super useful before running)
```

`rvpm install` and `rvpm add` are declared but **not yet implemented** — don't suggest them as a working solution.

The `raven` binary must be on PATH for `rvpm run` to delegate. On developer machines that's usually `target/release/raven` from the compiler repo.

## Recommended workflow when writing Raven

1. Decide module layout (`src/main.rv` plus helpers).
2. Sketch types first: structs, enums.
3. Implement helpers as small `export`ed functions.
4. Write `fun main()` and call it on the last line of `main.rv`.
5. Run `raven src/main.rv -c` (or `rvpm run` if you want execution). Type-check first — it's fast and catches the bulk of mistakes.
6. If you hit a parse error, the most likely cause is one of the pitfalls in the list at the top: `elseif`, comment-before-else, missing type annotation, missing `;`.
7. Run `rvpm fmt` before committing.

## Style

- 4-space indent.
- Order in a file: imports → types (struct/enum/impl) → free functions → top-level invocation.
- One blank line between top-level declarations.
- Name functions in `snake_case`, types in `PascalCase`, constants and variables in `snake_case`.
- Prefer `export fun` for anything another module needs; keep helpers private (no `export`).
- Use `format("{}", x)` over string `+` concatenation when mixing types.
- Keep `if`/`elseif`/`else` chains tight: no blank lines or comments between branches.

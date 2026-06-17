# Raven grammar and parser notes (v2)

The reference for "what does the parser actually accept?" Use it when something parses surprisingly.

The authoritative source is `src/lexer/`, `src/parser/`, `src/ast/`, and `src/tycheck/` in the Raven repo. Read those when this document and the compiler disagree.

---

## Lexical structure

### Keywords (reserved)

```
let const fun struct impl enum trait
if else while for loop in return match
break continue defer spawn import as extern macro
true false
```

Type names (`Int`, `Float`, `Bool`, `String`, `Char`, `Unit`, `List`, `Map`, `Set`, `Option`, `Result`, the `C*` FFI types) are PascalCase identifiers, not keywords. `const` declares an immutable compile-time constant and parses both inside a function body (`const X = 5`) and at module level (`const MAX: Int = 100`).

### Identifiers

`[A-Za-z_][A-Za-z0-9_]*`, case-sensitive. Types and enum variants are conventionally PascalCase; functions and variables snake_case.

### Literals

| Kind     | Examples                                              |
| -------- | ---------------------------------------------------- |
| Int      | `0`, `42`, `-5` (unary minus)                        |
| Float    | `0.5`, `3.14`, `1.0`                                 |
| Bool     | `true`, `false`                                      |
| Char     | `'R'`, `'\n'`                                         |
| String   | `"hello"`, `"with ${expr}"`, `"""block string"""`    |
| C string | `c"hello"` (null-terminated, FFI)                    |
| List     | `[]`, `[1, 2, 3]`                                    |
| Map      | `["a": 1, "b": 2]`, `[:]` (empty)                    |
| Set      | `{1, 2, 3}`                                          |

### String interpolation

`"${expr}"` splices `expr` into the string. The expression can be arbitrary:

- A nested `"`-delimited string literal is fine: `"${greet("x")}"`.
- A macro invocation is fine: `"${m!(1)}"`.
- A struct value that derives `ToString` interpolates directly: `"${v}"` (equivalent to `"${v.to_string()}"`).

### Comments

- `// to end of line`
- `/* block, does not nest */`

### Operators

```
Arithmetic:  + - * / %
Comparison:  == != < > <= >=
Logical:     && || !          (the words and/or/not are NOT operators)
Bitwise:     & | ^ << >>
Assignment:  =  +=  -=  *=  /=  %=  &=  |=  ^=  <<=  >>=
Range:       ..  (exclusive)   ..=  (inclusive)
Error prop:  ?    (postfix, on Result/Option)
Arrow:       ->   (function return type)
Match arm:   ->   (arm body)     (arms separated by commas or newlines)
Member:      .    (field, method, and qualified enum: EnumName.Variant)
Generic:     <T>  at definitions and call sites (no turbofish ::<>)
Attribute:   @derive(...), @repr(C)
Macro call:  name!(...)
```

There is no statement terminator. Newlines and braces separate statements. A `;` is not used.

---

## Operator precedence (low to high)

1. `||`
2. `&&`
3. `== !=`
4. `< > <= >=`
5. `| ^ &` (bitwise)
6. `<< >>`
7. `+ -`
8. `* / %`
9. unary `-`, `!`
10. postfix `?`
11. call / index / field / method chain: `()`, `[]`, `.`

Use parentheses to override. `&&` and `||` short-circuit.

---

## Items (top level)

A program is a set of top-level items. Order does not matter for resolution (functions, types, and traits are visible across the whole module). `fun main()` is the entry point and is called by the runtime, not by you.

### Variable declaration (inside function bodies)

```
let <ident> = <expr>
let <ident>: <type> = <expr>
const <ident> = <expr>
const <ident>: <type> = <expr>
```

`let` bindings are mutable and reassignable; `const` bindings are immutable. Both also work at module level (though reassigning a module-level binding from a function mis-compiles). Empty collection literals need a type: `let xs: List<Int> = []`.

### Function declaration

```
fun <ident>(<params>) -> <type> { <stmts> }
fun <ident>(<params>) { <stmts> }              // returns Unit
fun <ident>(<params>) -> <type> = <expr>       // single-expression body
fun <ident><<generics>>(<params>) -> <type> { ... }   // generic, with optional bounds <T: Trait>
```

`<params>` are comma-separated `name: Type`.

### Struct

```
struct <Name> {
    field1: <Type>,
    field2: <Type>,
}
struct <Name><<generics>> { ... }
struct <Name> {}                  // empty
```

Construct with `Name { field: value, ... }`. Field punning is allowed: `Name { x }` when a local `x` is in scope.

### Trait and impl

```
trait <Name> {
    fun method(self) -> <Type>
    fun method2(self, other: <Type>) -> <Type>
}

impl <Name> { fun inherent(self) -> <Type> { ... } }     // inherent methods
impl <Trait> for <Name> { fun method(self) -> <Type> { ... } }   // trait impl
impl<T> <Name><T> { ... }                                 // generic impl
```

The first method parameter is `self`. Static (associated) methods omit `self`: `fun new() -> <Name> { ... }`, called as `Name.new()`.

### Enum (sum type)

```
enum <Name> {
    UnitVariant,
    Tuple(Type, Type),
    Single(Type),
}
```

Construct qualified: `Name.UnitVariant`, `Name.Tuple(a, b)`. Match patterns are bare: `Tuple(a, b) -> ...`.

### Import

```
import std/<module>                 // bundled stdlib module (types/methods merge)
import std/<module> { a, b }        // selective: free functions and types by name
import "./relative"                 // sibling file ./relative.rv
import "./relative" { a, b }
import "github.com/user/repo"       // package root (lib.rv), fetched by rvpm
import "github.com/user/repo" { a } // selective
import "github.com/user/repo/sub" { a }   // sub-module sub.rv
```

Importing a free function by name (`import std/fs { write }` then `write(...)`) is the form that always works. A `module.function()` qualified call works for some runtime-backed modules (`import std/fs` then `fs.exists(...)`) but not all (`import std/math` then `math.sqrt(...)` is rejected), so prefer the selector form.

### extern (C FFI)

```
extern "C" {
    fun c_function(x: CInt) -> CInt
}
```

### macro

```
macro <name> { (<matcher>) => { <template> } }
```

Metavariables `$x:expr`, `$x:ident`; repetition `$(...)*` and `$(...)+`. Invoke as `name!(args)`.

### Attributes

```
@derive(Eq, Hash, Ord, ToString, Debug, ToJson, FromJson)
@repr(C)
```

Placed immediately before a `struct` or `enum`.

---

## Statements and expressions

### Control flow

```
if <cond> { ... }
if <cond> { ... } else if <cond> { ... } else { ... }
// `if` is also an expression: let s = if c { a } else { b }

while <cond> { ... }
loop { ...; break }
for <name> in <iterable> { ... }        // 0..n, 1..=n, or a List

break
continue
return <expr>
return
defer <expr>                            // runs at function exit, LIFO
```

### match

```
match <scrutinee> {
    Pattern1 -> <expr>,
    Pattern2(a, b) -> { <stmts> },
    Some(x) -> ...,
    None -> ...,
    _ -> ...,            // wildcard
}
```

Exhaustive at compile time. Arms are separated by commas (and/or newlines). Matching on `String` literal patterns works (`match s { "a" -> ..., _ -> ... }`).

### Closures

```
fun(x: Int) -> Int = x * 2        // expression-bodied closure
fun(x: Int) -> Int { return x*2 } // block-bodied
```

Closures capture their environment.

### Calls, fields, indexing

```
f(a, b)
obj.field
obj.method(args)
xs[i]
matrix[i][j]
xs.iter().map(...).filter(...)
Name.StaticMethod(args)
Enum.Variant(args)
generic_fn<Int>(x)
```

---

## Type system summary

- Primitives: `Int` (i64), `Float` (f64), `Bool`, `String`, `Char`, `Unit`.
- Collections: `List<T>`, `Map<K, V>`, `Set<T>`.
- Sum/optional: `Option<T>` (sugar `T?`), `Result<T, E>`.
- User types: `struct`, `enum`, plus `trait` bounds on generics.
- Trait objects: `dyn Trait` (single value; `List<dyn Trait>` is not yet supported).
- FFI: `CInt`, `CLong`, `CSize`, `CFloat`, `CDouble`, `CStr`, `CPtr<T>`, `CFnPtr`, `Any`, `Channel`.

Generics monomorphize (static dispatch). Type arguments use `<T>` at the call site, never turbofish.

### Conversions

No implicit numeric coercion and no cast syntax. Build strings with interpolation (`"${x}"`) or `to_string()` where derived. Parse with the relevant stdlib function (returning `Result`/`Option`).

### Equality and ordering

`==`/`!=` work on matching types (and on `String` by value). Ordering `< <= > >=` works on numbers and on `String` (lexicographic). Structs and enums can derive ordering with `@derive(Ord)`, which provides `compare(self, other) -> Int`.

---

## What exists in v2 that did not in v1

Reach for these freely now: generics, traits, `dyn Trait`, sum types with payloads, exhaustive `match`, `Option`/`Result`/`?`, closures, `break`/`continue`, compound assignment (`+=` …), `else if`, `for x in xs`, string interpolation, `defer`, goroutines (`spawn`) and channels, a C FFI, `@derive`, declarative macros, and compile-time/runtime reflection.

## Things that still don't work / don't exist

- `null`, `nil`, `None`-as-keyword (use the `None` value of `Option`).
- Module-level mutable state: a top-level `let`/`const` is readable from any function, but reassigning it from a function mis-compiles (`binop lhs used a Unit value`). Keep mutable state inside functions or a struct.
- Universal `module.function()` qualified calls (works for some runtime-backed modules, not all; prefer selector imports).
- `String.len()` (use `.length()`).
- `List<dyn Trait>`.
- Turbofish `::<T>`.
- Exceptions / `try`/`catch` (use `Result` and `?`).

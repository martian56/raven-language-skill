# Raven grammar and parser notes

This is the reference for "what does the parser actually accept?" Use it when something parses surprisingly or you need to write code that hits an edge case.

The authoritative source is `src/lexer.rs`, `src/parser.rs`, `src/ast.rs`, and `src/type_checker.rs` in the Raven repo. Read those when this document and the parser disagree.

---

## Lexical structure

### Keywords (all lowercase, reserved)

```
let const fun struct impl enum export
if elseif else while for return
import from
print
and or not
true false
int float bool string void
```

`const` is reserved by the lexer but **not handled** by the parser — using it gives an "Unexpected token" error.

### Identifiers

`[A-Za-z_][A-Za-z0-9_]*`. Case-sensitive.

### Literals

| Kind    | Examples                          |
| ------- | --------------------------------- |
| int     | `0`, `42`, `9223372036854775807`  |
| float   | `0.5`, `3.14`, `1.0`              |
| bool    | `true`, `false`                   |
| string  | `"hello"`, `""`, `"with \"quote\""`|
| array   | `[]`, `[1, 2, 3]`                 |

No scientific notation. No hex/oct/bin literals. No char type — character data is single-character strings.

### Negative numbers

`-5` lexes as two tokens: `Minus` then `IntLiteral(5)`. Resolved as unary minus during parsing.

### String escapes

Recognised: `\n`, `\r`, `\t`, `\0`, `\"`, `\\`. Anything else (e.g. `\x`) is preserved literally.

### Comments

- `// to end of line`
- `/* block — does not nest */`

Block comments end at the **first** `*/`; nesting is not supported.

### Operators

Arithmetic: `+ - * / %`
Comparison: `== != < > <= >=`
Logical: `&& || !` and the word forms `and or not`
Assignment: `=`
Type annotation: `:`
Statement end: `;`
Member access: `.`
Enum variant: `::`
Function return arrow: `->`
List separator: `,`
Range / slicing: `..` (parsed but currently used in slicing-style contexts only)

---

## Operator precedence (low to high)

1. `||` / `or`
2. `&&` / `and`
3. `==` `!=`
4. `<` `>` `<=` `>=`
5. `+` `-` (binary)
6. `*` `/` `%`
7. unary `-`, `!`, `not` (prefix, right-associative)
8. method/field/index chain — `.`, `[]`, `()`

Use parentheses to override.

**Both sides of `&&` and `||` are evaluated.** Short-circuit semantics are not implemented. This affects guard idioms like `if (i < len(a) and a[i] == 0)` — the second clause runs even when the first is false, so `a[i]` will out-of-range if `i >= len(a)`.

---

## Top-level statements

A program is a sequence of statements at module scope. They execute top to bottom, so functions/structs/enums must be defined before they are first called.

### Variable declaration

```
let <ident>: <type> = <expr>;
let <ident>: <type>;            // only valid for struct types — primitives must have an initialiser
```

### Function declaration

```
fun <ident>(<params>) -> <return_type> { <stmts> }
fun <ident>(<params>) { <stmts> }                // return type defaults to void
```

`<params>` is comma-separated `name: type` pairs. Trailing comma not necessary. The arrow `->` is required when an explicit return type is given.

### Struct declaration

```
struct <Name> {
    field1: <type>,
    field2: <type>,
}
```

Trailing comma allowed. Comments inside the body are preserved by the formatter.

### Impl block (methods)

```
impl <StructName> {
    fun <method>(self, <other params>) -> <type> { ... }
    ...
}
```

The first parameter MUST be `self`. Type annotation on `self` is optional (inferred). Methods chain by calling on the receiver.

### Enum declaration

```
enum <Name> {
    Variant1,
    Variant2,
}
```

No payloads. Construct with `Name::Variant1`. Compare with `==`.

### Import

```
import "module";                       // file lookup, exports merged into scope
import math;                           // bare identifier, namespaced as math.fn()
import alias from "module";            // namespaced under alias
import { f1, f2 } from "module";       // selective
```

### Export

```
export fun ...
export let ...
```

Wraps a declaration to make it visible to importers.

### Control flow

```
if (<cond>) { ... }
if (<cond>) { ... } elseif (<cond>) { ... }
if (<cond>) { ... } elseif (<cond>) { ... } else { ... }

while (<cond>) { ... }

for (let <name>: <type> = <init>; <cond>; <name> = <expr>) { ... }
```

`elseif` is **one** word. `else if` does not parse. Comments between `}` and `elseif`/`else` do not parse.

The for-loop init must be a `let` declaration with a type annotation. The increment must be an assignment. There's no for-in form.

No `break`, no `continue`. Exit early by setting a boolean flag and reading it in the loop condition, or by `return`ing out of the enclosing function.

### Return

```
return <expr>;
return;          // valid in void functions
```

### Print (statement form)

```
print(<args>);
```

Same as the function call.

### Expression statement

Any expression followed by `;`.

---

## Expressions

### Function calls

```
f()
f(a, b, c)
```

Arguments are evaluated left to right before the call.

### Method / field access

```
obj.field
obj.method()
obj.method(args)
arr[i]
arr[i].method()
matrix[i][j]
chain().of().calls()
```

The parser greedily continues with `.`, `[]`, and `()` until none applies.

### Struct construction

```
Name { field: value, field: value }
```

All fields required, order doesn't matter.

### Enum variant

```
EnumName::VariantName
```

### Array literal

```
[]
[1, 2, 3]
```

Empty literal needs the variable to be typed (`let xs: int[] = [];`). Nested literals (`[[1,2],[3,4]]`) need the variable to be `int[][]`.

---

## Type system summary

Types accepted in annotations:

- Primitive: `int`, `float`, `bool`, `string`, `void`
- User-defined: any in-scope `struct` or `enum` name
- Array: append `[]` for each dimension. `int[]`, `string[][]`, `Cell[][]`

No generics, no union types, no optionals.

### Conversions and coercions

| From → To       | Implicit? | How to do it explicitly |
| --------------- | --------- | ----------------------- |
| `int` → `float` | No        | `x + 0.0` (in arithmetic, the result is float) |
| `float` → `int` | No        | None built in. Use `math.floor` then accept loss of float-ness, or do it via string |
| anything → `string` | No (except in `+` with a string operand) | `format("{}", x)` |
| `string` → `int` | No        | `parse_int(s)` (returns 0 on failure) |

`+` with a string operand on either side concatenates and stringifies the other operand — this is the one implicit conversion. Other arithmetic operators error on type mismatches.

### Equality

`==` and `!=` work on all types and require both sides to have the same type.

### Order comparisons

`<`, `>`, `<=`, `>=` require numeric types (int or float).

---

## Runtime semantics worth knowing

### Scope

There are no block scopes. Variables declared inside `if`/`while`/`for` bodies live in the enclosing function's scope. Re-declaring with the same name overwrites.

### Functions

Calls save the caller's variable map, install the parameters as locals, run the body, then restore. There are no closures — functions cannot capture variables from enclosing scopes.

### Method calls and `self`

When calling `obj.method()` where `obj` is a variable, mutations to `self` inside the method persist on `obj`. When called on an expression (e.g. a freshly-constructed struct), the mutated `self` is discarded.

### Arrays

Cloned on assignment and on parameter passing. `arr.push(x)` on a local variable mutates that local. `arr.push(x)` inside a function that received `arr` as a parameter typically mutates only the local copy — design helpers to **return** their array instead of mutating.

### Strings

Immutable. All methods return new strings.

### Modules

Loaded once and cached. The first `import "x"` parses and runs `x.rv` to completion (top-level side effects happen). Subsequent imports just bring symbols into scope.

---

## Things that don't exist

A short list of "don't reach for these":

- `null`, `nil`, `None`, `undefined`
- `try`/`catch`, exceptions, `Result`/`Option`
- Generics
- Pattern matching
- Closures / lambdas
- `break`, `continue`
- `++`, `--`, compound assignment (`+=`, `-=`, etc.)
- `else if` (use `elseif`)
- `for x in xs` (use a C-style index loop)
- Single-statement `if`/`while`/`for` bodies (always need `{ }`)
- Type casting syntax — no `(int)x`, no `x as int`
- Operator overloading
- Visibility modifiers other than `export`
- Default parameter values
- Variadic parameters in user functions (`format`/`print` are special-cased in the interpreter)
- String interpolation outside of `format`/`print` (no `${}` or `f"..."`)

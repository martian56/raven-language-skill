# Raven grammar notes (3.0)

Read this when something parses surprisingly. The full reference is
`docs/guide/language-reference.md` in the Raven repo.

## Lexical structure

- Keywords: `let var fun return if else while for in loop break continue
  match case class enum trait impl import from as pub extern defer spawn
  and or not true false self Self macro dyn`. `assert` is a statement only
  at the start of a statement, so it still works as a variable name.
- Identifiers: letters, digits, `_`; snake_case for values, PascalCase for
  types. A class name may start with `_`.
- Literals: `42`, `0xff`, `0b1010`, `0o17`, `1_000_000`, `3.14`, `'c'`,
  `"text"`, `"""raw, keeps newlines"""`, `c"C string"` (FFI), `true`, `()`.
- Strings: escapes `\n \t \\ \" \u{1F600}`; `${expr}` interpolates and may
  contain nested strings; `${x:.2}`, `${s:>8}`, `${n:05}`, `${s:*^9}`
  format.
- Comments: `// line` and `/* block */`.
- Newlines end statements; `;` is optional. A line continues after an
  operator, a comma, or an open bracket.

## Operator precedence (low to high)

1. `a if cond else b`
2. `or`
3. `and`
4. `not`
5. `==  !=  <  <=  >  >=  in  not in` (comparisons do not chain)
6. `|`  then `^` then `&`
7. `<<  >>`
8. `..  ..=`
9. `+  -`
10. `*  /  %`
11. unary `-`
12. `**` (right associative; `-x ** 2` is `-(x ** 2)`)
13. `as`
14. postfix: call `f(x)`, field `x.f`, index `x[i]`, `?`

## Declarations

```text
let name = expr                     immutable binding (module level: constant
let name: T = expr                    when expr is a literal expression)
var name = expr                     reassignable
fun name[T: Bound](a: T, b: int = 3) -> R { ... }
fun name(x: int) -> int = x * x     one-expression body
class Name[T]: TraitA, TraitB {
    field: T = default
    fun method(self) -> R { ... }   method; no self means static: Name.f()
}
enum Name { Unit, Tuple(int, str), Named(x: float) }
trait Name { fun req(self) -> R  fun default(self) -> R { ... } }
impl Trait for Type { ... }         for a type declared elsewhere
impl Type { ... }                   extra methods on any type, even `int`
from std.json import parse, stringify as dump
import std.fs                       then fs.read(...)
from "./models" import User         local file models.rv
from "github.com/user/repo" import Thing
pub fun / pub class                 accepted; every top-level item is importable
extern "C" { fun puts(s: CStr) -> CInt }
@derive(Ord, ToJson, FromJson)      before a class or enum
@repr(C)                            C memory layout for FFI
macro name { ($x:expr) => { ... } }
```

## Statements

```text
if c { } else if d { } else { }
while c { }       loop { }       for x in xs { }       for (k, v) in d { }
break / continue / return expr
match v { case P { } case Q if guard { } case _ { } }
if let Some(x) = opt { }          while let Some(x) = next() { }
let Some(x) = opt else { return }
let (a, b) = pair                 (a, b) = (b, a)
x op= v                           for + - * / % & | ^ << >>
defer expr                        runs when the function returns
spawn { }                         goroutine
assert cond, "message"
```

`if` and `match` are statements; use `a if c else b` for a value.

## Patterns

Literals (`0`, `"a"`, `'c'`, `true`), ranges (`1..=9`), `_`, bindings,
qualified or bare variants (`Shape.Circle(r)`, `Some(x)`, `None`), tuples
`(a, _)`, nesting `Some(Ok(n))`, and guards `case n if n > 0`. Matches are
checked for exhaustiveness, including through nested patterns.

## Expressions

- Construction: `Point(1.0, 2.0)`, `Point(x: 1.0)`, `Point(..p, y: 0.0)`.
- Lambdas: `x => x + 1`, `(a, b) => a + b`, `(x: int) -> int => x * 2`,
  `() => { ... }`. A lambda's parameter types come from the expected
  function type or from later calls.
- Collections: `[a, b]`, `{k: v}`, `{a, b}`, `{a}` (set), `{}` (dict),
  `set()`, `(a, b)` tuple, `()` unit; comprehensions
  `[e for x in xs if c]`, `{e for ...}`, `{k: v for ...}`.
- Indexing and slicing: `xs[i]`, `xs[-1]`, `xs[a..b]`, `xs[a..]`, `xs[..b]`,
  `xs[a..=b]`, `d[k]`.
- `t.0` reads a tuple item; `?` propagates `Err`/`None` (in `main` it stops
  the program with the error).

## Types

`int float bool str char`, `list[T] dict[K, V] set[T]`, `Option[T]` (also
`T?`), `Result[T, E]`, tuples `(A, B)`, function types `fun(A, B) -> R`,
class/enum names with arguments `Box[int]`, a trait as a type (a trait
object, also spelled `dyn Trait`), and C types for FFI (`CInt`, `CLong`,
`CDouble`, `CStr`, `CPtr[T]`, ...). There are no implicit conversions;
use `as` or `int()`/`float()`/`str()`.

## Not in 3.0

Sized integers (`i32`, `u8`) and overflow checks, `from x import *`,
inheritance between classes, value (inline) classes, exceptions.

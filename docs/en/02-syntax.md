# Syntax

## Keywords

| Keyword | Meaning |
|---|---|
| `let` | create a variable |
| `fn` | computation: returns an answer, the answer must be consumed |
| `sub` | action: returns a check — information about where the data went |
| `link` | create a reference (reading, dies on `mut`) or connect a file (`link file = "file.lang"`) |
| `link&` | long-lived reference to a slot (survives `mut`, the slot's existence is checked at runtime) |
| `link*` | long-lived reference with freeze (the data cannot be changed while the reference is alive) |
| `mut` | change |
| `box` | use the data (the compiler decides copy/move) |
| `const` | constant (the value is known at compile time) |
| `use` | import a module (std, packages) |

## Variables

```text
let name = value
let name: type = value
```

An explicit type is written with a colon:

```text
let x = 10
let user: User = get_user()
```

A created variable can be changed — the right exists because you created it:

```text
mut x = 15
```

## Strings

`"..."` is a string literal of type `str`. There are no other string literals.

`str` is a sequence of characters: `s[i]` yields a `str` of one character, `s.len` is the number of characters. There is no separate `char`, and no `String` either.

```text
let s: str = "abc"
let c = s[0]        // c: str, "a"
```

## Structs

A struct is created only like this:

```text
struct Person box {
    x,
}
```

`Person` is the struct name, `x` is a variable. A struct holds variables with data, not separate fields: a field name is a variable name. There is no need to write the type — it is already known from the variable. If it is written explicitly, it must match the variable's type. `box` is mandatory.

## Computations and actions

Two different entities, not interchangeable:

```text
fn name(params) -> type { ... }
sub name(params) -> type or E { ... }
```

`fn` declares a computation: it returns an answer. `sub` declares an action: with `or E` it returns a check — information about where the data went. The `fn` answer must be consumed.

```text
main() -> Sys

fn sum(box a: i32, box b: i32) -> i32 {
    a + b
}

sub main() {
    let r = sum(box(3, 5))
    ignore!(print(r))
}
```

- The answer of `fn` must be consumed: bound, used, passed, or returned. A bare `fn` call is an error.
- The check of `sub` is information about where the data went. Needed — bind it or handle it; not needed — do not take it, or `ignore!`.
- A `sub` without `or E` cannot fail: there is no error, the call is a statement.
- `ignore!` explicitly does not take the check: `ignore!(print(x))`.
- `out` is a parameter for writing into someone else's memory: `read_into(path, out(buf))`.
- `fn` does not call `sub`: a computation does not perform actions. Calls go in one direction only — `sub` calls `fn`, but not the other way around.
- `fn` can return `T or E`; the caller must handle the result at the call site via `switch` or `match`. There is no propagation.
- `fn` can be a value of type `Fn`; `sub` is not a value.
- The entry is declared at the top of the file: `main() -> Sys`. It says where the data goes, not a return type. `Sys` is a sink, not a type. For embedded systems the sink is different, its name is provided by the platform.

## Sugar

A form with `!` is a convenient notation, not a new entity:

```text
getline!(line)
```

The expansion is visible in the IDE and the documentation. The tag comes from the expansion: if an action is inside, the form is an action, and it cannot be called from an `fn`.

## Templates and blueprints

A template is a ready function with a concrete signature: all types are known, it can be called as is.

A blueprint is a function without concrete types: placeholders are left instead of them. There is no separate syntax; once the placeholders are filled, a template results.

Blueprints are not part of v1.

## Conditionals

```text
if cond { ... } else if ... { ... } else { ... }
```

`if` is an expression (more explicit than a ternary):

```text
let value = if cond { 1 } else { 2 }
```

## Switch

```text
switch x {
    1 => ...,
    else => ...
}
```

`switch` is an expression, it yields a value. It compares ordinary values: numbers, strings, variants without data. The default branch is `else`. Branching (running different code without a value) is done via `if`/`else if`.

## Enum

An `enum` is a choice: one of the listed items, a second one cannot be chosen. A variant may carry its own data:

```text
enum Shape {
    Circle(f64),
    Empty,
    Rect(f64, f64),
}
```

`Circle` carries one number, `Rect` two, `Empty` none. The data belongs to the variant: `Circle` has only a radius, `Rect` only width and height. A circle with a width cannot be built.

Without data a variant is just a label: `enum TokenKind { Num, Add, Mul }`.

## Match

```text
let area = match shape {
    Circle(r) => 3.14 * r * r,
    Rect(w, h) => w * h,
    Empty => 0,
}
```

`match` is an expression. It looks at the variant and gives access to that variant's data: `r`, `w`, `h` are names for reading, they live only inside their branch. The data is not copied or taken, `match` only reads.

- The match must be complete: all variants listed. Forgetting a branch is a compile error. `else` is not needed.
- Inside a branch only its data is visible: in the `Circle` branch there is `r`, but no `w`.
- The names `r`, `w`, `h` are for reading only and do not escape the branch.

`switch` compares values, `match` destructures variants. For variants without data either works; `match` is needed where a variant has data.

## Loops

```text
for i in 0..10 { ... }      // range
for item in list { ... }    // over elements
while cond { ... }          // while the condition is true
```

Control:

```text
break
continue
```

## Zones

Code is divided into zones by `---`:

```text
--- zone 1
zone 1 code

--- zone 2
zone 2 code
```

- `---` is official language syntax.
- Zones are flat: exactly as written in the file, they are not nested anywhere and must not be.
- Zones do not affect code execution — this is pure structure, not behavior.

After `---` on the same line you can write anything — a zone name or description:

```text
--- reading a file
code

--- processing data
code
```

This is not a comment: the description is attached to the zone. The compiler sees it (parses it, does not report an error) and does nothing with it.

The main purpose of zones is to separate imports from receiving/passing and from code, so that this is super obvious. Right now the compiler parses zones. Tools can use the markup: for example, a separate test zone. In the future, the compiler may be able to mark zones in the AST into finer structures and interact with them.

The accepted order in a file: first the `use` zone (imports), then the `link file` zone (file connections), then code. This way `link` is not confused: in the imports zone `link file = "..."` is a file connection, while in code `link b = a` is a reference. This is the order the language suggests using.
# Syntax

## Keywords

| Keyword | Meaning |
|---|---|
| `let` | create a variable |
| `fn` | computation: returns an answer, the answer must be consumed |
| `sub` | action: no answer, called as a statement |
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
sub name(params) { ... }
```

`fn` declares a computation: it always has an answer. `sub` declares an action: no answer, the call is a statement. `let` is not needed in either place.

```text
fn sum(box a: i32, box b: i32) -> i32 {
    a + b
}

sub main() {
    let r = sum(box(3, 5))
    std.print(r)
}
```

- The answer of `fn` must be consumed: bound, used, passed, or returned. A bare `fn` call is an error.
- `sub` gives no answer. It cannot be assigned, passed, returned, or used in an expression. Only a call as a statement.
- `fn` can return `T or E`; the caller must immediately handle the result via `switch`. `sub` does not pass an error out.
- `fn` can be a value of type `Fn`; `sub` is not a value.

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

`switch` is an expression, it yields a value. Branching (running different code without a value) is done via `if`/`else if`. It covers 90% of pattern-matching needs.

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

This is not a comment: the description is attached to the zone. The compiler sees it (parses it, does not report an error) but does nothing with it — like a comment: it does not need to know, and that's it.

The main purpose of zones is to separate imports from receiving/passing and from code, so that this is super obvious. Right now the compiler parses zones, but they do not affect code execution — this is structure and convenience. Tools can use the markup: for example, a separate test zone. In the future, the compiler may be able to mark zones in the AST into finer structures and interact with them.

The accepted order in a file: first the `use` zone (imports), then the `link file` zone (file connections), then code. This way `link` is not confused: in the imports zone `link file = "..."` is a file connection, while in code `link b = a` is a reference. This is the order the language suggests using; if you do not want to, it is not forbidden, but then you will have to figure out for yourself where is what.
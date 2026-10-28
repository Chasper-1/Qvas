# Types

## Static, 100%

Types are static, 100%.

## Numeric types

Signed integers: `i8`, `i16`, `i32`, `i64`.

Unsigned integers: `u8`, `u16`, `u32`, `u64`.

Floating point: `f32`, `f64`.

`bool` and `str` are separate basic types.

Literals without context: integers — `i32`, floating point — `f64`.

There is no `usize`/`isize`: the bit width is fixed at build time, machine width is the compiler's business. Lengths and indices in v1 are `i32`.

## Strings

`str` is the only string type. There is no `String`: memory is automatic, a separate owned type is not needed.

A string is a sequence of characters. `s[i]` returns a `str` of one character, `s.len` is the number of characters.

```text
let s: str = "abc"
let c = s[0]        // c: str, "a"
```

There is no separate `char`. String literals are only in double quotes: `"..."`.

Byte access to a string (`s.bytes`, `u8`) is not in v1. `u8` remains a numeric type for bytes, but `str[i]` does not return a byte.

## Type labels

`i32:id` is an `i32` with the label `id`. Not a new type: there is no declaration, the representation and operations are the same. The label is checked statically and erased at runtime.

This is not a newtype. `type Id = i32` is an alias, not a new type. A separate struct is a new type. `i32:id` is neither: the type remains `i32`, the label is visible right on the type.

It is created by a value:

```text
let user_id: i32:id = 42
```

Now 42 is an `i32:id`. 42 does not cease to be an `i32` — all `i32` operations work. But the value has the label `id`.

A computation or action requests a type with a label:

```text
fn get_user(i32:id user_id) -> User { ... }
```

### Where it is checked

At boundaries: passing, returning, fields, assignment. The label must match. Inside an expression the value behaves as an `i32`, but at boundaries the label is checked.

| Situation | Allowed |
|---|---|
| `i32:id` → `i32:id` | yes |
| `i32:id` → `i32` | no: a named type where it is not expected |
| `i32` → `i32:id` | no: there is no label |
| `i32:id` → `i32:userId` | no: the labels differ |
| literal `42` → `i32:id` | yes: the literal accepts the expected label |

### Arithmetic

A plain `i32` is a neutral operand. The label is preserved.

| Expression | Result |
|---|---|
| `i32:id + i32` | `i32:id` |
| `i32 + i32:id` | `i32:id` |
| `i32:id + i32:id` | `i32:id` |
| `i32:id + i32:userId` | error |
| `i32 + i32` | `i32` |

The same for `-`, `*`, `/`, `%`.

### Comparisons

Labels do not participate. Values are compared, not types.

```text
i32:id == i32          // bool
i32:id < 100           // bool
i32:id == i32:userId   // bool
```

## Sources: `i32:=id`

`i32:=id` is not a type and not a label. It is a binding to a source: an `i32` from the variable `id` is accepted. "Who is speaking" is fixed: another source will not do.

It is used only at the place of acceptance:

```text
sub print(i32:=data) { ... }
```

Inside, the data is accessible by the name `data`. `i32:=data` cannot be used in `let`, `return`, as a struct field, in arithmetic, or in comparison.

The source can also be a function: `i32:id:=f()` — at the acceptance boundary, where the type and source match. The answer is accepted where it is expected, and nowhere else.

## Units of measurement

Units of measurement are just a label: `i32:meters`. There is no special unit arithmetic (meters², meters/second are not computed). You declared `i32:meters` — and pass it where needed. The contract is the label.

## Ranges are not types

A range check is a dumb comparison, easy to write explicitly:

```text
if x > 100 { ... }
```

A type label would hide the action and make it less clear, and would also allow the action to be performed in different ways. If needed, there is an explicit way to write about a range (a variable, a computation or action signature), not a type label.

## Principle

> Only what cannot be checked by ordinary code goes into types.

`i32:id` is a semantic label of a value, not checked by ordinary code → mandatory. `i32:=id` is a binding to a source, also not checked by ordinary code → mandatory. Ranges are checked by ordinary code (`if x > 100`) → not in types.

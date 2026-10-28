# Status

## Stage

The core concept is decided. Parallelism: the concept is fixed, the scheduling mechanism is in development. By blocks:

- types;
- `mut` + operators;
- nested references;
- error handling;
- structs;
- lists;
- computations and actions;
- external operations;
- computations as values;
- modules;
- basic syntax;
- parallelism;
- ecosystem;
- error messages;
- scopes.

What is closed: the language core, basic syntax, ecosystem, error messages, scopes. Parallelism: the concept is decided, the mechanism is in development.

## What is decided by blocks

- **Types** — static, 100%. Numeric: `i8/i16/i32/i64`, `u8/u16/u32/u64`, `f32/f64`; `bool`, `str`. `str` is a sequence of characters: `s[i]` → a `str` of one character, `.len` — the number of characters; there is no `char` and no `String`, byte access — later. Lengths and indices — `i32`, no `usize/isize`. Labels `i32:id` — a refinement of a value, not a new type; `i32:=id` — a binding to a source variable.
- **Structs** — a way to assemble several different values into one, accessed by field name.
- **Lists and arrays** — created and indexed as everywhere.
- **`mut` + operators** — `mut` and move do not mix: `mut` changes the value, a move changes the name responsible for the data. Either one or the other.
- **Invalidation of nested references** — by path: a mutation kills references whose path intersects the changed one. `mut a[1]` does not kill the `link` on `a[0]`, but kills references to `a` as a whole.
- **References do not escape** — a computation or action that accepts a reference reads the data directly. Passing the reference further is not a chain; escape is forbidden: return, storage in a struct/list/closure, use longer than the data lives. Deeper — via `box` or an index/arena.
- **`link&`** — a reference to a slot/place: survives `mut`; the slot's existence is checked at runtime, failure — panic; the slot's contents are not analyzed.
- **External operations** — files, APIs, system calls — these are just calls to external code. The compiler needs their signature — as everywhere.
- **Error handling** — `fn` can return `T or E`; the caller must immediately handle the result via `switch`. There is no `?`. `sub` does not return an error outward. The contents — any type; lazy mode — auto-generated error number; detailed — you write it yourself.
- **Modules and imports** — a module is a file. `use std`, `link file = "file.lang"`, call `std.print()` / `file.sum()`, reference to a reference `file::match.sum()`. Chains are allowed, cycles are a compile error. Passing is explicit. Two guarantees: you see the giving and the receiving side.
- **Computations as values** — `fn` can be passed into an `fn` as a value of type `Fn`.
- **Actions are not values** — `sub` cannot be assigned, passed, or returned; only a call as a statement.
- **`fn` and `sub`** — `fn` always returns an answer (must be consumed), `sub` returns no answer (call as a statement).
- **Ecosystem** — config format, compiler (Cranelift), aliases are not needed: labels right on the type (`i32:id`). Profiling: built-in, the mechanism depends on Cranelift's capabilities.
- **Compiler error messages** — a dead reference is explained like this: "here is the reference that you use here, here is how it was killed — that is, at the point of use it no longer exists". Compile-time error format: reference use site → kill site and method (`mut`, move, slot death) → conclusion. For `link&` with a slot that disappeared at runtime — panic with diagnostics: use site → slot disappeared → panic.
- **`const` — a universal "known at compile time" modifier** — applicable to everything that can or cannot be known at compile time: variables, structs, computations, actions, loops, expressions. `const struct` — fixed variables and values known to the compiler; `struct Name box { variable, ... }` — the struct holds variables with data, field names are variable names; the explicit type is optional and must match the variable's type. Dynamic dictionaries (maps) are not in v1. `const` on a loop — the compiler tries to compute the loop's result in advance.
- **comptime** — metaprogramming: executing code at compile time (code generation, compile-time checks). It differs from `const`: `const` — "the value is known in advance", comptime — "code is executed at compile time".
- **Warnings** — overflow always produces a warning, release/debug makes no difference. Suppression — `no_warn` before an operator/expression/declaration: suppresses only warnings, never errors, applies to one thing.
- **Comments** — `//`.
- **Tests** — via `---` zones: a separate test zone.

## Syntactic sugar

**Take** (cheap, hides nothing): string interpolation, ranges + `for..in`, inclusive ranges (`0..=n`), `while`, if-expression, methods (`mut vec.push(x)`), type inference, `defer`, slices (`a[1:3]`), tuples / multiple return values, switch, type labels (`i32:id`), implicit return of the last expression, `break`/`continue`, `else if`, `+=`, explicit type casting, hex/bin/oct literals, tests via `---` zones, comptime.

**Do not take** (not necessary): operator overloading, exceptions, pattern matching (Rust-style), null operators, default/named arguments, macros, interfaces/traits, async/await, coroutines/generators, reflection, GC/RAII, enums with data (sum types), chained comparisons, loop labels, raw strings, multiline strings, pointers (raw pointers) — a reference (`link`) is the safe pointer, user panics/asserts — business errors via `T or E` on `fn`, panic only for runtime checks (references, bounds), recursive types — the arena (array + indices) covers it, dictionaries (maps) — not in v1; a fixed set of fields — `struct`, dynamic keys — later; blueprints — not in v1: it is an ordinary function without concrete types, there is no separate notation for it.

Principle: do not cut functionality. What we do not build in v1, we design so that it can be added later without breakage.

## Misc

- Enum in the config — single backticks `` `...` ``.
- Config format: strings `""`, numbers `''`, arrays `{}`, tables `[]`, enum `` `...` ``.
- Comments — `//`.
- Overflow — always a warning, release/debug makes no difference. Suppression — `no_warn`.
- `no_warn` — a prefix modifier before an operator/expression/declaration: suppresses warnings from the thing it stands before. Errors cannot be suppressed.

## Next

Implementation — the compiler on Cranelift.
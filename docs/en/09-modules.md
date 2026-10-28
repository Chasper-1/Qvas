# Modules

## A module is a file

A module is a file. Importing a module is `use std` — at the top of the file you write what you receive (as in Rust):

```text
use std
```

Connecting a file: `link file = "file.lang"` — you work with it as a library:

```text
link file = "file.lang"
```

According to the accepted order, `use` and `link file` live in their own zones at the top of the file, separated by `---`: first the `use` zone, then the `link file` zone, then code. This is a convention, not a rule: the compiler does not require it, but this way `link` is not confused with a reference to data.

## Calls

Calls of computations and actions are the same — name + dot + name of the callee:

```text
std.print()
file.sum()
```

## File proxy

```text
file::match.sum()
```

`::` is a transition into a file that is connected in this file, `.` is a call.

Chains are allowed, cycles (`a ↔ b`) are a compile error.

## Passing is explicit

At the call site you can see where you are passing. Two guarantees: you see the giving side and the receiving side — it is impossible to make a mistake. If you have not decided where to give it — then you do not need to pass it.

## Scopes

- The global scope is the whole file. Globals are simply visible by name in their scope.
- The local scope — a body creates its own scope (for example, a computation or action). Outside the body, what is written inside cannot be used in any way. Where the body ended — there the possibilities of using everything written before ended.

Between files, things are shared through computations and actions: a file does what it needs and passes the result to another file (`file.calc()`). There are no project-wide globals.
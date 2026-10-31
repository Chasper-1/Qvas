# Computations and Actions

## Two entities

```text
fn name(params) -> type { ... }
sub name(params) { ... }
```

`fn` is a computation: it returns an answer, the answer must be consumed. `sub` is an action: it returns a check. A check is information about where the data went.

## Contract

Parameters declare whether they read or consume. It is visible in the signature:

- a reading parameter — `link` (reference);
- a consuming parameter — `box` (the compiler decides copy or move by the number of uses at the caller).

```text
fn read(link data: Data) -> i32 { ... }             // computation: reads and returns an answer
sub print_data(link data: Data) -> () or E { ... }  // action: reads, returns a check
sub consume(box data: Data) -> () or E { ... }      // action: consumes, returns a check
```

At the call site the contract is observed explicitly:

```text
let n = read(a)                 // link parameter — a reference is passed, no box needed
ignore!(print_data(a))          // action: the check is not needed — not taken
ignore!(consume(box(a)))        // box parameter — box is written at the call
```

The signature shows what to write at the call site: a consuming parameter requires `box` at the call site.

There are no bare parameters: every parameter must be marked `link` (reading) or `box` (consumption). If there are several parameters, there is one `box` at the call: `sum(box(x, y))`.

A `link` parameter is reading: the computation or action receives a window into the data. Passing the reference further with a `link` parameter is not a chain; only escape (return, storage, use longer than the data lives) is forbidden. If the data must be passed deeper or for longer — via `box` or an index/arena.

## Computation

`fn` always returns an answer. The answer must be consumed: bound, used, passed, or returned. A bare `fn` call is an error.

```text
fn sum(box a: i32, box b: i32) -> i32 { a + b }

let r = sum(box(3, 5))      // answer consumed
```

Return is passing the data back, no separate mechanism is needed:

```text
fn make() -> Data {
    let d = ...
    d
}
```

Mutating data in a computation:

```text
fn process(box data: Data) -> Data {
    mut data.field = new_value
    data
}
```

The data arrived (consumption), was changed via `mut`, and was returned (return = passing). The answer is still mandatory: if there is no answer, it is not an `fn` but a `sub`.

## Action

A `sub` works with the world: writes, sends, draws, calls a syscall. The data goes into memory — the world is memory. The result is a check: what was written, where, how much, what was sent.

```text
sub print(link s: str) -> () or IoError
sub open(link path: str) -> FileId or IoError                     // FileId — where
sub read_into(link path: str, out buf: [u8]) -> usize or IoError  // usize — how much
```

A check is not a value for computations but information. Needed — bind it or handle it. Not needed — do not take it: the data is already placed, that was the work.

- `ignore!` — explicitly not take the check, visible in the code;
- `switch` / `match` — handle it in place.

```text
sub copy(link from: str, link to: str) -> () or FileError {
    let data = read_file(from)
    switch data {
        FileError::NotFound => ignore!(print("no file")),
        else => ignore!(write_file(to, data)),
    }
}
```

If an action can fail, it returns `or E`. The error is handled at the call site, as with `fn`. There is no propagation.

A `sub` without `or E` cannot fail: there is no error, the call is a statement.

```text
sub push(link list: [T], box value: T) { ... }

mut tokens.push(token)      // nothing can fail
```

## `out` — writing into someone else's memory

`link` only reads, `box` takes. `out` gives write access and is visible at the call site.

```text
sub greet(out buf: [u8]) -> usize { ... }

let buf: [u8] = {}
let n = greet(out(buf))
```

## `main`

The entry is declared at the top of the file:

```text
main() -> Sys
```

This is not a function's return type. It says where the data goes: `main` gives it to `Sys`. `Sys` is a sink, not a type.

For embedded systems the sink is different: `main() -> Metal`. The sink's name is provided by the platform; the language does not fix it.

`main` is an action with no caller in the language: the platform starts it and reads what went into the sink. Errors inside `main` are handled in place, as everywhere.

There is no reverse notation `Sys = main()`: the boundary is one, it cannot be crossed twice.

## Sugar

Convenient forms (`getline!`, `read_into!`) are a thin layer. Three rules:

- the expansion is visible: the IDE and the documentation show what the form expands into;
- the tag follows the expansion, not the name: if an action is inside, the whole form is an action, and it cannot be called from an `fn`;
- the expansion fits in one's head: "here is an action, here is a computation".

`getline!` expands into `read_into` plus a slice of the buffer; its tag is action.

## `mut` and move do not mix

`mut` changes the value, a move changes the name responsible for the data. Either one or the other.

## Computations as values

A computation can be passed into a computation — as everywhere:

```text
fn apply(link f: Fn, box value: i32) -> i32 {
    f(value)
}

let result = apply(my_func, box(42))
```

`sub` is not a value: it cannot be put into a variable or passed as `Fn`.

## Calls

`fn` are called from any `fn` or `sub` without specifying the call in the signature. A call is the use of ready logic: one function, many calls. There are no implicit inputs: data comes only through parameters according to the contract. The logic of a function does not depend on the logic of the caller.

`fn` does not call `sub`: a computation does not perform actions. Calls go in one direction only — `sub` calls `fn`, but not the other way around. If an action is needed inside a computation, it is an action, not a computation.

## Return

The answer is data, passing is a move. The answer is not a reference; it is not in two places at once.

## Recursion

A direct self-call is an impossible contract, not recursion: the function requires its answer right now, but it does not exist yet. Recursion is possible only through branching: the answer has a type obtained from the function itself, and another branch with different computations is launched.

# Computations and Actions

## Two entities

```text
fn name(params) -> type { ... }
sub name(params) { ... }
```

`fn` is a computation: it always returns an answer. `sub` is an action: no answer, the call is a statement. `let` is not needed in either place.

## Contract

Parameters declare whether they read or consume. It is visible in the signature:

- a reading parameter — `link` (reference);
- a consuming parameter — `box` (the compiler decides copy or move by the number of uses at the caller).

```text
fn read(link data: Data) -> i32 { ... }      // computation: reads and returns an answer
sub print_data(link data: Data) { ... }      // action: reads, no answer
sub consume(box data: Data) { ... }          // action: consumes, no answer
```

At the call site the contract is observed explicitly:

```text
let n = read(a)             // link parameter — a reference is passed, no box needed
print_data(a)               // action: call as a statement
consume(box(a))             // box parameter — box is written at the call
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

`sub` returns no answer. It cannot be assigned, passed, returned, or used in an expression. Only a call as a statement.

```text
sub main() {
    let data = ...
    consume(box(data))
    print_data(data)
}
```

An action is the top of the chain: it has no answer for anyone to consume. It itself consumes data and answers of computations.

`sub` does not return an error. If a computation that returned `T or E` is called inside an action, the result must be handled via `switch` at the call site, inside that action.

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

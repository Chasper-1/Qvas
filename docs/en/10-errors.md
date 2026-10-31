# Error handling

## Approach

An error is a feature, not a bug. An error is handled where it arose: at the call site. There is no propagation.

## An error is a value

An error is an ordinary value. What exactly it is — the programmer decides:

```text
enum FileError {
    NotFound,
    PermissionDenied,
}
```

## A computation can return "value or error"

```text
fn read_file(link path: str) -> Data or FileError {
    ...
}
```

`T or E` is a computation's answer. The caller must handle the result right at the call site, via `switch` or `match`.

## An action returns a check

A `sub` works with the world and returns a check — information about what was written, where, how much:

```text
sub write_file(link path: str, link data: Data) -> usize or FileError
```

A check is not a value for computations but information. Needed — bind it or handle it. Not needed — do not take it: the data is already placed, that was the work.

## Errors are handled in place

```text
sub show(link path: str) {
    let result = read_file(path)
    switch result {
        FileError::NotFound => ignore!(print("file not found")),
        FileError::PermissionDenied => ignore!(print("no access")),
        else => ignore!(print(result)),
    }
}
```

An `fn` returned `T or E` — the caller handles it, right at the call site. A `sub` is called only by another `sub`, so the calling `sub` handles the error there.

There is no error that cannot be handled in place. If one does not fit — the code is written badly.

## There is no propagation

The `?` operator does not exist in the language. It moves an error out of the place where it arose — the language does not do that.

## `ignore!` — explicitly not taking the check

```text
ignore!(print("x"))
```

The code visibly shows that the check is not used. A bare `print("x")` is the same thing, a call as a statement.

## Lazy and verbose modes

**Lazy mode:** the compiler generates the error number from the location (file, line, computation or action); you look the number up in the docs.

**Verbose mode:** you write yourself what to put into the error.

## `defer` — deferred execution

`defer` is deferred execution (closing resources and the like):

```text
sub work() {
    let f = open(path)
    defer ignore!(f.close())
    ...
}
```

The action call is postponed until the block exits. If the action has a check, it is taken or not taken right there.

# Error Handling

## Approach

An error is a feature, not a bug. An error is not propagated automatically: a result with an error is examined at the call site.

## An error is a value

An error is an ordinary value. What exactly it is, the programmer decides:

```text
enum FileError {
    NotFound,
    PermissionDenied,
}
```

## A computation can return "value or error"

```text
fn read_file() -> Data or FileError {
    ...
}
```

`T or E` is the answer of a computation. This is not permission to propagate the error: the caller must immediately examine the result.

## Handling on the spot

```text
sub main() {
    let result = read_file()
    switch result {
        FileError::NotFound => std.print("file not found"),
        FileError::PermissionDenied => std.print("permission denied"),
        else => std.print(result),
    }
}
```

If a computation calls another computation with `T or E`, it too must handle the result via `switch`. A bare call of such a computation is an error.

## There is no `?`

There is no `?` operator in the language. It moves the error out of the place where it originated — the language does not do that.

## Actions and errors

`sub` does not return an error. If a computation with `T or E` is called inside an action, the action must handle the result via `switch` inside itself. If the action cannot do this, it is a code design error, not a reason to return the error outward.

## Lazy and detailed modes

**Lazy mode:** the compiler generates the error number from the place (file, line, computation or action); you look up the number in the docs.

**Detailed mode:** you write yourself what to put into the error.

## `defer` — deferred execution

`defer` is deferred execution (closing resources, etc.):

```text
sub work() {
    let f = open(path)
    defer f.close()
    ...
}
```

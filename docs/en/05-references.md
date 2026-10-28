# Reference Checking

## What the compiler checks

The compiler tracks all references (`link`, `link&`, and `link*`) and checks four rules:

1. A reference does not outlive the object/slot it points to (point of last use, end of scope) — compile-time.
2. A state-bound reference (`link`) is not used after mutation of the path it reads — compile-time.
3. While `link*` is alive, mutation of the object/path is forbidden — compile-time.
4. The slot that `link&` looks at exists at the moment of use — runtime check, if the slot's death is not statically provable.

The first three checks are performed at compile time, on all possible execution paths. The fourth is a runtime assert: the compiler does not analyze the slot's contents, it only checks that the path is valid. Failure is a panic, not UB.

## A reference does not escape

A computation or action that accepts a reference reads the data directly. Passing the reference further with a `link` parameter is not a chain: the same window into the data is passed, not a reference to a reference. What is forbidden is precisely escape: returning a reference, storing it in a struct/list/closure, or using it longer than the data lives. If the data is needed deeper or longer — via `box` or an index/arena.

## Conditionals and branches

Branches are not a fundamental problem for checking.

```text
if condition {
    use(x)
} else {
    discard(x)
}
```

The compiler knows both possible paths and builds a CFG:

```text
       condition
        /      \
      true    false
       |        |
     use(x)  discard(x)
        \      /
          end
```

The checking task is to ensure that on all possible paths references do not outlive the paths/slots they read.

## Loops

Loops may have an unknown number of iterations:

```text
while condition {
    use(x)
}
```

This does not mean checking is impossible. The compiler uses standard data-flow/fixpoint analysis.

## Runtime and compile-time

The compiler does not need to know in advance which branch exactly will execute. It needs to know which branches exist at all and which actions are possible on each of them.

Runtime chooses the concrete path. Compile-time analyzes the set of possible paths. Separately from path choice, `link&` checks the slot's existence at the moment of use; this check does not depend on which branch executed.

## External operations

If data leaves the ordinary program context, this must happen through an explicit operation:

- a computation or action;
- a struct;
- a file;
- an API;
- a device;
- a system call;
- another external boundary.

For such operations the compiler needs a contract. External code is not obliged to be fully analyzed by the compiler, but the boundary between it and the language must be described explicitly.

## Checking cost

Reference checking is standard data-flow analysis for life, state-binding, and freeze; it is performed at compile time. The existence of a slot for `link&` is checked at runtime because it depends on data; the slot's contents are not analyzed. Slots and references are explicit, so the analysis itself is simpler than lifetime inference.

Preference is given to:

> more work at compile time → fewer errors at runtime.

## Error format

A dead reference is explained like this: "here is the reference that you use here, here is how it was killed — that is, at the point of use it no longer exists".

Compile-time error format:

> reference use site → kill site and method (`mut`, move, slot death) → conclusion.

Runtime panic format for `link&`:

> reference use site → slot disappeared (removal from a container, index out of bounds) → panic.
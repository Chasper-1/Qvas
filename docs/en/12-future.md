# Future Ideas

## Explicit size at vector creation — not in v1

When a container is created, the allocation size is mandatory to write down — in objects, not in bytes:

```text
let v = vec [T; 64]        // size known: 64 objects, the compiler multiplies by the type size itself
let v = vec [T; none]      // size unknown: empty vector, allocated gradually
```

Rules:

- the size is always mandatory — either a number or `none`; creating a vector without a size is a compile error;
- the size is written in objects — the translation into bytes is done by the compiler, not the programmer;
- the vector mechanics do not change: `none` is the current behavior (empty + gradual allocation); an explicit number is a capacity claim, not a contract;
- if there turn out to be more actual objects than claimed — the vector grows as usual, that is not an error;
- both modes are written in one place: know the size — pay one exact allocation and see it in the code; don't know — get gradual growth honestly.

The point: vector creation is the moment an allocation is born, and writing the size confirms the programmer knows this. Over-allocation is visible by eye in the code — no benchmark or profiler is needed to figure out that somebody reserved ten times more than they use.

## Blueprints — not in v1

A blueprint is a function in which concrete types are not specified: placeholders are left instead of them. There is no separate syntax: it is an ordinary function without concrete types.

When the placeholders are filled, a template results — a ready function with concrete types.

Blueprints are not part of v1: the C level is enough without them.

## JIT interpreter (built into the compiler)

- One language, two ways to run: an AOT compiler and a JIT interpreter. One backend — Cranelift.
- The frontend hands data to the JIT as it becomes ready (per computation and action), Cranelift compiles on the fly — REPL mode, fast iteration.
- Automatic memory makes this natural: no GC, compiler-inserted `free` calls work in the JIT too.
- Value: the language is its own scripting language — write it, check it, then compile it into a fast binary.

## Comptime via JIT

The JIT backend also works as a comptime engine: executing code at compile time (constants, code generation, compile-time checks).

Zig does comptime by interpretation — we use a JIT, which is faster.

One backend, three applications: AOT, JIT interpretation, comptime.

## Self-hosting

When rewriting the compiler in its own language, the backend (Cranelift) will have to be written by ourselves — our own code generator.
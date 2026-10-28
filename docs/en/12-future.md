# Future Ideas

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
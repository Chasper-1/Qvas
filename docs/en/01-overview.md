# Qvas Language Overview

## What it is

A low-level compiled language without GC:

- about as simple as Go;
- memory-safe no worse than Rust;
- explicit like Zig.

The compiler checks the program at compile time and generates ordinary low-level code.

## Goals

- near-native performance;
- predictable behavior;
- no need for a heavy runtime/GC;
- automatic memory management with reference checking: statically, plus a runtime assert for `link&`;
- the simplest possible state-change model;
- explicitness of important actions in the source code;
- memory safety without having to reproduce Rust's model one-to-one.

## See also

- [02-syntax.md](02-syntax.md) — syntax
- [03-data-model.md](03-data-model.md) — data model
- [04-memory.md](04-memory.md) — memory
- [05-references.md](05-references.md) — reference checking
- [06-parallelism.md](06-parallelism.md) — parallelism
- [07-functions.md](07-functions.md) — computations and actions
- [08-types.md](08-types.md) — types
- [09-modules.md](09-modules.md) — modules
- [10-errors.md](10-errors.md) — error handling
- [11-compiler.md](11-compiler.md) — compiler
- [12-future.md](12-future.md) — future ideas
- [13-status.md](13-status.md) — status
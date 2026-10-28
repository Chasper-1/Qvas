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

## Key formula

> **Maximum analysis at compile time, minimum mandatory work at runtime.**

The exception is references — `link&`: when used, it checks that the slot exists; failure is a panic. `link*` does not require this check because it freezes the data.

State-change model:

> **Not "I allow this to be changed", but "I am changing this right now".**

## Philosophy: 10 principles

1. **Immutability by default.** Data does not change without an explicit action.
2. **`mut` denotes an action.** `mut` is not a permanent permission but a concrete state transition.
3. **References are bound to state by default.** `link b = a` creates a contract with the current state.
4. **A state change breaks old state-bound contracts.** After `mut a = ...`, old ordinary references to the previous state become invalid.
5. **A long-lived reference must be explicit.** `link& b = a` expresses binding to a slot/place.
6. **No default state: reading or using.** Every interaction with data is either reading or using. Which one is determined by the operation's contract that the data is passed to. Reading is not consumption: you can read as many times as you like.
7. **Memory is automatic, the compiler checks references.** The compiler manages memory automatically and checks that references do not outlive the object/slot they point to; the existence of a slot for `link&` is checked at runtime.
8. **Independent actions can run in parallel.** Dependencies form a graph that can be used for automatic scheduling.
9. **The language must be explicit.** The compiler analyzes the semantics expressed in the program. It must not guess the programmer's intent where this is not required.
10. **Remove the superfluous, not the functionality.** Everything actually needed remains: read, copy, move, mutate, pass. Only the ways to do it wrongly or redundantly are removed: reference chains, references hidden in containers, mutation through a reference.

This is not cutting functionality but cleaning it up: the functionality is the same, but the ways to break it are zero.

## Learning

One of the main goals is that the basic model of the language can be understood in about five minutes and applied immediately. The model boils down to a few rules:

1. Data is immutable by default.
2. `mut` is a change event, not a permission.
3. There is no default state: reading or using is determined by the operation's contract.
4. `let b = a` — a copy (create an independent value).
5. `link b = a` — a reference (reading). `link& b = a` — a reference to a slot (reading, bound to a slot/place, survives `mut`; the existence of the slot is checked at runtime, failure → panic). `link* b = a` — a reference with freeze (reading, the data cannot be changed while the reference is alive; static check).
6. `box(expr)` — using the real data. 1 consumption → move; 2+ → copies, a move is possible on the last use but not guaranteed; there are never two moves.
7. `const a = 10` — a constant, the value is known at compile time.
8. Using a dead reference is a compile error; if a `link&` slot disappeared at runtime — panic.
9. A reference is reading: you cannot mutate through it.
10. References are direct, without chains; references do not hide and do not escape a computation or action (they are not returned, not stored in structs/lists). Passing a reference to another computation or action is not a chain.
11. A move kills references: the source variable is dead — references to it are dead.
12. Memory is automatic: the compiler allocates and frees it itself and checks references.
13. Parallelism: `parallel { ... }` is a block; the compiler builds a dependency graph and schedules it itself *(mechanism in development)*.
14. `fn` — a computation: returns an answer, the answer must be consumed.
15. `sub` — an action: no answer, called as a statement. An action is not a value.

Two learning paths are possible: for beginners — with basic concepts; for experienced programmers — without re-explaining the obvious basics.

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
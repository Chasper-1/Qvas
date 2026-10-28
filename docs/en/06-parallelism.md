# Parallelism

> **Mechanism in development:** how the compiler builds the dependency graph is still a plan.

## Dependency graph

Syntax: `parallel { ... }` — a block in which you write what to do in parallel. The compiler builds a dependency graph and schedules it itself.

```text
parallel {
    let x = compute_x()
    let files = load_files()
    let result = process(x, files)
}
```

Graph:

```text
compute_x ── x ──→ process
load_files ──────→ process
```

`compute_x` and `load_files` are independent — they run in parallel. `process` starts only after the necessary results are obtained.

## One-way dependencies

A dependency must exist only where it is actually needed.

If the result of `A` is needed by `B`, then this is enough:

```text
A
↓
B
```

This does not mean `B → A`. Independent parts of the program must not artificially block each other.

## Automatic scheduling

The compiler can transform dependencies into a task graph and then choose how to execute them:

- sequentially;
- in parallel;
- on separate threads;
- via a worker pool;
- via another lightweight scheduler.

The main idea:

> the programmer describes semantic dependencies, not manually placing every synchronization point for independent operations.

Low-level primitives such as mutex/atomic/channel can still exist if needed.

## Parallelism is not the same as memory safety

Parallel execution is considered a separate level.

A program can be:

- memory-safe but logically incorrect;
- correct in terms of dependencies;
- incorrect due to improperly organized parallel operations.

Memory safety and correctness of parallel logic do not necessarily have to be the same system of rules.

Shared data in a parallel block is read through references. `link*` freezes the data for the duration of parallel reading — nobody can change it.

```text
let data = load_data()
link* view = data

parallel {
    process_a(view)
    process_b(view)
}
```

`process_a` and `process_b` read `view` directly. Passing `view` to them is not a chain but a use of the same window into the data; a reference cannot be returned or stored, but calling another reading computation or action with it is allowed.

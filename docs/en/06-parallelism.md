# Parallelism

## Mechanism solved

The mechanism (on top of the graph concept) is solved. The solution — take existing, proven schedulers and make them part of the language semantics: the language is responsible for correctness (who depends on whom), the scheduler is responsible for execution (how to spread work across cores). Each does its own job.

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

Graph construction comes from the written order and data dependencies: reading what a previous node computed is an edge. The language rules (`mut`, `box`, `link`) already supply every edge:

- **`mut` inside the block** — mutation is a dependency: if two nodes write the same variable, they serialize by written order. Independent writing tasks — each writes its own variable.
- **Block boundary** — values created inside live past the block; joins happen automatically through dependencies. There is no separate `await`/`join`: manual synchronization does not come back.
- **Written order by default** — priority: if a node depends on the next one somehow, it is written top-down; mostly the block contains independent parts.

## fn → rayon, sub → tokio

Routing graph nodes to executors is the simplest part, because it is already visible on the entity:

- **`fn` — a computation** (pure: never calls `sub`, returns an answer) is a node that mostly *computes*. Its place is **rayon**: work-stealing, splitting across cores, workers.
- **`sub` — an action** (work with the world: files, network, I/O) is a node that mostly *waits*. Its place is **tokio**: asynchronous execution, waiting is cheap.

The block itself never changes: it is always the same thing — a graph. The choice of "async or threads" is not a property of the code but the route of a node, and the route follows from what is written in the node: `sub` → tokio, `fn` → rayon. Mixed graphs split across two executors, and the junction (compute after load) is just an edge between them.

The conditions making this safe already exist in the language model:

- `fn` never calls `sub` (07-functions.md) — graph tasks are pure → execution order does not affect the result → scheduler nondeterminism is safe;
- `link*` freeze — several tasks read the same data without conflicts, and this is proven by the compiler, not "hoping for Sync";
- `box` passing — data moves into the task, the source dies: an ownership race cannot exist by construction.

Consequence: asynchrony does not change the surface syntax (no async viral coloring, no colored functions) — like Go at the source level, but with a real scheduler that sees the whole graph.

## Shared data and references

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

## One-way dependencies

A dependency must exist only where it is actually needed.

If the result of `A` is needed by `B`, then this is enough:

```text
A
↓
B
```

This does not mean `B → A`. Independent parts of the program must not artificially block each other.

## What the programmer describes and what is decided for them

> the programmer describes semantic dependencies, not manually placing every synchronization point for independent operations.

Layers:

1. **Graph** — built from the written order and data dependencies (the `mut`/`box`/`link` rules supply the edges themselves).
2. **Node routing** — by entity: a `sub` with I/O waits → tokio, a `fn` computes → rayon.
3. **Execution** — the scheduler already exists and is proven: rayon for the computing, tokio for the waiting.

Low-level primitives such as mutex/atomic/channel can still exist if needed.

## Parallelism is not the same as memory safety

Parallel execution is considered a separate level.

A program can be:

- memory-safe but logically incorrect;
- correct in terms of dependencies;
- incorrect due to improperly organized parallel operations.

Memory safety and correctness of parallel logic do not necessarily have to be the same system of rules.

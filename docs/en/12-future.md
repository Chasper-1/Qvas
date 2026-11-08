# Future Ideas

## Iterator speed without iterators

An iterator as a language entity is not needed: it requires values with state, which are banned in this language (`sub` is not a value, links are not hidden in closures). But its speed is required: going through a "plain loop" must not lose to an iterator, as it does in other languages.

Why iterators in Rust beat "plain loops" at all — three things the compiler does with iterator code after inlining: removes bounds checks (the cursor has an honest range, overflow is excluded by construction), removes indirection (next() replaced by an index increment), turns write-read-write into a straight pipeline with no intermediate allocations. Adapters like map/filter are just a way to *express* all this so the compiler sees the whole chain at once. A `while` loses to an iterator exactly on these points: nothing says `n` will not change, `i` will not overflow, `a[i]` will not go out of bounds.

## Lowering rules

1. `for item in list` lowers into IR as a single pattern "cursor with fixed base+len", not as an expansion into a `while`: out-of-bounds through the cursor is impossible by construction — the check is not needed, it is not merely cheap.
2. The same lowering rule applies to the `while` version of the same code if the compiler proved freeze/immutability (see the index→pointer rule below). There are no fast and slow ways to walk a container — there is a single lowered pattern and the syntax in which a person writes it.
3. Common patterns are built-in `for` forms lowered into the same IR without intermediate lists:

```text
for x in list.map(mul2) where x > 0 { ... }   // fusion: no intermediate list
for pair in zip(a, b) { ... }                 // two streams, one iteration
for w in list.windows(3) { ... }              // sliding window = cursor with offset
```

Sugar, not entities: the forms have no values, no state, they cannot be stored or passed. Their expansion is visible, like `!`-forms (see 02-syntax.md).

4. The check for mutating the container inside the body — the existing link-kill rules: if the container is mutated, len/index is re-read and the loop honestly rechecks. The programmer does not pay for the case that did not happen.

## Indices as the primary mechanism for heap access

The idea: heap access is built on indices, not pointers. An index is a value: it is simpler than a pointer, it cannot be swapped to point at someone else's memory, and it is easy to store, compare, and pass around. Safety checks (bounds, slot existence, container membership) are performed on the index. And then — the key step:

> wherever the compiler has proven access safety, the index turns into a direct pointer.

Checked once → from then on the code runs like C/Rust, with zero runtime cost. Not everywhere — everywhere where the proof is cheap, and that is the majority of cases.

### Why the optimization ceiling is higher than with raw pointers

The main expensive unsolved problem of optimizers in C and Rust is aliasing: "do two pointers point into the same memory?". Because of it the compiler cannot hoist a load out of a loop, reorder stores, or turn copies into memcpy. C solves it with `restrict` (a promise), Rust with borrow checking (an expensive analysis with lifetimes).

An index is a value; access always goes through the explicit channel "container + index". In a region where safety is proven, the compiler automatically gets:

- elements are not mutated under its feet;
- there is no reallocation;
- bounds are checked;
- aliasing is absent constructively.

No promises, no separate analysis — this is a consequence of the language rules, not a separate feature.

### What gives safety without lifetimes

An index by itself is meaningless — it is tied to its source container. That binding is handled by type labels: `i32:id` (08-types.md).

```text
let user_i: i32:users = ...
fn get(i32:users idx) -> User { ... }
```

The label ties the index to the container and is checked at boundaries; it is erased at runtime — zero cost. This is half of Rust's lifetimes, and for indices that is enough:

- the label = "which container";
- the life of the container itself is covered by the existing analysis.

The pointer-caching rule is not a separate feature but a consequence of existing rules:

> An index checked at least once in a region where the container is frozen (`link*`) or provably immutable is equivalent to a direct pointer and may be cached by the compiler. Any mutation of the container kills both the freeze and the cached pointers — like an ordinary link kill.

It cannot be written incorrectly: `link*` freezes the data, so the slot does not vanish and the contract is explicit. The optimizer gets the proof from the language semantics, for free.

### Where to squeeze first

The rule: squeeze the maximum, but if it gets too complicated — drop it. The language stays simple; a compiler decision where the proof is cheap, the ordinary checked path where it is expensive.

1. **Loop body over a container.** The bound is checked once (len fixed at the start of the iteration), then a cached pointer to the base or straight sequential access — direct auto-vectorization inside the loop (the millions-of-iterations case).
2. **Frozen data (`link*`) read from several places.** All readers get direct pointers with no repeated checks (tied to parallelism, 06-parallelism.md).
3. **Arenas / self-referential structures** (trees, graphs). Indices instead of recursive types — already decided. Dereferencing a node via `node.children[j]` is three hops through memory; cache-line locality is where the payoff is critical.

### Where to drop it (first tier of "it gets complicated")

- caching pointers across function boundaries — everything that needs interprocedural proof;
- tricky removal patterns (if anything is taken later — one generations rule, not inference);
- "clever" static bounds for loops with unknown len — a runtime check is enough.

### Open questions

- Indices under `box`-move of a container: memory does not move, only the responsible name changes — an index is meaningful for "container as data", not "variable name": this needs a formal rule.
- Removal from non-stable containers: one option is generational indices (index + generation packed into one 64-bit value), the slot is not reused immediately; a natural extension of `i32:id`.
- Indices are not for everything: value data (stack, struct fields) is statically checked already — do not force indices there. Indices replace exactly the class where pointers used to be and where the bans were: link chains, links hidden in containers — what could not be done with a link is done with a checked index.

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
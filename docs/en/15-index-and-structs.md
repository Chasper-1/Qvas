# Pointer, Structs, and Reference Direction

## Pointer (`index`)

**Definition.** A pointer is a value that contains information about where other data is located. It does not contain the data itself and does not give access to it. It is location information.

**Spelling.** `index`, abbreviations: `idx`, `ind`, `indx`, `inx`.

**Use.** Data and structs only. Its purpose is to build connected things: linked lists, graphs, trees. In Qvas — via an arena: an array + indices.

**Properties:**

- an ordinary value;
- not a reference: liveness is not checked, `mut` does not kill it;
- copied like a value, stored in structs and lists;
- access to the data — only through a container: `nodes[i]`;
- nothing can be done with it: neither dereferenced, nor used to change data through it;
- only what it passes needs to be checked (whether the slot exists at the moment of reading), not its logic.

**What it is not:**

- not `link` / `link&` / `link*` — those give a window into the data and are checked;
- not a raw pointer — the language has no raw pointers;
- not ownership and not a responsible name.

## Struct with pointers

Rule: `()` — action/call, `[]` — place/data. Struct parameters are declared in square brackets.

```text
struct Node[right_i: nodes[indx], left_i: nodes[indx]] box {
    id:    i32
    link& right: right_i
    link& left:  left_i
}
```

Read line by line:

- `right_i: nodes[indx]` — a pointer: an index in `nodes`;
- `link& right: right_i` — a reference through this pointer;
- `id: i32` — ordinary data;
- `indx` is the index label, `nodes` is the container, `Node` is the struct type.

Three parts of a struct:

| Part | Where | What it contains |
|---|---|---|
| addressing | `[...]` | pointers: where the node looks |
| data | body | what is stored in the node |
| links | body | references created through the pointers |

### The reference is created in place

A reference is declared right in the struct and does not go outside: it is not passed into the struct from outside, not extracted, not returned, not stored elsewhere. Born inside — dies together with the struct.

This does not violate the principle "references do not hide": what is forbidden is carrying a reference into a container from outside, not creating one inside.

A pointer costs nothing. It only passes along the reference, which is created inside and dies there.

### Order

1. A pointer is passed in `[...]` — `right_i: nodes[indx]`.
2. The pointer is bound to a reference — `link& right: right_i`.
3. The concrete data is read through the reference — `right`.

## Reference direction

`main` is level 0. A function called from `main` is level 1, its callee is level 2. The level is call depth.

- A reference is born at level N.
- It can be read only at levels above N — inside calls growing out of the creation site.
- Passing — one step per call: N → N+1 → N+2.
- Downward, to the caller (N-1, N-2) — forbidden.
- Sideways, into another call branch — forbidden.

**Why this works.** Frame N is alive as long as any frame N+k is alive. Therefore the data the reference looks at is guaranteed alive at every use. In the opposite direction there is no guarantee: frame N-1 will outlive N, and the reference will become dead. Hence the downward prohibition.

**Checking.** At every call exactly one thing is checked: the reference's creation site is an ancestor of the call site. A function is checked once, calls — by one edge. No long chains and no lifetime inference over the whole graph.

**Consequences:**

- recursion — a self-call is a step up; returning a reference from recursion is forbidden, reading inside is allowed;
- `Fn` values — a call through a variable is still a call edge; a reference is not put into a closure;
- `parallel {}` — the stack level does not work; a reference is created in a frame that is an ancestor of the block, inside — reading only, `link*` as the guarantee.

## Mutation and the choice of reference

| | `link` | `link&` | `link*` |
|---|---|---|---|
| After `mut` | dies | lives | `mut` is forbidden |
| What it reads | the old state | the current slot | the frozen state |
| Meaning of the link | fixed | may diverge from the data | always matches |
| Check | static | static, otherwise runtime | static |
| Cost | 0 | possible runtime | 0 |

Invalidation follows the path: `mut parent.left` does not kill the `link` on `node`, because the paths do not intersect. Therefore, when rebalancing, the links of neighbors change while references to the nodes themselves live on.

## Where `link&` is needed

Only when the path itself is mutated but must keep being read:

1. **Accumulator in a loop** — a reference to a field that changes every iteration.
2. **An element that survives a change to the container** — `link& item = list[i]`, then `mut list.push(...)`; `link` would die, `link*` would forbid `push`.
3. **A field observer** — cache, counter, currently selected element: a long-lived reference that must see the new value.

It is not needed for trees and graphs: links there are recreated, and neighbor paths do not intersect — an ordinary `link` is enough.

**Selection rule:**

- the path is not mutated → `link` (or `link*` if a guarantee of immutability is needed);
- the path is mutated, the reference can be recreated → `link`, recreate it;
- the path is mutated often, recreating is inconvenient or costly in code → `link&`;
- a guarantee that the data will not change is needed → `link*`;
- a live link with in-place change is needed → `link&` with a runtime check and the risk of semantic divergence.

`link&` is not for trees and graphs but for live observation of a mutable slot.

## Graphs: modes

1. **Build** — indices only, mutation allowed, nodes are added.
2. **Freeze** — structs with `link*` to children are created; the arena is not mutated while any such struct is alive.
3. **Read** — traversal, parallel reading, no checks or panics.

Changing a graph = a new version: remove the `link*` structs, mutate the arena, rebuild.

## Cost of `link&`

A runtime check is needed only where the static analysis has not proven the slot's existence:

| Situation | Check |
|---|---|
| the slot is a local variable in scope | static, no runtime |
| the slot is a field of a live struct | static, no runtime |
| the slot can disappear due to data (`pop`, `clear`, `remove`) | runtime |
| the index comes from outside and may be out of bounds | runtime |

Mutation itself adds no cost: the reference only reads, and the point of change is known. Either 3 or 4 is seen — there is no half-written state, memory is safe.

In parallel execution there is no order, and "the moment of mutation is known" ceases to be free: `link*` (freeze) or explicit synchronization is needed.

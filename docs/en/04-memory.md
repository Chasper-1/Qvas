# Memory

## Automatic management

Memory is managed by the compiler. The programmer does not write `alloc` and `free` — the compiler itself decides where to allocate (stack or heap, escape analysis) and when to free.

Freeing happens at the point of the data's last use (like NLL in Rust). The lifetime analysis that checks references already knows when the data is dead: inserting `free` does not require a separate pass.

Automatic management is deterministic: no GC, no reference counting, no pauses. It is just compiler-inserted `free` calls at known points.

## The compiler checks references

The compiler checks references: a reference cannot outlive the object/slot it points to. Data dies at the point of last use (or at the end of the scope), and the reference cannot be used after that. The life of the base is checked statically; the existence of a slot for `link&` is checked at runtime if it depends on data.

```text
let a = ...
link b = a        // reference to the value
std.print(b)      // last use of the data
// here the data is dead — the compiler has freed the memory
```

Referring to something that does not exist is impossible by definition. If the slot's death is provable, it is a compile error. For `link&` the slot can disappear only at runtime (for example, an element is removed from a container) — then the compiler inserts a slot-existence check, and failure of the check is a panic, not UB.

## Why automatic management

The programmer should not have to think about memory when it already works deterministically. Manual allocation requires knowing exactly where memory is needed, and a mistake there is a logic error the compiler does not catch.

Automatic management + reference checking give:

- predictable behavior — freeing at the point of last use, without pauses or a collector;
- simple checks — lifetime analysis is needed for reference checking anyway, inserting `free` is free;
- memory safety — the compiler itself decides where to allocate and when to free.

The goal of the language is not to let you shoot yourself in the foot while writing. The compiler catches obvious errors at compile time and manages memory itself.

## Move and copy

`let b = a` is always a copy (the programmer's explicit intent). `box(a)` — the compiler decides copy or move.

The rule for `box`:

- the data is used once → move: the data is not copied, the old name dies, the memory stays in place;
- the data is used 2+ times → copies: all uses except at most one become copies. A move may be assigned to the actually last use, but is not guaranteed; there are never two moves.

Only **consuming** uses (`box`) are counted. Readings through references (`link`) are not counted — you can read as many times as you like, that is not consumption.

This is passing, not ownership. Ownership is passing plus a lot of extra. Passing is passing, no more and no less: the responsible name changes, the old name is dead, the memory does not move. References to it die together with the old name.

For large objects a copy can be expensive; the optimizer may turn safe copies into moves, but the decision is made by the compiler, not the programmer.
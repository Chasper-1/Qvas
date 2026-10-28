# Data Model

## Mechanisms for working with data

| Mechanism | Meaning | What happens |
|---|---|---|
| `let b = a` | copy | create an independent value, `a` stays alive |
| `link b = a` | reference | reading, `a` stays alive, dies on `mut a` |
| `link& b = a` | reference to a slot | reading, bound to a slot/place, survives `mut a`; the slot's existence is checked at runtime, failure → panic |
| `link* b = a` | reference with freeze | reading, bound to a slot/place, `mut a` is forbidden while the reference is alive; static check |
| `box(expr)` | real data | the compiler chooses copy or move |

The operation's contract determines what the operation accepts: a reference (reading) or box (using). All operations are checked by the compiler. Using a dead reference is an error: if the death is provable, it is a compile error; for `link&` with a slot that disappeared at runtime — panic.

## No default state

Every interaction with data is either reading or using. There is no "default".

```text
let b = a
```

creates a copy. `b` is an independent value, `a` stays alive.

```text
link b = a
```

creates a reference. `b` reads the data, `a` stays alive.

```text
let sum = box(a + b)
```

uses the real data. The compiler chooses copy or move based on the number of consuming uses:

- 1 use → move: the data is not copied, the old name dies;
- 2+ uses → copies: all uses except at most one become copies. A move may be assigned to the actually last use, but is not guaranteed — it depends on the code. There are never two moves.

Memory does not move on a move: only the name responsible for the data changes.

```text
let result = add(a, b)                // reading — references, no box
let result = add(box(a, b))           // use — box
```

### Reading is not consumption

Reading is implemented via a reference (`link`, `link&`, or `link*`), changes nothing, and consumes nothing. You can read as many times as you like.

Consumption is passing data via box. The compiler decides copy or move by the number of consuming uses (the rule above).

> Reading is just looking. Taking is box.

## Operation contracts

The contract belongs to the operation, not to the value. Every operation declares in advance whether it reads or consumes. No guessing — the guarantee is written down.

**Read (accept a reference, nothing needs to be written at the call site):**

- `print`, output;
- length, `isEmpty`, `contains`, and other checks;
- comparisons (`==`, `!=`, `<`, `>`, `<=`, `>=`);
- conditions (`if cond`);
- reading by index (`a[0]`);
- iteration (`for item in list` — reads the list, `item` is a reference).

**Consume (require `box`):**

- arithmetic (`a + b`, `a * b` — create a new value from the data);
- string concatenation;
- `push` into a list, insertion, storing into a struct;
- passing to a consuming computation or action (a `box` parameter);
- `return` — passing the data back.

**Everything else is decided on the spot.** Slices, strings, and composite values do nothing by themselves: whichever contract they are passed into applies. Into a reference — they are read; into `box` — they are consumed. There is no single answer to "is a slice reading or consumption", and none is needed.

**User computations and actions** do not need a list: the contract is in the signature — `link` = reading, `box` = consumption. It is visible immediately.

## Mutation breaks the state-bound contract

```text
let a = 20
link b = a
```

Both names are bound to the same current state of the data. The relation `link b = a` is a contract with a specific state: `b` has the right to access the current state of `a`.

If then this happens:

```text
mut a = 30
```

a new state is created. The old contract `link b = a₀` no longer exists. `b` does not turn into a dangling reference — the old state version simply ceases to be valid for further use through this contract. Using `b` after `mut a` is a compile error.

To get `b` again, you must explicitly write:

```text
link b = a
```

## `link& b = a` — an explicit long-lived reference

A separate syntax for a reference that is bound not to a specific state version but to a slot/place:

```text
let a = 20
link& b = a

mut a = 30
```

In this case `b` remains bound to the same slot/place and after changing `a` sees the new value.

The difference:

```text
link b = a
```

is a reference to a specific state.

```text
link& b = a
```

is an explicit binding to a slot/place.

`link&` does not make the reference immortal: when the slot dies (for example, a variable goes out of scope or an object is freed), the reference dies with it. If the slot's death is provable at compile time, using the reference after it is a compile error. If the slot can disappear only at runtime (for example, an element is removed from a container), the compiler inserts a slot-existence check; failure of the check is a panic, not UB.

The main idea:

> An ordinary reference is short-lived and bound to state. A long-lived reference must be expressed with separate syntax.

## `link* b = a` — a reference with freeze

`link*` is the strongest guarantee: while the reference is alive, the data it points to cannot be changed at all. Neither through the reference nor through the original.

```text
let a = 20
link* b = a

mut a = 30    //// ERROR: a is frozen while link* b is alive
```

When `link*` dies, the freeze is lifted. While `link*` is alive, any mutation is forbidden, so the slot cannot disappear: the check is static, with no runtime cost.

The difference:

```text
link& b = a    // "I want to see the slot's changes" — mut a is allowed, b follows the changes
link* b = a    // "the data will not change while I am looking" — mut a is forbidden
```

`link&` is a statement by the programmer: "at this moment the slot exists and the path is valid". The compiler does not analyze what exactly is in the slot — it only checks that the path exists. If the slot disappeared, a runtime check triggers and the program panics. If the slot exists but contains something other than the programmer expected, that is a logic error in the code, not a reason for a panic.

Why `link*` is needed:
- parallel access — several threads read, nobody mutates;
- invariants — the data will not change while it is being processed;
- caching — the value is guaranteed valid.

Using a dead `link*` is a compile error: `link*` freezes the data, so the slot cannot disappear unnoticed.

Control is attached to the consumer, not to the data: while nobody reads the data, it is free. A consumer appeared (`link*`) — the data is frozen. The reference died — the control is switched off.

## References to elements

A reference can be created directly to the desired element:

```text
link b = a[0]
```

means a reference to the corresponding element of the current state. The reference is bound to a path (base + index/field), not to the whole object.

If `a` is then changed as a whole (`mut a = ...`), the ordinary relation to the previous state of `a[0]` ceases. Even if after the change `a[0]` happens to hold the same value, this is already a new state.

Invalidation follows the path: a mutation kills only those state-bound references whose path intersects the changed one. `mut a[1]` kills the `link` on `a[1]` and on `a` as a whole, but not on `a[0]`; `mut a` kills all references derived from `a`. If binding to a slot/place must be preserved, the long-lived reference form is used.

`link& b = a[0]` is a logical path (base + index), not the address of a cell in a buffer. Buffer reallocation does not affect `link&`: the path is resolved anew on every use. If the element is removed (for example, `pop` or `clear`), the runtime check does not find the slot and the program panics. If the element exists but a different value now sits at the index, this is a logic error by the programmer, not a language error.

## References are direct, without chains

A reference always points directly to data. A reference can only be created from a variable that holds data. From another reference — it cannot:

```text
link b = a
link c = b    // forbidden: a reference from a reference
```

If you want a reference to the data — write it directly on the variable with the data:

```text
link c = a    // correct
```

Chains are impossible in principle — not because the compiler catches them, but because they cannot be written. A reference always points directly to data, and this is guaranteed by the language itself.

Passing a reference to a computation or action is not a chain: the receiver gets the same window into the data and reads it directly. A chain is a reference to a variable that holds a reference; there is no such thing in the language.

## A reference does not escape a computation or action

A computation or action can accept a reference and read the data. Passing the reference further with a `link` parameter is not a chain: the same window into the data is passed, not a reference to a reference. The compiler does not need to analyze it anew — the same slot or the same state is used.

What is forbidden is precisely the escape of a reference:

- returning a reference from a computation or action;
- storing it in a struct, list, or closure;
- using it longer than the data lives.

If the data is needed deeper or longer, it is passed via `box` (copy or move) or via an index/arena, not via a reference.

```text
sub read(link data: Data) { ... }   // reads directly; returning a reference is impossible
```

## References do not hide

A reference does not escape: `link` lives where it was created, `link&`/`link*` — as long as they are alive and the target slot exists. A reference is not put into lists and not stored in structs — that is not done.

If data is needed by someone for a long time, it is not hidden in a reference but passed via box (move or copy). The data is either yours, or passed to someone else, or copied. There is no third option.

## A move kills references

On a move (when box decides on a move), the source variable dies: the new name becomes responsible for the data. Memory does not move. All references to it die together with the old name:

```text
let a = big_object
link b = a            // reference to a
let c = box(a)    // a: 1 use → move, a is dead
print(b)              // error: b is dead together with a
```

The rule is simple: the source variable died — references to it are dead. This also applies to `link&`: a move kills the slot, and with it the reference.

## Copy and independence

`let b = a` is **always a copy**. This is the programmer's explicit intent: "I need an independent copy". `a` remains alive.

`b` and `a` are no longer related in any way: mutating `a` does not affect `b`, and vice versa. A copy is the only way to get an independent value.

A copy is made from a variable with data: `let c = a`, not through a reference.

`box` is different: the compiler decides copy or move by the number of **consuming** uses. Readings through references (`link`) are not counted — you can read as many times as you like, that is not consumption.

## Choosing a mechanism is a statement of intent

The choice of mechanism expresses the programmer's intent:

- `let b = a` — "I need an independent copy";
- `link b = a` — "I need to read the data";
- `link& b = a` — "I want to see the slot's changes";
- `box(expr)` — "I am using the real data, the compiler will decide copy or move".

The operation's contract determines what to accept: a reference (reading) or box (using). If an object replacement is planned, the compiler will catch any use after it.

## A reference is reading

Through a reference you can only read. You cannot change or mutate through a reference — that is not its role. Mutation is `mut` on the variable itself where the data lives. A reference is incapable of this in principle.

An ordinary reference (`link`) is safe by definition: physically nothing can be broken through it, because it only reads. And it dies on mutation, because the data it looked at has changed — and it was reading the old state. `link&` survives mutation, therefore it checks the slot's existence at runtime; `link*` strengthens this to a freeze: while the reference is alive, the data cannot be changed even through the original.

## Logic errors are the code's responsibility

The compiler guarantees memory safety: a dead reference cannot be used. If the death is provable, it is a compile error; for `link&` with a slot that disappeared at runtime — panic. The compiler does not solve logic errors:

```text
let a = 10
link& b = a

mut a = new_object
```

Memory is fine: `b` is alive, the slot is alive, `b` sees the new value. But if `b` expected the data of the old object, this is a semantic error in the code, not in the language. If the slot has ceased to exist (for example, an element was removed from a container), the `link&` runtime check triggers — panic.
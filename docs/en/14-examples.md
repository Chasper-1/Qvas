# Examples

Two real examples: a Reverse Polish Notation calculator and a recursive descent parser. Each is explained step by step — what happens at what moment and which language mechanisms are involved.

---

## Example 1: Calculator (Reverse Polish Notation)

The expression `3 4 + 2 *` = `(3 + 4) * 2` = `14`.

```text
use std

--- types

enum TokenKind {
    Num,
    Add,
    Minus,
    Mul,
    Div,
}

struct Token box {
    kind,
    value,
}

--- errors

enum CalcError {
    UnexpectedChar,
    DivByZero,
    BadExpr,
}

--- lexer

fn digit_value(link c: str) -> i32 {
    switch c {
        "0" => 0,
        "1" => 1,
        "2" => 2,
        "3" => 3,
        "4" => 4,
        "5" => 5,
        "6" => 6,
        "7" => 7,
        "8" => 8,
        "9" => 9,
        else => -1,
    }
}

fn is_digit(link c: str) -> bool {
    digit_value(c) >= 0
}

fn tokenize(link input: str) -> [Token] or CalcError {
    let tokens: [Token] = {}
    let i = 0

    while i < input.len {
        let c = input[i]

        if c == " " {
            mut i += 1
            continue
        }

        if is_digit(c) {
            let num = 0
            while i < input.len and is_digit(input[i]) {
                mut num = num * 10 + digit_value(input[i])
                mut i += 1
            }
            let kind = TokenKind.Num
            let value = num
            mut tokens.push(struct Token box {
                kind,
                value,
            })
            continue
        }

        let kind = switch c {
            "+" => TokenKind.Add,
            "-" => TokenKind.Minus,
            "*" => TokenKind.Mul,
            "/" => TokenKind.Div,
            else => return CalcError::UnexpectedChar,
        }
        let value = 0

        mut tokens.push(struct Token box {
            kind,
            value,
        })
        mut i += 1
    }

    tokens
}

--- computation

fn eval(link tokens: [Token]) -> i32 or CalcError {
    let stack: [i32] = {}

    for token in tokens {
        if token.kind == TokenKind.Num {
            mut stack.push(token.value)
            continue
        }

        if stack.len < 2 {
            return CalcError::BadExpr
        }

        let b = mut stack.pop()
        let a = mut stack.pop()

        if token.kind == TokenKind.Div and b == 0 {
            return CalcError::DivByZero
        }

        let result = switch token.kind {
            Add => a + b,
            Minus => a - b,
            Mul => a * b,
            Div => a / b,
            else => return CalcError::BadExpr,
        }
        mut stack.push(result)
    }

    if stack.len != 1 {
        return CalcError::BadExpr
    }
    stack[0]
}

--- entry point

sub main() {
    let input = "3 4 + 2 *"
    let tokens = tokenize(input)
    switch tokens {
        CalcError::UnexpectedChar => std.print("unknown character"),
        else => {
            let result = eval(tokens)
            switch result {
                CalcError::DivByZero => std.print("division by zero"),
                CalcError::BadExpr => std.print("bad expression"),
                else => std.print(result),
            }
        }
    }
}
```

### How it works, step by step

**`use std`** — connect the standard library. After that `std.print()` is available by name.

**Zone `--- types`** — declare the data the program works with:
- `TokenKind` — an enum: which kinds of tokens exist (a number and four operations).
- `Token` — a struct: a token = kind + value (for a number — the number itself, for an operation — 0).

**Zone `--- errors`** — `CalcError` — an enum of possible errors. Computations will return "value or error".

**Zone `--- lexer`** — turns a string into a list of tokens:
- `fn digit_value(link c: str) -> i32` — the value of a digit: `"0"` → 0, …, `"9"` → 9, otherwise -1.
- `fn is_digit(link c: str) -> bool` — checks whether a character is a digit: `digit_value(c) >= 0`. The parameter `c` is a reference; the computation only reads it.
- `fn tokenize(link input: str) -> [Token] or CalcError`:
  - `link input: str` — a reference parameter: the computation **reads** the string, does not consume it. The caller can pass the string without `box`.
  - `[Token] or CalcError` — the answer: a list of tokens **or** an error. The error is handled by the caller via `switch`.
  - `let tokens: [Token] = {}` — create an empty list.
  - `while i < input.len` — loop over the string. `input.len` — reading the length (contract: length reads).
  - `let c = input[i]` — reading a character by index (contract: `a[0]` reads), `c` — a `str` of one character.
  - `if c == " "` — comparison (reads). Skip the space: `mut i += 1` — mutate the counter, `continue`.
  - `if is_digit(c)` — call a computation (reads). If it is a digit, assemble the number:
    - `mut num = num * 10 + digit_value(input[i])` — `digit_value` turns the character into a number; `mut` — the event of changing `num`.
    - `mut tokens.push(...)` — `push` **consumes** the value (box), `mut` — the event of changing the list.
  - `switch c { ... }` — a switch expression: character → token kind. `else => return CalcError::UnexpectedChar` — unknown character: return an error.
  - `tokens` — implicit return of the last expression: the list is ready.

**Zone `--- computation`** — `fn eval(link tokens: [Token]) -> i32 or CalcError`:
- `link tokens` — we read the token list (do not consume it).
- `let stack: [i32] = {}` — the number stack.
- `for token in tokens` — iteration over the list: `token` is a **reference** to an element, the list is not consumed.
- `if token.kind == TokenKind.Num` — a number: put it on the stack, `continue` — on to the next loop iteration.
- `if stack.len < 2 { return CalcError::BadExpr }` — before popping two numbers, check that there are enough of them on the stack; otherwise return `BadExpr`. `pop` on an empty stack is a runtime error (panic), so we do not let it happen.
- `let b`, `let a` — take two numbers from the stack: just take the top one, the next one becomes the top itself.
- `if ... Div and b == 0` — division-by-zero check, separately from the `switch`.
- `let result = switch token.kind { ... }` — `switch` as an expression: yields the result of the arithmetic operation. Each branch is one expression: `a + b`, `a - b`, `a * b`, `a / b`.
- `mut stack.push(result)` — put the result on the stack.
- `if stack.len != 1 { return CalcError::BadExpr }` — after all operations exactly one result must remain on the stack.
- `stack[0]` — reading the result (implicit return).

**Zone `--- entry point`** — `sub main()`:
- `let input = "3 4 + 2 *"` — an expression.
- `let tokens = tokenize(input)` — the lexer. `tokenize` returns `[Token] or CalcError`.
- `switch tokens { ... }` — handling the lexer error: `CalcError::UnexpectedChar` — unknown character; `else` — success, `tokens` is the token list.
- `let result = eval(tokens)` — the computation. `eval` reads `tokens` via `link` — passing without `box`.
- `switch result { ... }` — handling the computation error: `CalcError::DivByZero` — division by zero; `CalcError::BadExpr` — bad expression; `else` — success, `result` is a number.
- `std.print(result)` — output. `print` is a `sub`: no answer, call as a statement.

**Data flow:** `main` → `tokenize` (string → tokens) → `eval` (tokens → number) → `print`. Each computation and action does one thing, errors are handled explicitly via `switch`, contracts are visible in the signatures.

---

## Example 2: Recursive descent parser

Parses the expression `3 + 4 * (2 - 1)` into a tree of nodes (arena: nodes live in an array, references are indices).

```text
use std

--- types

enum TokenKind { Num, Add, Minus, Mul, Div, LParen, RParen, Eof }

struct Token box {
    kind,
    value,
}

enum NodeKind { Num, Bin, Neg }

struct Node box {
    kind,
    op,
    value,
    left,
    right,
}

struct Parser box {
    tokens,
    nodes,
    pos,
}

enum ParseError {
    BadExpr,
}

--- lexer

fn digit_value(link c: str) -> i32 {
    switch c {
        "0" => 0,
        "1" => 1,
        "2" => 2,
        "3" => 3,
        "4" => 4,
        "5" => 5,
        "6" => 6,
        "7" => 7,
        "8" => 8,
        "9" => 9,
        else => -1,
    }
}

fn is_digit(link c: str) -> bool {
    digit_value(c) >= 0
}

fn lex(link input: str) -> [Token] or ParseError {
    let tokens: [Token] = {}
    let i = 0

    while i < input.len {
        let c = input[i]

        if c == " " {
            mut i += 1
            continue
        }

        if is_digit(c) {
            let num = 0
            while i < input.len and is_digit(input[i]) {
                mut num = num * 10 + digit_value(input[i])
                mut i += 1
            }
            let kind = TokenKind.Num
            let value = num
            mut tokens.push(struct Token box {
                kind,
                value,
            })
            continue
        }

        let kind = switch c {
            "+" => TokenKind.Add,
            "-" => TokenKind.Minus,
            "*" => TokenKind.Mul,
            "/" => TokenKind.Div,
            "(" => TokenKind.LParen,
            ")" => TokenKind.RParen,
            else => return ParseError::BadExpr,
        }
        let value = 0

        mut tokens.push(struct Token box {
            kind,
            value,
        })
        mut i += 1
    }

    let kind = TokenKind.Eof
    let value = 0
    mut tokens.push(struct Token box {
        kind,
        value,
    })
    tokens
}

--- parser helpers

fn peek(link p: Parser) -> Token {
    p.tokens[p.pos]
}

fn advance(box p: Parser) -> Parser {
    mut p.pos += 1
    p
}

fn emit(box p: Parser, box node: Node) -> (i32, Parser) {
    let idx = p.nodes.len
    mut p.nodes.push(node)
    (idx, p)
}

--- recursive descent

fn parse_factor(box p: Parser) -> (i32, Parser) or ParseError {
    let tok = peek(p)

    switch tok.kind {
        Num => {
            let p1 = advance(box(p))
            let kind = NodeKind.Num
            let op = Num
            let value = tok.value
            let left = -1
            let right = -1
            let node = struct Node box {
                kind,
                op,
                value,
                left,
                right,
            }
            emit(box(p1, node))
        },
        Minus => {
            let p1 = advance(box(p))
            let r = parse_factor(box(p1))
            let (operand, p2) = switch r {
                ParseError::BadExpr => return ParseError::BadExpr,
                else => r,
            }
            let kind = NodeKind.Neg
            let op = Minus
            let value = 0
            let left = operand
            let right = -1
            let node = struct Node box {
                kind,
                op,
                value,
                left,
                right,
            }
            emit(box(p2, node))
        },
        LParen => {
            let p1 = advance(box(p))
            let r = parse_expr(box(p1))
            let (inner, p2) = switch r {
                ParseError::BadExpr => return ParseError::BadExpr,
                else => r,
            }
            let closing = peek(p2)
            if closing.kind != RParen {
                return ParseError::BadExpr
            }
            let p3 = advance(box(p2))
            (inner, p3)
        },
        else => ParseError::BadExpr,
    }
}

fn parse_term(box p: Parser) -> (i32, Parser) or ParseError {
    let r = parse_factor(box(p))
    let (left, p1) = switch r {
        ParseError::BadExpr => return ParseError::BadExpr,
        else => r,
    }
    term_tail(box(left, p1))
}

fn term_tail(box left: i32, box p: Parser) -> (i32, Parser) or ParseError {
    let tok = peek(p)
    if tok.kind != Mul and tok.kind != Div {
        return (left, p)
    }
    let p1 = advance(box(p))
    let r = parse_factor(box(p1))
    let (right, p2) = switch r {
        ParseError::BadExpr => return ParseError::BadExpr,
        else => r,
    }
    let kind = NodeKind.Bin
    let op = tok.kind
    let value = 0
    let node = struct Node box {
        kind,
        op,
        value,
        left,
        right,
    }
    let (idx, p3) = emit(box(p2, node))
    term_tail(box(idx, p3))
}

fn parse_expr(box p: Parser) -> (i32, Parser) or ParseError {
    let r = parse_term(box(p))
    let (left, p1) = switch r {
        ParseError::BadExpr => return ParseError::BadExpr,
        else => r,
    }
    expr_tail(box(left, p1))
}

fn expr_tail(box left: i32, box p: Parser) -> (i32, Parser) or ParseError {
    let tok = peek(p)
    if tok.kind != Add and tok.kind != Minus {
        return (left, p)
    }
    let p1 = advance(box(p))
    let r = parse_term(box(p1))
    let (right, p2) = switch r {
        ParseError::BadExpr => return ParseError::BadExpr,
        else => r,
    }
    let kind = NodeKind.Bin
    let op = tok.kind
    let value = 0
    let node = struct Node box {
        kind,
        op,
        value,
        left,
        right,
    }
    let (idx, p3) = emit(box(p2, node))
    expr_tail(box(idx, p3))
}

--- entry point

fn parse(box input: str) -> (i32, [Node]) or ParseError {
    let tokens_result = lex(input)
    let tokens = switch tokens_result {
        ParseError::BadExpr => return ParseError::BadExpr,
        else => tokens_result,
    }
    let nodes = {}
    let pos = 0
    let p = struct Parser box {
        tokens,
        nodes,
        pos,
    }
    let root_result = parse_expr(box(p))
    let (root, p1) = switch root_result {
        ParseError::BadExpr => return ParseError::BadExpr,
        else => root_result,
    }
    let last = peek(p1)
    if last.kind != Eof {
        return ParseError::BadExpr
    }
    (root, p1.nodes)
}

sub main() {
    let result = parse(box("3 + 4 * (2 - 1)"))
    switch result {
        ParseError::BadExpr => std.print("parse error"),
        else => {
            let (root, nodes) = result
            std.print(root)
            std.print(nodes.len)
        }
    }
}
```

### How it works, step by step

**Zone `--- types`** — the parser's data:
- `TokenKind` — token kinds, including parentheses and `Eof` (end of input).
- `Token` — a token: kind + value.
- `NodeKind` — kinds of tree nodes: a number, a binary operation, a unary minus.
- `Node` — a node: kind, operation, value, and the **indices** of the left/right subtrees (`left`/`right`). The tree is stored in an array, links are indices, not pointers. `-1` — "no node".
- `Parser` — parser state: tokens, the node array (arena), position.
- `ParseError` — possible errors.

**Zone `--- lexer`** — `fn lex(link input: str) -> [Token] or ParseError`:
- Works like `tokenize` from example 1, but additionally an `Eof` token is appended at the end — the end-of-input marker by which the parser will understand that the expression has ended.

**Zone `--- parser helpers`** — three computations:
- `fn peek(link p: Parser) -> Token` — **reads** the current token (`p.tokens[p.pos]`), does not move the position. A reference parameter: the parser is not consumed.
- `fn advance(box p: Parser) -> Parser` — **consumes** the parser, moves the position (`mut p.pos += 1`) and returns the new state. Here the model "data arrived (consumption), was changed via `mut`, was returned (return = passing)" works. The compiler decides copy or move by the number of uses.
- `fn emit(box p: Parser, box node: Node) -> (i32, Parser)` — adds a node to the arena (`mut p.nodes.push(node)`), returns a **tuple**: the index of the new node + the new parser state.

**Zone `--- recursive descent`** — the grammar:
- `parse_factor` — the lowest level: a number, a unary minus, or an expression in parentheses.
  - `Num` → create a number node, `emit` puts it into the arena.
  - `Minus` → unary minus: parse the operand recursively (`parse_factor(box(p1))`), check the result immediately via `switch`, create a `Neg` node.
  - `LParen` → parse the expression inside the parentheses (`parse_expr(box(p1))`), check the result via `switch`, then check the closing parenthesis (`if closing.kind != RParen`), otherwise — an error.
- `parse_term` / `term_tail` — multiplication and division (left-associative tail):
  - `parse_term` parses the first factor and calls `term_tail`.
  - `term_tail` looks: if the next token is `*` or `/` — parses the right factor, creates a `Bin` node and recursively calls itself. If not — returns as is. This is classic tail recursion for left associativity.
- `parse_expr` / `expr_tail` — the same for `+` and `-`.
- Precedence follows by itself: `parse_expr` calls `parse_term`, which calls `parse_factor`. Therefore `3 + 4 * (2 - 1)` is parsed as `3 + (4 * (2 - 1))`, not `(3 + 4) * (2 - 1)`.

**Zone `--- entry point`**:
- `fn parse(box input: str) -> (i32, [Node]) or ParseError`:
  - `let tokens_result = lex(input)` — the lexer. `parse` consumes the string via `box`; `lex` reads `input` directly and does not return a reference — the result is its own token list. The error is examined immediately via `switch`.
  - `let p = struct Parser box { tokens, nodes, pos }` — the initial parser state.
  - `let root_result = parse_expr(box(p))` — parsing the whole expression; the result is examined immediately via `switch`. `root` is the index of the root node in the arena.
  - `let last = peek(p1)` — after the expression there must be `Eof`, otherwise there are extra tokens in the expression.
  - `(root, p1.nodes)` — return: the root index + the whole node arena.
- `sub main()`:
  - `let result = parse(box("3 + 4 * (2 - 1)"))` — parsing. `parse` returns `(i32, [Node]) or ParseError`.
  - `switch result { ... }` — handling the error: `ParseError::BadExpr` — a message; `else` — success, `result` is the tuple `(i32, [Node])`.
  - `let (root, nodes) = result` — destructuring the tuple: the root node index and the node arena.
  - `std.print(root)` — the root node index.
  - `std.print(nodes.len)` — how many nodes are in the arena.

**Data flow:** `main` → `parse` → `lex` (string → tokens) → `parse_expr` → `parse_term` → `parse_factor` (tokens → node arena). Parser state is passed explicitly: each computation receives the parser via `box`, mutates it, and returns the new state — no global variables.

---

## Which language mechanisms are involved

| Mechanism | Where | What it does |
|---|---|---|
| `use std` | both | module import, access to `std.print()` |
| `--- zone` | both | file structure: types / errors / lexer / computation / entry point |
| `enum X { ... }` | both | creating an enum (token kinds, node kinds, errors) |
| `struct X box { ... }` | both | creating a struct (token, node, parser) |
| `fn name(params) -> T or E` | both | computation: returns a value or an error, the answer is mandatory |
| `sub name(params)` | both | action: no answer (`main`, `print`, `push`) |
| `link param: T` | both | reference parameter: the computation or action reads, does not consume |
| `box param: T` | example 2 | consuming parameter: the compiler decides copy/move. At the call, one `box(x, y)` for all consuming arguments |
| `[T]` | both | list (tokens, stack, node arena) |
| `return Error::Variant` | both | returning an error |
| `switch x { ... }` | both | switch expression (yields a value), `else` — the default branch |
| `mut x += 1` | both | mutation: a change event |
| `mut list.push(x)` | both | mutating method: `push` consumes the value |
| `for item in list` | example 1 | iteration: `item` is a reference to an element |
| `a[i]` | both | reading by index (reference) |
| `a.len` | both | reading the length (contract: length reads) |
| `struct Token box { kind, value }` | both | creating a struct |
| `(i32, Parser)` | example 2 | tuple: multiple return values |
| implicit return of the last expression | both | `tokens`, `stack[0]`, `(idx, p)` |
| `std.print()` | both | calling an action from std |
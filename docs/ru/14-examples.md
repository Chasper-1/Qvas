# Примеры

Два реальных примера: калькулятор на обратной польской записи и парсер рекурсивного спуска. Каждый разобран по шагам — что происходит в какой момент и какие механизмы языка задействованы.

---

## Пример 1: Калькулятор (обратная польская запись)

Выражение `3 4 + 2 *` = `(3 + 4) * 2` = `14`.

```text
use std

--- типы

enum TokenKind {
    Num,
    Add,
    Minus,
    Mul,
    Div,
}

struct Token {
    kind: TokenKind,
    value: i32,
}

--- ошибки

enum CalcError {
    UnexpectedChar,
    DivByZero,
    BadExpr,
}

--- лексер

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
            mut tokens.push(Token { kind: TokenKind.Num, value: num })
            continue
        }

        let kind = switch c {
            "+" => TokenKind.Add,
            "-" => TokenKind.Minus,
            "*" => TokenKind.Mul,
            "/" => TokenKind.Div,
            else => return CalcError::UnexpectedChar,
        }

        mut tokens.push(Token { kind: kind, value: 0 })
        mut i += 1
    }

    tokens
}

--- вычисление

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

--- точка входа

sub main() {
    let input = "3 4 + 2 *"
    let tokens = tokenize(input)
    switch tokens {
        CalcError::UnexpectedChar => std.print("неизвестный символ"),
        else => {
            let result = eval(tokens)
            switch result {
                CalcError::DivByZero => std.print("деление на ноль"),
                CalcError::BadExpr => std.print("неверное выражение"),
                else => std.print(result),
            }
        }
    }
}
```

### Как это работает по шагам

**`use std`** — подключаем стандартную библиотеку. Дальше `std.print()` будет доступен по имени.

**Зона `--- типы`** — объявляем данные, с которыми работает программа:
- `TokenKind` — enum: какие виды токенов бывают (число и четыре операции).
- `Token` — структура: токен = вид + значение (для числа — само число, для операции — 0).

**Зона `--- ошибки`** — `CalcError` — enum возможных ошибок. Вычисления будут возвращать «значение или ошибку».

**Зона `--- лексер`** — превращает строку в список токенов:
- `fn digit_value(link c: str) -> i32` — значение цифры: `"0"` → 0, …, `"9"` → 9, иначе -1.
- `fn is_digit(link c: str) -> bool` — проверка, цифра ли символ: `digit_value(c) >= 0`. Параметр `c` — ссылка, вычисление его только читает.
- `fn tokenize(link input: str) -> [Token] or CalcError`:
  - `link input: str` — параметр-ссылка: вычисление **читает** строку, не потребляет. Вызывающий может передать строку без `box`.
  - `[Token] or CalcError` — ответ: список токенов **или** ошибка. Ошибку обрабатывает вызывающий через `switch`.
  - `let tokens: [Token] = {}` — создаём пустой список.
  - `while i < input.len` — цикл по строке. `input.len` — чтение длины (контракт: длина читает).
  - `let c = input[i]` — чтение символа по индексу (контракт: `a[0]` читает), `c` — `str` из одного символа.
  - `if c == " "` — сравнение (читает). Пробел пропускаем: `mut i += 1` — мутация счётчика, `continue`.
  - `if is_digit(c)` — вызов вычисления (читает). Если цифра — собираем число:
    - `mut num = num * 10 + digit_value(input[i])` — `digit_value` превращает символ в число; `mut` — событие изменения `num`.
    - `mut tokens.push(...)` — `push` **потребляет** значение (box), `mut` — событие изменения списка.
  - `switch c { ... }` — switch-выражение: символ → вид токена. `else => return CalcError::UnexpectedChar` — неизвестный символ: возврат ошибки.
  - `tokens` — неявный возврат последнего выражения: список готов.

**Зона `--- вычисление`** — `fn eval(link tokens: [Token]) -> i32 or CalcError`:
- `link tokens` — читаем список токенов (не потребляем).
- `let stack: [i32] = {}` — стек чисел.
- `for token in tokens` — итерация по списку: `token` — **ссылка** на элемент, список не потребляется.
- `if token.kind == TokenKind.Num` — число: кладём на стек, `continue` — дальше по циклу.
- `if stack.len < 2 { return CalcError::BadExpr }` — перед снятием двух чисел проверяем, что на стеке их достаточно; иначе возвращаем `BadExpr`. `pop` на пустом стеке — ошибка времени выполнения (panic), поэтому до неё не доводим.
- `let b`, `let a` — берём два числа со стека: просто берём верхний, следующий сам становится верхним.
- `if ... Div and b == 0` — проверка деления на ноль, отдельно от `switch`.
- `let result = switch token.kind { ... }` — `switch` как выражение: выдаёт результат арифметической операции. Каждая ветка — одно выражение: `a + b`, `a - b`, `a * b`, `a / b`.
- `mut stack.push(result)` — кладём результат на стек.
- `if stack.len != 1 { return CalcError::BadExpr }` — после всех операций на стеке должен остаться ровно один результат.
- `stack[0]` — чтение результата (неявный возврат).

**Зона `--- точка входа`** — `sub main()`:
- `let input = "3 4 + 2 *"` — выражение.
- `let tokens = tokenize(input)` — лексер. `tokenize` возвращает `[Token] or CalcError`.
- `switch tokens { ... }` — обработка ошибки лексера: `CalcError::UnexpectedChar` — неизвестный символ; `else` — успех, `tokens` — список токенов.
- `let result = eval(tokens)` — вычисление. `eval` читает `tokens` через `link` — передача без `box`.
- `switch result { ... }` — обработка ошибки вычисления: `CalcError::DivByZero` — деление на ноль; `CalcError::BadExpr` — неверное выражение; `else` — успех, `result` — число.
- `std.print(result)` — вывод. `print` — `sub`: ответа нет, вызов как оператор.

**Поток данных:** `main` → `tokenize` (строка → токены) → `eval` (токены → число) → `print`. Каждое вычисление и действие делает одно дело, ошибки обрабатываются явно через `switch`, контракты видны в сигнатурах.

---

## Пример 2: Парсер рекурсивного спуска

Разбирает выражение `3 + 4 * (2 - 1)` в дерево узлов (арена: узлы лежат в массиве, ссылки — индексы).

```text
use std

--- типы

enum TokenKind { Num, Add, Minus, Mul, Div, LParen, RParen, Eof }

struct Token {
    kind: TokenKind,
    value: i32,
}

enum NodeKind { Num, Bin, Neg }

struct Node {
    kind: NodeKind,
    op: TokenKind,
    value: i32,
    left: i32,
    right: i32,
}

struct Parser {
    tokens: [Token],
    nodes: [Node],
    pos: i32,
}

enum ParseError {
    BadExpr,
}

--- лексер

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
            mut tokens.push(Token { kind: TokenKind.Num, value: num })
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

        mut tokens.push(Token { kind: kind, value: 0 })
        mut i += 1
    }

    mut tokens.push(Token { kind: TokenKind.Eof, value: 0 })
    tokens
}

--- хелперы парсера

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

--- рекурсивный спуск

fn parse_factor(box p: Parser) -> (i32, Parser) or ParseError {
    let tok = peek(p)

    switch tok.kind {
        Num => {
            let p1 = advance(box(p))
            emit(box(p1, Node { kind: NodeKind.Num, op: Num, value: tok.value, left: -1, right: -1 }))
        },
        Minus => {
            let p1 = advance(box(p))
            let r = parse_factor(box(p1))
            let (operand, p2) = switch r {
                ParseError::BadExpr => return ParseError::BadExpr,
                else => r,
            }
            emit(box(p2, Node { kind: NodeKind.Neg, op: Minus, value: 0, left: operand, right: -1 }))
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
    let (idx, p3) = emit(box(p2, Node { kind: NodeKind.Bin, op: tok.kind, value: 0, left: left, right: right }))
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
    let (idx, p3) = emit(box(p2, Node { kind: NodeKind.Bin, op: tok.kind, value: 0, left: left, right: right }))
    expr_tail(box(idx, p3))
}

--- точка входа

fn parse(box input: str) -> (i32, [Node]) or ParseError {
    let tokens_result = lex(input)
    let tokens = switch tokens_result {
        ParseError::BadExpr => return ParseError::BadExpr,
        else => tokens_result,
    }
    let p = Parser { tokens: tokens, nodes: {}, pos: 0 }
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
        ParseError::BadExpr => std.print("ошибка разбора"),
        else => {
            let (root, nodes) = result
            std.print(root)
            std.print(nodes.len)
        }
    }
}
```

### Как это работает по шагам

**Зона `--- типы`** — данные парсера:
- `TokenKind` — виды токенов, включая скобки и `Eof` (конец входа).
- `Token` — токен: вид + значение.
- `NodeKind` — виды узлов дерева: число, бинарная операция, унарный минус.
- `Node` — узел: вид, операция, значение и **индексы** левого/правого поддерева (`left`/`right`). Дерево хранится в массиве, связи — индексы, а не указатели. `-1` — «нет узла».
- `Parser` — состояние парсера: токены, массив узлов (арена), позиция.
- `ParseError` — возможные ошибки.

**Зона `--- лексер`** — `fn lex(link input: str) -> [Token] or ParseError`:
- Работает как `tokenize` из примера 1, но дополнительно в конец добавляется токен `Eof` — маркер конца входа, по которому парсер поймёт, что выражение закончилось.

**Зона `--- хелперы парсера`** — три вычисления:
- `fn peek(link p: Parser) -> Token` — **читает** текущий токен (`p.tokens[p.pos]`), не двигает позицию. Параметр-ссылка: парсер не потребляется.
- `fn advance(box p: Parser) -> Parser` — **потребляет** парсер, двигает позицию (`mut p.pos += 1`) и возвращает новое состояние. Здесь работает модель «данные пришли (потребление), поменялись через `mut`, вернулись (return = передача)». Компилятор решает копия или move по числу использований.
- `fn emit(box p: Parser, box node: Node) -> (i32, Parser)` — добавляет узел в арену (`mut p.nodes.push(node)`), возвращает **кортеж**: индекс нового узла + новое состояние парсера.

**Зона `--- рекурсивный спуск`** — грамматика:
- `parse_factor` — самый низкий уровень: число, унарный минус или выражение в скобках.
  - `Num` → создаём узел числа, `emit` кладёт его в арену.
  - `Minus` → унарный минус: разбираем операнд рекурсивно (`parse_factor(box(p1))`), результат сразу проверяем через `switch`, создаём узел `Neg`.
  - `LParen` → разбираем выражение внутри скобок (`parse_expr(box(p1))`), результат проверяем через `switch`, затем проверяем закрывающую скобку (`if closing.kind != RParen`), иначе — ошибка.
- `parse_term` / `term_tail` — умножение и деление (левый ассоциативный хвост):
  - `parse_term` разбирает первый множитель и вызывает `term_tail`.
  - `term_tail` смотрит: если следующий токен `*` или `/` — разбирает правый множитель, создаёт узел `Bin` и рекурсивно вызывает себя. Если нет — возвращает как есть. Это классическая хвостовая рекурсия для левой ассоциативности.
- `parse_expr` / `expr_tail` — то же самое для `+` и `-`.
- Приоритет получается сам собой: `parse_expr` вызывает `parse_term`, тот — `parse_factor`. Поэтому `3 + 4 * (2 - 1)` разбирается как `3 + (4 * (2 - 1))`, а не `(3 + 4) * (2 - 1)`.

**Зона `--- точка входа`**:
- `fn parse(box input: str) -> (i32, [Node]) or ParseError`:
  - `let tokens_result = lex(input)` — лексер. `parse` потребляет строку через `box`; `lex` читает `input` напрямую и не возвращает ссылку — результат собственный список токенов. Ошибка сразу разбирается через `switch`.
  - `let p = Parser { tokens: tokens, nodes: {}, pos: 0 }` — начальное состояние парсера.
  - `let root_result = parse_expr(box(p))` — разбор всего выражения; результат сразу разбирается через `switch`. `root` — индекс корневого узла в арене.
  - `let last = peek(p1)` — после выражения должен быть `Eof`, иначе в выражении лишние токены.
  - `(root, p1.nodes)` — возврат: индекс корня + вся арена узлов.
- `sub main()`:
  - `let result = parse(box("3 + 4 * (2 - 1)"))` — разбор. `parse` возвращает `(i32, [Node]) or ParseError`.
  - `switch result { ... }` — обработка ошибки: `ParseError::BadExpr` — сообщение; `else` — успех, `result` — кортеж `(i32, [Node])`.
  - `let (root, nodes) = result` — разбор кортежа: индекс корневого узла и арена узлов.
  - `std.print(root)` — индекс корневого узла.
  - `std.print(nodes.len)` — сколько узлов в арене.

**Поток данных:** `main` → `parse` → `lex` (строка → токены) → `parse_expr` → `parse_term` → `parse_factor` (токены → арена узлов). Состояние парсера передаётся явно: каждое вычисление получает парсер через `box`, мутирует и возвращает новое состояние — никаких глобальных переменных.

---

## Какие механизмы языка задействованы

| Механизм | Где | Что делает |
|---|---|---|
| `use std` | оба | импорт модуля, доступ к `std.print()` |
| `--- зона` | оба | структура файла: типы / ошибки / лексер / вычисление / точка входа |
| `enum X { ... }` | оба | создание enum (виды токенов, узлов, ошибок) |
| `struct X { ... }` | оба | создание структуры (токен, узел, парсер) |
| `fn name(params) -> T or E` | оба | вычисление: отдаёт значение или ошибку, ответ обязателен |
| `sub name(params)` | оба | действие: ответа нет (`main`, `print`, `push`) |
| `link param: T` | оба | параметр-ссылка: вычисление или действие читает, не потребляет |
| `box param: T` | пример 2 | потребляющий параметр: компилятор решает копия/move. На вызове один `box(x, y)` на все потребляющие аргументы |
| `[T]` | оба | список (токены, стек, арена узлов) |
| `return Error::Variant` | оба | возврат ошибки |
| `switch x { ... }` | оба | switch-выражение (выдаёт значение), `else` — ветка по умолчанию |
| `mut x += 1` | оба | мутация: событие изменения |
| `mut list.push(x)` | оба | мутирующий метод: `push` потребляет значение |
| `for item in list` | пример 1 | итерация: `item` — ссылка на элемент |
| `a[i]` | оба | чтение по индексу (ссылка) |
| `a.len` | оба | чтение длины (контракт: длина читает) |
| `Token { kind: ..., value: ... }` | оба | создание структуры |
| `(i32, Parser)` | пример 2 | кортеж: несколько возвращаемых значений |
| неявный возврат последнего выражения | оба | `tokens`, `stack[0]`, `(idx, p)` |
| `std.print()` | оба | вызов действия из std |
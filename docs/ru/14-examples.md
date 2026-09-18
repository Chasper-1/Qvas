# Примеры

Два реальных примера: калькулятор на обратной польской записи и парсер рекурсивного спуска. Каждый разобран по шагам — что происходит в какой момент и какие механизмы языка задействованы.

---

## Пример 1: Калькулятор (обратная польская запись)

Выражение `3 4 + 2 *` = `(3 + 4) * 2` = `14`.

```text
use std

--- типы

let TokenKind = enum {
    Num,
    Add,
    Sub,
    Mul,
    Div,
}

let Token = struct {
    kind: TokenKind,
    value: int,
}

--- ошибки

let CalcError = enum {
    UnexpectedChar,
    DivByZero,
    BadExpr,
}

--- лексер

fn is_digit(link c: u8) -> bool {
    c >= '0' and c <= '9'
}

fn tokenize(link input: str) -> [Token] or CalcError {
    let tokens: [Token] = {}
    let i = 0

    while i < input.len {
        let c = input[i]

        if c == ' ' {
            mut i += 1
            continue
        }

        if is_digit(c) {
            let num = 0
            while i < input.len and is_digit(input[i]) {
                mut num = num * 10 + (input[i] - '0')
                mut i += 1
            }
            mut tokens.push(Token { kind: TokenKind.Num, value: num })
            continue
        }

        let kind = switch c {
            '+' => TokenKind.Add,
            '-' => TokenKind.Sub,
            '*' => TokenKind.Mul,
            '/' => TokenKind.Div,
            else => return CalcError::UnexpectedChar,
        }

        mut tokens.push(Token { kind: kind, value: 0 })
        mut i += 1
    }

    tokens
}

--- вычисление

fn eval(link tokens: [Token]) -> int or CalcError {
    let stack: [int] = {}

    for token in tokens {
        if token.kind == TokenKind.Num {
            mut stack.push(token.value)
            continue
        }

        let b = mut stack.pop()
        let a = mut stack.pop()

        if token.kind == TokenKind.Div and b == 0 {
            return CalcError::DivByZero
        }

        let result = switch token.kind {
            Add => a + b,
            Sub => a - b,
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

fn.nothing main() {
    let input = "3 4 + 2 *"
    let tokens = tokenize(input)
    let result = eval(tokens)
    std.print(result)
}
```

### Как это работает по шагам

**`use std`** — подключаем стандартную библиотеку. Дальше `std.print()` будет доступен по имени.

**Зона `--- типы`** — объявляем данные, с которыми работает программа:
- `TokenKind` — enum: какие виды токенов бывают (число и четыре операции).
- `Token` — структура: токен = вид + значение (для числа — само число, для операции — 0).

**Зона `--- ошибки`** — `CalcError` — enum возможных ошибок. Функции будут возвращать «значение или ошибку».

**Зона `--- лексер`** — превращает строку в список токенов:
- `fn is_digit(link c: u8) -> bool` — проверка, цифра ли символ. Параметр `c` — ссылка, функция его только читает.
- `fn tokenize(link input: str) -> [Token] or CalcError`:
  - `link input: str` — параметр-ссылка: функция **читает** строку, не потребляет. Вызывающий может передать строку без `databox`.
  - `[Token] or CalcError` — возврат: список токенов **или** ошибка. Механизм ошибок как в Rust.
  - `let tokens: [Token] = {}` — создаём пустой список.
  - `while i < input.len` — цикл по строке. `input.len` — чтение длины (контракт: длина читает).
  - `let c = input[i]` — чтение символа по индексу (контракт: `a[0]` читает), `c` — копия символа.
  - `if c == ' '` — сравнение (читает). Пробел пропускаем: `mut i += 1` — мутация счётчика, `continue`.
  - `if is_digit(c)` — вызов функции (читает). Если цифра — собираем число:
    - `mut num = num * 10 + (input[i] - '0')` — арифметика **потребляет** значения и создаёт новое; `mut` — событие изменения `num`.
    - `mut tokens.push(...)` — `push` **потребляет** значение (databox), `mut` — событие изменения списка.
  - `switch c { ... }` — switch-выражение: символ → вид токена. `else => return CalcError::UnexpectedChar` — неизвестный символ: возврат ошибки.
  - `tokens` — неявный возврат последнего выражения: список готов.

**Зона `--- вычисление`** — `fn eval(link tokens: [Token]) -> int or CalcError`:
- `link tokens` — читаем список токенов (не потребляем).
- `let stack: [int] = {}` — стек чисел.
- `for token in tokens` — итерация по списку: `token` — **ссылка** на элемент, список не потребляется.
- `if token.kind == TokenKind.Num` — число: кладём на стек, `continue` — дальше по циклу.
- `let b`, `let a` — берём два числа со стека: просто берём верхний, следующий сам становится верхним. Пустой стек — логическая ошибка кода, не проверяется.
- `if ... Div and b == 0` — проверка деления на ноль, отдельно от `switch`.
- `let result = switch token.kind { ... }` — `switch` как выражение: выдаёт результат арифметической операции. Каждая ветка — одно выражение: `a + b`, `a - b`, `a * b`, `a / b`.
- `mut stack.push(result)` — кладём результат на стек.
- `if stack.len != 1 { return CalcError::BadExpr }` — после всех операций на стеке должен остаться ровно один результат.
- `stack[0]` — чтение результата (неявный возврат).

**Зона `--- точка входа`** — `fn.nothing main()`:
- `let input = "3 4 + 2 *"` — выражение.
- `let tokens = tokenize(input)` — лексер. 
- `let result = eval(tokens)` — вычисление. `eval` читает `tokens` через `link` — передача без `databox`.
- `std.print(result)` — вывод. `print` — `fn.nothing`: ответ НИЧЕГО, вызов как оператор.

**Поток данных:** `main` → `tokenize` (строка → токены) → `eval` (токены → число) → `print`. Каждая функция делает одно дело, ошибки пробрасываются через `?`, контракты видны в сигнатурах.

---

## Пример 2: Парсер рекурсивного спуска

Разбирает выражение `3 + 4 * (2 - 1)` в дерево узлов (арена: узлы лежат в массиве, ссылки — индексы).

```text
use std

--- типы

let TokenKind = enum { Num, Add, Sub, Mul, Div, LParen, RParen, Eof }

let Token = struct {
    kind: TokenKind,
    value: int,
}

let NodeKind = enum { Num, Bin, Neg }

let Node = struct {
    kind: NodeKind,
    op: TokenKind,
    value: int,
    left: int,
    right: int,
}

let Parser = struct {
    tokens: [Token],
    nodes: [Node],
    pos: int,
}

let ParseError = enum {
    UnexpectedChar,
    UnexpectedToken,
    UnexpectedEof,
    BadExpr,
}

--- лексер

fn is_digit(link c: u8) -> bool {
    c >= '0' and c <= '9'
}

fn lex(link input: str) -> [Token] or ParseError {
    let tokens: [Token] = {}
    let i = 0

    while i < input.len {
        let c = input[i]

        if c == ' ' {
            mut i += 1
            continue
        }

        if is_digit(c) {
            let num = 0
            while i < input.len and is_digit(input[i]) {
                mut num = num * 10 + (input[i] - '0')
                mut i += 1
            }
            mut tokens.push(Token { kind: TokenKind.Num, value: num })
            continue
        }

        let kind = switch c {
            '+' => TokenKind.Add,
            '-' => TokenKind.Sub,
            '*' => TokenKind.Mul,
            '/' => TokenKind.Div,
            '(' => TokenKind.LParen,
            ')' => TokenKind.RParen,
            else => return ParseError::UnexpectedChar,
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

fn advance(databox p: Parser) -> Parser {
    mut p.pos += 1
    p
}

fn emit(databox p: Parser, databox node: Node) -> (int, Parser) {
    let idx = p.nodes.len
    mut p.nodes.push(node)
    (idx, p)
}

--- рекурсивный спуск

fn parse_factor(databox p: Parser) -> (int, Parser) or ParseError {
    let tok = peek(p)

    switch tok.kind {
        Num => {
            let p1 = advance(databox(p))
            emit(databox(p1, Node { kind: NodeKind.Num, op: Num, value: tok.value, left: -1, right: -1 }))
        },
        Sub => {
            let p1 = advance(databox(p))
            let (operand, p2) = parse_factor(databox(p1))?
            emit(databox(p2, Node { kind: NodeKind.Neg, op: Sub, value: 0, left: operand, right: -1 }))
        },
        LParen => {
            let p1 = advance(databox(p))
            let (inner, p2) = parse_expr(databox(p1))?
            let closing = peek(p2)
            if closing.kind != RParen {
                return ParseError::UnexpectedToken
            }
            let p3 = advance(databox(p2))
            (inner, p3)
        },
        else => ParseError::UnexpectedToken,
    }
}

fn parse_term(databox p: Parser) -> (int, Parser) or ParseError {
    let (left, p1) = parse_factor(databox(p))?
    term_tail(databox(left, p1))
}

fn term_tail(databox left: int, databox p: Parser) -> (int, Parser) or ParseError {
    let tok = peek(p)
    if tok.kind != Mul and tok.kind != Div {
        return (left, p)
    }
    let p1 = advance(databox(p))
    let (right, p2) = parse_factor(databox(p1))?
    let (idx, p3) = emit(databox(p2, Node { kind: NodeKind.Bin, op: tok.kind, value: 0, left: left, right: right }))
    term_tail(databox(idx, p3))
}

fn parse_expr(databox p: Parser) -> (int, Parser) or ParseError {
    let (left, p1) = parse_term(databox(p))?
    expr_tail(databox(left, p1))
}

fn expr_tail(databox left: int, databox p: Parser) -> (int, Parser) or ParseError {
    let tok = peek(p)
    if tok.kind != Add and tok.kind != Sub {
        return (left, p)
    }
    let p1 = advance(databox(p))
    let (right, p2) = parse_term(databox(p1))?
    let (idx, p3) = emit(databox(p2, Node { kind: NodeKind.Bin, op: tok.kind, value: 0, left: left, right: right }))
    expr_tail(databox(idx, p3))
}

--- точка входа

fn parse(databox input: str) -> (int, [Node]) or ParseError {
    let tokens = lex(input)?
    let p = Parser { tokens: tokens, nodes: {}, pos: 0 }
    let (root, p1) = parse_expr(databox(p))?
    let last = peek(p1)
    if last.kind != Eof {
        return ParseError::UnexpectedToken
    }
    (root, p1.nodes)
}

fn.nothing main() {
    let (root, nodes) = parse(databox("3 + 4 * (2 - 1)"))
    std.print(root)
    std.print(nodes.len)
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

**Зона `--- хелперы парсера`** — три функции:
- `fn peek(link p: Parser) -> Token` — **читает** текущий токен (`p.tokens[p.pos]`), не двигает позицию. Параметр-ссылка: парсер не потребляется.
- `fn advance(databox p: Parser) -> Parser` — **потребляет** парсер, двигает позицию (`mut p.pos += 1`) и возвращает новое состояние. Здесь работает модель «данные пришли (потребление), поменялись через `mut`, вернулись (return = передача)». Компилятор решает копия или move по числу использований.
- `fn emit(databox p: Parser, databox node: Node) -> (int, Parser)` — добавляет узел в арену (`mut p.nodes.push(node)`), возвращает **кортеж**: индекс нового узла + новое состояние парсера.

**Зона `--- рекурсивный спуск`** — грамматика:
- `parse_factor` — самый низкий уровень: число, унарный минус или выражение в скобках.
  - `Num` → создаём узел числа, `emit` кладёт его в арену.
  - `Sub` → унарный минус: разбираем операнд рекурсивно (`parse_factor(databox(p1))?`), создаём узел `Neg`.
  - `LParen` → разбираем выражение внутри скобок (`parse_expr(databox(p1))?`), проверяем закрывающую скобку (`if closing.kind != RParen`), иначе — ошибка.
- `parse_term` / `term_tail` — умножение и деление (левый ассоциативный хвост):
  - `parse_term` разбирает первый множитель и вызывает `term_tail`.
  - `term_tail` смотрит: если следующий токен `*` или `/` — разбирает правый множитель, создаёт узел `Bin` и рекурсивно вызывает себя. Если нет — возвращает как есть. Это классическая хвостовая рекурсия для левой ассоциативности.
- `parse_expr` / `expr_tail` — то же самое для `+` и `-`.
- Приоритет получается сам собой: `parse_expr` вызывает `parse_term`, тот — `parse_factor`. Поэтому `3 + 4 * (2 - 1)` разбирается как `3 + (4 * (2 - 1))`, а не `(3 + 4) * (2 - 1)`.

**Зона `--- точка входа`**:
- `fn parse(databox input: str) -> (int, [Node]) or ParseError`:
  - `let tokens = lex(input)?` — лексер. `parse` потребляет строку через `databox`, `lex` — конечный читатель: проброса ссылки нет.
  - `let p = Parser { tokens: tokens, nodes: {}, pos: 0 }` — начальное состояние парсера.
  - `let (root, p1) = parse_expr(databox(p))?` — разбор всего выражения. `root` — индекс корневого узла в арене.
  - `let last = peek(p1)` — после выражения должен быть `Eof`, иначе в выражении лишние токены.
  - `(root, p1.nodes)` — возврат: индекс корня + вся арена узлов.
- `fn.nothing main()`:
  - `let (root, nodes) = parse(databox("3 + 4 * (2 - 1)"))` — разбор.
  - `std.print(root)` — индекс корневого узла.
  - `std.print(nodes.len)` — сколько узлов в арене.

**Поток данных:** `main` → `parse` → `lex` (строка → токены) → `parse_expr` → `parse_term` → `parse_factor` (токены → арена узлов). Состояние парсера передаётся явно: каждая функция получает парсер через `databox`, мутирует и возвращает новое состояние — никаких глобальных переменных.

---

## Какие механизмы языка задействованы

| Механизм | Где | Что делает |
|---|---|---|
| `use std` | оба | импорт модуля, доступ к `std.print()` |
| `--- зона` | оба | структура файла: типы / ошибки / лексер / вычисление / точка входа |
| `let X = enum { ... }` | оба | создание enum (виды токенов, узлов, ошибок) |
| `let X = struct { ... }` | оба | создание структуры (токен, узел, парсер) |
| `fn name(params) -> T or E` | оба | функция, возвращающая значение или ошибку |
| `fn.nothing name(params)` | оба | функция, отдающая ответ НИЧЕГО (`main`, `print`, `push`) |
| `link param: T` | оба | параметр-ссылка: функция читает, не потребляет |
| `databox param: T` | пример 2 | потребляющий параметр: компилятор решает копия/move. На вызове один `databox(x, y)` на все потребляющие аргументы |
| `[T]` | оба | список (токены, стек, арена узлов) |
| `expr?` | оба | проброс ошибки одной командой |
| `return Error::Variant` | оба | возврат ошибки |
| `switch x { ... }` | оба | switch-выражение (выдаёт значение), `else` — ветка по умолчанию |
| `mut x += 1` | оба | мутация: событие изменения |
| `mut list.push(x)` | оба | мутирующий метод: `push` потребляет значение |
| `for item in list` | пример 1 | итерация: `item` — ссылка на элемент |
| `a[i]` | оба | чтение по индексу (ссылка) |
| `a.len` | оба | чтение длины (контракт: длина читает) |
| `Token { kind: ..., value: ... }` | оба | создание структуры |
| `(int, Parser)` | пример 2 | кортеж: несколько возвращаемых значений |
| неявный возврат последнего выражения | оба | `tokens`, `stack[0]`, `(idx, p)` |
| `std.print()` | оба | вызов функции из std |
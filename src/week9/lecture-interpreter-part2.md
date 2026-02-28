# Паттерн «Интерпретатор» — Часть 2: Реализация LISP и AST на JavaScript/TypeScript

## Введение

В этой лекции мы подробно разберём реализацию LISP-интерпретатора на JavaScript и TypeScript. Будет рассмотрено три варианта реализации: классический (на классах с наследованием), упрощённый (без абстрактного класса, с коллекцией операторов) и расширенный (с поддержкой операций `car`, `cdr`, `list`, `eq`).

LISP — это язык, основанный на списках. Программа на LISP — это вложенные списки, где первый элемент списка является оператором, а остальные — операндами. Например:

```
(+ 1 2)          ; сложение: 1 + 2 = 3
(* 2 3)          ; умножение: 2 * 3 = 6
(- 10 3)         ; вычитание: 10 - 3 = 7
(+ (* 2 3) 1)    ; вложенное: (2 * 3) + 1 = 7
(+ x y)          ; переменные из контекста
```

Интерпретатор принимает такое выражение, строит AST (Abstract Syntax Tree — абстрактное синтаксическое дерево) и вычисляет результат.

---

## Реализация LISP-интерпретатора

### Архитектура

Интерпретатор состоит из трёх слоёв:

1. **Токенизатор (Tokenizer)** — разбивает строку на токены (скобки, числа, идентификаторы).
2. **Парсер (Parser)** — преобразует плоский список токенов в дерево вложенных массивов, а затем в экземпляры классов выражений.
3. **Вычислитель (Evaluator)** — рекурсивно обходит дерево и вычисляет значение каждого узла.

---

## AST

AST (Abstract Syntax Tree) — абстрактное синтаксическое дерево. Каждый узел дерева представляет одну синтаксическую конструкцию языка. В нашем случае узлами являются:

- `NumberExpression` — числовой литерал;
- `VariableExpression` — ссылка на переменную из контекста;
- `OperationExpression` — список с оператором и операндами.

Все узлы реализуют единый интерфейс: метод `interpret(context)`, который возвращает вычисленное значение.

```
Дерево для выражения (+ (* x 2) y):

        OperationExpression(+)
       /                      \
OperationExpression(*)     VariableExpression(y)
      /        \
VariableExpression(x)   NumberExpression(2)
```

---

## Парсер

### Шаг 1 — Токенизатор

Токенизатор принимает строку с LISP-выражением и превращает её в плоский массив токенов, а затем строит дерево вложенных массивов по скобкам.

```typescript
// Тип токена: число, строка-идентификатор или вложенный список
type Token = number | string | Token[];

/**
 * Токенизирует строку LISP-программы.
 * Разделяет по пробелам и скобкам, строит дерево вложенности.
 *
 * Пример: "(+ 1 2)" => ["+", 1, 2]
 *         "(+ (* 2 3) 1)" => ["+", ["*", 2, 3], 1]
 */
function tokenize(input: string): Token[] {
  // Добавляем пробелы вокруг скобок, чтобы split работал корректно
  const tokens = input
    .replace(/\(/g, ' ( ')
    .replace(/\)/g, ' ) ')
    .trim()
    .split(/\s+/)
    .filter(Boolean);

  const stack: Token[][] = [];
  let current: Token[] = [];

  for (const token of tokens) {
    if (token === '(') {
      // Открывающая скобка: сохраняем текущий уровень в стек,
      // начинаем новый вложенный список
      stack.push(current);
      current = [];
    } else if (token === ')') {
      // Закрывающая скобка: завершаем текущий список,
      // возвращаемся к родительскому уровню
      const completed = current;
      current = stack.pop()!;
      current.push(completed);
    } else {
      // Число или идентификатор
      const num = Number(token);
      current.push(isNaN(num) ? token : num);
    }
  }

  return current[0] as Token[];
}
```

### Шаг 2 — Узлы AST на TypeScript (классический подход с наследованием)

```typescript
// Контекст: словарь переменных
type Context = Record<string, number>;

/**
 * Абстрактный базовый класс для всех выражений.
 * Все наследники обязаны реализовать метод interpret.
 */
abstract class Expression {
  abstract interpret(context: Context): number | number[];
}

/**
 * Числовой литерал.
 * Пример: 42
 */
class NumberExpression extends Expression {
  constructor(private value: number) {
    super();
  }

  interpret(_context: Context): number {
    // Возвращаем хранимое значение, контекст не нужен
    return this.value;
  }
}

/**
 * Переменная из контекста.
 * Пример: x — берётся из context["x"]
 */
class VariableExpression extends Expression {
  constructor(private name: string) {
    super();
  }

  interpret(context: Context): number {
    if (!(this.name in context)) {
      throw new Error(`Переменная "${this.name}" не найдена в контексте`);
    }
    return context[this.name];
  }
}

/**
 * Выражение с оператором и операндами.
 * Пример: (+ 1 2) => оператор "+", операнды [1, 2]
 */
class OperationExpression extends Expression {
  constructor(
    private operator: string,
    private operands: Expression[]
  ) {
    super();
  }

  interpret(context: Context): number {
    // Рекурсивно вычисляем все операнды
    const values = this.operands.map((op) => op.interpret(context) as number);

    switch (this.operator) {
      case '+':
        return values.reduce((acc, v) => acc + v, 0);
      case '-':
        // Вычитаем все остальные из первого элемента
        return values.slice(1).reduce((acc, v) => acc - v, values[0]);
      case '*':
        return values.reduce((acc, v) => acc * v, 1);
      case '/':
        return values.slice(1).reduce((acc, v) => acc / v, values[0]);
      default:
        throw new Error(`Неизвестный оператор: "${this.operator}"`);
    }
  }
}
```

### Шаг 3 — Парсер: токены → AST

```typescript
/**
 * Преобразует токены (вложенный массив) в дерево Expression-объектов.
 */
function parse(tokens: Token[]): Expression {
  if (!Array.isArray(tokens)) {
    // Атомарный токен: число или переменная
    if (typeof tokens === 'number') {
      return new NumberExpression(tokens);
    }
    return new VariableExpression(tokens as string);
  }

  // Список: первый элемент — оператор, остальные — операнды
  const [operator, ...rest] = tokens as [string, ...Token[]];
  const operands = rest.map((token) => parse(token));
  return new OperationExpression(operator, operands);
}
```

---

## Вычислитель (Evaluator)

Вычислитель в классическом варианте — это просто вызов `interpret` на корневом узле AST. Контекст передаётся при вычислении и содержит значения переменных.

```typescript
/**
 * Evaluator: принимает корень AST и контекст,
 * возвращает результат вычисления.
 */
class Evaluator {
  evaluate(expression: Expression, context: Context): number | number[] {
    return expression.interpret(context);
  }
}

// --- Полный пример использования ---

const source = '(+ (* x 2) y)';

// 1. Токенизируем
const tokens = tokenize(source);
// => ["+", ["*", "x", 2], "y"]

// 2. Строим AST
const ast = parse(tokens);

// 3. Вычисляем с контекстом { x: 3, y: 5 }
const evaluator = new Evaluator();
const result = evaluator.evaluate(ast, { x: 3, y: 5 });

console.log(`Результат: ${result}`); // => Результат: 11
// (3 * 2) + 5 = 6 + 5 = 11
```

---

## Упрощённая реализация на JavaScript (без наследования)

В JavaScript нет необходимости в абстрактном классе. Вместо иерархии классов используется коллекция операторов — обычный объект-словарь. Это делает систему легко расширяемой: чтобы добавить новый оператор, достаточно добавить запись в словарь.

```javascript
// Коллекция операторов: имя => функция
const operators = {
  '+': (values) => values.reduce((acc, v) => acc + v, 0),
  '-': (values) => values.slice(1).reduce((acc, v) => acc - v, values[0]),
  '*': (values) => values.reduce((acc, v) => acc * v, 1),
  '/': (values) => values.slice(1).reduce((acc, v) => acc / v, values[0]),
};

// NumberExpression — просто объект с методом interpret
class NumberExpression {
  constructor(value) {
    this.value = value;
  }
  interpret(_context) {
    return this.value;
  }
}

// VariableExpression — читает переменную из контекста
class VariableExpression {
  constructor(name) {
    this.name = name;
  }
  interpret(context) {
    if (!(this.name in context)) {
      throw new Error(`Переменная "${this.name}" не найдена в контексте`);
    }
    return context[this.name];
  }
}

// OperationExpression — использует коллекцию operators
class OperationExpression {
  constructor(operator, operands) {
    this.operator = operator;
    this.operands = operands;
  }
  interpret(context) {
    // Рекурсивно интерпретируем каждый операнд
    const values = this.operands.map((op) => op.interpret(context));
    const fn = operators[this.operator];
    if (!fn) throw new Error(`Неизвестный оператор: "${this.operator}"`);
    return fn(values);
  }
}

// Токенизатор и парсер — те же, что в TypeScript-версии (адаптированы под JS)
function tokenize(input) {
  const tokens = input
    .replace(/\(/g, ' ( ')
    .replace(/\)/g, ' ) ')
    .trim()
    .split(/\s+/)
    .filter(Boolean);

  const stack = [];
  let current = [];

  for (const token of tokens) {
    if (token === '(') {
      stack.push(current);
      current = [];
    } else if (token === ')') {
      const completed = current;
      current = stack.pop();
      current.push(completed);
    } else {
      const num = Number(token);
      current.push(isNaN(num) ? token : num);
    }
  }

  return current[0];
}

function parse(tokens) {
  if (!Array.isArray(tokens)) {
    return typeof tokens === 'number'
      ? new NumberExpression(tokens)
      : new VariableExpression(tokens);
  }
  const [operator, ...rest] = tokens;
  return new OperationExpression(operator, rest.map(parse));
}

// Расширяем коллекцию операторов без изменения движка:
operators['max'] = (values) => Math.max(...values);
operators['min'] = (values) => Math.min(...values);
```

---

## Расширение LISP: операции над списками

LISP исторически строится вокруг операций над списками. Добавим базовые операции: `list`, `car` (голова списка), `cdr` (хвост списка), `eq` (сравнение).

```javascript
// Вспомогательные функции
const isList = (x) => Array.isArray(x) && x.length > 0;
const car  = (lst) => isList(lst) ? lst[0] : null;       // head
const cdr  = (lst) => isList(lst) ? lst.slice(1) : null; // tail

// Расширяем коллекцию операторов списковыми операциями
const lispOperators = {
  // Арифметика
  '+': (vals) => vals.reduce((a, v) => a + v, 0),
  '-': (vals) => vals.slice(1).reduce((a, v) => a - v, vals[0]),
  '*': (vals) => vals.reduce((a, v) => a * v, 1),
  '/': (vals) => vals.slice(1).reduce((a, v) => a / v, vals[0]),

  // Конструирование списка
  // (list 7 3 1) => [7, 3, 1]
  'list': (vals) => vals,

  // Проверка равенства
  // (eq 3 3) => true
  'eq': (vals) => vals[0] === vals[1],

  // Голова списка
  // (car (list 7 3 1)) => 7
  'car': (vals) => car(vals[0]),

  // Хвост списка
  // (cdr (list 7 3 1)) => [3, 1]
  'cdr': (vals) => cdr(vals[0]),
};

// OperationExpression с поддержкой lispOperators
class LispOperationExpression {
  constructor(operator, operands) {
    this.operator = operator;
    this.operands = operands;
  }
  interpret(context) {
    const values = this.operands.map((op) => op.interpret(context));
    const fn = lispOperators[this.operator];
    if (!fn) throw new Error(`Неизвестный оператор: "${this.operator}"`);
    return fn(values);
  }
}

function parseLisp(tokens) {
  if (!Array.isArray(tokens)) {
    return typeof tokens === 'number'
      ? new NumberExpression(tokens)
      : new VariableExpression(tokens);
  }
  const [operator, ...rest] = tokens;
  return new LispOperationExpression(operator, rest.map(parseLisp));
}

// Запуск LISP-программы
function run(source, context = {}) {
  const tokens = tokenize(source);
  const ast = parseLisp(tokens);
  return ast.interpret(context);
}
```

---

## Примеры

```javascript
// Арифметика
console.log(run('(+ 2 5)'));           // 7
console.log(run('(* 3 4)'));           // 12
console.log(run('(- 10 3 2)'));        // 5

// Переменные из контекста
console.log(run('(+ x y)', { x: 3, y: 5 }));  // 8

// Вложенные выражения
console.log(run('(+ (* 2 3) 1)'));     // 7
console.log(run('(* (+ 1 2) (- 5 2))')); // 9

// Переменные в сложных выражениях
console.log(run('(+ (* x 2) y)', { x: 3, y: 5 })); // 11

// Операции над списками
console.log(run('(list 7 3 1)'));
// => [7, 3, 1]

console.log(run('(car (list 7 3 1))'));
// => 7

console.log(run('(cdr (list 7 3 1))'));
// => [3, 1]

console.log(run('(eq 3 3)'));
// => true

console.log(run('(eq 3 5)'));
// => false

// Добавление нового оператора без изменения движка:
lispOperators['pow'] = (vals) => Math.pow(vals[0], vals[1]);
console.log(run('(pow 2 10)'));        // 1024
```

---

## Транслятор LISP → JavaScript (концепция)

Одно из практических применений AST — трансляция между языками. AST-дерево можно обойти и сгенерировать эквивалентный JavaScript-код, который затем выполняется на движке V8 с нативной скоростью.

```typescript
/**
 * Транслирует AST LISP-выражения в строку JavaScript-кода.
 */
function transpileToJS(tokens: Token[]): string {
  if (!Array.isArray(tokens)) {
    // Атом: число или имя переменной
    return String(tokens);
  }

  const [operator, ...operands] = tokens as [string, ...Token[]];
  const jsOperands = operands.map(transpileToJS);

  // Арифметические операторы транслируются напрямую
  if (['+', '-', '*', '/'].includes(operator)) {
    return `(${jsOperands.join(` ${operator} `)})`;
  }

  // Для специальных операторов — вызов функций
  if (operator === 'list') {
    return `[${jsOperands.join(', ')}]`;
  }
  if (operator === 'car') {
    return `(${jsOperands[0]})[0]`;
  }
  if (operator === 'cdr') {
    return `(${jsOperands[0]}).slice(1)`;
  }

  // Неизвестный оператор — вызов функции
  return `${operator}(${jsOperands.join(', ')})`;
}

// Пример трансляции
const lisp1 = tokenize('(+ (* x 2) y)');
console.log(transpileToJS(lisp1));
// => ((x * 2) + y)

const lisp2 = tokenize('(+ 1 (* 3 4))');
console.log(transpileToJS(lisp2));
// => (1 + (3 * 4))

// Выполняем транслированный код
const code = transpileToJS(tokenize('(+ (* x 2) y)'));
const fn = new Function('x', 'y', `return ${code};`);
console.log(fn(3, 5)); // => 11
```

---

## Полный пример: TypeScript-интерпретатор

Ниже приведён полный самодостаточный пример на TypeScript, объединяющий все части:

```typescript
// === types ===
type Token = number | string | Token[];
type Context = Record<string, number | number[]>;

// === AST Nodes ===
interface Expression {
  interpret(context: Context): number | number[] | boolean;
}

class NumberExpression implements Expression {
  constructor(private value: number) {}
  interpret(_ctx: Context) { return this.value; }
}

class VariableExpression implements Expression {
  constructor(private name: string) {}
  interpret(ctx: Context) {
    if (!(this.name in ctx)) throw new Error(`Переменная не найдена: ${this.name}`);
    return ctx[this.name] as number;
  }
}

// Коллекция операторов — легко расширяется
const ops: Record<string, (vals: any[]) => any> = {
  '+':    (v) => v.reduce((a: number, x: number) => a + x, 0),
  '-':    (v) => v.slice(1).reduce((a: number, x: number) => a - x, v[0]),
  '*':    (v) => v.reduce((a: number, x: number) => a * x, 1),
  '/':    (v) => v.slice(1).reduce((a: number, x: number) => a / x, v[0]),
  'list': (v) => v,
  'eq':   (v) => v[0] === v[1],
  'car':  (v) => Array.isArray(v[0]) ? v[0][0] : null,
  'cdr':  (v) => Array.isArray(v[0]) ? v[0].slice(1) : null,
};

class OperationExpression implements Expression {
  constructor(
    private operator: string,
    private operands: Expression[]
  ) {}

  interpret(ctx: Context) {
    const values = this.operands.map((op) => op.interpret(ctx));
    const fn = ops[this.operator];
    if (!fn) throw new Error(`Неизвестный оператор: ${this.operator}`);
    return fn(values);
  }
}

// === Tokenizer ===
function tokenize(input: string): Token[] {
  const raw = input.replace(/\(/g, ' ( ').replace(/\)/g, ' ) ')
    .trim().split(/\s+/).filter(Boolean);
  const stack: Token[][] = [];
  let cur: Token[] = [];
  for (const t of raw) {
    if (t === '(') { stack.push(cur); cur = []; }
    else if (t === ')') { const done = cur; cur = stack.pop()!; cur.push(done); }
    else { const n = Number(t); cur.push(isNaN(n) ? t : n); }
  }
  return cur[0] as Token[];
}

// === Parser ===
function parse(tokens: Token[]): Expression {
  if (!Array.isArray(tokens)) {
    return typeof tokens === 'number'
      ? new NumberExpression(tokens)
      : new VariableExpression(tokens as string);
  }
  const [op, ...rest] = tokens as [string, ...Token[]];
  return new OperationExpression(op, rest.map(parse));
}

// === Runner ===
function run(src: string, ctx: Context = {}): any {
  return parse(tokenize(src)).interpret(ctx);
}

// === Тесты ===
console.log(run('(+ 2 5)'));                        // 7
console.log(run('(+ (* x 2) y)', { x: 3, y: 5 })); // 11
console.log(run('(list 7 3 1)'));                   // [7, 3, 1]
console.log(run('(car (list 7 3 1))'));             // 7
console.log(run('(cdr (list 7 3 1))'));             // [3, 1]
console.log(run('(eq 3 3)'));                       // true

// Добавляем оператор без изменения движка
ops['pow'] = (v) => Math.pow(v[0], v[1]);
console.log(run('(pow 2 10)'));                     // 1024
```

---

## Итог

Паттерн **Interpreter** реализует следующие принципы:

1. **Разделение ответственности** — токенизатор, парсер и вычислитель независимы.
2. **Открытость/закрытость (OCP)** — движок не меняется при добавлении новых операторов: достаточно расширить коллекцию `ops`.
3. **Рекурсивная структура** — AST обходится рекурсивно; каждый узел умеет вычислять себя сам.
4. **Иммутабельность** — в функциональном стиле каждая операция возвращает новое значение, не мутируя входные данные.

### Когда применять

| Ситуация | Пример |
|---|---|
| DSL для конфигураций | Описание правил бизнес-логики |
| Распределённые вычисления | Передача выражений между сервисами |
| Кроссплатформенность | Портирование выражений на мобильный движок |
| Анализ кода | Обход AST для линтинга и трансформаций |
| Расширяемый синтаксис | Макросы, пользовательские операторы |

Ключевое преимущество подхода: **движок интерпретатора занимает менее 100 строк**, при этом сам язык можно расширять до любой сложности через коллекцию операторов — без модификации ядра.

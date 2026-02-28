# Паттерн «Интерпретатор» (Interpreter) — Часть 1: AST, DSL, LISP

> Лекция по курсу «Паттерны проектирования GoF».
> Тема: доменные языки (DSL), абстрактное синтаксическое дерево (AST),
> язык LISP и поведенческий паттерн Interpreter из «Банды четырёх».

---

## Введение

В этой лекции мы разберём понятие **доменного языка (DSL — Domain-Specific Language)** и посмотрим на несколько конкретных примеров таких языков:

- **LISP** — один из старейших и самых элегантных функциональных языков;
- **MetaSequoia** — внутренний DSL фреймворка для описания схем данных, API и экранных форм;
- DSL для описания **бизнес-процессов** на базе Markdown;
- DSL для **электронных таблиц** (формульный движок поверх JavaScript);
- DSL для построения **SQL-запросов** через шаблонные строки и query-builder.

Все эти языки объединяет одно: за ними стоит поведенческий паттерн **Interpreter** («Интерпретатор») из книги «Банды четырёх» (GoF). В следующей части лекции мы реализуем интерпретатор LISP глубоко и подробно.

---

## Что такое DSL

**DSL (Domain-Specific Language)** — это язык программирования или спецификации, заточенный под конкретную предметную область. В отличие от универсальных языков (JavaScript, Python, Java), DSL:

- выражает понятия предметной области напрямую;
- читается и пишется специалистами этой области, а не только разработчиками;
- снижает количество кода и риск ошибок для типовых задач домена.

DSL бывают двух видов:

| Вид | Описание | Пример |
|-----|----------|--------|
| **Внешний (External DSL)** | Отдельный синтаксис, требует парсера | SQL, Markdown, регулярные выражения |
| **Внутренний (Internal DSL)** | Надстройка над существующим языком | Query-builder в JavaScript, Fluent API |

В лекции рассматриваются преимущественно **внутренние DSL** на базе JavaScript/TypeScript.

---

## Что такое AST

**AST (Abstract Syntax Tree — абстрактное синтаксическое дерево)** — промежуточное представление программы в виде дерева узлов, которое компилятор или интерпретатор строит из исходного текста.

Любой язык программирования при вычислении выражения обязан:

1. Прочитать текст программы (строку).
2. Разобрать (распарсить) её в AST.
3. Либо скомпилировать AST в машинный код, либо интерпретировать обход дерева.

```
Исходный текст          Токены               AST
─────────────────       ──────────────────   ─────────────────
"(+ 2 (* 3 4))"  ──►   +, 2, *, 3, 4  ──►      (+)
                                              /     \
                                             2      (*)
                                                   /   \
                                                  3     4
```

Именно поэтому запись LISP **и есть** практически готовое AST: структура выражения явно видна в тексте через вложенные скобки.

### Связь JavaScript с AST

Когда движок V8 исполняет JavaScript, он строит AST из вашего кода. Это значит, что **любое JavaScript-выражение можно рассматривать как фрагмент AST**. Инструменты вроде Babel, ESLint и TypeScript работают именно с AST.

```typescript
// Посмотреть AST любого JS-кода можно на https://astexplorer.net/
// Выражение:
const x = 2 + 3 * 4;

// Соответствующий узел AST (упрощённо):
// {
//   type: "BinaryExpression",
//   operator: "+",
//   left:  { type: "NumericLiteral", value: 2 },
//   right: {
//     type: "BinaryExpression",
//     operator: "*",
//     left:  { type: "NumericLiteral", value: 3 },
//     right: { type: "NumericLiteral", value: 4 }
//   }
// }
```

---

## Что такое LISP

**LISP (LISt Processing)** — один из старейших языков программирования (1958, Джон Маккарти). Его главная особенность — **префиксная нотация** и единообразная структура: весь код и данные представлены как **списки**.

### Синтаксис LISP

```lisp
; Форма: (оператор аргумент1 аргумент2 ...)
(+ 2 3)           ; => 5  (сложение)
(* 3 4)           ; => 12 (умножение)
(+ 2 (* 3 4))     ; => 14 (вложенность = рекурсия)
(+ x (* y 4))     ; => вычисляет с переменными x и y
```

Каждая скобка определяет **список**. Первый элемент списка — это **оператор** (что делаем), остальные элементы — **операнды** (над чем делаем). Вложенность произвольная, вычисление рекурсивное.

Такая форма записи **идентична AST**:

```
(+ 2 (* 3 4))

       +
      / \
     2   *
        / \
       3   4
```

### Трансляция LISP ↔ JavaScript

LISP-выражения легко транслируются в JavaScript и обратно:

```typescript
// LISP:  (+ 2 (* 3 4))
// JS:    2 + 3 * 4
//
// LISP:  (+ x (* y 4))
// JS:    x + y * 4

// Если нам нужно только подмножество JS с математическими операторами,
// его можно разобрать в AST — и мы сразу получим LISP.
```

V8 предоставляет API `vm.Script` / `vm.createContext`, с помощью которого
можно исполнять JavaScript-выражения в изолированном контексте:

```typescript
import vm from 'node:vm';

// Безопасное вычисление выражения в изолированном контексте
function evalExpression(expression: string, context: Record<string, unknown>): unknown {
  const script = new vm.Script(`(${expression})`);
  const sandbox = vm.createContext({ ...context });
  return script.runInContext(sandbox);
}

// Использование
const result = evalExpression('x + y * 4', { x: 2, y: 3 });
console.log(result); // 14
```

---

## Паттерн «Интерпретатор» (Interpreter)

### Описание паттерна

**Interpreter** — поведенческий паттерн GoF. Он задаёт **грамматику** некоторого языка и предоставляет **интерпретатор** для работы с предложениями этого языка.

**Участники:**

| Роль | Описание |
|------|----------|
| `AbstractExpression` | Интерфейс с методом `interpret(context)` |
| `TerminalExpression` | Терминальный узел (переменная, литерал) |
| `NonTerminalExpression` | Составной узел (операция над подвыражениями) |
| `Context` | Содержит глобальную информацию для интерпретатора |
| `Client` | Строит AST и запускает интерпретацию |

**Когда применять:**

- Грамматика языка простая и стабильная.
- Эффективность не критична (паттерн не оптимален для сложных грамматик).
- Нужно расширять набор выражений без изменения существующих классов.

### UML-структура

```
«interface»
AbstractExpression
────────────────────
+ interpret(ctx): T
       ▲
       │
  ┌────┴──────────────────────┐
  │                           │
TerminalExpression     NonTerminalExpression
──────────────────     ─────────────────────
+ interpret(ctx): T    - left: AbstractExpression
                       - right: AbstractExpression
                       + interpret(ctx): T
```

---

## Реализация

### 1. Типы узлов AST

```typescript
// Типы узлов нашего мини-AST
type NumberLiteral = {
  type: 'number';
  value: number;
};

type Variable = {
  type: 'variable';
  name: string;
};

type BinaryOp = {
  type: 'binary';
  operator: '+' | '-' | '*' | '/';
  left: ASTNode;
  right: ASTNode;
};

type ASTNode = NumberLiteral | Variable | BinaryOp;
```

### 2. Контекст (переменные)

```typescript
// Контекст хранит значения переменных
type Context = Record<string, number>;
```

### 3. Интерпретатор

```typescript
/**
 * Рекурсивно интерпретирует AST-узел в числовое значение.
 * Реализует паттерн Interpreter: обход дерева с вычислением.
 */
function interpret(node: ASTNode, ctx: Context): number {
  switch (node.type) {
    // Терминальные выражения
    case 'number':
      return node.value;

    case 'variable': {
      const val = ctx[node.name];
      if (val === undefined) {
        throw new Error(`Неизвестная переменная: ${node.name}`);
      }
      return val;
    }

    // Нетерминальное выражение
    case 'binary': {
      const left  = interpret(node.left,  ctx);
      const right = interpret(node.right, ctx);
      switch (node.operator) {
        case '+': return left + right;
        case '-': return left - right;
        case '*': return left * right;
        case '/':
          if (right === 0) throw new Error('Деление на ноль');
          return left / right;
      }
    }
  }
}
```

### 4. Простой парсер LISP-выражений

```typescript
/**
 * Токенизатор: разбивает строку LISP-выражения на токены.
 * Пример: "(+ 2 (* x 4))" => ["(", "+", "2", "(", "*", "x", "4", ")", ")"]
 */
function tokenize(input: string): string[] {
  return input
    .replace(/\(/g, ' ( ')
    .replace(/\)/g, ' ) ')
    .trim()
    .split(/\s+/)
    .filter(Boolean);
}

/**
 * Рекурсивный парсер: строит AST из списка токенов.
 */
function parse(tokens: string[]): ASTNode {
  const token = tokens.shift()!;

  // Начало списка — нетерминальное выражение
  if (token === '(') {
    const operator = tokens.shift() as BinaryOp['operator'];
    const left  = parse(tokens);
    const right = parse(tokens);
    tokens.shift(); // убираем закрывающую ')'
    return { type: 'binary', operator, left, right };
  }

  // Число
  const num = Number(token);
  if (!isNaN(num)) {
    return { type: 'number', value: num };
  }

  // Переменная
  return { type: 'variable', name: token };
}

/**
 * Объединяет токенизацию и парсинг.
 */
function parseLisp(source: string): ASTNode {
  const tokens = tokenize(source);
  return parse(tokens);
}
```

### 5. Полный пример использования

```typescript
// Вычисляем: (+ 2 (* x 4))  при x = 3
// Ожидаем:   2 + 3 * 4 = 14

const source  = '(+ 2 (* x 4))';
const context: Context = { x: 3 };

const ast    = parseLisp(source);
const result = interpret(ast, context);

console.log(`Выражение: ${source}`);
console.log(`Контекст:  x = ${context.x}`);
console.log(`Результат: ${result}`);
// Выражение: (+ 2 (* x 4))
// Контекст:  x = 3
// Результат: 14
```

### 6. Расширяемый LISP-интерпретатор на основе таблицы операторов

Лектор подчёркивает: операторы LISP удобно добавлять как **записи в объект** (структуру данных). Ключ — имя оператора, значение — функция-реализация. Это делает интерпретатор легко расширяемым без изменения его ядра.

```typescript
// Таблица операторов: имя -> реализация
type OperatorFn = (args: number[]) => number;

const operators: Record<string, OperatorFn> = {
  '+': (args) => args.reduce((acc, v) => acc + v, 0),
  '-': (args) => args.reduce((acc, v, i) => i === 0 ? v : acc - v),
  '*': (args) => args.reduce((acc, v) => acc * v, 1),
  '/': (args) => args.reduce((acc, v, i) => i === 0 ? v : acc / v),
  'max': (args) => Math.max(...args),
  'min': (args) => Math.min(...args),
  'abs': ([a]) => Math.abs(a),
};

// Узел расширенного AST поддерживает произвольное число аргументов
type LispNode =
  | { type: 'num'; value: number }
  | { type: 'var'; name: string }
  | { type: 'call'; op: string; args: LispNode[] };

function evalLisp(node: LispNode, ctx: Context): number {
  if (node.type === 'num') return node.value;
  if (node.type === 'var') {
    const val = ctx[node.name];
    if (val === undefined) throw new Error(`Неизвестная переменная: ${node.name}`);
    return val;
  }
  // node.type === 'call'
  const fn = operators[node.op];
  if (!fn) throw new Error(`Неизвестный оператор: ${node.op}`);
  const evaluatedArgs = node.args.map((arg) => evalLisp(arg, ctx));
  return fn(evaluatedArgs);
}

// Пример: (max (* x 2) (+ y 1))  при x=3, y=5
const tree: LispNode = {
  type: 'call',
  op: 'max',
  args: [
    { type: 'call', op: '*', args: [{ type: 'var', name: 'x' }, { type: 'num', value: 2 }] },
    { type: 'call', op: '+', args: [{ type: 'var', name: 'y' }, { type: 'num', value: 1 }] },
  ],
};

console.log(evalLisp(tree, { x: 3, y: 5 })); // max(6, 6) = 6

// Добавить новый оператор — одна строка:
operators['sum-of-squares'] = (args) => args.reduce((acc, v) => acc + v * v, 0);
```

---

## Применение

В лекции показано, что паттерн Interpreter и идея DSL используются повсеместно.

### Query Builder (SQL DSL)

Вместо написания строк SQL вручную используется **цепочка методов** (Fluent API / Builder) — это тоже форма DSL поверх JavaScript:

```typescript
// Внутренний DSL для построения SQL (пример MetaSQL / Knex-подобного API)
const query = db
  .insert({ name: 'Alice', age: 30 })
  .into('users')
  .returning(['id', 'name']);

// Внутри библиотеки это выражение транслируется в SQL:
// INSERT INTO users (name, age) VALUES ('Alice', 30) RETURNING id, name;
```

### SQL через шаблонные строки (Tagged Template Literals)

```typescript
// Tagged template literal — ещё один вид внутреннего DSL
// Функция-тег получает массив строк и массив вставленных значений
function sql(strings: TemplateStringsArray, ...values: unknown[]): { query: string; params: unknown[] } {
  const query  = strings.raw.join('$' + (strings.length)); // упрощённо
  return { query, params: values };
}

const userId = 42;
const minAge = 18;

// Значения НЕ интерполируются напрямую — они передаются как параметры,
// что защищает от SQL-инъекций
const result = sql`SELECT * FROM users WHERE id = ${userId} AND age > ${minAge}`;
// result.query  = "SELECT * FROM users WHERE id = $1 AND age > $2"
// result.params = [42, 18]
```

### Формульный движок (DSL для электронных таблиц)

```typescript
// Мини-движок электронных таблиц: каждая ячейка — выражение на JavaScript
// Ячейки ссылаются друг на друга по имени (A1, B1, C1 или произвольно)

type CellDef = {
  [cellName: string]: string; // имя ячейки -> выражение
};

class Spreadsheet {
  private cache = new Map<string, number>();

  constructor(private cells: CellDef) {}

  // Ленивое вычисление с кэшированием
  get(name: string): number {
    if (this.cache.has(name)) return this.cache.get(name)!;

    const expr = this.cells[name];
    if (expr === undefined) throw new Error(`Ячейка не найдена: ${name}`);

    // Строим контекст: все остальные ячейки доступны по имени
    const ctx = new Proxy({} as Record<string, number>, {
      get: (_, key: string) => this.get(key),
    });

    // Вычисляем выражение через Function (в реальности — через vm.Script)
    const fn = new Function(...Object.keys(ctx), `return (${expr})`);
    const value = fn(...Object.keys(ctx).map((k) => this.get(k)));
    this.cache.set(name, value);
    return value;
  }
}

// Использование
const sheet = new Spreadsheet({
  A1: '10',
  B1: '3',
  C1: 'A1 * B1',           // 10 * 3 = 30
  D1: 'Math.log(C1)',       // Math.log(30) ≈ 3.401
});

console.log(sheet.get('C1')); // 30
console.log(sheet.get('D1')); // ~3.401
```

### DSL для описания бизнес-процессов

В лекции показан пример описания бизнес-процессов на Markdown-подобном синтаксисе:

```markdown
# Процесс: Бронирование (Reservation)

* validateOrder(order) -> balance
* checkBalance(order, balance) -> instances
* createReservation(order, balance, instances) -> reservation
```

Каждый шаг — вызов функции. Результат предыдущего шага передаётся в следующий.
Диаграмма рендерится любым Markdown-совместимым инструментом (GitHub, IDE).
Внутри каждый блок может быть реализован на JavaScript, WebAssembly или
подключён как плагин.

---

## Итог

**Паттерн Interpreter** решает задачу интерпретации языка через дерево объектов, каждый из которых умеет «вычислить» себя. Ключевые выводы лекции:

1. **AST — основа всего.** Любой язык строит AST перед исполнением. Понимание AST открывает дорогу к написанию парсеров, трансформеров и интерпретаторов.

2. **LISP — это «живое» AST.** Синтаксис LISP с префиксными списками буквально отражает структуру дерева, что делает его идеальной учебной моделью паттерна Interpreter.

3. **DSL повышают выразительность.** Вместо императивного кода предметная область описывается декларативно. Пользователи (не только разработчики) могут писать формулы, бизнес-правила, схемы.

4. **Расширяемость через таблицу операторов.** Добавить новый оператор в интерпретатор = добавить одну запись в объект. Это классический пример открытости для расширения без модификации существующего кода (Open/Closed Principle).

5. **JavaScript — мощная база для DSL.** Tagged template literals, Proxy, vm.Script, Fluent API — все эти механизмы позволяют строить богатые внутренние DSL на чистом JavaScript/TypeScript.

В следующей части лекции будет реализован **полноценный интерпретатор LISP**: лексер, парсер, evaluator с несколькими стратегиями и поддержкой переменных.

# Паттерн «Прототип» (Prototype) — GoF Patterns

## Обзор

Паттерн **Прототип** (Prototype) — это порождающий паттерн проектирования из каталога Gang of Four, который позволяет копировать объекты, включая те, что имеют приватные поля или скрытое состояние в замыканиях. В контексте JavaScript более точным названием для этого паттерна было бы **Clone** (клонирование), поскольку его основная задача — создание копий объектов.

> **Важно**: Не путайте паттерн Prototype с прототипным наследованием JavaScript! Это совершенно разные концепции. Паттерн GoF Prototype занимается клонированием экземпляров объектов, а не наследованием.

---

## Проблема

В JavaScript существует несколько уровней приватности данных:
- **Приватные поля через `#`** (современный стандарт)
- **Замыкания** (функциональный подход к инкапсуляции)
- **Условно приватные поля** (через подчеркивание `_field` или просто соглашение)
- **Внешнее хранилище состояния** (Map, WeakMap, Symbol)

При клонировании объектов снаружи невозможно получить доступ к:
- Приватным полям (`#field`)
- Переменным в замыканиях
- Состоянию, хранящемуся во внешних структурах данных

---

## Решение

**Встроить процедуру клонирования непосредственно в саму абстракцию**, поскольку только изнутри класса или функции есть доступ ко всему внутреннему состоянию.

### Основные принципы:

1. **Метод `clone()`** добавляется в класс или возвращаемый объект
2. Метод имеет доступ ко всем приватным полям и замыканиям
3. Все классы, поддерживающие клонирование, реализуют единый интерфейс

### В TypeScript можно определить интерфейс:

```typescript
interface Cloneable {
  clone(): this;
}
```

---

## Преимущества паттерна Prototype

1. **Доступ к приватным полям**: Метод `clone()` имеет доступ к полям с `#` и переменным в замыканиях
2. **Не плодится количество классов**: Не нужно создавать отдельные классы для клонирования
3. **Все экземпляры поддерживают клонирование**: Клоны также имеют метод `clone()` и могут быть клонированы снова
4. **Рекурсивное клонирование**: Можно создавать глубокие копии сложных структур

---

## Недостатки паттерна Prototype

1. **Усложнение класса**: Добавление логики клонирования внутрь класса увеличивает его сложность
2. **Скрытая сложность**: Логика клонирования изолирована внутри класса, что может затруднить понимание
3. **Необходимость реализации для каждого класса**: Каждая клонируемая абстракция должна реализовывать метод `clone()`

---

## Реализация на классах с приватными полями

### Пример 1: Точка на плоскости

```javascript
class Point {
  #x;
  #y;

  constructor(x, y) {
    this.#x = x;
    this.#y = y;
  }

  move(dx, dy) {
    this.#x += dx;
    this.#y += dy;
  }

  toString() {
    return `Point(${this.#x}, ${this.#y})`;
  }

  // Метод клонирования с доступом к приватным полям
  clone() {
    return new Point(this.#x, this.#y);
  }
}

// Использование
const p1 = new Point(10, 20);
console.log(p1.toString()); // Point(10, 20)

const p2 = p1.clone();
p2.move(5, -10);

console.log(p1.toString()); // Point(10, 20) - оригинал не изменился
console.log(p2.toString()); // Point(15, 10) - клон изменён
```

**Ключевые моменты**:
- Приватные поля `#x` и `#y` недоступны снаружи класса
- Метод `clone()` имеет доступ к приватным полям через `this.#x` и `this.#y`
- Клон создаётся вызовом конструктора с текущими значениями
- Оригинал и клон — полностью независимые объекты

---

## Реализация с использованием замыканий

### Пример 2: Точка через замыкания

```javascript
function Point(x, y) {
  // x и y хранятся в замыкании - "приватные" переменные

  const move = (dx, dy) => {
    x += dx;
    y += dy;
  };

  const toString = () => {
    return `Point(${x}, ${y})`;
  };

  const clone = () => {
    // Доступ к замкнутым переменным x и y
    return Point(x, y);
  };

  // Возвращаем объект с методами
  return {
    move,
    toString,
    clone
  };
}

// Использование
const p1 = Point(10, 20);
console.log(p1.toString()); // Point(10, 20)

const c1 = p1.clone();
c1.move(-5, 10);

console.log(p1.toString()); // Point(10, 20) - оригинал
console.log(c1.toString()); // Point(5, 30) - клон
```

**Преимущества замыканий**:
- Переменные `x` и `y` действительно приватны (нет способа получить к ним доступ снаружи)
- Каждый вызов `Point()` создаёт новое замыкание
- Метод `clone()` создаёт новое замыкание с теми же значениями
- Это не функциональное программирование (мутабельный state), но удобный паттерн

---

## Рекурсивное клонирование: композиция объектов

### Пример 3: Линия из двух точек

```javascript
class Point {
  #x;
  #y;

  constructor(x, y) {
    this.#x = x;
    this.#y = y;
  }

  move(dx, dy) {
    this.#x += dx;
    this.#y += dy;
  }

  toString() {
    return `Point(${this.#x}, ${this.#y})`;
  }

  clone() {
    return new Point(this.#x, this.#y);
  }
}

class Line {
  #start;
  #end;

  constructor(start, end) {
    this.#start = start;
    this.#end = end;
  }

  toString() {
    return `Line[${this.#start.toString()} -> ${this.#end.toString()}]`;
  }

  // Рекурсивное клонирование: клонируем вложенные точки
  clone() {
    return new Line(
      this.#start.clone(),  // Клонируем начальную точку
      this.#end.clone()     // Клонируем конечную точку
    );
  }
}

// Использование
const p1 = new Point(0, 0);
const p2 = new Point(10, 10);
const line1 = new Line(p1, p2);

console.log(line1.toString());
// Line[Point(0, 0) -> Point(10, 10)]

const line2 = line1.clone();
console.log(line2.toString());
// Line[Point(0, 0) -> Point(10, 10)]

// line1 и line2 — независимые объекты
// p1 и точки внутри line2 — тоже разные объекты
```

**Важно**: Благодаря тому, что метод называется одинаково (`clone()`) везде, мы можем создавать глубокие копии сложных иерархий:
- Ломаные линии из массива точек
- Многоугольники из линий
- Сложные фигуры, состоящие из множества элементов

### Пример 4: Коллекция линий

```javascript
class Polyline {
  #lines;

  constructor(lines = []) {
    this.#lines = lines;
  }

  addLine(line) {
    this.#lines.push(line);
  }

  toString() {
    return `Polyline[\n${this.#lines.map(l => '  ' + l.toString()).join('\n')}\n]`;
  }

  // Глубокое клонирование всей коллекции
  clone() {
    const clonedLines = this.#lines.map(line => line.clone());
    return new Polyline(clonedLines);
  }
}

// Использование
const poly1 = new Polyline();
poly1.addLine(new Line(new Point(0, 0), new Point(5, 5)));
poly1.addLine(new Line(new Point(5, 5), new Point(10, 0)));

const poly2 = poly1.clone();
// poly2 содержит полные копии всех линий и точек
```

---

## Альтернативные способы клонирования в JavaScript

### 1. JSON сериализация (ограниченная)

```javascript
const obj = { x: 10, y: 20, nested: { z: 30 } };

// Клонирование через JSON
const clone = JSON.parse(JSON.stringify(obj));

console.log(clone); // { x: 10, y: 20, nested: { z: 30 } }
console.log(obj === clone); // false
console.log(obj.nested === clone.nested); // false - глубокое копирование
```

**Ограничения JSON сериализации**:
- ❌ Не копирует методы
- ❌ Не копирует приватные поля (`#field`)
- ❌ Не копирует Symbol-ключи
- ❌ Теряет прототип (методы из класса)
- ❌ Не работает с `undefined`, `Function`, `Symbol`, `Date`, `RegExp`
- ❌ Циклические ссылки вызывают ошибку

**Решение проблемы с прототипом**:

```javascript
const obj = { x: 10, y: 20 };
const clone = JSON.parse(JSON.stringify(obj));

// Восстановление прототипа
Object.setPrototypeOf(clone, Object.getPrototypeOf(obj));
```

---

### 2. V8 сериализация (Node.js)

```javascript
const v8 = require('v8');

const obj = {
  x: 10,
  date: new Date(),
  buffer: Buffer.from('hello'),
  nested: { y: 20 }
};

// Сериализация и десериализация
const clone = v8.deserialize(v8.serialize(obj));

console.log(clone);
// Поддерживает больше типов, чем JSON
```

**Преимущества**:
- ✅ Бинарный формат (более эффективен)
- ✅ Поддерживает больше типов (Date, Buffer, TypedArray и т.д.)
- ✅ Глубокое копирование

**Ограничения**:
- ❌ Не копирует приватные поля (`#field`)
- ❌ Не копирует методы
- ❌ Требует восстановления прототипа

---

### 3. Структурное клонирование (Web API / Node.js)

```javascript
// В браузере и современном Node.js
const obj = {
  x: 10,
  date: new Date(),
  map: new Map([['key', 'value']]),
  set: new Set([1, 2, 3]),
  nested: { y: 20 }
};

const clone = structuredClone(obj);

console.log(clone);
console.log(obj === clone); // false
console.log(obj.nested === clone.nested); // false
console.log(obj.date === clone.date); // false
```

**Преимущества**:
- ✅ Современный стандарт (Web API)
- ✅ Поддерживает множество типов: `Date`, `Map`, `Set`, `ArrayBuffer`, `TypedArray`, и др.
- ✅ Обрабатывает циклические ссылки
- ✅ Глубокое копирование

**Ограничения**:
- ❌ Не копирует приватные поля (`#field`)
- ❌ Не копирует функции и методы
- ❌ Не копирует Symbol-ключи

---

## Когда встроенные методы не работают

Все описанные альтернативы (JSON, V8, structuredClone) **не могут скопировать приватные поля**. Только паттерн Prototype с методом `clone()` внутри класса решает эту проблему.

### Сравнительная таблица

| Метод | Приватные поля | Методы | Глубокое копирование | Специальные типы |
|-------|---------------|---------|---------------------|------------------|
| **Паттерн Prototype** | ✅ | ✅ | ✅ (если реализовано) | ✅ |
| JSON serialize | ❌ | ❌ | ✅ | ❌ |
| V8 serialize | ❌ | ❌ | ✅ | ✅ (частично) |
| structuredClone | ❌ | ❌ | ✅ | ✅ (многие типы) |
| Spread operator | ❌ | ✅ (по ссылке) | ❌ (поверхностное) | Частично |

---

## Расширенный пример: TypeScript с интерфейсом

```typescript
// Интерфейс для клонируемых объектов
interface Cloneable<T> {
  clone(): T;
}

// Точка с приватными полями
class Point implements Cloneable<Point> {
  #x: number;
  #y: number;

  constructor(x: number, y: number) {
    this.#x = x;
    this.#y = y;
  }

  move(dx: number, dy: number): void {
    this.#x += dx;
    this.#y += dy;
  }

  toString(): string {
    return `Point(${this.#x}, ${this.#y})`;
  }

  clone(): Point {
    return new Point(this.#x, this.#y);
  }

  getCoordinates(): { x: number; y: number } {
    return { x: this.#x, y: this.#y };
  }
}

// Круг с приватными полями
class Circle implements Cloneable<Circle> {
  #center: Point;
  #radius: number;

  constructor(center: Point, radius: number) {
    this.#center = center;
    this.#radius = radius;
  }

  toString(): string {
    return `Circle(center: ${this.#center.toString()}, radius: ${this.#radius})`;
  }

  // Рекурсивное клонирование
  clone(): Circle {
    return new Circle(
      this.#center.clone(), // Клонируем центр
      this.#radius
    );
  }
}

// Использование
const circle1 = new Circle(new Point(10, 20), 5);
const circle2 = circle1.clone();

console.log(circle1.toString());
// Circle(center: Point(10, 20), radius: 5)
console.log(circle2.toString());
// Circle(center: Point(10, 20), radius: 5)

// Изменение клона не влияет на оригинал
```

---

## Паттерн Prototype с обработкой циклических ссылок

В сложных структурах данных могут возникать циклические ссылки. Вот пример, как с ними работать:

```typescript
class Node implements Cloneable<Node> {
  value: string;
  #next: Node | null = null;
  #visited: WeakMap<Node, Node> = new WeakMap();

  constructor(value: string) {
    this.value = value;
  }

  setNext(node: Node): void {
    this.#next = node;
  }

  clone(cache: Map<Node, Node> = new Map()): Node {
    // Проверяем, не клонировали ли уже этот узел
    if (cache.has(this)) {
      return cache.get(this)!;
    }

    // Создаём клон
    const cloned = new Node(this.value);

    // Сохраняем в кэш до рекурсивного клонирования
    cache.set(this, cloned);

    // Рекурсивно клонируем связанный узел
    if (this.#next) {
      cloned.#next = this.#next.clone(cache);
    }

    return cloned;
  }
}

// Создаём циклическую структуру
const node1 = new Node('A');
const node2 = new Node('B');
const node3 = new Node('C');

node1.setNext(node2);
node2.setNext(node3);
node3.setNext(node1); // Цикл!

// Клонируем с обработкой циклов
const clonedNode1 = node1.clone();
```

---

## Практические применения паттерна Prototype

### 1. Прототипирование UI компонентов

```javascript
class Widget {
  #config;
  #state;

  constructor(config) {
    this.#config = { ...config };
    this.#state = {};
  }

  clone() {
    const cloned = new Widget(this.#config);
    cloned.#state = { ...this.#state };
    return cloned;
  }
}

// Создаём шаблон виджета
const template = new Widget({ color: 'blue', size: 'large' });

// Клонируем для создания множества похожих виджетов
const widget1 = template.clone();
const widget2 = template.clone();
const widget3 = template.clone();
```

### 2. Снимки состояния (Memento pattern)

```javascript
class GameState {
  #level;
  #score;
  #inventory;

  constructor(level, score, inventory) {
    this.#level = level;
    this.#score = score;
    this.#inventory = [...inventory];
  }

  // Сохранение состояния через клонирование
  createSnapshot() {
    return this.clone();
  }

  clone() {
    return new GameState(this.#level, this.#score, this.#inventory);
  }
}

const gameState = new GameState(5, 1000, ['sword', 'shield']);
const snapshot = gameState.createSnapshot();

// Можно восстановить состояние позже
```

### 3. Кэширование сложных объектов

```javascript
class ComplexObject {
  #data;

  constructor(data) {
    this.#data = this.expensiveProcessing(data);
  }

  expensiveProcessing(data) {
    // Сложная обработка данных
    return data;
  }

  clone() {
    const cloned = Object.create(Object.getPrototypeOf(this));
    cloned.#data = { ...this.#data };
    return cloned;
  }
}

// Создаём один раз
const template = new ComplexObject(rawData);

// Быстро клонируем вместо пересоздания
const instance1 = template.clone();
const instance2 = template.clone();
```

---

## Ключевые выводы

1. **Паттерн Prototype ≠ Прототипное наследование JavaScript** — это разные концепции
2. **Основная задача** — клонирование объектов с приватными полями и скрытым состоянием
3. **Метод `clone()`** должен быть встроен в класс для доступа к приватным полям
4. **Рекурсивное клонирование** позволяет копировать сложные структуры объектов
5. **Альтернативы** (JSON, V8, structuredClone) не работают с приватными полями
6. **Единый интерфейс** `Cloneable` обеспечивает согласованность API
7. **Замыкания** можно использовать вместо классов для инкапсуляции

---

## Дополнительные материалы

### Связанные паттерны

- **Factory Method** — может использовать Prototype для создания объектов
- **Memento** — часто использует Prototype для сохранения состояния
- **Object Pool** — может клонировать прототип вместо создания с нуля

### Когда использовать Prototype

✅ **Используйте, когда**:
- Нужно копировать объекты с приватными полями
- Создание объекта дорого (можно клонировать шаблон)
- Нужна независимость от конкретных классов при копировании
- Требуется глубокое копирование сложных структур

❌ **Избегайте, когда**:
- Объекты простые и нет приватных полей (используйте spread или Object.assign)
- Не требуется независимость клонов (можно использовать ссылки)
- Структура объектов плоская (достаточно поверхностного копирования)

---

## Заключение

Паттерн Prototype — мощный инструмент для клонирования объектов в JavaScript/TypeScript, особенно когда речь идёт о приватных полях, замыканиях или сложных структурах данных. Несмотря на название, которое может вызвать путаницу с прототипным наследованием, паттерн решает важную задачу копирования состояния объектов, к которому невозможно получить доступ снаружи.

Правильная реализация паттерна Prototype обеспечивает:
- Инкапсуляцию логики клонирования
- Независимость клонов от оригиналов
- Возможность глубокого копирования
- Единообразный интерфейс для всех клонируемых объектов

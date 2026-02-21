# Ссылочная прозрачность (Referential Transparency)
## ФП, ООП в JavaScript и TypeScript

---

## Оглавление
1. [Введение](#введение)
2. [Базовые концепции](#базовые-концепции)
3. [Применение в разных парадигмах](#применение-в-разных-парадигмах)
4. [Оптимизация компилятора](#оптимизация-компилятора)
5. [Иммутабельность](#иммутабельность)
6. [Copy-on-Write стратегия](#copy-on-write-стратегия)
7. [Ownership концепция](#ownership-концепция)
8. [Практические примеры](#практические-примеры)
9. [Выводы](#выводы)

---

## Введение

**Ссылочная прозрачность** — это свойство программного кода, которое позволяет безопасно заменять выражения и функции на их вычисленные значения без изменения поведения программы.

### Ключевая идея
Традиционно ссылочная прозрачность ассоциируется с **функциональным программированием**, но на самом деле она применима во **всех парадигмах**: ФП, ООП и процедурном программировании.

### Основное определение
> Если у нас есть иммутабельные объекты, структуры данных, функции или замыкания, в которых не изменяется состояние, то мы можем передать ссылки на эти абстракции в разные части программы и не бояться коррапции данных.

---

## Базовые концепции

### 1. Замена выражений на значения

В функциональном программировании ссылочная прозрачность означает возможность заменить:
- Выражение на его вычисленное значение
- Функцию на результат её вызова
- Мемоизированное значение из кэша

```javascript
// Ссылочно прозрачное выражение
const x = 5 + 3;  // можно заменить на 8
const y = x * 2;  // можно заменить на 16

// Ссылочно прозрачная функция
const add = (a, b) => a + b;
const result = add(5, 3);  // можно заменить на 8
```

### 2. Контейнерные типы

Если ссылки на контейнерные типы (объекты, массивы) распространились по приложению, они всегда могут быть заменены на:
- Конкретное вычисленное значение
- Ссылку на другой иммутабельный объект

```javascript
// Иммутабельный объект - ссылочно прозрачный
const point = Object.freeze({ x: 10, y: 20 });

// Можно безопасно использовать в разных местах
function drawPoint(p) { /* ... */ }
function savePoint(p) { /* ... */ }
function logPoint(p) { /* ... */ }

// Все функции получат одинаковые данные
drawPoint(point);
savePoint(point);
logPoint(point);
```

---

## Применение в разных парадигмах

### Функциональное программирование

```javascript
// Чистая функция - ссылочно прозрачная
const multiply = (a, b) => a * b;

// Можно заменить вызов на результат
const result1 = multiply(3, 4);  // 12
const result2 = 12;  // эквивалентно
```

### Объектно-ориентированное программирование

```javascript
class Point {
  #x;
  #y;

  constructor(x, y) {
    this.#x = x;
    this.#y = y;
  }

  // Читающий метод - безопасен для ссылочной прозрачности
  getX() {
    return this.#x;
  }

  // Метод возвращает НОВЫЙ объект - ссылочно прозрачный
  shift(dx, dy) {
    return new Point(this.#x + dx, this.#y + dy);
  }

  // Метод НЕ ссылочно прозрачный - изменяет состояние
  // setX(x) {
  //   this.#x = x;
  // }
}

// Использование
const p1 = new Point(10, 20);
const p2 = p1.shift(5, 5);  // Создан новый объект

// p1 остался неизменным - безопасно использовать везде
console.log(p1.getX());  // 10
console.log(p2.getX());  // 15
```

### Процедурное программирование

```javascript
// Иммутабельные структуры данных
const polyline = Object.freeze([
  Object.freeze({ x: 0, y: 0 }),
  Object.freeze({ x: 10, y: 10 }),
  Object.freeze({ x: 20, y: 5 })
]);

// Процедура с иммутабельными данными
function shiftPolyline(line, dx, dy) {
  return line.map(point => ({
    x: point.x + dx,
    y: point.y + dy
  }));
}

// Оригинал не изменился
const shifted = shiftPolyline(polyline, 5, 5);
console.log(polyline[0].x);  // 0 (не изменилось)
console.log(shifted[0].x);   // 5 (новая линия)
```

---

## Оптимизация компилятора

### Автоматическая оптимизация в JavaScript

Оптимизирующий компилятор JavaScript может:
- Разбивать выражения на подвыражения
- Кэшировать промежуточные результаты
- Переиспользовать вычисленные значения

```javascript
// Компилятор может оптимизировать
function calculate(data) {
  const result1 = data.config.value * 2;
  // ... другой код без изменения data.config.value
  const result2 = data.config.value * 3;

  // Компилятор видит, что data.config.value не изменился
  // и может закэшировать его значение
}
```

### Пример с циклами

```javascript
// Неоптимально - обращение к свойству на каждой итерации
for (let i = 0; i < items.length; i++) {
  const value = config.settings.maxValue;  // Читается каждый раз
  if (items[i] > value) {
    // ...
  }
}

// Оптимизатор может вынести за цикл
const maxValue = config.settings.maxValue;  // Один раз
for (let i = 0; i < items.length; i++) {
  if (items[i] > maxValue) {
    // ...
  }
}
```

---

## Иммутабельность

### Способы создания иммутабельных объектов

#### 1. Object.freeze()

```javascript
const user = Object.freeze({
  name: 'John',
  age: 30
});

// Попытка изменить - не сработает
user.age = 31;  // В strict mode - ошибка
console.log(user.age);  // 30
```

#### 2. Приватные поля (#)

```javascript
class User {
  #name;
  #age;

  constructor(name, age) {
    this.#name = name;
    this.#age = age;
  }

  getName() {
    return this.#name;
  }

  // Возвращает новый объект
  withAge(newAge) {
    return new User(this.#name, newAge);
  }
}

const user1 = new User('John', 30);
const user2 = user1.withAge(31);

console.log(user1.getName());  // John
// user1.#name - ошибка: приватное поле
```

#### 3. Замыкания

```javascript
function createPoint(x, y) {
  // x и y защищены замыканием
  return {
    getX: () => x,
    getY: () => y,
    shift: (dx, dy) => createPoint(x + dx, y + dy),
    toString: () => `Point(${x}, ${y})`
  };
}

const point = createPoint(10, 20);
console.log(point.getX());  // 10

const shifted = point.shift(5, 5);
console.log(point.getX());    // 10 (не изменилось)
console.log(shifted.getX());  // 15
```

---

## Copy-on-Write стратегия

### Проблема

При работе с иммутабельными данными каждое изменение создает новую копию всей структуры, что неэффективно по памяти.

```javascript
// Неэффективно - полная копия
class Polyline {
  constructor(points) {
    this.points = [...points];  // Копируем все точки
  }

  shiftPoint(index, dx, dy) {
    const newPoints = [...this.points];  // Копируем ВСЕ точки
    newPoints[index] = {
      x: this.points[index].x + dx,
      y: this.points[index].y + dy
    };
    return new Polyline(newPoints);
  }
}

// Если 100 точек, а меняем одну - копируем все 100!
```

### Решение: Copy-on-Write

Копируем только измененные части, остальное переиспользуем.

```javascript
class Polyline {
  constructor(points) {
    this.points = points;  // Используем исходный массив
  }

  shiftPoint(index, dx, dy) {
    // Копируем только массив ссылок, не сами точки
    const newPoints = [...this.points];

    // Заменяем только одну точку
    newPoints[index] = {
      x: this.points[index].x + dx,
      y: this.points[index].y + dy
    };

    return new Polyline(newPoints);
  }
}

// Если 100 точек, копируем массив ссылок (дешево)
// и создаем только одну новую точку
```

### Copy-on-Write через прототипы

```javascript
function createCOWContainer(initialData) {
  let data = initialData;

  return {
    get(key) {
      return data[key];
    },

    set(key, value) {
      // Создаем новый объект с предыдущим как прототипом
      const newData = Object.create(data);
      newData[key] = value;
      data = newData;
    },

    getData() {
      return data;
    }
  };
}

// Использование
const container = createCOWContainer({ name: 'John', age: 30 });

console.log(container.get('name'));  // John

container.set('age', 31);  // Создан новый объект в цепочке прототипов
console.log(container.get('age'));   // 31
console.log(container.get('name'));  // John (из прототипа)
```

### Баланс: Память vs CPU

```
┌─────────────────────────────────────────┐
│ Подход          │ Память │ CPU          │
├─────────────────┼────────┼──────────────┤
│ Полная копия    │ Много  │ Мало         │
│ Copy-on-Write   │ Мало   │ Чуть больше  │
│ Мутации         │ Мало   │ Мало         │
│                 │        │ (НЕ безопасно)│
└─────────────────────────────────────────┘
```

---

## Ownership концепция

### Идея из Rust

Каждый объект имеет одного владельца (owner). Когда владелец выходит из области видимости, объект уничтожается.

### Реализация через `using` (ES2024)

```javascript
class OwnedResource {
  constructor(data) {
    this.data = data;
    this.disposed = false;
  }

  [Symbol.dispose]() {
    if (!this.disposed) {
      console.log('Disposing resource:', this.data);
      this.data = null;
      this.disposed = true;
    }
  }

  getData() {
    if (this.disposed) {
      throw new Error('Resource already disposed');
    }
    return this.data;
  }
}

function processData() {
  using resource = new OwnedResource({ value: 42 });
  console.log(resource.getData());  // { value: 42 }
  // resource автоматически dispose при выходе из функции
}

processData();
// "Disposing resource: { value: 42 }"
```

### Передача владения

```javascript
class OwnershipContainer {
  constructor(data) {
    this.data = data;
    this.owned = true;
  }

  transfer() {
    if (!this.owned) {
      throw new Error('Ownership already transferred');
    }
    this.owned = false;
    return new OwnershipContainer(this.data);
  }

  getData() {
    if (!this.owned) {
      throw new Error('No longer owns this data');
    }
    return this.data;
  }

  [Symbol.dispose]() {
    if (this.owned) {
      console.log('Cleaning up:', this.data);
      this.data = null;
    }
  }
}

// Использование
function example() {
  using container1 = new OwnershipContainer({ id: 1 });

  // Передаем ownership
  const container2 = container1.transfer();

  // container1.getData();  // Ошибка: ownership передан
  console.log(container2.getData());  // OK
}
```

### Ownership решает те же задачи

Вместо копирования данных (иммутабельность), мы гарантируем, что:
- Только один владелец может изменять данные
- Все старые ссылки аннулируются при передаче владения
- Нет race conditions и коррапции данных

---

## Практические примеры

### Пример 1: Безопасная работа со структурами данных

```javascript
// Проблема: мутабельная структура
class MutablePolyline {
  constructor(points) {
    this.points = points;
  }

  shift(dx, dy) {
    // ОПАСНО: изменяет оригинал
    this.points.forEach(p => {
      p.x += dx;
      p.y += dy;
    });
  }
}

const line = new MutablePolyline([
  { x: 0, y: 0 },
  { x: 10, y: 10 }
]);

// Передали ссылку в разные части программы
renderLine(line);
saveLine(line);

// Изменили - сломали данные везде!
line.shift(5, 5);

// Теперь renderLine и saveLine видят измененные данные
```

```javascript
// Решение: иммутабельная структура
class ImmutablePolyline {
  constructor(points) {
    this.points = Object.freeze([...points.map(p => Object.freeze({...p}))]);
  }

  shift(dx, dy) {
    // Создаем НОВУЮ ломаную с новыми точками
    const newPoints = this.points.map(p => ({
      x: p.x + dx,
      y: p.y + dy
    }));
    return new ImmutablePolyline(newPoints);
  }
}

const line1 = new ImmutablePolyline([
  { x: 0, y: 0 },
  { x: 10, y: 10 }
]);

// Безопасно передаем везде
renderLine(line1);
saveLine(line1);

// Создаем новую ломаную
const line2 = line1.shift(5, 5);

// line1 не изменилась - безопасно!
console.log(line1.points[0].x);  // 0
console.log(line2.points[0].x);  // 5
```

### Пример 2: Ссылочная прозрачность в асинхронном коде

```javascript
// ОПАСНО: мутабельное состояние + асинхронность
class AsyncCounter {
  constructor() {
    this.count = 0;
  }

  async increment() {
    const current = this.count;
    await delay(100);  // Симуляция async операции
    this.count = current + 1;  // Race condition!
  }
}

const counter = new AsyncCounter();

// Параллельные вызовы - race condition
Promise.all([
  counter.increment(),
  counter.increment(),
  counter.increment()
]);

// Результат непредсказуем: может быть 1, 2 или 3
```

```javascript
// БЕЗОПАСНО: иммутабельное состояние
class ImmutableCounter {
  constructor(count = 0) {
    this.count = count;
    Object.freeze(this);
  }

  async increment() {
    await delay(100);
    return new ImmutableCounter(this.count + 1);
  }
}

// Использование
async function safeIncrement() {
  let counter = new ImmutableCounter();

  // Каждый вызов создает новый объект
  counter = await counter.increment();  // 1
  counter = await counter.increment();  // 2
  counter = await counter.increment();  // 3

  console.log(counter.count);  // Всегда 3
}
```

### Пример 3: Ссылочная прозрачность относительно оперативной памяти

```javascript
// Функция с выводом, но ссылочно прозрачная относительно памяти
function calculateWithLogging(a, b) {
  const result = a + b;

  // Вывод не влияет на состояние в памяти
  console.log(`Calculated: ${a} + ${b} = ${result}`);

  // Состояние программы не изменилось
  return result;
}

// Можно вызывать много раз - состояние предсказуемо
const x = calculateWithLogging(5, 3);  // 8
const y = calculateWithLogging(5, 3);  // 8

// x === y, несмотря на console.log
```

---

## Выводы

### Основные принципы

1. **Ссылочная прозрачность не только для ФП**
   - Применима в ООП и процедурном программировании
   - Помогает избежать багов во всех парадигмах

2. **Иммутабельность = безопасность**
   - Данные не меняются - нет race conditions
   - Можно безопасно передавать ссылки везде
   - Упрощается отладка и тестирование

3. **Инструменты для иммутабельности**
   - `Object.freeze()` - простая заморозка
   - Приватные поля (`#field`) - инкапсуляция
   - Замыкания - защита состояния
   - `using` - автоматическое управление ресурсами

4. **Оптимизация**
   - Copy-on-Write экономит память
   - Ownership контролирует жизненный цикл
   - Компилятор может оптимизировать ссылочно прозрачный код

5. **Баланс производительности**
   - Иммутабельность: больше памяти, безопаснее
   - Copy-on-Write: компромисс память/CPU
   - Ownership: безопасность без копирования

### Когда использовать

**Используйте ссылочную прозрачность когда:**
- Работаете с асинхронным кодом
- Данные передаются между компонентами
- Нужна предсказуемость и отсутствие побочных эффектов
- Важна возможность отката состояния (undo/redo)

**Допустимы мутации когда:**
- Локальные временные данные в функции
- Узкая область видимости
- Критична производительность
- Гарантирован один владелец данных

### Практические рекомендации

```javascript
// ✅ ХОРОШО: Явная иммутабельность
const point = Object.freeze({ x: 10, y: 20 });

// ✅ ХОРОШО: Возврат новых объектов
function movePoint(point, dx, dy) {
  return { x: point.x + dx, y: point.y + dy };
}

// ✅ ХОРОШО: Приватные поля
class Point {
  #x; #y;
  constructor(x, y) { this.#x = x; this.#y = y; }
  getX() { return this.#x; }
}

// ❌ ПЛОХО: Мутация аргумента
function movePoint(point, dx, dy) {
  point.x += dx;  // Изменяет оригинал!
  point.y += dy;
}

// ❌ ПЛОХО: Публичные изменяемые поля
class Point {
  constructor(x, y) {
    this.x = x;  // Можно изменить снаружи
    this.y = y;
  }
}
```

---

## Дополнительные ресурсы

### Темы для дальнейшего изучения

1. **Copy-on-Write подробнее**
   - Persistent data structures
   - Structural sharing
   - Библиотеки: Immer, Immutable.js

2. **Ownership в JavaScript**
   - Explicit Resource Management (using)
   - WeakMap и WeakSet
   - Finalizers

3. **Функциональное программирование**
   - Чистые функции
   - Монады и функторы
   - Композиция функций

4. **Паттерны иммутабельности**
   - Event Sourcing
   - CQRS
   - Redux / Zustand

### Полезные библиотеки

```javascript
// Immer - упрощает работу с иммутабельными данными
import { produce } from 'immer';

const state = { count: 0, items: [1, 2, 3] };

const newState = produce(state, draft => {
  draft.count += 1;  // Пишем как будто мутируем
  draft.items.push(4);
});

// state не изменился, newState - новый объект

// Immutable.js - persistent data structures
import { Map } from 'immutable';

const map1 = Map({ a: 1, b: 2 });
const map2 = map1.set('a', 10);

console.log(map1.get('a'));  // 1
console.log(map2.get('a'));  // 10
```

---

## Глоссарий

- **Ссылочная прозрачность (Referential Transparency)** - свойство кода, позволяющее заменять выражения на их значения без изменения поведения
- **Иммутабельность (Immutability)** - неизменяемость данных после создания
- **Copy-on-Write (COW)** - стратегия создания копий только при изменении
- **Ownership** - концепция единоличного владения ресурсом
- **Side effect (Побочный эффект)** - изменение состояния вне функции
- **Pure function (Чистая функция)** - функция без побочных эффектов
- **Мемоизация** - кэширование результатов функции
- **Structural sharing** - переиспользование неизменённых частей структуры

---

**Автор конспекта:** Claude Code
**Дата:** 2025-12-06
**Основано на лекции:** "Referential Transparency — Ссылочная прозрачность"

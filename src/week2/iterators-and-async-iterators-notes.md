# Итераторы и Асинхронные Итераторы в JavaScript

## Обзор

Эта лекция охватывает фундаментальные концепции итераторов в JavaScript - как синхронных, так и асинхронных. Мы рассмотрим контракты Iterator и Iterable, научимся создавать собственные итераторы различными способами, и поймем, как генераторы упрощают работу с итерируемыми объектами.

---

## Часть 1: Синхронные Итераторы

### 1.1 Контракт Iterator

**Iterator** - это контракт (соглашение), который определяет, как объект должен предоставлять последовательный доступ к своим элементам.

#### Требования контракта Iterator:

Объект является итератором, если у него есть:
- **Метод `next()`** - возвращает следующее значение в последовательности
- Метод `next()` должен возвращать объект со структурой:
  - `value` - текущее значение
  - `done` - булево значение (`true` когда итерация завершена, `false` в противном случае)

#### Пример 1: Простейший Iterator (литерал объекта)

```javascript
// Создаем простой итератор вручную
const iterator = {
  current: 0,

  next() {
    // Возвращаем объект с value и done
    if (this.current < 3) {
      return {
        value: this.current++,
        done: false
      };
    }
    return {
      value: undefined,
      done: true
    };
  }
};

// Используем итератор
console.log(iterator.next()); // { value: 0, done: false }
console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

**Ключевой момент**: Когда `done: true`, значение `value` должно игнорироваться - итерация завершена.

---

### 1.2 Контракт Iterable

**Iterable** - это более сложный контракт, который позволяет объекту быть итерируемым с помощью циклов `for...of` и spread-оператора.

#### Требования контракта Iterable:

Объект является итерируемым (iterable), если у него есть:
- **Метод с именем `Symbol.iterator`** - специальный символьный ключ
- Этот метод должен возвращать **Iterator** (объект с методом `next()`)

#### Пример 2: Iterable объект (литерал)

```javascript
// Создаем iterable объект
const iterable = {
  // Метод Symbol.iterator возвращает iterator
  [Symbol.iterator]() {
    let current = 0;

    // Возвращаем iterator
    return {
      next() {
        if (current < 3) {
          return {
            value: current++,
            done: false
          };
        }
        return {
          value: undefined,
          done: true
        };
      }
    };
  }
};

// Способ 1: Получаем iterator вручную
const iter = iterable[Symbol.iterator]();
console.log(iter.next()); // { value: 0, done: false }
console.log(iter.next()); // { value: 1, done: false }
console.log(iter.next()); // { value: 2, done: false }
console.log(iter.next()); // { value: undefined, done: true }

// Способ 2: Используем цикл for...of
for (const value of iterable) {
  console.log(value); // 0, 1, 2
}

// Способ 3: Используем spread оператор
const array = [...iterable];
console.log(array); // [0, 1, 2]
```

**Важно**:
- Метод `Symbol.iterator` создает **новый** iterator при каждом вызове
- `for...of` и spread-оператор работают только с **iterable** объектами
- Значения с `done: true` автоматически игнорируются

---

### 1.3 Iterable класс с замыканиями

Литералы объектов неудобны для повторного использования. Давайте создадим класс.

#### Пример 3: Класс Counter как Iterable

```javascript
class Counter {
  constructor(begin, end, step = 1) {
    this.begin = begin;
    this.end = end;
    this.step = step;
  }

  // Делаем класс iterable
  [Symbol.iterator]() {
    // Сохраняем текущие значения begin и end
    // на случай, если они изменятся во время итерации
    let current = this.begin;
    const end = this.end;
    const step = this.step;

    // Возвращаем iterator
    return {
      next() {
        if (current < end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
}

// Использование класса
const counter = new Counter(0, 3);

// Способ 1: Получаем iterator вручную
const iterator = counter[Symbol.iterator]();
console.log(iterator.next()); // { value: 0, done: false }
console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: undefined, done: true }

// Способ 2: for...of
for (const num of counter) {
  console.log(num); // 0, 1, 2
}

// Способ 3: spread оператор
const numbers = [...counter];
console.log(numbers); // [0, 1, 2]
```

**Преимущества класса**:
- Переиспользуемый код
- Возможность наследования
- Инкапсуляция логики
- Параметризация через конструктор

**Техника замыкания**: Мы сохраняем `begin` и `end` в локальные переменные функции `Symbol.iterator`, чтобы защититься от изменений во время итерации.

---

### 1.4 Встроенные Iterable объекты

Многие встроенные типы JavaScript уже реализуют контракт Iterable.

#### Пример 4: Массивы как Iterable

```javascript
const array = [0, 1, 2];

// Массив уже является iterable!
// У него есть Symbol.iterator

// Способ 1: Получаем iterator вручную
const iterator = array[Symbol.iterator]();
console.log(iterator.next()); // { value: 0, done: false }
console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: undefined, done: true }

// Способ 2: for...of работает!
for (const item of array) {
  console.log(item); // 0, 1, 2
}

// Способ 3: spread оператор работает!
const copy = [...array];
console.log(copy); // [0, 1, 2]
```

**Откровение**: Теперь понятно, почему `for...of` и spread-оператор работают с массивами - они реализуют контракт Iterable через `Symbol.iterator`!

**Другие встроенные Iterable**:
- `String` - итерация по символам
- `Set` - итерация по значениям
- `Map` - итерация по парам [key, value]
- `arguments` - итерация по аргументам функции
- Typed Arrays (Uint8Array и т.д.)

---

### 1.5 Генераторы - синтаксический сахар для Iterable

Создание итераторов вручную требует много кода. **Генераторы** - это специальные функции, которые автоматически создают iterable объекты.

#### Синтаксис генератора:

```javascript
function* generatorName() {
  yield value1;
  yield value2;
  // ...
}
```

- `function*` - объявление функции-генератора (со звездочкой)
- `yield` - возвращает значение и приостанавливает выполнение
- Генератор возвращает iterable объект

#### Пример 5: Генератор вместо класса

```javascript
// Императивный подход с классом (22 строки кода)
class Counter {
  constructor(begin, end, step = 1) {
    this.begin = begin;
    this.end = end;
    this.step = step;
  }

  [Symbol.iterator]() {
    let current = this.begin;
    const end = this.end;
    const step = this.step;

    return {
      next() {
        if (current < end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
}

// VS

// Декларативный подход с генератором (9 строк кода)
function* counter(begin, end, step = 1) {
  for (let i = begin; i < end; i += step) {
    yield i;
  }
}

// Использование одинаковое!
const gen = counter(0, 3);

// Способ 1: Вручную
console.log(gen.next()); // { value: 0, done: false }
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: undefined, done: true }

// Способ 2: for...of
for (const num of counter(0, 3)) {
  console.log(num); // 0, 1, 2
}

// Способ 3: spread
const numbers = [...counter(0, 3)];
console.log(numbers); // [0, 1, 2]
```

**Ключевое отличие**:
- **Императивный** (класс): мы сами создаем объект, метод `next()`, управляем состоянием
- **Декларативный** (генератор): мы описываем последовательность значений, JavaScript создает итератор автоматически

---

### 1.6 Делегирование в генераторах (yield*)

Генераторы могут делегировать итерацию другим iterable объектам с помощью `yield*`.

#### Пример 6: Делегирование массиву

```javascript
// Генератор, который делегирует итерацию массиву
function* gen() {
  yield* [0, 1, 2]; // yield* делегирует массиву
}

const iterator = gen();

// Способ 1: Вручную
console.log(iterator.next()); // { value: 0, done: false }
console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: undefined, done: true }

// Способ 2: for...of
for (const value of gen()) {
  console.log(value); // 0, 1, 2
}

// Способ 3: spread
const array = [...gen()];
console.log(array); // [0, 1, 2]
```

**Что делает `yield*`**:
- Берет iterable объект (массив, другой генератор, строку и т.д.)
- Проходит по всем его значениям
- Возвращает каждое значение через `yield`
- Это краткая запись для цикла:
  ```javascript
  // yield* [0, 1, 2] эквивалентно:
  for (const value of [0, 1, 2]) {
    yield value;
  }
  ```

---

## Часть 2: Асинхронные Итераторы

### 2.1 Контракт Async Iterator

**Async Iterator** - это итератор, который работает с асинхронными операциями (Promise).

#### Требования контракта Async Iterator:

Объект является async iterator, если у него есть:
- **Асинхронный метод `next()`** (объявленный с `async` или возвращающий Promise)
- Метод возвращает **Promise**, который разрешается объектом с полями:
  - `value` - текущее значение
  - `done` - булево значение

#### Пример 7: Простейший Async Iterator

```javascript
// Async Iterator вручную
const asyncIterator = {
  current: 0,

  // Асинхронный метод next
  async next() {
    if (this.current < 3) {
      return {
        value: this.current++,
        done: false
      };
    }
    return {
      value: undefined,
      done: true
    };
  }
};

// Использование
async function run() {
  console.log(await asyncIterator.next());
  // Promise { value: 0, done: false }

  console.log(await asyncIterator.next());
  // Promise { value: 1, done: false }

  console.log(await asyncIterator.next());
  // Promise { value: 2, done: false }

  console.log(await asyncIterator.next());
  // Promise { value: undefined, done: true }
}

run();
```

**Ключевое отличие от синхронного**: Метод `next()` возвращает **Promise**, а не обычный объект.

---

### 2.2 Контракт Async Iterable

**Async Iterable** - это объект, который можно итерировать асинхронно.

#### Требования контракта Async Iterable:

Объект является async iterable, если у него есть:
- **Метод `Symbol.asyncIterator`** - специальный символьный ключ для асинхронной итерации
- Этот метод возвращает **Async Iterator** (объект с async методом `next()`)

#### Пример 8: Async Iterable объект

```javascript
// Async Iterable объект
const asyncIterable = {
  [Symbol.asyncIterator]() {
    let current = 0;

    // Возвращаем async iterator
    return {
      async next() {
        // Можем делать асинхронные операции здесь
        // Например, запросы к API, чтение файлов и т.д.

        if (current < 3) {
          return {
            value: current++,
            done: false
          };
        }
        return {
          value: undefined,
          done: true
        };
      }
    };
  }
};

// Использование
async function run() {
  // Способ 1: Вручную
  const iter = asyncIterable[Symbol.asyncIterator]();
  console.log(await iter.next()); // { value: 0, done: false }
  console.log(await iter.next()); // { value: 1, done: false }
  console.log(await iter.next()); // { value: 2, done: false }
  console.log(await iter.next()); // { value: undefined, done: true }

  // Способ 2: for await...of (специальный цикл для async iterable)
  for await (const value of asyncIterable) {
    console.log(value); // 0, 1, 2
  }
}

run();
```

**Важно**:
- Для async iterable используется `for await...of` (не просто `for...of`)
- Spread-оператор **НЕ работает** с async iterable
- Каждая итерация ожидает разрешения Promise

---

### 2.3 Async Iterable класс

Создадим класс для удобного переиспользования.

#### Пример 9: Класс AsyncCounter

```javascript
class AsyncCounter {
  constructor(begin, end, step = 1) {
    this.begin = begin;
    this.end = end;
    this.step = step;
  }

  // Делаем класс async iterable
  [Symbol.asyncIterator]() {
    let current = this.begin;
    const end = this.end;
    const step = this.step;

    return {
      async next() {
        // Здесь могут быть асинхронные операции
        // Например, await fetch(), await fs.readFile() и т.д.

        if (current < end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
}

// Использование
async function run() {
  const counter = new AsyncCounter(0, 3);

  // Способ 1: Вручную
  const iterator = counter[Symbol.asyncIterator]();
  console.log(await iterator.next()); // { value: 0, done: false }
  console.log(await iterator.next()); // { value: 1, done: false }
  console.log(await iterator.next()); // { value: 2, done: false }

  // Способ 2: for await...of
  for await (const num of counter) {
    console.log(num); // 0, 1, 2
  }

  // Способ 3: Spread НЕ работает с async iterable!
  // const arr = [...counter]; // ОШИБКА!
}

run();
```

**Практическое применение**: Async iterable идеально подходит для:
- Чтения больших файлов по частям
- Получения данных из API с пагинацией
- Обработки потоков данных (streams)
- Любых последовательных асинхронных операций

---

### 2.4 Async генераторы

Как и с синхронными итераторами, создание async iterable вручную требует много кода. **Async генераторы** решают эту проблему.

#### Синтаксис async генератора:

```javascript
async function* asyncGeneratorName() {
  yield value1;
  yield value2;
  // ...
}
```

- `async function*` - объявление async генератора (async + звездочка)
- `yield` - возвращает значение через Promise
- Возвращает async iterable объект

#### Пример 10: Async генератор вместо класса

```javascript
// С классом (много кода)
class AsyncCounter {
  constructor(begin, end, step = 1) {
    this.begin = begin;
    this.end = end;
    this.step = step;
  }

  [Symbol.asyncIterator]() {
    let current = this.begin;
    const end = this.end;
    const step = this.step;

    return {
      async next() {
        if (current < end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
}

// VS

// С async генератором (компактно!)
async function* asyncCounter(begin, end, step = 1) {
  for (let i = begin; i < end; i += step) {
    // Здесь можно делать await операции
    // await someAsyncOperation();
    yield i;
  }
}

// Использование одинаковое
async function run() {
  // Способ 1: Вручную
  const gen = asyncCounter(0, 3);
  console.log(await gen.next()); // { value: 0, done: false }
  console.log(await gen.next()); // { value: 1, done: false }
  console.log(await gen.next()); // { value: 2, done: false }

  // Способ 2: for await...of
  for await (const num of asyncCounter(0, 3)) {
    console.log(num); // 0, 1, 2
  }

  // Способ 3: Promise.all для массива
  const gen2 = asyncCounter(0, 4);
  const promises = [];
  for (let i = 0; i < 4; i++) {
    promises.push(gen2.next());
  }
  const results = await Promise.all(promises);
  console.log(results);
  // [
  //   { value: 0, done: false },
  //   { value: 1, done: false },
  //   { value: 2, done: false },
  //   { value: 3, done: false }
  // ]
}

run();
```

**Преимущества async генератора**:
- Лаконичный синтаксис
- Естественная работа с `await` внутри
- Автоматическое создание async iterable
- Декларативный стиль

---

### 2.5 Делегирование в async генераторах (yield*)

Async генераторы тоже поддерживают делегирование через `yield*`.

#### Пример 11: Async делегирование массиву

```javascript
// Async генератор с делегированием
async function* asyncGen() {
  // yield* работает с любым iterable
  // значения автоматически оборачиваются в Promise
  yield* [0, 1, 2];
}

async function run() {
  // Способ 1: Вручную
  const iterator = asyncGen();
  console.log(await iterator.next()); // { value: 0, done: false }
  console.log(await iterator.next()); // { value: 1, done: false }
  console.log(await iterator.next()); // { value: 2, done: false }

  // Способ 2: for await...of
  for await (const value of asyncGen()) {
    console.log(value); // 0, 1, 2
  }

  // Способ 3: Promise.all
  const gen = asyncGen();
  const promises = [
    gen.next(),
    gen.next(),
    gen.next()
  ];
  const results = await Promise.all(promises);
  console.log(results);
}

run();
```

**Что происходит**:
- `yield* [0, 1, 2]` берет обычный массив (синхронный iterable)
- Каждое значение автоматически оборачивается в Promise
- Получается async итерация по синхронным данным

---

## Сравнительная таблица

| Характеристика | Синхронный Iterator | Async Iterator |
|---|---|---|
| **Метод next()** | Обычная функция | `async` функция |
| **Возвращает** | `{ value, done }` | `Promise<{ value, done }>` |
| **Символ** | `Symbol.iterator` | `Symbol.asyncIterator` |
| **Цикл** | `for...of` | `for await...of` |
| **Spread оператор** | ✅ Работает | ❌ Не работает |
| **Генератор** | `function*` | `async function*` |
| **yield** | Возвращает значение | Возвращает Promise |
| **Делегирование** | `yield*` | `yield*` |
| **Применение** | Синхронные данные | Асинхронные операции |

---

## Ключевые моменты

### Контракты

1. **Iterator** = объект с методом `next()` → `{ value, done }`
2. **Iterable** = объект с методом `[Symbol.iterator]()` → Iterator
3. **Async Iterator** = объект с `async next()` → `Promise<{ value, done }>`
4. **Async Iterable** = объект с `[Symbol.asyncIterator]()` → Async Iterator

### Способы создания

**Синхронные итераторы:**
1. Вручную через литералы объектов
2. Через классы с `[Symbol.iterator]()`
3. Через генераторы `function*`

**Асинхронные итераторы:**
1. Вручную через литералы с `async next()`
2. Через классы с `[Symbol.asyncIterator]()`
3. Через async генераторы `async function*`

### Использование

**Синхронные:**
- `iterator.next()` - ручное получение
- `for...of` - автоматическая итерация
- `[...iterable]` - spread в массив

**Асинхронные:**
- `await iterator.next()` - ручное получение
- `for await...of` - автоматическая итерация
- Spread **не поддерживается**

---

## Практические примеры применения

### Пример: Чтение больших файлов построчно

```javascript
// Async генератор для чтения файла построчно
async function* readLines(filePath) {
  const fs = require('fs').promises;
  const fileHandle = await fs.open(filePath, 'r');

  try {
    let buffer = '';
    const chunk = Buffer.alloc(1024);

    while (true) {
      const { bytesRead } = await fileHandle.read(chunk, 0, 1024);
      if (bytesRead === 0) break;

      buffer += chunk.toString('utf8', 0, bytesRead);
      const lines = buffer.split('\n');
      buffer = lines.pop(); // Последняя неполная строка

      for (const line of lines) {
        yield line;
      }
    }

    if (buffer.length > 0) {
      yield buffer;
    }
  } finally {
    await fileHandle.close();
  }
}

// Использование
async function processFile() {
  for await (const line of readLines('huge-file.txt')) {
    console.log('Line:', line);
    // Обрабатываем каждую строку по мере чтения
    // Не загружаем весь файл в память!
  }
}
```

### Пример: Пагинация API

```javascript
// Async генератор для получения всех страниц API
async function* fetchAllPages(baseUrl) {
  let page = 1;

  while (true) {
    const response = await fetch(`${baseUrl}?page=${page}`);
    const data = await response.json();

    // Отдаем текущую страницу
    yield data.items;

    // Проверяем, есть ли еще страницы
    if (!data.hasNextPage) break;
    page++;
  }
}

// Использование
async function loadAllData() {
  for await (const pageItems of fetchAllPages('/api/users')) {
    console.log('Page items:', pageItems);
    // Обрабатываем каждую страницу по мере загрузки
  }
}
```

### Пример: Бесконечная последовательность

```javascript
// Генератор бесконечной последовательности чисел Фибоначчи
function* fibonacci() {
  let a = 0, b = 1;

  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

// Использование с ограничением
function* take(iterable, limit) {
  let count = 0;
  for (const value of iterable) {
    if (count >= limit) break;
    yield value;
    count++;
  }
}

// Получаем первые 10 чисел Фибоначчи
const first10 = [...take(fibonacci(), 10)];
console.log(first10);
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

---

## Распространенные ошибки и как их избежать

### Ошибка 1: Попытка использовать spread с async iterable

```javascript
async function* asyncGen() {
  yield 1;
  yield 2;
}

// ❌ ОШИБКА: Spread не работает с async iterable
// const arr = [...asyncGen()];

// ✅ ПРАВИЛЬНО: Используйте for await...of
async function collectToArray() {
  const arr = [];
  for await (const value of asyncGen()) {
    arr.push(value);
  }
  return arr;
}
```

### Ошибка 2: Забыть await в async генераторе

```javascript
async function* fetchData() {
  // ❌ ОШИБКА: Забыли await
  const data = fetch('/api/data'); // Promise!
  yield data; // Вернет Promise, а не данные

  // ✅ ПРАВИЛЬНО
  const data2 = await fetch('/api/data');
  yield data2;
}
```

### Ошибка 3: Не проверять done при ручном вызове next()

```javascript
const iterator = [1, 2, 3][Symbol.iterator]();

// ❌ ОШИБКА: Не проверяем done
console.log(iterator.next().value); // 1
console.log(iterator.next().value); // 2
console.log(iterator.next().value); // 3
console.log(iterator.next().value); // undefined (должны были остановиться!)

// ✅ ПРАВИЛЬНО
let result;
while (!(result = iterator.next()).done) {
  console.log(result.value);
}
```

### Ошибка 4: Переиспользование итератора

```javascript
function* gen() {
  yield 1;
  yield 2;
}

const iterator = gen();

// Первый проход
for (const value of iterator) {
  console.log(value); // 1, 2
}

// ❌ Второй проход не сработает - итератор исчерпан!
for (const value of iterator) {
  console.log(value); // Ничего не выведет
}

// ✅ ПРАВИЛЬНО: Создавайте новый итератор
for (const value of gen()) {
  console.log(value); // 1, 2
}
```

---

## Резюме

### Что мы изучили:

1. **Контракты Iterator и Iterable** - фундаментальные интерфейсы для итерации
2. **Ручное создание итераторов** - через литералы и классы
3. **Генераторы (`function*`)** - синтаксический сахар для создания iterable
4. **Async Iterator и Async Iterable** - для асинхронных операций
5. **Async генераторы (`async function*`)** - упрощенное создание async iterable
6. **Делегирование (`yield*`)** - переиспользование других iterable
7. **`for...of` и `for await...of`** - циклы для итерации
8. **Spread оператор** - работает только с синхронными iterable

### Почему это важно:

- **Единый интерфейс** для работы с последовательностями
- **Ленивые вычисления** - данные генерируются по требованию
- **Память эффективность** - не нужно загружать все данные сразу
- **Композиция** - можно комбинировать итераторы
- **Асинхронность** - естественная работа с async операциями

### Когда использовать:

**Синхронные итераторы:**
- Обработка коллекций данных
- Математические последовательности
- Генерация комбинаций/перестановок
- Обход структур данных (деревья, графы)

**Асинхронные итераторы:**
- Чтение больших файлов
- Работа с API (пагинация)
- Обработка потоков данных
- Последовательные асинхронные задачи

---

## Дополнительные материалы для изучения

### Связанные темы:

1. **Iterator Helpers** (ES2024) - методы `.map()`, `.filter()`, `.take()` для итераторов
2. **Observable** - реактивные потоки данных (RxJS)
3. **Streams API** - работа с потоками данных в браузере
4. **Node.js Streams** - потоки в Node.js
5. **Transducers** - композируемые преобразования данных

### Спецификации:

- [Iteration protocols (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols)
- [Generators (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator)
- [for await...of (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of)

---

**Конец конспекта лекции**

_Удачи в изучении итераторов и генераторов! Практикуйтесь на реальных задачах для закрепления материала._
